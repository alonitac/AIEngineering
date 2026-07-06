# Intro to Kubernetes 


In this section you'll provision a Kubernetes cluster consists by **one control plane node** and **one worker node**.
Although AWS offers EKS, a managed service that simplifies cluster setup and management, we will manually set up the cluster using `kubeadm`.
This hands-on approach will allow you to gain a deeper understanding of Kubernetes architecture and the processes involved in cluster creation and configuration. 

The underlying nodes infrastructure are based on **Ubuntu EC2 instances**. Let's get started. 

## Kubernetes Architecture 

A Kubernetes cluster consists of a set of worker machines, called **nodes**, that run containerized applications (known as **Pods**).
Every cluster has at least one **worker node** and one **control plane node**.

The control plane manages the worker nodes and the Pods in the cluster.
In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability.

![][k8s_components]

#### Control Plane main components

The control plane's components make global decisions about the cluster (for example, scheduling), as well as detecting and responding to cluster events (for example, starting up a new pod when a deployment's replicas field is unsatisfied).

- **kube-apiserver**: The API server is the front end for the Kubernetes control plane.
- **etcd**: Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data.
- **kube-scheduler**: Watches for newly created Pods with no assigned node, and selects a node for them to run on.
- **kube-controller-manager**: Runs [controllers](https://kubernetes.io/docs/concepts/overview/components/#kube-controller-manager). There are many different types of controllers. Some examples of them are:
  - Responsible for noticing and responding when nodes go down.
  - Responsible for noticing and responding when a Deployment is not in its desired state.

#### Node components

Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment.

- **kubelet**: An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod.
- **kube-proxy**: kube-proxy is a network proxy that runs on each node in your cluster. It allows network communication to your Pods from network sessions inside or outside your cluster.
- **Container runtime**: It is responsible for managing the execution and lifecycle of containers ([containerd](https://containerd.io/) or [CRI-O](https://cri-o.io/)).

## Provision 2 nodes Kubernetes cluster in AWS using kubeadm

### Prepare infrastructure 



2. Launch two `t2` or `t3.medium` Ubuntu instances with `30GB` disk, naming them `<your-name>-control-plane` and `<your-name>-worker`. Make sure to attach the `kubeadm-cluster-node-role` role to both instances, and the `kubeadm-cluster-node-sg` security group.

   <details>
   <summary>Step-by-step: launching an EC2 instance</summary>

   Repeat these steps twice - once for the control plane node, once for the worker node.

   1. Open the Amazon EC2 console at [https://console.aws.amazon.com/ec2/](https://console.aws.amazon.com/ec2/).
   2. Choose **Launch instance**.
   3. Under **Name and tags**, enter `<your-name>-control-plane` (or `<your-name>-worker` for the second instance).
   4. Under **Application and OS Images**, choose **Quick Start** → **Ubuntu**.
   5. Under **Instance type**, select `t3.medium` (or `t2.medium` if unavailable in your region).
   6. Under **Key pair (login)**, select your existing `.pem` key pair from the dropdown.
   7. Under **Network settings**, click **Edit**:
      - **VPC**: choose the default VPC.
      - **Subnet**: choose any available subnet.
      - **Firewall (security groups)**: choose **Select existing security group** and select `kubeadm-cluster-node-sg` (this is a dedicated security group for the Kubernetes cluster nodes we created earlier).
   8. Under **Advanced details** → **IAM instance profile**, select `kubeadm-cluster-node-role` (this is a dedicated IAM role for the Kubernetes cluster nodes we created earlier. It allows the cluster nodes to interact with AWS services securely).
   9. Under **Configure storage**, set the root volume to **30 GiB**.
   10. Review the **Summary** panel and click **Launch instance**.

   Once both instances are running, connect to each one:

   ```bash
   ssh -i "</path/to/key.pem>" ubuntu@<instance-public-ip>
   ```

   </details>
3. SSH into each instance and execute the below script **in both machines** as root. 

First switch to root user by 

```bash
sudo su -
```

Then

```bash
set -euo pipefail

kubernetes_version=v1.35

# Log all output
exec > >(tee /var/log/user-data.log)
exec 2>&1

# Update system
apt-get update
apt-get install jq unzip ebtables ethtool -y

# Install AWS CLI
echo "Installing AWS CLI..."
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
rm -rf awscliv2.zip aws/


# Enable IPv4 packet forwarding. sysctl params required by setup, params persist across reboots
echo "Configuring kernel parameters..."
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sysctl --system

# Install cri-o kubelet kubeadm kubectl
echo "Installing Kubernetes ${kubernetes_version} components..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/${kubernetes_version}/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${kubernetes_version}/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list

apt-get update
apt-get install -y software-properties-common apt-transport-https ca-certificates curl gpg
apt-get install -y cri-o kubelet kubeadm kubectl amazon-ecr-credential-helper
apt-mark hold kubelet kubeadm kubectl amazon-ecr-credential-helper

# Start the CRIO container runtime and kubelet
echo "Starting container runtime and kubelet..."
systemctl start crio.service
systemctl enable --now crio.service
systemctl enable --now kubelet

# Disable swap memory
echo "Disabling swap..."
swapoff -a

# Add the command to crontab to make it persistent across reboots
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab -

# Create a marker file to indicate setup completion
echo "Setup completed at $(date)" | tee /var/log/k8s-setup-complete
```


The script essentially installs: 

- `kubeadm` - the [official Kubernetes tool](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) that initializes the cluster and make it up and running.
- `kubelet` - a Linux service that runs on each node in the cluster, responsible for Pods lifecycle.
- `cri-o` as the container runtime.
- `kubectl` - to control the cluster.

> [!WARNING]
> In this cluster we don't use Docker as the **container runtime** (i.e. the engine that responsible to run your containers).
> We do this because Docker itself is not implementing the [Container Runtime Interface (CRI)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md) required by Kubernetes. Using Docker would require an additional adapter to be installed. 
> 
> To avoid this headache, we use [CRI-O](https://github.com/cri-o/cri-o), which is another great container runtime.
> 
> This should not affect the operation of your Kubernetes cluster in any way.
> Anyway, do not install Docker, as it may cause conflicts with the setup.


### Initialize the cluster 

1. From the **control-plane node**, initialize the cluster by:

```bash
sudo kubeadm init --cri-socket unix:///var/run/crio/crio.sock
```


2. **Carefully** read the output to understand how to start using your cluster, and how to join the worker node to be part of the cluster. 


   The `kubeadm init` command essentially does the below:
   
   1. Runs pre-flight checks.
   2. Creates certificates that used by different components for secure communication.
   3. Generates `kubeconfig` files for cluster administration.
   4. Deploys the `etcd` db as a Pod.
   5. Deploys the control plane components as Pods (`apiserver`, `controller-manager`, `scheduler`).
   6. Starts the `kubelet` as a Linux service.
   7. Install addons in the cluster (`coredns` and `kube-proxy`).

   ![][k8s_architecture_kubeadm]

   Make sure you understand the [role of each component in the architecture](https://kubernetes.io/docs/concepts/architecture/).


3. `kubectl` is the command-line tool for controlling Kubernetes clusters. To start using it:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```




> [!NOTE]
> - When running the `kubeadm join` command on the worker node, you may need to append the `--cri-socket` flag, just as you did with `kubeadm init`:
>   ```bash
>   sudo <join command> --cri-socket unix:///var/run/crio/crio.sock
>   ```
> - To run `kubeadm init` again, you must first [tear down the cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tear-down).
> - The join token is valid for 24 hours. Read here [how to generate a new one](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes) if needed.
> - For more information about initializing a cluster using `kubeadm`, [read the official docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).


3. Make sure you have two (not yer ready) nodes cluster by executing from the control-plane machine:

```bash
kubectl get nodes
```

### Install Pod networking plugin

When deploying a Kubernetes cluster, there are 2 layers of networking communication:

![][k8s_cni]

- Communication between Nodes (denoted by the green line). This is managed for us by the AWS VPC.
- Communication between Pods (denoted by the purple line). Although the communication is done on top of the VPC, communication between pods using their own cluster-internal IP address is **not** implemented for us by AWS.   

You must deploy a [Container Network Interface (CNI)](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) network add-on so that your Pods can communicate with each other. 
There are many [addons that implement the CNI](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy). 

We'll install [Calico](https://docs.tigera.io/calico/latest/about/), simply by:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico.yaml
```

Make sure you have nodes are ready:

```bash
kubectl get nodes
```


## Deploy application in the cluster

Let's see Kubernetes cluster in all his glory! 

**Online Boutique** is a microservices demo application, consists of an 11-tier microservices.
The application is a web-based e-commerce app where users can browse items, add them to the cart, and purchase them.

Here is the app architecture and description of each microservice:

![k8s_online-boutique-arch][k8s_online-boutique-arch]


| Service                                              | Language      | Description                                                                                                                       |
| ---------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| frontend                           | Go            | Exposes an HTTP server to serve the website. Does not require signup/login and generates session IDs for all users automatically. |
| cartservice                     | C#            | Stores the items in the user's shopping cart in Redis and retrieves it.                                                           |
| productcatalogservice | Go            | Provides the list of products from a JSON file and ability to search products and get individual products.                        |
| currencyservice             | Node.js       | Converts one money amount to another currency. Uses real values fetched from European Central Bank. It's the highest QPS service. |
| paymentservice               | Node.js       | Charges the given credit card info (mock) with the given amount and returns a transaction ID.                                     |
| shippingservice             | Go            | Gives shipping cost estimates based on the shopping cart. Ships items to the given address (mock)                                 |
| emailservice                   | Python        | Sends users an order confirmation email (mock).                                                                                   |
| checkoutservice             | Go            | Retrieves user cart, prepares order and orchestrates the payment, shipping and the email notification.                            |
| recommendationservice | Python        | Recommends other products based on what's given in the cart.                                                                      |
| adservice                         | Java          | Provides text ads based on given context words.                                                                                   |
| loadgenerator                 | Python/Locust | Continuously sends requests imitating realistic user shopping flows to the frontend.                                              |


To deploy the app in you cluster, perform the below command: 

```bash 
kubectl apply -f https://raw.githubusercontent.com/alonitac/MicroservicesBGU26/main/k8s/online-boutique.yaml
```

By default, **applications running within the cluster are not accessible from outside the cluster.**
There are various techniques available to enable external access, we will cover some of them later on.

Using **port forwarding** allows developers to establish a temporary tunnel for debugging purposes and access applications running inside the cluster from their local machines.

```bash
kubectl port-forward svc/frontend 8080:80 --address 0.0.0.0
```

Visit the service in `http://<control-plane-public-ip>:8080`

To delete the app from the cluster:

```bash
kubectl delete -f https://raw.githubusercontent.com/alonitac/MicroservicesBGU26/main/k8s/online-boutique.yaml
```

## Pods and namespaces


Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.
A Pod is a group of **one or more containers**, with **shared storage** and **network resources**, and a specification for how to run the containers.

You can list the Online Boutique pods by:

```bash
kubectl get pods
```

Make sure all pods are `Running`. 

Pods and other resources are aggrandized in **namespaces**.
Namespace isolates resources within a cluster, usually for better organization and access control.

If you don't specify namespace, the `default` namespace is used. To list pods from all namespaces:

```bash
kubectl get pods -A
```

# Exercises 

> [!TIP]
> ### `kubectl` quick reference
> 
> | Description                                | Examples                                                |
> |--------------------------------------------|---------------------------------------------------------|
> | List cluster objects - basic information   | `kubectl get pods`, `kubectl get nodes`                 |
> | List cluster objects - from all namespaces | `kubectl get pods -A`.                                  |
> | List cluster objects - certain namespace   | `kubectl get pods -n kube-system`.                      |
> | List cluster objects - wider information   | `kubectl get pods -o wide`, `kubectl get nodes -o wide` |
> | Get full description of an object          | `kubectl describe pod POD_NAME`                         |
> | Apply a YAML manifest                      | `kubectl apply -f my-manifest.yaml`.                    |
> | Apply all manifests in a given dir         | `kubectl apply -f dir/`.                                |
> | Delete an applied manifest                 | `kubectl delete -f my-manifest.yaml`                    |
> | Watch pod logs                             | `kubectl logs my-pod`                                   |


### :pencil2: Using `kubectl`

Use `kubectl` to answer the following questions: 

1. How many pod replicas does the **frontend** microservice have? 
2. How many **containers** does a **frontend** pod have? 
3. Use the `kubectl describe` command to get the IP address of the **frontend** pod. 
4. For the single **frontend** running pod, how many environment variables does a container named `server` have? 
5. For the single **frontend** running pod, what is the Docker image the container named `server` based on? 
6. What is the node name that the **emailservice** pod was scheduled on (by the k8s scheduler)?
7. What is the port do the **checkoutservice** pods listend on? 



[k8s_architecture_kubeadm]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_architecture_kubeadm.png
[k8s_cni]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_cni.png



[k8s_components]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_components.png
[k8s_online-boutique-arch]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_online-boutique-arch.png