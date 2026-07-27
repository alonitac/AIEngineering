# Kubernetes Networking

The Kubernetes network model is built out of several pieces:

1. Communication achieved by 2 layers of networking: **Pod network**, **Node network**.
   - The Pod network is managed by the [CNI (Container Network Interface) plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/), which ensures connectivity between pods across nodes in the cluster.
     In out cluster, we've installed **Calico**. 
   - The Node network handles communication between the nodes themselves. In out cluster, we use AWS VPC. 

2. Each **pod** in a cluster gets its own unique **cluster-wide IP address**.
   - The pod network namespace is **shared by all of the containers within the pod**: containers share the same network interface, and communicate with each other by `localhost`. 
   - All pods can communicate with all other pods, whether they are on the same node or on different nodes. 
3. Pod-to-Service communications: this is covered by **Services**.
   - Services act as **stable endpoints** that abstract the dynamic IPs of pods.
   - When a pod communicates with a service, traffic is routed to one of the service's backend pods.
     `kube-proxy` manages this routing by maintaining IP tables or rules to forward traffic to the correct pod.


## Expose applications outside the cluster using a Service of type `NodePort`

So far, all our created **Service** objects in the cluster were of type `ClusterIP`, which is the default option in Kubernetes:

```console
$ kubectl describe svc <one-of-your-services>
Name:              some-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Type:              ClusterIP     <------ Service type
```

A `ClusterIP` service make the application accessible **only within the cluster**.


Kubernetes allows you to create a Service of type `NodePort`.
When you create a `NodePort` service, **every node in the cluster** configures itself to listen on that assigned port and to forward traffic to one of the Pods associated with that Service.

For example (**no need** to apply the below example in your cluster):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001  # Optional field, if not specified, the control plane will allocate a port from a range (default: 30000-32767)
```

Now every node in the cluster listens on port `30001`, so you'll be able to access the service from outside the cluster by:

```bash
http://<any-cluster-node-ip>:30001
```

![][k8s_networking_np_service]

## Integrate an external Load Balancer

All that remains is to add a mechanism for routing the traffic across the different cluster's nodes.

AWS Elastic Load balancer is perfectly fit here: 

![][k8s_networking_np_lb_service]


## Ingress and Ingress controller 

A Service of type `NodePort` (or `LoadBalancer`) is the core mechanism that allows you to expose application to clients outside the cluster. 

Should we create a dedicated Load Balancer (or multiple listeners in the same LB) for each service we want to expose outside the cluster?
Wouldn't that be overly complex and wasteful in terms of resources and cost?

It is. Let's introduce an **Ingress** and **Ingress Controller**.

Ingress Controller is an application that runs in the Kubernetes cluster, that manage external traffic to services within the cluster. 
Sound familiar? It's essentially a **web server**, like Nginx, but with a twist.

There are [many Ingress Controller implementations](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) for different usages and clusters. 

[Nginx ingress controller](https://github.com/kubernetes/ingress-nginx) is one of the popular used one. 
Essentially, it's the same old good Nginx webserver, exposed to be available outside the cluster, and configured to route incoming traffic to different Services in the cluster (a.k.a. reverse proxy). 

![][k8s_networking_nginx_ic]

#### Let's deploy an Nginx Ingress Controller

Apply the below manifest: 

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/baremetal/deploy.yaml
   ```

The above manifest creates the Nginx ingress controller along with a dedicated `NodePort` service, and other related resources. 

To discover the ports exposed by the Nginx Ingress Controller, run:
 

```console
$ kubectl describe service ingress-nginx-controller -n ingress-nginx

Name:                     ingress-nginx-controller
Namespace:                ingress-nginx
Labels:                   app.kubernetes.io/component=controller
                          app.kubernetes.io/instance=ingress-nginx
                          app.kubernetes.io/name=ingress-nginx
                          app.kubernetes.io/part-of=ingress-nginx
                          app.kubernetes.io/version=1.12.0
Annotations:              <none>
Selector:                 app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.103.240.159
IPs:                      10.103.240.159
Port:                     http  80/TCP
TargetPort:               http/TCP
NodePort:                 http  31867/TCP      <---- Node port for HTTP traffic
Endpoints:                192.168.51.196:80
Port:                     https  443/TCP
TargetPort:               https/TCP
NodePort:                 https  30683/TCP     <---- Node port for HTTPS traffic
Endpoints:                192.168.51.196:443
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

We now would like to create a Application Load Balancer that routes traffic across the Nginx Ingress Controller's NodePorts, so that we can access the applications running in the cluster from outside.


### Configure a target group

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

2. In the left navigation pane, under **Load Balancing**, choose **Target Groups**\.

3. Choose **Create target group**\.

4. In the **Basic configuration** section, set the following parameters:

    1. For **Choose a target type**, select **Instance** to specify targets by instance ID

    2. For **Target group name**, enter a name for the target group\.

    3. Set the **Port** and **Protocol** to the HTTP NodePort
       (e.g. `31867` in the above example output).

    4. For VPC, select your virtual private cloud \(VPC\)

    5. For **Protocol version**, select **HTTP1**.

5. In the **Health checks** section, modify the default settings as needed to perform a health checks to `/healthz`.

6. Choose **Next**\.
7. In the **Register targets** page, **no need** to add targets as instances will be registered automatically by Auto Scaling Group. 
8. Choose **Create target group**\.

### Configure a load balancer and a listener

1. In the navigation pane, under **Load Balancing**, choose **Load Balancers**\.

2. Choose **Create Load Balancer**\.

3. Under **Application Load Balancer**, choose **Create**\.

4. **Basic configuration**

    1. For **Load balancer name**, enter a name for your load balancer\.
    2. For **Scheme**, choose **Internet-facing**.
    3. For **IP address type**, choose **IPv4**.

5. **Network mapping**

    1. For **VPC**, select the VPC that you used for your EC2 instances\. As you selected **Internet\-facing** for **Scheme**, only VPCs with an internet gateway are available for selection\.

    1. For **Mappings**, select two or more Availability Zones and corresponding subnets\. Enabling multiple Availability Zones increases the fault tolerance of your applications\.

6. For **Security groups**, select an existing security group, or create a new one\. The rules in this Security Group would be applied to the Load Balancer itself (not to your instances). 

7. For Listeners and routing, the default listener accepts **HTTP** traffic on port **80**. 
9. Review your configuration, and choose **Create load balancer**\. 



Once the external load balancer routes the traffic to the Nginx Ingress Controller, 
we need to configure the Nginx to route the traffic to the different services of the cluster. 

Instead of configuring NGINX using a `.conf` files, as we've learned, we now use a dedicated Kubernetes object called `Ingress`,
which is a unified way to configure Ingress Controller in Kubernetes (no matter if it's Nginx, Traefik, or any other platform).

So no more working with Nginx configuration files to define routing rules. 
`Ingress` defines the **routing rules** for you in "Kubernetic" a way.

To route traffic to the **PolyAI frontend** service, apply the below `Ingress` (change values according to your configurations): 

```yaml
# k8s/ingress-demo.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
spec:
  ingressClassName: nginx
  rules:
  - host: # YOUR_ELB_or_ROUTE53_DOMAIN_HERE 
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: # YOUR_FRONTEND_SERVICE_HERE
            port:
              number: # YOUR_SERVICE_PORT_HERE
```

Your Nginx Ingress Controller is configured to **automatically** discover all `Ingress` objects where `ingressClassName: nginx` is present.

So all you need is to apply the manifest:

```bash 
kubectl apply -f k8s/ingress-demo.yaml
```

And try to visit the application using the ELB or Route53 domain name. 

> [!NOTE]
> #### The relation between **Ingress** and **Ingress Controller**:
> 
> **Ingress** only defines the *routing rules*, it is not responsible for the actual routing mechanism.  
> An Ingress controller is responsible for fulfilling the Ingress routing rules. 
> In order for the Ingress resource to work, the cluster must have an Ingress Controller running.


> [!NOTE]
> #### Service of type `LoadBalancer`
> 
> In addition to the `NodePort` service, Kubernetes allows you to create a Service of type `LoadBalancer` (**no need** to apply the below example):
> 
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: my-service
> spec:
>   type: LoadBalancer
>   selector:
>     app: MyApp
>   ports:
>     - protocol: TCP
>       port: 80
>       targetPort: 80
> ```
> 
> Applying this Service will **automatically** provision the Load Balancer (ELB), register your cluster nodes, and route the traffic according to your Service manifest.
> This Service takes effect only on cloud providers which support external load balancers (like AWS ELB).
> 
> Why can't we use a `LoadBalancer` service for our cluster out of the box?
> Even though our cluster is running in AWS, it consists of "raw" VMs with Kubernetes manually installed.
> As a result, the cluster isn't "aware" that it's running in AWS.
> To enable the use of a `LoadBalancer` service, the [AWS Cloud Controller Manager](https://github.com/kubernetes/cloud-provider-aws) should be installed.  





# Exercises 

### :pencil2: Canary using Nginx 

You can add [nginx annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) to specific `Ingress` objects to customize their behavior.

In some cases, you may want to "canary" a new set of changes by sending a small number of requests to a different service than the production service. 
The [canary annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary) enables the Ingress spec to act as an alternative service for requests to route to depending on the rules applied.

In this exercise we'll deploy a canary for the PolyAI frontend service. 

- Deploy the frontend service in a version which is not your most up-to-date (e.g. `0.8.0` instead of `0.9.0`). 
- Now you want to deploy the newer app version (e.g. `0.9.0`) but you don't confident with this deployment.
  Create other (separated) YAML manifests for the new version of the service, call then `frontend-canary`. 
- Create another `Ingress` pointing to your canary Deployment, as follows: 

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"
spec:
  ingressClassName: nginx
  rules:
     # TODO ... Make sure the `host` entry is the same as the existed frontend Ingress. 
```

This Ingress routes 5% of the traffic to the canary deployment. 

Test your configurations by periodically access the application:

```bash
/bin/sh -c "while sleep 0.05; do (wget -q -O- http://LOAD_BALANCER_DOMAIN &); done"
```

**Bonus**: Use [different annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) to perform a canary deployment which routes users based on a request header `FOO=bar`, instead of specific percentage.


[k8s_networking_lb_service]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_lb_service.png
[k8s_networking_nginx_ic]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_nginx_ic.png
[k8s_networking_np_service]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_np_service.png
[k8s_networking_np_lb_service]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_np_lb_service.png

[^1]: Created in the [previous tutorial](aws_elb_asg.md). 