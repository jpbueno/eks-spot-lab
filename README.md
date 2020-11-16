# [LAB] JP's Amazon EKS with Spot instances

### Create, connect to your Cloud9 environment, and open a terminal.

Instructions: https://docs.aws.amazon.com/cloud9/latest/user-guide/environments.html

### Install eksctl
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Create an EKS cluster with a managed node group having 3 On-Demand t3.medium nodes and a label called lifecycle=OnDemand

```bash
eksctl create cluster --version=1.18 --name=eks-spot-lab --node-private-networking --managed --nodes=3 --alb-ingress-access --region=us-east-1 --node-type t3.medium --node-labels="lifecycle=OnDemand" --asg-access --zones=us-east-1a,us-east-1b
```
### kube-ops-view

***Install kube-ops-view***

```bash
git clone https://github.com/hjacobs/kube-ops-view.git
cd kube-ops-view
kubectl apply -k deploy
```

***Open kube-ops-view***

```bash
kubectl port-forward service/kube-ops-view 8080:80
```

_Note: Open kube-ops-view by accessinig http://localhost:8080/ in the browser. To increase size, append /#scale=2.0 in the end of URL_

### Spot Worker Nodes

***Create spot worker nodes***

```bash
eksctl create nodegroup -f spot-nodegroups.yaml
git clone https://github.com/hjacobs/kube-ops-view.git
cd ~/environment/eks-spot-lab/kube-ops-view/deploy
kubectl apply -k ./
```

***Confirm these nodes were added to the cluster***

```bash
kubectl get nodes --show-labels --selector=lifecycle=Ec2Spot
```

### Node Termination Handler

***Install Node Termination Handler***

```bash
kubectl apply -f https://github.com/aws/aws-node-termination-handler/releases/download/v1.10.0/all-resources.yaml
```

***Verify Node Termination Handler is running***

```bash
kubectl get daemonsets --all-namespaces
```

### Cluster Autoscaler

***Deploy Cluster Autoscaler***

```bash
eksctl create nodegroup -f spot_nodegroups.yaml
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

***Add cluster-autoscaler.kubernetes.io/safe-to-evictannotation to the deployment***

```bash
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
kubectl get nodes --show-labels --selector=lifecycle=Ec2Spot
```

***View cluster-autoscaler logs***

```bash
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

### AWS ALB Ingress Controller

***Create Ingress with AWS ALB Ingress Controller***

```bash
kubectl apply -f https://github.com/aws/aws-node-termination-handler/releases/download/v1.10.0/all-resources.yaml
```

***Deploy the relevant RBAC roles and role bindings as required by the AWS ALB Ingress controller***

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v2.0.1/docs/examples/rbac-role.yaml
```

***Create an IAM policy named ALBIngressControllerIAMPolicy to allow the ALB Ingress controller to make AWS API calls on your behalf.***

```bash
kubectl get daemonsets --all-namespaces
aws iam create-policy --policy-name ALBIngressControllerIAMPolicy --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/iam-policy.json
```

***Create a Kubernetes service account and an IAM role (for the pod running the AWS ALB Ingress controller) by substituting \$PolicyARN with the recorded value from the previous step.***

```bash
eksctl create iamserviceaccount --cluster=eks-spot-demo --namespace=kube-system --name=alb-ingress-controller --attach-policy-arn=\$PolicyARN --override-existing-serviceaccounts --approve
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

***Deploy the AWS ALB Ingress controller***

```bash
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

***View cluster-autoscaler logs***

```bash
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

***Verify that the deployment was successful and the controller started***

```bash
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)
```

### Sample App

***Create namespace***

```bash
kubectl create ns lightbulb-jp-ns
kubectl apply -f lightbulb-jp-deploy.yaml -n lightbulb-jp-ns
```

***Create deployment with three replicas, which land on one of the Spot Instance node groups due to the nodeSelector choosing lifecycle: Ec2Spot.***

```bash
kubectl apply -f lightbulb-jp-deploy.yaml -n lightbulb-jp-ns
kubectl apply -f lightbulb-jp-service.yaml -n lightbulb-jp-ns
```

***Create service***

```bash
kubectl apply -f lightbulb-jp-service.yaml -n lightbulb-jp-ns
```

***Deploy an Ingress resource for the lightbulb-jp app***

```bash
kubectl apply -f lightbulb-jp-ingress.yaml -n lightbulb-jp-ns
```

### Scale deployment out

```bash
kubectl scale --replicas=30 lightbulb-jp -n lightbulb-jp-ns
```