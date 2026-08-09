# Amazon EKS - Managed Kubernetes

## Intro

So far you've built a Kubernetes cluster **by hand** with `kubeadm` - provisioning EC2 instances, installing `cri-o`/`kubelet`/`kubeadm`, running `kubeadm init`/`join`, and installing a CNI plugin yourself.

**Amazon EKS (Elastic Kubernetes Service)** is a *managed* Kubernetes offering. AWS runs and operates the control plane for you, across multiple AZs.

> [!IMPORTANT]
> EKS is **not free** - the control plane alone costs money per hour, on top of the EC2 worker nodes and the Load Balancer you'll create.
> **Delete everything at the end of this tutorial**. Instructions are at the bottom - don't skip them.


## Create the IAM roles

EKS needs two IAM roles: one for the **control plane** itself, and one for the **worker nodes**.

<details>
<summary>Step-by-step: creating the cluster IAM role</summary>

1. Open the [IAM console](https://console.aws.amazon.com/iam/) → **Roles** → **Create role**.
2. **Trusted entity type**: `AWS service`. **Use case**: search for `EKS` and choose **EKS - Cluster**.
3. Click **Next** - the console automatically pre-attaches the `AmazonEKSClusterPolicy` permission policy.
4. **Role name**: `eksClusterRole`. Click **Create role**.

</details>

<details>
<summary>Step-by-step: creating the node IAM role</summary>

1. IAM console → **Roles** → **Create role**.
2. **Trusted entity type**: `AWS service`. **Use case**: `EC2`.
3. Attach these permission policies: `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy`.
4. **Role name**: `eksNodeRole`. Click **Create role**.

</details>

## Create the cluster

1. Open the [EKS console](https://console.aws.amazon.com/eks/) → **Clusters** → **Create cluster**.
2. **Name**: `<your-name>-eks`. **Kubernetes version**: leave the default.
3. **Cluster service role**: select `eksClusterRole`.
4. Under **Networking**: select your **VPC** and at least **2 subnets in different AZs**. Leave **Cluster endpoint access** as `Public`.
5. Skip the **Add-ons** step for now (we'll add the EBS CSI driver ourselves later).
6. Review and click **Create**.

This takes **10-15 minutes** - the cluster status moves from `Creating` to `Active`. EKS is provisioning a highly available control plane behind the scenes. Grab a coffee.

Notice there's no control plane node to SSH into, no `etcd` Pod, no `kube-apiserver` Pod to find with `kubectl get pods -n kube-system` - **AWS runs the control plane for you**, outside of your account's visible EC2 resources.

### Connect `kubectl` to the cluster

The console doesn't offer a downloadable kubeconfig file, so a single one-off command is required to fetch the cluster's connection details (endpoint + certificate) into `~/.kube/config`. This command **does not create or modify any AWS resource**, it only reads data already produced by the console:

```bash
aws eks update-kubeconfig --name <your-name>-eks --region us-east-1
```

```bash
kubectl get svc
kubectl get nodes   # empty - no worker nodes yet
```

## Node group

A **node group** is a set of EC2 instances that AWS provisions, joins to the cluster, and keeps healthy, based on an Auto Scaling Group under the hood - same idea as the ASG you designed manually in the Terraform task, but managed by EKS.

1. On the cluster page, go to the **Compute** tab → **Add node group**.
2. **Name**: `workers`. **Node IAM role**: select `eksNodeRole`.
3. **Instance type**: `t3.medium`. **Disk size**: leave the default (`20 GiB`).
4. **Scaling configuration**: Desired size `2`, Minimum size `1`, Maximum size `3`.
5. **Subnets**: select the same subnets used by the cluster.
6. Review and click **Create**. Wait for the node group status to become `Active` (a few minutes).

Verify the nodes joined the cluster:

```bash
kubectl get nodes
```

Notice the nodes are `Ready` immediately - no `kubeadm join`, no manual CNI installation. EKS pre-configures the [Amazon VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s) add-on for you (assigning each Pod a real VPC IP address, instead of an overlay network like Calico).

## Install the EBS CSI driver

Just like your manual cluster, EKS doesn't provision the EBS CSI driver by default - but since it's such a common need, EKS offers it as a managed **add-on**, and it can be authorized entirely from the console using **[EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)** - no manual OIDC provider setup required.

1. Cluster page → **Add-ons** tab → **Get more add-ons** → select **Amazon EKS Pod Identity Agent** → **Next** → **Create**. This installs the small in-cluster agent that hands out AWS credentials to Pods.
2. IAM console → **Roles** → **Create role**. **Trusted entity type**: `AWS service`. **Use case**: search for `EKS` and choose **EKS - Pod Identity**. Attach the `AmazonEBSCSIDriverPolicy` permission policy. **Role name**: `AmazonEKS_EBS_CSI_DriverRole`. Click **Create role**.
3. Cluster page → **Add-ons** tab → **Get more add-ons** → select **Amazon EBS CSI Driver** → **Next**.
4. Under **Pod Identity Association**, select service account `ebs-csi-controller-sa` (namespace `kube-system`) and IAM role `AmazonEKS_EBS_CSI_DriverRole`.
5. Click **Next** → **Create**.

Create the same `StorageClass` you used before:

```yaml
# k8s/ebs-storage-class.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
```

```bash
kubectl apply -f k8s/ebs-storage-class.yaml
```

## Deploy an app - the Yolo service

Deploy the familiar Yolo Deployment and Service:

```bash
kubectl apply -f k8s/deployment-demo.yaml
kubectl apply -f k8s/service-demo.yaml
kubectl get pods
```

Nothing new here - Deployments, Pods and Services behave **identically** to your manual cluster. That's the whole point of Kubernetes as a portable API.

## Deploy the Nginx Ingress Controller - watch the LB appear automatically

This is where EKS really shines. Install the Ingress controller using its AWS-specific manifest (it creates a Service of type `LoadBalancer`, not `NodePort` like you used before):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/aws/deploy.yaml
```

Now watch:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller --watch
```

Within a minute or two, the `EXTERNAL-IP` column fills in with a real AWS **Classic/Network Load Balancer** hostname:

```console
NAME                       TYPE           EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   a1b2c3d4e5f6.elb.us-east-1.amazonaws.com
```

Recall the manual cluster: you had to create the Target Group and ALB **yourself**, register instances, configure health checks... Here, applying a Service of type `LoadBalancer` was enough - the [AWS Cloud Controller Manager](https://github.com/kubernetes/cloud-provider-aws), built into EKS, watched for the Service and provisioned the Load Balancer for you.

Create an `Ingress` for the Yolo service, exactly as before:

```yaml
# k8s/ingress-demo.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yolo
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: yolo-svc
            port:
              number: 8080
```

```bash
kubectl apply -f k8s/ingress-demo.yaml
curl http://<the-elb-hostname>/
```

## Deploy a monitoring stack - kube-prometheus-stack

Same Helm chart you already know:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

```bash
kubectl get pods -n monitoring
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open `http://localhost:3000` (user `admin`) and browse the same dashboards as before. Helm charts don't care whether the cluster is `kubeadm` or EKS - it's the same Kubernetes API underneath.

## Clean up - delete everything

Please delete the cluster and all its resources when you're done, to avoid ongoing charges!