# Three Tier Architecture Deployment on AWS EKS

Stan's Robot Shop is a sample microservice application you can use as a sandbox to test and learn containerised application orchestration and monitoring techniques. It is not intended to be a comprehensive reference example of how to write a microservices application, although you will better understand some of those concepts by playing with Stan's Robot Shop. To be clear, the error handling is patchy and there is not any security built into the application.

You can get more detailed information from my [blog post](https://www.instana.com/blog/stans-robot-shop-sample-microservice-application/) about this sample microservice application.

This sample microservice application has been built using these technologies:
- NodeJS ([Express](http://expressjs.com/))
- Java ([Spring Boot](https://spring.io/))
- Python ([Flask](http://flask.pocoo.org))
- Golang
- PHP (Apache)
- MongoDB
- Redis
- MySQL ([Maxmind](http://www.maxmind.com) data)
- RabbitMQ
- Nginx
- AngularJS (1.x)


### Understanding the Basics:

The 3-tier architecture is composed of three primary layers, each with distinct responsibilities:sponsibilities:

1. Presentation Layer:

* Also known as the user interface layer, this tier is responsible for interacting with end-users.
  
* Encompasses the user interface components, such as web pages, mobile apps, or any other interface through which users interact with the application.
  
* The goal is to provide a seamless and intuitive user experience while keeping the presentation logic separate from the business logic.
  
2. Application (or Business Logic) Layer:

* Positioned between the presentation and data layers, the application layer contains the business logic that processes and manages user requests.

* It acts as the brain of the application, handling tasks such as data validation, business rules implementation, and decision-making.

* Separating the business logic from the presentation layer promotes code reusability, maintainability, and adaptability to changes.

3. Data Layer:

* The data layer is responsible for managing and storing the application’s data.

*  It includes databases, data warehouses, or any other data storage solutions.

* This layer ensures data integrity, security, and efficient data retrieval for the application.

By isolating data-related operations, developers can optimize data access and storage mechanisms independently of the rest of the application.


### Benefits of 3-Tier Architecture:

1. Scalability:
   
The modular nature of 3-tier architecture allows for independent scaling of each layer. This enables efficient resource allocation, ensuring that specific components can be scaled based on demand without affecting the entire application.

2. Maintainability:

With clear separation of concerns, developers can make changes to one layer without impacting others. This facilitates easier debugging, updates, and maintenance, as modifications can be confined to the relevant layer.

3. Flexibility and Adaptability:

The architecture accommodates technology changes and updates without disrupting the entire system. New technologies can be integrated into specific layers, allowing the application to evolve over time.



### Prerequisites for this setup include:

1. kubectl: A command-line tool for managing Kubernetes clusters. It allows users to deploy, inspect, and manage Kubernetes resources such as pods, deployments, services, and more. Kubectl enables users to perform operations such as creating, updating, deleting, and scaling Kubernetes resources.

Run the following steps to install kubectl on your local machine / VM(EC2 instance - Ubuntu image).

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Use [official documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) for kubectl installation on other OS


Add execute permission to the binary:
```
chmod +x ./kubectl
```

Move the binary to a directory in your path:
```
sudo mv ./kubectl /usr/local/bin/kubectl
```

Verify the installation:

```
kubectl version --client
```

The output would look like this,

![image](https://github.com/user-attachments/assets/223e9a45-66bf-4f20-8f7f-1fc439ba7797)


2. eksctl: A command-line tool designed for working with EKS clusters, streamlining various tasks. The eksctl tool uses CloudFormation under the hood, creating one stack for the EKS master control plane and another stack for the worker nodes.

Install and set up eksctl - Download and extract the latest release of eksctl with the following command.

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp &&

sudo mv /tmp/eksctl /usr/local/bin
```

You can check the eksctl version using,

```
eksctl version
```

Use [official documentation](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/setting-up-eksctl.html) for eksctl installation on other OS.

The output would look like this,

![image](https://github.com/user-attachments/assets/ea963b1c-4c7d-407a-b300-b930a44be52f)


3. AWS CLI: A command-line interface for interacting with AWS services, including Amazon EKS. To install, update, and uninstall AWS CLI, follow the instructions in the AWS Command Line Interface User Guide. After installation, it is advisable to configure the AWS CLI using the guidelines outlined in the AWS Command Line Interface User Guide under “Quick configuration with aws configure.”

To install the AWS CLI, run the following commands.

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" unzip awscliv2.zip

sudo ./aws/install
```

Use [official documentation]([https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/setting-up-eksctl.html) for eksctl installation on other OS


### Steps:


#### Step-1: Create an EKS Cluster


```
eksctl create cluster \
  --name <your-cluster-name> \
  --region <your-region> \
  --nodegroup-name <your-nodegroup-name> \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
```

The output will be,

![1 2](https://github.com/user-attachments/assets/0b6bb282-88a4-42c0-b5ab-d6a4398a7abb)

This will create an EKS Cluster along with the node-group,

![1 3](https://github.com/user-attachments/assets/8a67c423-114c-4316-97e4-4e02a8d58520)

![1 4](https://github.com/user-attachments/assets/797c8c9f-e5b5-4f63-a925-cbb32c055a1c)


#### Step-2: Configure IAM OIDC provider

1. Export Cluster Name and assign oidc_id

```
export cluster_name=<CLUSTER-NAME>

oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```


2. Check if there is an IAM OIDC provider configured already

```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

3. If not, run the below command.

```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

![1 1](https://github.com/user-attachments/assets/8a29167b-10fb-47c3-8f98-e6f84dfc3900)



We will use Ingress(ALB Resource) and Ingress Controller(ALB Resource Controller) to expoose the application to extrenal users instead of using Load Balancer Service type to expose the application .

Kubernets has default created service accounts. So we can connect an IAM user in AWS with a service account using IAM Policies for the pods in Kuberbetes to access IAM resources in AWS. So 

#### Step-3: Setup ALB Add-On



1. Download IAM policy.
 
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

2. Create IAM Policy.

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

3. Create IAM Role.

```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

The ouput will be as below,

![1 7](https://github.com/user-attachments/assets/963e8b69-0fc2-460b-bb92-459a60234616)


#### Step-4: Deploy ALB controller

Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
```

2. Update the repo.

```
helm repo update eks
```

3. Install the chart.

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<your-cluster-name> \--set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=<region> --set vpcId=<your-vpc-id>
```

4. Verify that the deployments are running.

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

![1 9](https://github.com/user-attachments/assets/832acb84-f98f-45fd-8c94-efa69cc3f7f0)


#### Step-5: EBS CSI Plugin configuration.


When we cretae PVC(Persistent Volume Claim), the storage class will look at the claim and manage to cretae a EBS volume. 
In EKS, EBS CSI plugin with SStorage Class  will automatically create an EBS volume when a PVS is created and attached it to Stateful Set where in memory storage is required between sessions in the application.

1. The Amazon EBS CSI plugin requires IAM permissions to make calls to AWS APIs on your behalf.

2. Create an IAM role and attach a policy. AWS maintains an AWS managed policy or you can create your own custom policy. You can create an IAM role and attach the AWS managed policy with the following command. Replace my-cluster with the name of your cluster. The command deploys an AWS CloudFormation stack that creates an IAM role and attaches the IAM policy to it.

```
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <YOUR-CLUSTER-NAME> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

The output will be,

![2 0](https://github.com/user-attachments/assets/463dffa4-7acb-4105-a7f5-00a95234a2e8)

3. Run the following command. Replace with the name of your cluster, with your account ID.

```
eksctl create addon --name aws-ebs-csi-driver --cluster <YOUR-CLUSTER-NAME> --service-account-role-arn arn:aws:iam::<AWS-ACCOUNT-ID>:role/AmazonEKS_EBS_CSI_DriverRole --force
```


#### Step-6: Deploy the entire project as Helm Chart:

Advantages using Helm - It makes kubernetes resources easily deployable and manageable. For example if I need change the image repository of each micro service manually, I need to use their relavant deployment and services filemanifests and change each file to the relavant image. But if I use Helm, I can change the image repository only in the values.yaml file in Helm and it will effect to all micro-services defined as deployment and services.

1. Initially Clone the GitHub Repo:

```
git clone https://github.com/RavDas/Three-Tier-Architecture-Application-Deployment.git
```

2. Then Navigate to the path where Chart.yaml exists

```
cd RobotShop-Project/EKS/helm
```

3. Create a namespace and then install the helm chart in the namespace.

```
kubectl create ns robot-shop
helm install robot-shop --namespace robot-shop .
```

4. Ensure all the pods are running if not troubleshoot the issues.

```
get pods -n robot-shop
```

![2 1](https://github.com/user-attachments/assets/bc67ea59-b1ea-4ae7-a220-08a155399ada)


#### Step 7: Create Ingress to expose for external users

Goto,

```
cd /RobotShop-Project/EKS/helm
```
```
 kubectl apply -f ingress.yaml
 ```

This will create a Load Balancer on the AWS console.

![2 2](https://github.com/user-attachments/assets/e78e0aca-ab87-4dd1-bb07-ecfd6e5151f1)


#### Step 8: ACcess the deployed application

Paste the DNS-name on your favorite browser and access the application.

![2 4](https://github.com/user-attachments/assets/1927d388-c6c4-40dc-b888-80123051c909)

Dashboard of the application.

![2 5](https://github.com/user-attachments/assets/6e03271b-4083-40fe-bb35-bc9101bab391)

Get Register and login into the application.

![2 6](https://github.com/user-attachments/assets/5ded073c-7577-4f25-a109-5bbd527613a3)

Add to cart the robots.

![2 7](https://github.com/user-attachments/assets/be838a14-96d7-4109-b816-34e6108a26d7)






==============================================================================================================



The various services in the sample application already include all required Instana components installed and configured. The Instana components provide automatic instrumentation for complete end to end [tracing](https://docs.instana.io/core_concepts/tracing/), as well as complete visibility into time series metrics for all the technologies.

To see the application performance results in the Instana dashboard, you will first need an Instana account. Don't worry a [trial account](https://instana.com/trial?utm_source=github&utm_medium=robot_shop) is free.

## Build from Source
To optionally build from source (you will need a newish version of Docker to do this) use Docker Compose. Optionally edit the `.env` file to specify an alternative image registry and version tag; see the official [documentation](https://docs.docker.com/compose/env-file/) for more information.

To download the tracing module for Nginx, it needs a valid Instana agent key. Set this in the environment before starting the build.

```shell
$ export INSTANA_AGENT_KEY="<your agent key>"
```

Now build all the images.

```shell
$ docker-compose build
```

If you modified the `.env` file and changed the image registry, you need to push the images to that registry

```shell
$ docker-compose push
```

## Run Locally
You can run it locally for testing.

If you did not build from source, don't worry all the images are on Docker Hub. Just pull down those images first using:

```shell
$ docker-compose pull
```

Fire up Stan's Robot Shop with:

```shell
$ docker-compose up
```

If you want to fire up some load as well:

```shell
$ docker-compose -f docker-compose.yaml -f docker-compose-load.yaml up
```

If you are running it locally on a Linux host you can also run the Instana [agent](https://docs.instana.io/quick_start/agent_setup/container/docker/) locally, unfortunately the agent is currently not supported on Mac.

There is also only limited support on ARM architectures at the moment.

## Marathon / DCOS

The manifests for robotshop are in the *DCOS/* directory. These manifests were built using a fresh install of DCOS 1.11.0. They should work on a vanilla HA or single instance install.

You may install Instana via the DCOS package manager, instructions are here: https://github.com/dcos/examples/tree/master/instana-agent/1.9

## Kubernetes
You can run Kubernetes locally using [minikube](https://github.com/kubernetes/minikube) or on one of the many cloud providers.

The Docker container images are all available on [Docker Hub](https://hub.docker.com/u/robotshop/).

Install Stan's Robot Shop to your Kubernetes cluster using the [Helm](K8s/helm/README.md) chart.

To deploy the Instana agent to Kubernetes, just use the [helm](https://github.com/instana/helm-charts) chart.

## Accessing the Store
If you are running the store locally via *docker-compose up* then, the store front is available on localhost port 8080 [http://localhost:8080](http://localhost:8080/)

If you are running the store on Kubernetes via minikube then, find the IP address of Minikube and the Node Port of the web service.

```shell
$ minikube ip
$ kubectl get svc web
```

If you are using a cloud Kubernetes / Openshift / Mesosphere then it will be available on the load balancer of that system.

## Load Generation
A separate load generation utility is provided in the `load-gen` directory. This is not automatically run when the application is started. The load generator is built with Python and [Locust](https://locust.io). The `build.sh` script builds the Docker image, optionally taking *push* as the first argument to also push the image to the registry. The registry and tag settings are loaded from the `.env` file in the parent directory. The script `load-gen.sh` runs the image, it takes a number of command line arguments. You could run the container inside an orchestration system (K8s) as well if you want to, an example descriptor is provided in K8s directory. For End-user Monitoring ,load is not automatically generated but by navigating through the Robotshop from the browser .For more details see the [README](load-gen/README.md) in the load-gen directory.  

## Website Monitoring / End-User Monitoring

### Docker Compose

To enable Website Monioring / End-User Monitoring (EUM) see the official [documentation](https://docs.instana.io/website_monitoring/) for how to create a configuration. There is no need to inject the JavaScript fragment into the page, this will be handled automatically. Just make a note of the unique key and set the environment variable `INSTANA_EUM_KEY` and `INSTANA_EUM_REPORTING_URL` for the web image within `docker-compose.yaml`.

### Kubernetes

The Helm chart for installing Stan's Robot Shop supports setting the key and endpoint url required for website monitoring, see the [README](K8s/helm/README.md).

## Prometheus

The cart and payment services both have Prometheus metric endpoints. These are accessible on `/metrics`. The cart service provides:

* Counter of the number of items added to the cart

The payment services provides:

* Counter of the number of items perchased
* Histogram of the total number of items in each cart
* Histogram of the total value of each cart

To test the metrics use:

```shell
$ curl http://<host>:8080/api/cart/metrics
$ curl http://<host>:8080/api/payment/metrics
```

