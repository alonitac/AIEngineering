# Kubernetes Cluster Provisioning as Code

## Overview

In this task, you'll provision a Kubernetes cluster on AWS using Terraform, and deploy the PolyAI stack with ArgoCD.


> [!NOTE]
> Provision only **one cluster for both `dev` and `prod`** (use namespaces to separate them).
> And still, you should follow the regular git workflow: feature branch → `dev` → PR to `main`.

## Part I: Provision the Infrastructure

All Terraform files go under `infra/tf/` in your repo:

```text
infra/tf/
├── tfvars/
│   └── us-east-1.tfvars
├── modules/
│   └── k8s-cluster/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── main.tf
├── outputs.tf
└── variables.tf
```

Create a `modules/k8s-cluster` local Terraform module. Inside it, provision:

### VPC

A VPC with **2 public subnets in different Availability Zones**. All instances go into these subnets. For that, use [the VPC module taught in class ](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest).

### Control plane

- An EC2 instance (`t3.medium`, Ubuntu, 20 GiB) for the control plane.
- An IAM role with `AmazonEKSClusterPolicy`, `AmazonEBSCSIDriverPolicy`, and `AmazonEC2ContainerRegistryReadOnly`.
- A security group allowing SSH (22), and all intra-VPC traffic.


The control plane must be initialized automatically when the instance first boots.

To do so, you need to install the necessary dependencies (`cri-o`, `kubelet`, `kubeadm`, `kubectl`) and execute `kubeadm init` automatically.
You can use a [user data script](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) and [EC2 AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html) with pre-installed dependencies.


### Worker nodes

Workers are managed by an [Auto Scaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling.html). It means that you don't create worker instances directly, but instead you just specify the desired number of workers, and the ASG creates and terminates instances automatically.

Read about [Launch Templates](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-templates.html) and [Auto Scaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling.html) in the AWS docs.


For the ASG, use `min_size = 1`, `max_size = 3`.
As for `desired_capacity`, you can set it to `> 0` when working on the cluster, and `0` when you finish, to avoid unnecessary costs. 

Keep in mind that new worker instances that the ASG launches, must join the cluster ( `kubeadm join`) with a valid token. Design a solution so worker nodes can join command the cluster at boot. Relevant services to explore:

- [ASG lifecycle hooks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-lifecycle.html)
- [SSM Parameter Store / Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/documents.html)
- [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)

Any working approach is acceptable - leave a short comment in the code explaining your choice.

> [!NOTE]
> When you manually scale down the ASG, terminated instances are removed from AWS but their corresponding Node objects remain in Kubernetes with a `NotReady` status. Try to find a solution that automatically removes the Node from the cluster on scale-down (e.g. using an ASG lifecycle hook) - **only implement it if you can fully understand and explain how it works**. If not, it's acceptable at that point to manually run `kubectl delete node <node-name>` when scaling down instances.


### Terraform Workspaces

The same configuration must work in any region by switching the workspace and `.tfvars` file. **You don't need to provision a second region!!!**, just design your Terraform code so it works in any region.

## Part II: Bootstrap the Cluster

After `terraform apply`, the control plane is already initialized. Run these remaining steps manually (you will automate them in Part III).

1. Install **Calico** as the CNI plugin.
2. Install **ArgoCD** and retrieve the initial admin password.
3. Create an ArgoCD `Application` for each microservice. ArgoCD will auto sync `dev` on every push to `dev`, and manual sync `prod` on every push to `main`.


## Part III: CI/CD Pipelines

Create two GitHub Actions workflows:

`.github/workflows/cluster.yaml` - triggered **manually** (use `workflow_dispatch` in the workflow YAML, with a `region` input). Two sequential jobs:
1. **Provision** - runs `terraform apply` for the selected region.
2. **Bootstrap** - SSHs into the control plane and **idempotently** installs Calico, ArgoCD, and the ArgoCD `Application` per microservice. Use `terraform output` to get the control plane IP.

`.github/workflows/cd.yaml` - triggered on push to `dev` and `main`. Updates the image tag for the changed service and commits the change back to the repo so ArgoCD picks it up and deploys the updated service.

> [!NOTE]
> After ArgoCD is up and the `Application` is created, ArgoCD handles all workload deployments. The bootstrap job runs once per cluster lifetime.

> [!NOTE]
> The best way to test your work is to `terraform destroy`, then re-run the `cluster.yaml` workflow to recreate everything from scratch. If that succeeds, your work is complete.