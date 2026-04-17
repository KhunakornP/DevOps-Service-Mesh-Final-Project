# DevOps-Service-Mesh-Final-Project
Scripts for creating service mesh platform using Consul for building the Kubernetes multi cluster, and multi cloud system with platform failover.

This project uses source code attributed to the following projects 
1. [Consul crash course video](https://www.youtube.com/watch?v=s3I1kKKfjtQ) on YouTube


## Setting up the Project
### Prerequisites
The following application/command line tool must be installed:

- AWS CLI
- kubectl

### Hosting the Application on AWS

1. Navigate to the ```terraform``` directory
	```
	cd terraform
	```


2. Create a .tfvars file for environment variables
	```
	# windows
	type nul > terraform.tfvars

	# Mac / Linux
	touch terraform.tfvars
	```


3. Configure the following environment variables
	```
	aws_access_key_id     = "your-aws-access-key-id"
	aws_secret_access_key = "your-aws-access-secret-key"
	```
	The following variables are optionally configurable
	```
	aws_region       = "ap-southeast-1" (The AWS region for resources)
	instance_types   = ["t3.small"]     (AWS free tier supports t3, t4 small for accounts after 2015)
	k8s_cluster_name = "my-k8-cluster"  (Kubenetes cluster name containing letters, 0-9, or -)
	k8_version       = "1.30"           (The Kubernetes version to deploy. 1.33 < is depricated after of June 2026)
	```


4. Create the AWS resources
	```
	terraform init -upgrade
	terraform apply -var-file {your .tfvars file here}
	```

### Demo project accompanying a [Consul crash course video](https://www.youtube.com/watch?v=s3I1kKKfjtQ) on YouTube


Terraform commands to execute the script

```sh
# initialise project & download providers
terraform init

# preview what will be created with apply & see if any errors
terraform plan

# exeucute with preview
terraform apply -var-file terraform.tfvars

# execute without preview
terraform apply -var-file terraform.tfvars -auto-approve

# destroy everything
terraform destroy

# show resources and components from current state
terraform state list
```

#### Get access to EKS cluster
```sh
# install and configure awscli with access creds
aws configure

# check existing clusters list
aws eks list-clusters --region eu-central-1 --output table --query 'clusters'

# check config of specific cluster - VPC config shows whether public access enabled on cluster API endpoint
aws eks describe-cluster --region eu-central-1 --name myapp-eks-cluster --query 'cluster.resourcesVpcConfig'

# create kubeconfig file for cluster in ~/.kube
aws eks update-kubeconfig --region eu-central-1 --name myapp-eks-cluster

# test configuration
kubectl get svc
```
