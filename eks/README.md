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
  enable_aws_cloudwatch_metrics       = false

  tags = local.tags

  depends_on = [module.eks_blueprints.managed_node_groups]
}
```





