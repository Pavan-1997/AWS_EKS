# AWS EKS

Elastic Kubernetes Service is a Managed AWS Kubernetes offering. Where AWS manages the Control Plane and allows users to manage the Data Plane.

### Advantages:
```
Managed Control Plane: EKS takes care of managing the Kubernetes control plane components, such as the API server, controller manager, and etcd. AWS handles upgrades, patches, and ensures high availability of the control plane.

Automated Updates: EKS automatically updates the Kubernetes version, eliminating the need for manual intervention and ensuring that the cluster stays up-to-date with the latest features and security patches.

Scalability: EKS can automatically scale the Kubernetes control plane based on demand, ensuring the cluster remains responsive as the workload increases.

AWS Integration: EKS seamlessly integrates with various AWS services, such as AWS IAM for authentication and authorization, Amazon VPC for networking, and AWS Load Balancers for service exposure.

Security and Compliance: EKS is designed to meet various security standards and compliance requirements, providing a secure and compliant environment for running containerized workloads.

Monitoring and Logging: EKS integrates with AWS CloudWatch for monitoring cluster health and performance metrics, making it easier to track and troubleshoot issues.

Ecosystem and Community: Being a managed service, EKS benefits from continuous improvement, support, and contributions from the broader Kubernetes community.
```

### Disadvantages:
```
Cost: EKS is a managed service, and this convenience comes at a cost. Running an EKS cluster may be more expensive compared to self-managed Kubernetes, especially for large-scale deployments.

Less Control: While EKS provides a great deal of automation, it also means that you have less control over the underlying infrastructure and some Kubernetes configurations.
```

---
### Self-Managed Kubernetes on EC2 Instances

### Advanatages:
```
Cost-Effective: Self-managed Kubernetes allows you to take advantage of EC2 spot instances and reserved instances, potentially reducing the overall cost of running Kubernetes clusters.

Flexibility: With self-managed Kubernetes, you have full control over the cluster's configuration and infrastructure, enabling customization and optimization for specific use cases.

EKS-Compatible: Self-managed Kubernetes on AWS can still leverage various AWS services and features, enabling integration with existing AWS resources.

Experimental Features: Self-managed Kubernetes allows you to experiment with the latest Kubernetes features and versions before they are officially supported by EKS.
```

### Disadvantages:
```
Complexity: Setting up and managing a self-managed Kubernetes cluster can be complex and time-consuming, especially for those new to Kubernetes or AWS.

Maintenance Overhead: Self-managed clusters require manual management of Kubernetes control plane updates, patches, and high availability.

Scaling Challenges: Scaling the control plane of a self-managed cluster can be challenging, and it requires careful planning to ensure high availability during scaling events.

Security and Compliance: Self-managed clusters may require additional effort to implement best practices for security and compliance compared to EKS, which comes with some built-in security features.

Lack of Automation: Self-managed Kubernetes requires more manual intervention and scripting for certain operations, which can increase the risk of human error.
```

---
# Creating AWS EKS Cluster with Data Plane on AWS Fargate and Deploying 2048 Game using a Ubuntu machine

`Fargate - Its an AWS Serverless similar to AWS Lambda( but lambda is for small workloads)`
 
1. Now goto EC2 from AWS Console -> Click on Launch instance

	Give a name
	
	Use Ubuntu as an image
	
	Instance type as t2.micro
	
	Create a key pair if already present use existing one
	
	Click on Launch instance
	
	Connect to the instance


2. Install Docker
```
sudo apt update -y

sudo apt install apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"

apt-cache policy docker-ce

sudo apt install docker-ce

sudo usermod -aG docker $USER

sudo systemctl status docker 
```
```
sudo reboot
```
`Reboot the instance for Ubuntu user to execute docker commands`




3. Install AWS CLI
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

sudo apt install unzip

unzip awscliv2.zip

sudo ./aws/install

aws --version
```

4. Configure AWS CLI
```
aws configure
```
`Just give Access Key and Secret Key followed by ENTER-ENTER`


5. Install Kubectl

curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.6/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

kubectl version


6. Install Eksctl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp 

sudo mv /tmp/eksctl /usr/local/bin

eksctl version


7. Create a EKS Cluster on AWS Fargate

eksctl create cluster --name demo-cluster-1 --region us-east-2 --fargate


8. Now download the Kubeconfig file and saves the configuration to the path (/home/ubuntu/.kube/config)

aws eks update-kubeconfig --name demo-cluster-1 --region us-east-2


9. Now creating a Fargate profile for attaching the namespace 2048

eksctl create fargateprofile \
    --cluster demo-cluster-1 \
    --region us-east-2 \
    --name alb-sample-app \
    --namespace game-2048


10. Now Deploy the Namespace, Deployment, Service and Ingress

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml


11. Deploying an Ingress Controller

	a) Configuring IAM OIDC provider for ALB controller (is a K8s pod) to access the AWS Application LB
		
		export cluster_name=demo-cluster-1

		oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5) 
		
		eksctl utils associate-iam-oidc-provider --cluster demo-cluster-1 --region us-east-2 --approve 

	b) Now Setting up ALP Addon

		i) Download the IAM Policy

			curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

		ii) Create IAM Policy

			aws iam create-policy \
  			  --policy-name AWSLoadBalancerControllerIAMPolicy \
    			  --policy-document file://iam_policy.json
		
		iii) Create IAM Role (Make sure you give the cluster name and AWS Account ID)

			eksctl create iamserviceaccount \
 			 --cluster=demo-cluster-1 \
 			 --namespace=kube-system \
 			 --name=aws-load-balancer-controller \
  			--role-name AmazonEKSLoadBalancerControllerRole \
  			--attach-policy-arn=arn:aws:iam::317548628032:policy/AWSLoadBalancerControllerIAMPolicy \
			--region=us-east-2 \
  			--approve


	c) Deploy the ALB Controller using Helm


		i) Install Helm

			curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

			chmod 700 get_helm.sh

			./get_helm.sh

			helm version


		ii) Add Helm Repo

			helm repo add eks https://aws.github.io/eks-charts

		iii) Update the Helm Repo
		
			helm repo update eks



		iv) Install (Give your cluster name, region and VPC ID which can be found in the below image)

			helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  				--set clusterName=demo-cluster-1 \
 				--set serviceAccount.create=false \
  				--set serviceAccount.name=aws-load-balancer-controller \
  				--set region=us-east-2 \
  				--set vpcId=vpc-098cfa51f260d77f8

		v) Verify

			kubectl get deployment -n kube-system aws-load-balancer-controller


12. Now check the Pods for ALB Controller running

kubectl get deploy -n kube-system


13. Checking the Ingress resource which now shows address

kubectl get ingress -n game-2048

