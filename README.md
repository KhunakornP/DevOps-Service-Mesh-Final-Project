# DevOps-Service-Mesh-Final-Project
Scripts for creating service mesh platform using Consul for building the Kubernetes multi cluster, and multi cloud system with platform failover.

This project uses source code attributed to the following projects 
1. [Consul crash course video](https://www.youtube.com/watch?v=s3I1kKKfjtQ) on YouTube
2. [Google Cloud Platform Demo](https://github.com/GoogleCloudPlatform/microservices-demo)

## Setting up the Project
### Prerequisites
The following application/command line tool must be installed:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)
- [kubectl](https://kubernetes.io/docs/setup/)
- [HELM](https://helm.sh/docs/intro/install/)

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

5. After the resources are created, run the application using Kubernetes
	```
	# go to the kubernetes directory (assuming current directory is terraform)
	cd ..
	cd kubernetes

	# authenticate to AWS CLI
	aws login
	# or
	aws config

	# run the images
	kubectl apply -f config.yaml
	```

6. Check the status of the application
	```
	# check pods
	kubectl get pods

	# check services
	kubeclt get services
	```
	Notice that the paymentservice has the status CrashLoopBackOff.
	We will need to configure Consul to get the pod running.

7. Create an EBS (Elastic Block Storage) for Consul
	```
	kubectl apply -f config-gp3.yaml
	```

8. Creating the Consul data center
	```
	# register the Consul repository
	helm repo add hashicorp https://helm.releases.hashicorp.com

	# install Consul on Kubernetes
	helm install eks hashicorp/consul --version 1.0.0 --values consul-values.yaml --set global.datacenter=eks
	```

9. Check your Kubernetes pods
	```kubectl get pods``` should report similar pods, depending on what you name your data center
	```
	NAME                                               READY   STATUS    RESTARTS   AGE
	eks-consul-connect-injector-65cb48f9bb-bwcht       1/1     Running   0          45s
	eks-consul-mesh-gateway-66bf4dd558-gwkfz           1/1     Running   0          23m
	eks-consul-server-0                                1/1     Running   0          23m
	eks-consul-webhook-cert-manager-845479654f-92pth   1/1     Running   0          23m
	```

10. Recreate the containers to inject Consul sidecar
	```
	# destroy existing services
	kubectl delete -f config.yaml

	# re-create the services with a Consul ready configuration
	kubectl apply -f config-consul.yaml
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
