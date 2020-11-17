# [LAB] JP's Amazon EKS with Spot instances

### You can run this lab either from your local computer or for AWS Cloud9. If you opt for AWS Cloud9, please follow the link bellow to create, connect to your Cloud9 environment, and open a terminal to get started.

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
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
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

### AWS Load Balancer Controller

***Create an IAM OIDC provider and associate it with the cluster***

```bash
eksctl utils associate-iam-oidc-provider --cluster eks-spot-lab --approve
```

***Download and create an IAM policy for the AWS load balancer controller***

```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

***Create an IAM role and Kubernetes service account named aws-load-balancer-controller in the kube-system namespace, a cluster role, and a cluster role binding for the load balancer controller***

```bash
eksctl create iamserviceaccount \                                 
  --cluster=eks-spot-lab \ 
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::arn:aws:iam::220713292402:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

***Install the controller and verify if it is installed***

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml
curl -o v2_0_0_full.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/v2_0_0_full.yaml
kubectl apply -f v2_0_0_full.yaml
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Sample App

***Deploy the game 2048 as a sample application. The deployment is created with five replicas, which land on one of the Spot Instance node groups due to the nodeSelector choosing lifecycle: Ec2Spot.***

```bash
kubectl apply -f 2048_full.yaml
```

***After a few minutes, verify that the Ingress resource was created with the following command.***

```bash
kubectl get ingress/ingress-2048 -n game-2048
```

***Open a browser and navigate to the ADDRESS URL from the previous command output to see the sample application.***

### Scale deployment out

```bash
kubectl scale --replicas=30 lightbulb-jp -n lightbulb-jp-ns
```