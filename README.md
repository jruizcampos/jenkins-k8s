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

Update your local kubeconfig file to access the EKS Cluster just created. Replace region-code with the name of the AWS region you're using (us-east-1 for example) and my-cluster with the name of your EKS cluster:
```
aws eks update-kubeconfig --region region-code --name my-cluster
```

Verify you can successfully connect to the EKS Cluster:
```
kubectl get nodes
kubectl get nodes -o wide
```
![Kubectl connect to EKS Cluster](https://johnruizcampos.com/wp-content/uploads/kubectl_eks_cluster.jpg)

## Install the Amazon EBS CSI driver
Official Amazon AWS documentation: https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html

- In the following code, replace **my-cluster** with the name of your EKS Cluster and run the command:
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```
Add the Amazon EBS CSI Driver using the AWS Console:
- Go to your EKS Cluster, select **_Add-ons section_** and click on **_Get more add-ons_**:
![Get more add-ons](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_1.jpg)
- Select the **_Amazon EBS CSI Driver_**:
![Select the Amazon EBS CSI Driver](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_2.jpg)
- Select the **_AmazonEKS_EBS_CSI_DriverRole_**:
![Select the AmazonEKS_EBS_CSI_DriverRole](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_3.jpg)
- Create the **_AmazonEKS_EBS_CSI_DriverRole_**:
![Create the AmazonEKS_EBS_CSI_DriverRole](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_4.jpg)

## Install the AWS Load Balancer Controller
Official Amazon AWS documentation: https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

Run:
- `curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json`

- `aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json `

- In the following code, replace **my-cluster** with the name of your EKS Cluster and **arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy** with the ARN of your IAM Policy. Then run the command:
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

