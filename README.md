# Jenkins-K8s
Project about deploying Jenkins in Kubernetes (AWS EKS) platform

## Deploy Kubernetes EKS Cluster (AWS) using eksctl
Run in 2 steps:
```bash
eksctl create cluster --name my-cluster --region us-east-1 --zones "us-east-1a,us-east-1b" --version 1.24 --node-type "t2.small" --nodes 2 --nodes-min 1 --nodes-max 2 --spot
```
```bash
eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=my-cluster --approve
```
Or everything in one step:
```bash
eksctl create cluster --name my-cluster --region us-east-1 --zones "us-east-1a,us-east-1b" --version 1.24 --node-type "t2.small" --nodes 2 --nodes-min 1 --nodes-max 2 --with-oidc --alb-ingress-access --spot
```

Update your local kubeconfig file to access the EKS Cluster just created. Replace region-code with the name of the AWS region you're using (us-east-1 for example) and my-cluster with the name of your EKS cluster:

<pre><code>aws eks update-kubeconfig --region <b>region-code</b> --name <b>my-cluster</b></code></pre>

Verify you can successfully connect to the EKS Cluster:
```bash
kubectl cluster-info
kubectl get nodes
```
![Kubectl connect to EKS Cluster](https://johnruizcampos.com/wp-content/uploads/kubectl_eks_cluster.jpg)

## Install the Amazon EBS CSI driver
Official Amazon AWS documentation: https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html

- In the following code, replace **my-cluster** with the name of your EKS Cluster and run the command:

<pre><code>eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <b>my-cluster</b> \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
</code></pre>

Add the Amazon EBS CSI Driver using the AWS Console:
- Go to your EKS Cluster, select **_Add-ons section_** and click on **_Get more add-ons_**:
![Get more add-ons](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_1.jpg)
- Select the **_Amazon EBS CSI Driver_**:
![Select the Amazon EBS CSI Driver](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_2.jpg)
- Select the **_AmazonEKS_EBS_CSI_DriverRole_**:
![Select the AmazonEKS_EBS_CSI_DriverRole](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_3.jpg)
- Create the **_AmazonEKS_EBS_CSI_DriverRole_**:
![Create the AmazonEKS_EBS_CSI_DriverRole](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_4.jpg)

## Install Prometheus for Cluster Monitoring
Install the Prometheus helm chart for monitoring the Kubernetes Cluster:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
```bash
helm install prometheus prometheus-community/prometheus
```

## Install the AWS Load Balancer Controller
Official Amazon AWS documentation: https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

Run:
```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json
```

```bash
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```

In the following code, replace **my-cluster** with the name of your EKS Cluster and **111122223333** with your AWS Account ID. Then run the command:

<pre><code>eksctl create iamserviceaccount \
  --cluster=<b>my-cluster</b> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::<b>111122223333</b>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve 
</code></pre>

- `helm repo add eks https://aws.github.io/eks-charts `
- `helm repo update`

In the following code, replace **my-cluster** with the name of your EKS Cluster, **602401143452** and **region-code** with the values corresponding to the AWS region you're using ([Check here](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)), and run the command:

<pre><code>helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<b>my-cluster</b> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=<b>602401143452</b>.dkr.ecr.<b>region-code</b>.amazonaws.com/amazon/aws-load-balancer-controller
</code></pre>

## Installing the Jenkins deployment
```bash
kubectl apply -f namespace.yaml
namespace/devops-tools created
```
```bash
kubectl apply -f serviceAccount.yaml
role.rbac.authorization.k8s.io/jenkins-admin created
serviceaccount/jenkins-admin created
rolebinding.rbac.authorization.k8s.io/jenkins-role-binding created
```
```bash
kubectl apply -f volume.yaml 
persistentvolumeclaim/jenkins-pv-claim created
```
```bash
kubectl apply -f deployment.yaml 
deployment.apps/jenkins created
```
```bash
kubectl apply -f service.yaml 
service/jenkins-service created
```

## Setup the AWS ALB Ingress

Get the list of public subnets in the cluster.
- Using **eksctl** in the console:
<pre><code>eksctl get cluster my-cluster
NAME            VERSION STATUS  CREATED                 VPC                     SUBNETS                                                                                                 SECURITYGROUPS          PROVIDER
my-cluster      1.24    ACTIVE  2022-12-12T14:38:34Z    vpc-0d3480fcbf26b253b   subnet-04fb2b4252c3e38b2,subnet-0c4b6807f56cbd85d,subnet-0ce7fbd629a3db193,subnet-0d4585057389798f7     sg-0c47545165b23b5d0    EKS
</code></pre>
- Using the AWS Management Console:
![List of public subnets](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_5.jpg)

Edit the **ingress.yaml** file. In the ***alb.ingress.kubernetes.io/subnets*** param add the public subnet list. Have in account to add just one subnet por availability zone:
<pre><code>annotations:
  alb.ingress.kubernetes.io/subnets: <b>subnet-0c4b6807f56cbd85d,subnet-0ce7fbd629a3db193</b>
</code></pre>
Create the ingress:
```
kubectl apply -f ingress.yaml 
ingress.networking.k8s.io/jenkins-ingress created
```

Verify the correct ALB creation in the AWS Management Console:

![](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_6.jpg)

## Accessing Jenkins
Access Jenkins through the DNS name of the Application Load Balancer:

![Jenkins Initial Setup](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_7.jpg)

To get the initial administrator password we need to check the Jenkins pod logs.
To identify the Jenkins pods run:
<pre><code><b>kubectl get pods -n devops-tools</b>
NAME                       READY   STATUS    RESTARTS   AGE
<b>jenkins-5ddc766476-ll82g</b>   1/1     Running   0          88m
</code></pre>

Get the last 20 log lines of the jenkins pod:
<pre><code><b>kubectl logs jenkins-5ddc766476-ll82g --tail 20 -n devops-tools</b>

*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

<b>a14f39190fa44226af623ce9560b0d7b</b>

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
</code></pre>
Enter the password, install the suggested plugins, create the first administrator user and access Jenkins:

![Installing plugins](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_10.jpg)
![Jenkins Reaady](https://johnruizcampos.com/wp-content/uploads/aws_eks_cluster_k8s_13.jpg)
