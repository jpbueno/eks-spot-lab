## [LAB] JP's Amazon EKS with Spot instances.

**Create and connect to your Cloud9 environment.**

> Instructions: https://docs.aws.amazon.com/cloud9/latest/user-guide/environments.html


**EKS cluster with a managed node group having 3 On-Demand t3.medium nodes and labels lifecycle=OnDemand and intent=control-apps.**


> eksctl create cluster --version=1.16 --name=eks-spot-lab --node-private-networking --managed --nodes=3 --alb-ingress-access --region=us-east-1 --node-type t3.medium --node-labels="lifecycle=OnDemand" --asg-access

**Install kube-ops-view**


> git clone https://github.com/hjacobs/kube-ops-view.git

> cd ~/environment/eks-spot-lab/kube-ops-view/deploy

> kubectl apply -k ./

**Open kube-ops-view**


> kubectl port-forward service/kube-ops-view 8080:80

*To increase size, append #scale=2.0 in the end of URL*

**Create spot worker nodes**


> eksctl create nodegroup -f spot_nodegroups.yaml

**Confirm these nodes were added to the cluster**


> kubectl get nodes --show-labels --selector=lifecycle=Ec2Spot

**Install Node Termination Handler**


> kubectl apply -f https://github.com/aws/aws-node-termination-handler/releases/download/v1.3.1/all-resources.yaml

**Verify Node Termination Handler is running**


> kubectl get daemonsets --all-namespaces

**Deploy the Cluster Autoscaler**


> kubectl apply -f cluster-autoscaler-autodiscover.yaml

**Add cluster-autoscaler.kubernetes.io/safe-to-evictannotation to the deployment**


> kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"

**View cluster-autoscaler logs**


> kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler


**Create Ingress with AWS ALB Ingress Controller**

**Deploy the relevant RBAC roles and role bindings as required by the AWS ALB Ingress controller**


> kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml

**Create an IAM policy named ALBIngressControllerIAMPolicy to allow the ALB Ingress controller to make AWS API calls on your behalf.**

> aws iam create-policy --policy-name ALBIngressControllerIAMPolicy --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/iam-policy.json

**Create a Kubernetes service account and an IAM role (for the pod running the AWS ALB Ingress controller) by substituting $PolicyARN with the recorded value from the previous step**

> eksctl create iamserviceaccount --cluster=eks-spot-demo --namespace=kube-system --name=alb-ingress-controller --attach-policy-arn=$PolicyARN --override-existing-serviceaccounts --approve

**Deploy the AWS ALB Ingress controller**


> curl -sS "https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/alb-ingress-controller.yaml" \
     | sed "s/# - --cluster-name=devCluster/- --cluster-name=eks-spot-demo/g" \
     | kubectl apply -f -

**Verify that the deployment was successful and the controller started**

> kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)

**Deploy sample app**

**Create namespace**

> kubectl create ns lightbulb-jp-ns

**This deploys three replicas, which land on one of the Spot Instance node groups due to the nodeSelector choosing lifecycle: Ec2Spot. The “web-stateful” nodes are not fault-tolerant and not appropriate to be deployed on Spot Instances. So, you use nodeSelector again, and instead choose lifecycle: OnDemand. By guiding fault-tolerant pods to Spot Instance nodes, and stateful pods to On-Demand nodes, you can even use this to support multi-tenant clusters.**

> kubectl apply -f lightbulb-jp-deploy.yaml -n lightbulb-jp-ns

**Create service**

> kubectl apply -f lightbulb-jp-service.yaml -n lightbulb-jp-ns


**Deploy an Ingress resource for the lightbulb-jp app**

> kubectl apply -f lightbulb-jp-ingress.yaml

**Scale deployment out**


> kubectl scale --replicas=30 deployment/web-stateless
