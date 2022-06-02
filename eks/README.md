# LATAM Containers RoadShow EKS Workshop

In this workshop, we will explore multiple ways to configure VPC, ALB, and EC2 Kubernetes DataPlane Nodes, and Amazon Elastic Kubernetes Service (EKS).

# Modules

- Provisioning you EKS Cluster
- Cluster Autoscaler
- Better scale strategy with Karpenter
    - Usage of Graviton and Spot instances to reduce costs
- Observability with Amazon CloudWatch Container Insights
- CI/CD Pipelines

# Getting Started

### Installing kubectl

```bash
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

### Update awscli 

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Install jq, envsubst (from GNU gettext utilities) and bash-completion

```bash
sudo yum -y install jq gettext bash-completion moreutils
```

### Install yq for yaml processing 

```bash
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc
```

### Enable kubectl bash_completion

```bash
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

### Installing Terraform

```bash
wget https://releases.hashicorp.com/terraform/1.2.1/terraform_1.2.1_linux_386.zip
unzip terraform_1.2.1_linux_386.zip
sudo mv terraform /usr/local/bin/
sudo chmod +x /usr/local/bin/terraform
```

### Installing Helm CLI

```bash
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm repo add stable https://charts.helm.sh/stable
```

# Provisioning your EKS Cluster

For the cluster provision we are going to use [**EKS Blueprints**](https://aws.amazon.com/blogs/containers/bootstrapping-clusters-with-eks-blueprints/). EKS Blueprints is a collection of Infrastructure as Code (IaC) modules that will help you configure and deploy consistent, batteries-included EKS clusters across accounts and regions. You can use EKS Blueprints to easily bootstrap an EKS cluster with Amazon EKS add-ons as well as a wide range of popular open-source add-ons, including Prometheus, Karpenter, Nginx, Traefik, AWS Load Balancer Controller, Fluent Bit, Keda, Argo CD, and more. EKS Blueprints also helps you implement relevant security controls needed to operate workloads from multiple teams in the same cluster.


### Step 1: Clone the repo using the command below

```bash
git clone https://github.com/davivcgarcia/latam-containers-roadshow.git
```

### Step 2: Run Terraform INIT

Initialize a working directory with configuration files

```bash
cd eks/terraform/
terraform init
```

### Step 3: Run Terraform PLAN

Verify the resources created by this execution

```bash
terraform plan
```

### Step 4: Finally, Terraform APPLY

to create resources

```bash
terraform apply --auto-approve
```

## Configure kubectl and test cluster

EKS Cluster details can be extracted from terraform output or from AWS Console to get the name of cluster. This following command used to update the kubeconfig in your local machine where you run kubectl commands to interact with your EKS Cluster.

### Step 5: Run update-kubeconfig command

`~/.kube/config` file gets updated with cluster details and certificate from the below command

```bash
aws eks --region us-east-1 update-kubeconfig --name latam-containers-roadshow
```

### Step 6: List all the worker nodes by running the command below

```bash
kubectl get nodes
```

# Cluster Auto-Scaler

In this module, we will show patterns for scaling your worker nodes and applications deployments automatically.

Automatic scaling in K8s comes in two forms:

- **Horizontal Pod Autoscaler (HPA)** scales the pods in a deployment or replica set. It is implemented as a K8s API resource and a controller. The controller manager queries the resource utilization against the metrics specified in each HorizontalPodAutoscaler definition. It obtains the metrics from either the resource metrics API (for per-pod resource metrics), or the custom metrics API (for all other metrics).

- **Cluster Autoscaler (CA)** a component that automatically adjusts the size of a Kubernetes Cluster so that all pods have a place to run and there are no unneeded nodes.

## Install kube-ops-view

Before starting to learn about the various auto-scaling options for your EKS cluster we are going to install [Kube-ops-view](https://github.com/hjacobs/kube-ops-view) from [Henning Jacobs](https://github.com/hjacobs).

Kube-ops-view provides a common operational picture for a Kubernetes cluster that helps with understanding our cluster setup in a visual way.

```bash
helm install kube-ops-view \
stable/kube-ops-view \
--set service.type=LoadBalancer \
--set rbac.create=True
```

To check the chart was installed successfully:

```bash
helm list
```

should display:

```output
NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
kube-ops-view   1               Sun Sep 22 11:47:31 2019        DEPLOYED        kube-ops-view-1.1.0     0.11            default  
```

With this we can explore kube-ops-view output by checking the details about the newly service created.

```bash
kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'
```

This will display a line similar to Kube-ops-view URL = http://<URL_PREFIX_ELB>.amazonaws.com Opening the URL in your browser will provide the current state of our cluster.

<p align="center"> 
<img src="static/kube-ops-view.png">
</p>

##  Scale an Application with HPA

For scale applications with HPA we will need the metric server installed in our cluster, we already installed it using the EKS blueprints with terraform showing below:

```terraform
# Add-ons
  enable_aws_load_balancer_controller = true
  enable_metrics_server               = true
```

### Deploy a sample app

We will deploy an application and expose as a service on TCP port 80.

The application is a custom-built image based on the php-apache image. The index.php page performs calculations to generate CPU load.

```bash
kubectl create deployment php-apache --image=us.gcr.io/k8s-artifacts-prod/hpa-example
kubectl set resources deploy php-apache --requests=cpu=200m
kubectl expose deploy php-apache --port 80

kubectl get pod -l app=php-apache
```

### Create an HPA resource

This HPA scales up when CPU exceeds 50% of the allocated container resource.

```bash
kubectl autoscale deployment php-apache `#The target average CPU utilization` \
    --cpu-percent=50 \
    --min=1 `#The lower limit for the number of pods that can be set by the autoscaler` \
    --max=10 `#The upper limit for the number of pods that can be set by the autoscaler`
```
View the HPA using kubectl. You probably will see `<unknown>/50%` for 1-2 minutes and then you should be able to see `0%/50%`

```bash
kubectl get hpa
```

## Generate load to trigger scaling 

Open a new terminal in the Cloud9 Environment and run the following command to drop into a shell on a new container

```bash
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

Execute a while loop to continue getting http:///php-apache

```bash
while true; do wget -q -O - http://php-apache; done
```

In the previous tab, watch the HPA with the following command

```bash
kubectl get hpa -w
```

You will see HPA scale the pods from 1 up to our configured maximum (10) until the CPU average is below our target (50%)

<p align="center"> 
<img src="static/scaling-hpa-results.png">
</p>

You can now stop (Ctrl + C) load test that was running in the other terminal. You will notice that HPA will slowly bring the replica count to min number based on its configuration. You should also get out of load testing application by pressing Ctrl + D.

## Configure Cluster Autoscaler (CA)

Cluster Autoscaler for AWS provides integration with Auto Scaling groups. It enables users to choose from four different options of deployment:

    One Auto Scaling group
    Multiple Auto Scaling groups
    Auto-Discovery
    Control-plane Node setup

Auto-Discovery is the preferred method to configure Cluster Autoscaler.

Cluster Autoscaler will attempt to determine the CPU, memory, and GPU resources provided by an Auto Scaling Group based on the instance type specified in its Launch Configuration or Launch Template.

### Configure the ASG

You configure the size of your Auto Scaling group by setting the minimum, maximum, and desired capacity. When we created the cluster we set these settings to 2 in the terraform manifest.

```terraform
managed_node_groups = {
    mg_5 = {
      node_group_name = "managed-ondemand"
      instance_types  = ["m5.large"]
      desired_size = 2
      max_size     = 2
      min_size     = 2
      subnet_ids      = module.vpc.private_subnets
    }
  }
```

Now, increase the maximum capacity to 4 instances. Open the `terraform/main.tf` file and update the `max_size` from 2 to 4, you will have your manifest like this:

```terraform
managed_node_groups = {
    mg_5 = {
      node_group_name = "managed-ondemand"
      instance_types  = ["m5.large"]
      desired_size = 2
      max_size     = 4
      min_size     = 2
      subnet_ids      = module.vpc.private_subnets
    }
  }
```

Let's also enable the `Cluster-Autoscaler` in the add-ons section of our terraform manifest. Open the `terraform/main.tf` and change the `enable_cluster_autoscaler` from `false` to `true`.

```terraform
module "eks_blueprints_kubernetes_addons" {
  source = "github.com/aws-ia/terraform-aws-eks-blueprints//modules/kubernetes-addons?ref=v4.0.7"

  eks_cluster_id = module.eks_blueprints.eks_cluster_id

  # EKS Managed Add-ons
  enable_amazon_eks_vpc_cni    = true
  enable_amazon_eks_coredns    = true
  enable_amazon_eks_kube_proxy = true

  # Add-ons
  enable_aws_load_balancer_controller = true
  enable_metrics_server               = true
  enable_cluster_autoscaler           = true
  enable_karpenter                    = false
  enable_aws_cloudwatch_metrics       = false
  enable_aws_for_fluentbit            = false
  
  tags = local.tags

  depends_on = [module.eks_blueprints.managed_node_groups]
}
```

### Run Terraform PLAN

Verify the resources created by this execution

```bash
cd eks/terraform/
terraform plan
```

### Finally, Terraform APPLY

to create resources

```bash
terraform apply --auto-approve
```

Watch the logs to verify that Cluster Autoscaler was deployed successfully.

```bash
kubectl -n kube-system logs -f deployment/cluster-autoscaler-aws-cluster-autoscaler
```

## Create sample-app to test the CA

We will deploy an sample nginx application as a ReplicaSet of 1 Pod.

```bash
cat <<EoF> ~/environment/nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF

kubectl apply -f ~/environment/nginx.yaml

kubectl get deployment/nginx-to-scaleout
```

### Scale our ReplicaSet

```bash
kubectl scale --replicas=10 deployment/nginx-to-scaleout
```

Some pods will be in the `Pending` state, which triggers the cluster-autoscaler to scale out the EC2 fleet.

```bash
kubectl get pods -l app=nginx -o wide --watch
```

Check the [EC2 AWS Management Console](https://console.aws.amazon.com/ec2/home?#Instances:sort=instanceId) to confirm that the Auto Scaling groups are scaling up to meet demand. This may take a few minutes. You can also follow along with the pod deployment from the command line. You should see the pods transition from pending to running as nodes are scaled up.

Or verify it using `kubectl`

```bash
kubectl get nodes
```

Output will be similar like this:

```output

ip-192-168-12-114.us-east-2.compute.internal   Ready    <none>   3d6h   v1.17.7-eks-bffbac
ip-192-168-29-155.us-east-2.compute.internal   Ready    <none>   63s    v1.17.7-eks-bffbac
ip-192-168-55-187.us-east-2.compute.internal   Ready    <none>   3d6h   v1.17.7-eks-bffbac
ip-192-168-82-113.us-east-2.compute.internal   Ready    <none>   8h     v1.17.7-eks-bffbac
```

### Cleanup Scaling

```bash
kubectl delete -f ~/environment/nginx.yaml

kubectl delete hpa,svc php-apache
kubectl delete deployment php-apache
kubectl delete pod load-generator

helm uninstall kube-ops-view
```

# Flexible Computing with Karpenter

[Karpenter](https://karpenter.sh/) is an open-source autoscaling project built for Kubernetes. Karpenter is designed to provide the right compute resources to match your application’s needs in seconds, instead of minutes by observing the aggregate resource requests of unschedulable pods and makes decisions to launch and terminate nodes to minimize scheduling latencies.

Let's enable Karpenter and disable Cluster Autoscaler in our EKS cluster using terraform. Open the `eks/terraform/main.tf` and set `enable_cluster_autoscale` from `true` to `false`, also set the `enable_karpenter` from `false` to `true`:

```terraform
module "eks_blueprints_kubernetes_addons" {
  source = "github.com/aws-ia/terraform-aws-eks-blueprints//modules/kubernetes-addons?ref=v4.0.7"

  eks_cluster_id = module.eks_blueprints.eks_cluster_id

  # EKS Managed Add-ons
  enable_amazon_eks_vpc_cni    = true
  enable_amazon_eks_coredns    = true
  enable_amazon_eks_kube_proxy = true

  # Add-ons
  enable_aws_load_balancer_controller = true
  enable_metrics_server               = true
  enable_cluster_autoscaler           = false # Disable Cluster Autoscaler
  enable_karpenter                    = true # Enable Karpenter
  enable_aws_cloudwatch_metrics       = false
  enable_aws_for_fluentbit            = false
  
  tags = local.tags

  depends_on = [module.eks_blueprints.managed_node_groups]
}
```

Now, decrease the maximum capacity to 2 instances. Open the `eks/terraform/main.tf` file and update the `max_size` from 4 to 2, you will have your manifest like this:

```terraform
managed_node_groups = {
    mg_5 = {
      node_group_name = "managed-ondemand"
      instance_types  = ["m5.large"]
      desired_size = 2
      max_size     = 2
      min_size     = 2
      subnet_ids      = module.vpc.private_subnets
    }
  }
```

### Run Terraform PLAN

Verify the resources created by this execution

```bash
cd eks/terraform/
terraform plan
```

### Finally, Terraform APPLY

to create resources

```bash
terraform apply --auto-approve
```

Doing that will remove the Cluster Autoscaler and all related resources from our cluster and install Karpenter with all resources.

## Setup the Provisioner

Karpenter configuration comes in the form of a Provisioner CRD (Custom Resource Definition). A single Karpenter provisioner is capable of handling many different pod shapes. Karpenter makes scheduling and provisioning decisions based on pod attributes such as labels and affinity. A cluster may have more than one Provisioner, but for the moment we will declare just one: the default Provisioner.

One of the main objectives of Karpenter is to simplify the management of capacity. If you are familiar with other Auto Scalers, you will notice Karpenter takes a different approach. You may have heard the approached referred as group-less auto scaling. Other Solutions have traditionally used the concept of a node group as the element of control that defines the characteristics of the capacity provided (i.e: On-Demand, EC2 Spot, GPU Nodes, etc) and that controls the desired scale of the group in the cluster. In AWS the implementation of a node group matches with [Auto Scaling groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html). Over time, clusters using this paradigm, that run different type of applications requiring different capacity types, end up with a complex configuration and operational model where node groups must be defined and provided in advance.

Let’s deploy the following provisioner:

```bash
export CLUSTER_NAME='latam-containers-roadshow'

cat <<EOF> ~/environment/provisioner.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  labels:
    intent: apps
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: kubernetes.io/arch
      operator: In
      values:
      - amd64
      - arm64
  limits:
    resources:
      cpu: 1000
  provider:
    instanceProfile: ${CLUSTER_NAME}-managed-ondemand
    subnetSelector:
      Name: "${CLUSTER_NAME}-private*"
    securityGroupSelector:
      karpenter.sh/discovery/${CLUSTER_NAME}: ${CLUSTER_NAME}
  ttlSecondsAfterEmpty: 30
EOF

kubectl apply -f ~/environment/provisioner.yaml
```

The configuration for this provider is quite simple. For the moment let’s focus in a few of the settings used.

**Requirements Section:** The [Provisioner CRD](https://karpenter.sh/docs/provisioner-crd/) supports defining node properties like instance type and zone. For example, in response to a label of `topology.kubernetes.io/zone=us-east-1c`, Karpenter will provision nodes in that availability zone. In this example we are setting the `karpenter.sh/capacity-type` to search EC2 Spot instances. You can learn which other properties are [available here](https://karpenter.sh/v0.7.2/aws/provisioning/).

**Limits section:** Provisioners can define a limit in the number of CPU’s and memory allocated to that particular provisioner and part of the cluster.

**Provider section:** This provisioner uses `securityGroupSelector` and subnetSelector to discover resources used to launch nodes. It uses the tags that Karpenter attached to the subnets.

**ttlSecondsAfterEmpty:** value configures Karpenter to terminate empty nodes. This behavior can be disabled by leaving the value undefined. In this case we have set it for a quick demonstration to a value of 30 seconds.

**Tags:** Provisioners can also define a set of tags that the EC2 instances will have upon creation. This helps to enable accounting and governance at the EC2 level.

## Displaying Karpenter Logs

To read Karpenter logs from the console you can run the following command.

```bash
kubectl logs -f deployment/karpenter -c controller -n karpenter
```

## Provisioning new nodes with Karpenter

With Karpenter now active, we can begin to explore how Karpenter provisions nodes. In this section we are going to create some pods using a deployment we will watch Karpenter provision nodes in response.

For this workshop, we will use a simple nginx container with **more resources requests than needed**, and see how Karpenter deal with it.

```bash
cat <<EoF> ~/environment/karpenter-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-scale-karpenter
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-scale-karpenter
  template:
    metadata:
      labels:
        service: nginx-scale-karpenter
        app: nginx-scale-karpenter
    spec:
      containers:
      - image: nginx
        name: nginx-scale-karpenter
        resources:
          limits:
            cpu: 1000m
            memory: 1000Mi
          requests:
            cpu: 1000m
            memory: 1000Mi
EoF

kubectl apply -f ~/environment/karpenter-nginx.yaml
```

List all available pods in the default namespace:

```bash
kubectl get pods
```

Observe that 2 pods are with status `Pending`. The reason is that the cluster does not have any node availble required by the Pod's resource request.

Karpenter is monitoring the cluster and is provisioning a new node. Watch the node creation until the status is `Ready`:

```bash
watch kubectl get nodes
```

When the new node is `Ready`, list the pods to validate that the application is Running:

```bash
kubectl get pods
```

## Undeploy the application

Now that we understand how Karpenter works, let's undeploy the nginx application:

```bash
kubectl delete -f ~/environment/karpenter-nginx.yaml
```

Since the new node does not have pods running on it, Karpenter will remove the node. Watch the list of nodes until the new node is removed:

```bash
watch kubectl get nodes
```

## Deploy a ARM based application

When a provisioner does not specify the architecture and also the instance types on which it can create new nodes, by default all instances and all architectures (amd64 and arm64) will be used. This means that a [Graviton](https://aws.amazon.com/pm/ec2-graviton/) instance can be created, and with that, the application that is to be executed may not support the architecture of the instance, which can cause errors at execution time.

It's possible to utilize [well-known label](https://kubernetes.io/docs/reference/labels-annotations-taints/) kubernetes.io/arch in the application manifest with a *node selector*. In that way Karpenter will take in consideration the selector and provision the specific node for that use case.

Let's provision an `arm` application with a `arm64` node selector:

```bash
cat <<EoF> ~/environment/karpenter-arm64.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-arm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world-arm
  template:
    metadata:
      labels:
        service: hello-world-arm
        app: hello-world-arm
    spec:
      containers:
      - image: public.ecr.aws/nginx/nginx:1.21-arm64v8
        name: hello-world-arm
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
      nodeSelector:
        kubernetes.io/arch: arm64  
EoF

kubectl apply -f ~/environment/karpenter-arm64.yaml
```

List all available pods in th default namespace:

```bash
kubectl get pods
```

Observe that the pod is with status Pending. The reason is that the cluster does not have any node availble required by the Pod's nodeSelector (kubernetes.io/arch: arm64).

Karpenter is monitoring the cluster and is provisioning a new node. Watch the node creation until the status is `Ready`:

```bash
watch kubectl get nodes
```

Once the node is `Ready`, let's get the Node name which the Pod is running:

```bash
export ARM_NODE=$(kubectl get pods -owide | awk '{print $7}' | grep -vi node)
```

Now let's check the Node labels, with the labels we are able to see if the node was provisioned in the correct architecture.

```bash
kubectl get node $ARM_NODE --show-labels
```

The result will look like the following:

```output
NAME                          STATUS   ROLES    AGE   VERSION                LABELS
ip-10-0-11-211.ec2.internal   Ready    <none>   15m   v1.21.12-eks-5308cf7   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/instance-type=t4g.small,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-east-1,failure-domain.beta.kubernetes.io/zone=us-east-1b,intent=apps,karpenter.sh/capacity-type=spot,karpenter.sh/provisioner-name=default,kubernetes.io/arch=arm64,kubernetes.io/hostname=ip-10-0-11-211.ec2.internal,kubernetes.io/os=linux,node.kubernetes.io/instance-type=t4g.small,topology.kubernetes.io/region=us-east-1,topology.kubernetes.io/zone=us-east-1b
```

As you can see this Node has the label `kubernetes.io/arch=arm64`, showing that is a arm node powered by Graviton.

## Undeploy the ARM application

```bash
kubectl delete -f ~/environment/karpenter-arm64.yaml 
```

Since the new node does not have pods running on it, Karpenter will remove the node. Watch the list of nodes until the new node is removed:

```bash
watch kubectl get nodes
```

# Observability with Amazon Cloudwatch Container Insights

In this module we will learn and leverage the new CloudWatch Container Insights to see how you can use native CloudWatch features to monitor your EKS Cluster performance.

You can use CloudWatch Container Insights to collect, aggregate, and summarize metrics and logs from your containerized applications and microservices. Container Insights is available for Amazon Elastic Container Service, Amazon Elastic Kubernetes Service, and Kubernetes platforms on Amazon EC2. The metrics include utilization for resources such as CPU, memory, disk, and network. Container Insights also provides diagnostic information, such as container restart failures, to help you isolate issues and resolve them quickly.

Let's first enable CloudWatch and Fluent-bit add-ons in `eks/terraform/main.tf`, set the flag `enable_aws_cloudwatch_metrics` from `false` to `true`, and the flag `enable_aws_for_fluentbit` from `false` to `true`


```terraform
module "eks_blueprints_kubernetes_addons" {
  source = "github.com/aws-ia/terraform-aws-eks-blueprints//modules/kubernetes-addons?ref=v4.0.7"

  eks_cluster_id = module.eks_blueprints.eks_cluster_id

  # EKS Managed Add-ons
  enable_amazon_eks_vpc_cni    = true
  enable_amazon_eks_coredns    = true
  enable_amazon_eks_kube_proxy = true

  # Add-ons
  enable_aws_load_balancer_controller = true
  enable_metrics_server               = true
  enable_cluster_autoscaler           = false 
  enable_karpenter                    = true 
  enable_aws_cloudwatch_metrics       = true # Enable CloudWatch metrics
  enable_aws_for_fluentbit            = true # Enable fluent-bit
  
  tags = local.tags

  depends_on = [module.eks_blueprints.managed_node_groups]
}
```

### Run Terraform PLAN

Verify the resources created by this execution

```bash
cd eks/terraform/
terraform plan
```

### Finally, Terraform APPLY

to create resources

```bash
terraform apply --auto-approve
```

Doing that will create the `amazon-cloudwatch` namespace in our cluster with the Container Insights resources and the `logging` namespace with the fluent-bit resources.

## Verify CloudWatch Container Insights is working

To verify that data is being collected in CloudWatch, launch the CloudWatch Containers UI in your browser using the link generated by the command below:

```bash
export AWS_REGION='us-east-1'

echo "
Use the URL below to access Cloudwatch Container Insights in $AWS_REGION:

https://${AWS_REGION}.console.aws.amazon.com/cloudwatch/home?region=${AWS_REGION}#container-insights:infrastructure
"
```

<p align="center"> 
<img src="static/container-insights.png">
</p>

From here you can see the metrics are being collected and presented to `CloudWatch`. You can switch between various drop downs to see EKS Services, EKS Cluster and more.

## Checking application logs

To check the application logs using Container Insights you just need to select the application you want to inspect in the `List View`:

<p align="center"> 
<img src="static/karpenter-container-insights.png">
</p>

In this case we have selected Karpenter, after click on Karpenter a CloudWatch dashboard will be presented, with all metrics collected by the CW agent in our cluster:

<p align="center"> 
<img src="static/karpenter-metrics.png">
</p>

To check the logs scroll down the page and under `Pod performance`, select the `karpenter-xxx` pod:

<p align="center"> 
<img src="static/karpenter-pod.png">
</p>

Then click on `Actions` > `View application logs` and click on `Run query`:

<p align="center"> 
<img src="static/karpenter-logs.png">
</p>

Done, now we are able to see the logs generated by Karpenter application pod.

# CI/CD with GitOps

GitOps, a term coined by [Weaveworks](https://www.weave.works/), is a way to do [continuous delivery](https://aws.amazon.com/devops/continuous-delivery/). Git is used as single source of truth for deploying into your cluster. **This is easy for a development team as they are already familiar with git and do not need to know other tools.**

In this module of the workshop we are going to setup [Flux CD](https://fluxcd.io) in our EKS cluster and deploy a sample application using Flux.

1. Fork this GitHub repository to your GitHub Account. If you don't have a GitHub account create one using [this link](https://github.com/).

<p align="center"> 
<img src="static/fork-github.png">
</p>

2. Export you GitHub personal access token and username:

```
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

> **_NOTE:_** To create you GitHub personal access token, follow the instructions of [this link](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

3. Install Flux CLI:

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

4. Bootstrap, using the `flux bootstrap` command you can install Flux on a Kubernetes cluster and configure it to manage itself from a Git repository.

```bash
flux bootstrap github --owner=${GITHUB_USER} --repository=
latam-containers-roadshow
 --path=./eks/fluxcd-examples/clusters/my-cluster/ --read-write-key --branch=main --namespace=flux-system --components-extra=image-reflector-controller,image-automation-controller
```

The above command will generate an SSH key (defaults to RSA 2048 but can be changed with --ssh-key-algorithm), and it will prompt you to add the SSH public key as a deploy key to your repository.

