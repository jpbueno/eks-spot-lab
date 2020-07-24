## [LAB] JP's Amazon EKS with Spot instances.

**EKS cluster with a managed node group having 3 On-Demand t3.medium nodes and labels lifecycle=OnDemand and intent=control-apps.**

```bash
eksctl create cluster --version=1.16 --name=eks-spot-lab --node-private-networking --managed --nodes=3 --alb-ingress-access --region=us-east-1 --node-type t3.medium --node-labels="lifecycle=OnDemand" --asg-access

**Install kube-ops-view**

```bash
git clone https://github.com/hjacobs/kube-ops-view.git
cd ~/environment/eks-spot-lab/kube-ops-view/deploy
kubectl apply -k ./

**Open kube-ops-view**

```bash
kubectl port-forward service/kube-ops-view 8080:80

*To increase size, append #scale=2.0 in the end of URL*

**Create spot worker nodes**

```bash
eksctl create nodegroup -f spot_nodegroups.yaml

**Confirm these nodes were added to the cluster**

```bash
kubectl get nodes --show-labels --selector=lifecycle=Ec2Spot

**Install Node Termination Handler**

```bash
kubectl apply -f https://github.com/aws/aws-node-termination-handler/releases/download/v1.3.1/all-resources.yaml

**Verify Node Termination Handler is running**

```bash
kubectl get daemonsets --all-namespaces

**Deploy the Cluster Autoscaler**

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml

**Add cluster-autoscaler.kubernetes.io/safe-to-evictannotation to the deployment**

```bash
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"

**View cluster-autoscaler logs**

```bash
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler

**Deploy sample app**


*This deploys three replicas, which land on one of the Spot Instance node groups due to the nodeSelector choosing lifecycle: Ec2Spot. The “web-stateful” nodes are not fault-tolerant and not appropriate to be deployed on Spot Instances. So, you use nodeSelector again, and instead choose lifecycle: OnDemand. By guiding fault-tolerant pods to Spot Instance nodes, and stateful pods to On-Demand nodes, you can even use this to support multi-tenant clusters.*

```bash
kubectl apply -f web-app.yaml

**Scale deployment out**

```bash
kubectl scale --replicas=30 deployment/web-stateless
