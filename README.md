# Jenkins-K8s
Project about deploying Jenkins in Kubernetes (AWS EKS) platform

## Deploy Kubernetes Cluster (AWS EKS)
Personalize the cluster capacity modifying the **eks-cluster.tf** file:
```
eks_managed_node_groups = {
    my_node_group = {
      name = "my_node_group"

      instance_types = ["t2.small"]

      min_size     = 1
      max_size     = 2
      desired_size = 2

      pre_bootstrap_user_data = <<-EOT
      echo 'foo bar'
      EOT

      vpc_security_group_ids = [
        aws_security_group.node_group_one.id
      ]
    }
  }
```
Run:
- ` terraform validate `
- ` terraform plan -out plan.out `
- ` terraform apply plan.out `

## Install the AWS Load Balancer Controller
Official Amazon AWS documentation (https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)

Run:
- `curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json`

- `aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json `

```
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve 
```

- `helm repo add eks https://aws.github.io/eks-charts `
- `helm repo update`

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/amazon/aws-load-balancer-controller
```

- `helm ls`

