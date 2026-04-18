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

Additionally, to run the terraform script you will need an AWS account with sufficient credits.

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

	# change the kubernetes config to the EKS on AWS
	aws eks update-kubeconfig --region {your region} --name {your app name}
	# example
	aws eks update-kubeconfig --region ap-southeast-1 --name myapp-eks-cluster

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

### Hosting the Application on Linode
1. Create a new Kubernetes cluster on Linode
	![ln1](/static-files/linode%20setup%201.JPG)

2. Choose a name for the cluster, along with a region that is closest to the AWS cluster
	![ln2](/static-files/linode%20setup%202.JPG)

3. Opt out of extra services to save cost
	![ln3](/static-files/linode%20setup%203.JPG)

4. Create at least 1 worker node for the cluster (use shared CPU to save cost)
	![ln4](/static-files/linode%20setup%204.JPG)

5. After the cluster is operational, download the kubeconfig file

6. Open a new terminal and switch the Kubernetes context to the Linode context
	```
	# Mac / Linux
	export KUBECONFIG=/path/to/config.yaml

	# Windows PowerShell
	$env:KUBECONFIG="/path/to/config.yaml"

	# confirm the context
	kubectl config current-context
	```

7. Install Consul on the cluster
	```
	# ensure that your worker nodes are running
	kubectl get nodes

	# install Consul
	helm install lke hashicorp/consul --version 1.0.0 --values consul-values.yaml --set global.datacenter=lke
	```

8. Run the e-commerce images, ensure you are in the ```\kubernetes``` directory
	```
	kubectl apply -f config-consul.yaml
	```

### Setting the Service Mesh
Run ```kubectl apply -f consul-mesh-gateway.yaml``` on both AWS and Linode clusters

To validate the command run
```
kubectl get mesh
```
You should see 1 mesh for each cluster.

### Setting up the Consul Peering Connection
Navigate to the peer connection tab on both Consul instances.
To access Consul, use ```kubectl get services``` and find the consul-ui LoadBalancer services.
Then connect to the services using a browser on HTTPS by accessing the load balancer IP.

From the AWS `eks` service, create a new peer connection.

Copy the token and paste it in the Linode `lke` service.
Be certain to name the connections `lke` and `eks` on the AWS and Linode Consul instances respectively.
AWS Consul    = `eks`
Linode Consul = `lke`


If successful your peering connection status will change to success.
![success](/static-files/final%20project%2025.JPG)
Note: In the screenshot, I configure the Linode Consul service to be called `lks`.
If either Consul name is changed, you will have to edit `service-resolver.yaml` and `export-service.yaml` files.
So that you can peer to your Consul instances.

### Configuring Service Failover
1. On the AWS cluster run
	```
	kubectl apply -f service-resolver.yaml
	```

2. On the Linode cluster run
	```
	kubectl apply -f exported-service.yaml
	```

If successful, your Consul peer connection should report 1 exported service and 1 imported service.
![done](/static-files/final%20project%2028.JPG)

To test the failover, run `kubectl delete deployment shippingservice`. The shopping cart page should still be accessible.

## Deviations from the Original Source Code.

1. Swap to t3.small for the instance_type variable. As of 2015, AWS free accounts only have access to t3 and t4 small EC2 instances.
2. Add config-gp3.yaml script. The script sets up a gp3 EBS volume for Consul to run on as Consul cannot run on gp2 volumes.
3. Add crds explicitly to consul-values.yaml. For some reason, the lke Consul instance sometimes failed to register
crds to configure ExportedService. It is added as redundancy.
4. Change the default Kubernetes version to 1.30 as version 1.27 is already deprecated on AWS.
5. Change the terraform AWS provider version from "~> 5.3" to "~> 5.8"
