# Kubernetes core workloads

## Workloads

A **Workload** is an application running on Kubernetes.
Whether your workload is a single Docker container, or several containers that work together (e.g. HA mongo cluster), on Kubernetes you run it inside a set of **Pods**.

In this tutorial we will cover the following workloads:

- Pod
- ReplicaSet
- Deployment

> [!NOTE]
> Kubernetes has an [amazing docs](https://kubernetes.io/docs/concepts/workloads/pods/), use it. 

### Pods

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.
A Pod is a group of one or more containers (usually one), with **shared storage** and **network resources**, and a specification for how to run the containers.

The following is an example of a Pod which consists of a container running the [yolo-service](https://hub.docker.com/r/alonithuji/yolo-service) image.

```yaml
# k8s/pod-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: yolo-service
  labels:
    app: yolo-service
    env: prod
spec:
  containers:
  - name: server
    image: docker.io/alonithuji/yolo-service:0.0.1
    ports:
    - containerPort: 8080
```

To create the Pod shown above, run the following command (the above content can be found in `k8s/pod-demo.yaml`):

> [!TIP]
> Clone this repository to your control plane machine so you can apply all YAML manifests directly:
> ```bash
> git clone https://github.com/alonitac/MicroservicesBGU26.git
> cd MicroservicesBGU26
> ```
> Then use `kubectl apply -f k8s/pod-demo.yaml` from within the cloned directory.

```bash
kubectl apply -f k8s/pod-demo.yaml
```

When a Pod gets created, the new Pod is scheduled to run on a Node in your cluster.
The Pod remains on that node until somthing happens - the Pod finishes execution, the Pod is deleted, the Pod is evicted for lack of resources, or the node fails.

The `labels` attached to the pod can be used to list or describe it.

**Labels** are key/value pairs that are attached to objects such as Pods.
Labels are intended to be used to specify identifying attributes of objects that are meaningful and relevant to users, as well as organizing a subsets of objects.
[Read here](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) about labeling best practice.

```console
$ kubectl get pods -l env=prod
NAME                    READY   STATUS    RESTARTS   AGE
yolo-service            1/1     Running   0          14s

$ kubectl describe pod yolo-service
Name:             yolo-service
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Sun, 24 Sep 2023 16:34:48 +0000
Labels:           app=yolo-service
                  env=prod
Annotations:      <none>
Status:           Running
IP:               10.244.0.87
IPs:
  IP:  10.244.0.87
Containers:
  server:
    Container ID:   docker://b93064757aa593c7be63edbb4796cdb468ccf1f5d527f0872e7adc7692127f7c
    Image:          alonithuji/yolo-service:0.0.1
    Image ID:       docker-pullable://alonithuji/yolo-service:0.0.1@sha256:53d0416f79e3d4ba8d2092d7c48880375e3398f6e086996aa12b2e68d0a04976
    Port:           8080/TCP
    Host Port:      <none>
    State:          Running
      Started:      Sun, 24 Sep 2023 16:34:49 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lw7mq (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-lw7mq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  21m   default-scheduler  Successfully assigned default/yolo-service to minikube
  Normal  Pulled     21m   kubelet            Container image "alonithuji/yolo-service:0.0.1" already present on machine
  Normal  Created    21m   kubelet            Created container server
  Normal  Started    21m   kubelet            Started container server
```

In the above output we can see a detailed description of the pod.
Reviewing all the presented information can be overwhelmed, but here are a few important notes:

- If you don't specify in which **namespace** to create the Pod object, the `default` namespace is used.
- Each Pod is assigned a unique IP address.
- Pod **events** can help you debug your pod state. 

### Workload resources 

Usually you don't need to create Pods directly. 
Each Pod is meant to run a single instance of a given application. 
If you want to scale your application horizontally, you should replicate the Pod in different nodes. Replicated Pods are usually created and managed as a group by a **Workload Resource**.

Workload resources create and manage multiple Pods for you, with many added benefits.
They handle replication and rollout and automatic healing in case of Pod failure.
For example, if a Node fails, a workload resource notices that Pods on that Node have stopped working and creates a replacement Pod. 

Available workload resources are:

- **Deployment** (and, indirectly, **ReplicaSet**), the most common way to run an stateless applications on your cluster.
- A **StatefulSet** lets you run Pods connected to a persistent storage.
- A **DaemonSet** lets you run single Pod on each Node of your cluster.
- You can use a **Job** (or a **CronJob**) to define tasks that (periodically) run to completion and then stop.

**Note:** Workload resources don't run containers directly, the only object in Kubernetes that does so is the Pod.
All other workload resources **only manage Pods**. For that reason, workload resources are also known as **Controllers**.
In Kubernetes, controllers are control loops that watch the state of your Pods, then make or request changes where needed. Each controller tries to move the current state closer to the desired state.

In this tutorial we will focus on **Deployment** and **ReplicaSet**. 

### ReplicaSet

A **ReplicaSet**'s purpose is to maintain a stable set of replica Pods running at any given time.
As such, it is often used to guarantee the availability of a specified number of identical Pods.

Here is an example of ReplicaSet:

```yaml 
# k8s/replicaset-demo.yaml 

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: yolo-rs
  labels:
    app: yolo-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: yolo-service
  template:
    metadata:
      labels:
        app: yolo-service
    spec:
      containers:
      - name: server
        image: docker.io/alonithuji/yolo-service:0.0.1
        ports:
        - containerPort: 8080
```

In this example:

- The `.spec.template` field contains a [PodTemplates](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates). PodTemplates are specifications for creating Pods. As you can see it's very similar to `Pod` YAML described in the previous section.
  We can also notice that pods created from this template hold a label with key `app` and value `yolo-service`.
- The ReplicaSet creates three replicated Pods, indicated by the `.spec.replicas` field.
- The `.spec.selector.matchLabels` field defines how the created ReplicaSet finds which Pods to manage (remember that ReplicaSet is a controller that manage Pods). In this case, the ReplicaSet will manage all Pods that match the label `app: yolo-service`, corresponding to the label we gave in the Pod template. 

Let's apply this ReplicaSet in the cluster.

```bash 
kubectl apply -f k8s/replicaset-demo.yaml
```

You can then get the current ReplicaSets deployed:

```bash 
kubectl get rs
```

You can also check for the Pods brought up as part of the ReplicaSet:

```bash 
kubectl get pods -l app=yolo-service
```

Let's play with your ReplicaSet:

- Delete one of the pods owned by the ReplicaSet. What happened? 
- A ReplicaSet can be easily scaled up or down by simply updating the `.spec.replicas` field. Try it out...
- Create a Pod (separately to the ReplicaSet) in the cluster, with the same label key and value as you gave in the PodTemplate. What happened? Why? 
- Try to update the image version in the YAML manifest (e.g. to `alonithuji/yolo-service:0.0.2`), and perform `kubectl apply` with the updated YAML file. What happened? Why? 

Delete your ReplicaSet before moving on to the next section:

```bash
kubectl delete replicaset yolo-rs
```

### Deployment

We started our tutorial with a bare Pod, then moved to work with a ReplicaSet.
However, this is not good enough. For example, as you've probability noticed, Replicaset does not automatically update containers when image version is updates. 

A **Deployment** is a higher-level concept that manages ReplicaSets and provides declarative updates to Pods along with a lot of other useful features.
In most cases, **ReplicaSet is not being used directly**. Kubernetes' docs recommending using Deployments instead of directly using ReplicaSets.

The following is an example of a Deployment:

```yaml
# k8s/deployment-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: yolo-service
  labels:
    app: yolo-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: yolo-service
  template:
    metadata:
      labels:
        app: yolo-service
    spec:
      containers:
      - name: server
        image: docker.io/alonithuji/yolo-service:0.0.1
        ports:
        - containerPort: 8080
```

Apply it to your cluster, to see the Deployment rollout status, run `kubectl rollout status deployment/yolo-service`.

Under the hood, the Deployment object created a ReplicaSet (`rs`) for you:

```console
$ kubectl get rs
NAME                                      DESIRED   CURRENT   READY   AGE
yolo-service-75675f5897                   3         3         3       18s
```

#### Updating a Deployment 

A Deployment's rollout is triggered if and only if the Deployment's Pod template (that is, `.spec.template`) is changed.

Let's update the `yolo-service` Pods to use the `alonithuji/yolo-service:0.0.2` image instead of `alonithuji/yolo-service:0.0.1`.
In the Deployment YAML manifest, edit the `.spec.template.spec.containers[0].image` value, and apply again.

Run `kubectl get rs` to see that the Deployment updated the Pods by **creating a new ReplicaSet** and scaling it up to 3 replicas, as well as scaling down **the old ReplicaSet** to 0 replicas.
This actually means that the Deployment manipulate ReplicaSet objects for you.

This process is known as `RollingUpdate`. 

```console
$ kubectl describe deployments yolo-service
Name:                   yolo-service
Namespace:              default
CreationTimestamp:      Thu, 28 Sep 2023 20:10:41 +0000
Labels:                 app=yolo-service
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=yolo-service
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=yolo-service
  Containers:
   server:
    Image:        alonithuji/yolo-service:0.0.2
    Port:         8080/TCP
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  yolo-service-976d85c7c (0/0 replicas created)
NewReplicaSet:   yolo-service-599c84db5d (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  76s   deployment-controller  Scaled up replica set yolo-service-976d85c7c to 3
  Normal  ScalingReplicaSet  30s   deployment-controller  Scaled up replica set yolo-service-599c84db5d to 1
  Normal  ScalingReplicaSet  28s   deployment-controller  Scaled down replica set yolo-service-976d85c7c to 2 from 3
  Normal  ScalingReplicaSet  28s   deployment-controller  Scaled up replica set yolo-service-599c84db5d to 2 from 1
  Normal  ScalingReplicaSet  26s   deployment-controller  Scaled down replica set yolo-service-976d85c7c to 1 from 2
  Normal  ScalingReplicaSet  25s   deployment-controller  Scaled up replica set yolo-service-599c84db5d to 3 from 2
  Normal  ScalingReplicaSet  23s   deployment-controller  Scaled down replica set yolo-service-976d85c7c to 0 from 1
```

If you look at the above Deployment closely, you will see that it first creates a new Pod, then deletes an old Pod, and creates another new one.
It does not kill old Pods until a sufficient number of new Pods have come up, and does not create new Pods until a sufficient number of old Pods have been killed. 


Deployment ensures that only a certain number of Pods are down while they are being updated.
By default, it ensures that at least 75% of the desired number of Pods are up (25% **max unavailable**).

Deployment also ensures that only a certain number of Pods are created above the desired number of Pods.
By default, it ensures that at most 125% of the desired number of Pods are up (25% **max surge**).

When you updated the Deployment, it created a new ReplicaSet (`yolo-service-599c84db5d`) and scaled it up to 1 and waited for it to come up.
Then it scaled down the old ReplicaSet to 2 and scaled up the new ReplicaSet to 2 so that at least 3 Pods were available and at most 4 Pods were created at all times.
It then continued scaling up and down the new and the old ReplicaSet, with the same rolling update strategy.
Finally, you'll have 3 available replicas in the new ReplicaSet, and the old ReplicaSet is scaled down to 0.

> [!IMPORTANT]
> During a RollingUpdate, new versions of an application are gradually rolled out while old versions are gradually scaled down. This means that for a brief period, both the old and new versions of the application may be running concurrently in the cluster.
> this simultaneous running of multiple versions can potentially lead to compatibility issues. For example, if the new version of the application introduces changes to the data schema or format that are incompatible with the old version, it can lead to issues when both versions are accessing the same data store concurrently.


#### Failed Deployment

A Deployment enters various states during its lifecycle.
It can be progressing while rolling out a new ReplicaSet, it can be complete while the Deployment scaled up its newest ReplicaSet, or it can fail to progress.

Sometimes, your Deployment may get stuck trying to deploy its newest ReplicaSet without ever completing. 
This could happen due to various reasons, for example, insufficient resources (CPU or RAM), or image pull errors.

The Deployment will try to reach its desire state for 10 minutes (this value can be changed in `.spec.progressDeadlineSeconds`), after that, the Deployment will indicate (in the Deployment status) that the Deployment progress has stalled.

Let's play with your deployment:

- Scale up and down your deployment
- Try to update the Deployment to an invalid image tag, e.g. `alonithuji/yolo-service:9.9.9`. What happened? How did you resolve?

## Services

Every Pod in a cluster gets its own unique cluster-wide IP address, and Pods can communicate with all other pods on any other node. 

But using Pods IP is not practical. If you use a Deployment to run your app, that Deployment can create and destroy Pods dynamically. 
From one moment to the next, you don't know how many of those Pods are working and healthy; you might not even know what those healthy Pods are named. 
Kubernetes Pods are created and destroyed to match the desired state of your cluster.
Pods are ephemeral resources (you should not expect that an individual Pod is reliable and durable).

Enter **Services**.

The Service is an abstraction to help you expose **groups of Pods** over a network.
Each Service object defines a logical set of **Endpoints** (usually these endpoints are Pods) along with a policy about how to make those pods accessible.

The set of Pods targeted by a Service is usually determined by a `selector` that you define.

Here is an example for Service exposing port `8080` for all Pods in the cluster labelled by the `app: yolo-service` key and value (this was the Pod label in the `yolo-service` Deployment):

```yaml
# k8s/service-demo.yaml

apiVersion: v1
kind: Service
metadata:
  name: yolo-svc
spec:
  selector:
    app: yolo-service
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

Apply the above Service, and describe the created object:

```console
$ kubectl apply -f k8s/service-demo.yaml
service/yolo-svc created

$ kubectl describe svc yolo-svc
Name:              yolo-svc
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=yolo-service
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.100.220.135
IPs:               10.100.220.135
Port:              <unset>  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.244.0.48:8080,10.244.0.51:8080,10.244.0.58:8080 + 1 more...
Session Affinity:  None
Events:            <none>
```

`Endpoints` are the set of Pods IP that the Service is routing traffic to. 
The controller for that Service continuously scans for Pods that match this selector.

You should now be able to communicate with the `yolo-svc` Service on `10.100.220.135:8080` (change the IP to your service IP) from any node in your cluster.
The above created service can be used only for consumption inside your cluster.

The `kube-proxy` (running in every node) ensures that connections to the service's IP are properly routed (or **load balanced**) to the correct pods that belong to the service.


Let's use the `kubectl run` command to create a **temporary** pod name `busybox-client` (based on the useful [`busybox` docker image](https://hub.docker.com/_/busybox)) that will communicate with our yolo service: 

```bash
kubectl run busybox-client --rm --image=docker.io/busybox --restart=Never -- sh -c 'while true; do wget -qO- http://yolo-svc:8080/health; sleep 5; done'
```

### Service DNS 

Kubernetes is shipped with an internal DNS server, called [CoreDNS](https://coredns.io/), for resolving addresses and service discovery within the cluster. 
The CoreDNS server is running as a Pod in the `kube-system` namespace. 
It watches the Kubernetes API for new Services and creates a set of DNS records for each one.

Instead of accessing your service by its IP address, you can simply use the Service name as the domain name (to be running from one of the Pods in the cluster):

```bash 
curl yolo-svc:8080
```

## ConfigMap

A **ConfigMap** is a mechanism for storing non-sensitive configuration data in key-value pairs.
ConfigMaps provide a convenient way to inject configuration settings into applications, allowing for easy changes without modifying the container image or rebuilding it.

ConfigMaps can be consumed as **environment variables** or **mounted as files** within Pods.

### Mounting a ConfigMap as a file

In the Docker tutorial, you ran Prometheus by manually creating a `prometheus.yml` file on the host and bind-mounting it into the container with `-v`.
On Kubernetes, the equivalent is to store the configuration in a ConfigMap and mount it as a file inside the Pod.

Here is the Prometheus scrape configuration as a ConfigMap:

```yaml
# k8s/configmap-demo.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  # "file-like" key - the "|" in YAML allows multi-line values
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'yolo-service'
        static_configs:
          - targets: ['yolo-svc:8080']
```

Notice that unlike in Docker (where you needed the container IP), here you can use the Service DNS name `yolo-svc` directly - CoreDNS resolves it within the cluster.

After applying the ConfigMap, mount it into the Prometheus Deployment:

```yaml
# k8s/deployment-demo-configmap-mount.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: docker.io/prom/prometheus
        ports:
        - containerPort: 9090
        volumeMounts:
          - name: prometheus-config
            mountPath: /etc/prometheus/
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
```

The `volumes` field declares a volume backed by the `prometheus-config` ConfigMap.
The `volumeMounts` field mounts that volume into `/etc/prometheus/` inside the container - the same path Prometheus reads its config file from by default.

Apply both manifests and exec into the Prometheus pod to confirm the file was mounted:

```bash
kubectl exec -it <prometheus-pod-name> -- cat /etc/prometheus/prometheus.yml
```

ConfigMaps can also be consumed as **environment variables**, similarly to how you set `CONFIDENCE_THRESHOLD` in the Docker tutorial - useful for simple key-value settings that don't require a config file.

## Kubernetes core objects summary

![][k8s_core_objects]

# Exercises

### :pencil2: Deploy the yolo service

Deploy the yolo object-detection API you already know from the Docker tutorials - this time on Kubernetes.

1. Create a `Deployment` named `yolo-service` based on the [`alonithuji/yolo-service:0.0.2`](https://hub.docker.com/r/alonithuji/yolo-service) image. The container listens on port `8080`. Run **2 replicas**.
2. Expose it (internally to the cluster) with a `Service` named `yolo-svc` on port `8080`.
3. Verify the pods are running and the service has endpoints:
   ```bash
   kubectl get pods -l app=yolo-service
   kubectl describe svc yolo-svc
   ```
4. Forward the service to your local machine and send a test prediction request:
   ```bash
   kubectl port-forward service/yolo-svc 8080:8080 --address 0.0.0.0
   ```
   Then in another terminal:
   ```bash
   curl -X POST -F "file=@beatles.jpeg" http://<control-plane-ip>:8080/predict
   ```

### :pencil2: Deploy the frontend

In this exercise you add the [frontend](https://hub.docker.com/r/alonithuji/yolo-frontend) web UI to your cluster and connect it to the backend.

1. Create a `Deployment` named `yolo-frontend` based on the [`alonithuji/yolo-frontend:0.0.2`](https://hub.docker.com/r/alonithuji/yolo-frontend) image. The container listens on port `3000`.  
   The frontend discovers the backend via the `YOLO_API_URL` environment variable - set it to `http://yolo-svc:8080` (the Service DNS name from the previous exercise).
2. Expose it with a `Service` named `yolo-frontend-svc` on port `3000`.
3. Forward the frontend service and open it in your browser:
   ```bash
   kubectl port-forward service/yolo-frontend-svc 3000:3000 --address 0.0.0.0
   ```
   Open `http://<your-control-plane-ip>:3000` and upload an image - you should see the detection results returned by the backend.

### :pencil2: Deploy the agent

In this exercise you deploy your AI agent to the cluster and connect it to the yolo service.

1. Create a `Deployment` named `agent` using your agent's Docker image. The container listens on port `8080`. Set the following environment variables:
   - `YOLO_API_URL=http://yolo-svc:8080` - so the agent can reach the yolo service by its DNS name.
2. Expose it with a `Service` named `agent-svc` on port `8080`.
3. Verify the agent is running:
   ```bash
   kubectl get pods -l app=agent
   kubectl describe svc agent-svc
   ```
4. Forward the agent service and send a test chat request:
   ```bash
   kubectl port-forward service/agent-svc 8080:8080 --address 0.0.0.0
   ```
   Then in another terminal:
   ```bash
   curl -X POST http://<control-plane-ip>:8080/chat \
     -H "Content-Type: application/json" \
     -d '{"message": "How many people are in the image?"}'
   ```

### :pencil2: Deploy the img-proc-mcp (if already implemented)

If you have already built the image processing MCP server (`services/img-proc-mcp`) in the previous task, deploy it to the cluster as well.

1. Build and push the `img-proc-mcp` Docker image to your registry.
2. Create a `Deployment` named `img-proc-mcp` using that image. The MCP server listens on port `9000`.
3. Expose it with a `Service` named `img-proc-mcp-svc` on port `9000`.
4. Update the agent `Deployment` to add the environment variable:
   - `IMG_PROC_MCP_URL=http://img-proc-mcp-svc:9000/mcp`
5. Apply the updated manifests and verify the full stack is running:
   ```bash
   kubectl get deployments
   kubectl get services
   ```

### :pencil2: Deploy the Monitoring Stack

In this exercise you deploy [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/) to monitor the yolo service, exactly as you did in the Docker tutorial - but now as Kubernetes `Deployment`s.

> [!NOTE]
> In production, both Prometheus and Grafana benefit from persistent storage so that metrics and dashboards survive pod restarts. Persistent volumes and StatefulSets will be covered in a later session. For now, plain `Deployment`s are sufficient.

#### Prometheus

1. Create a `ConfigMap` named `prometheus-config` that holds the `prometheus.yml` scrape configuration targeting `yolo-svc:8080` (as shown in the ConfigMap tutorial section above).
2. Create a `Deployment` named `prometheus` based on the [`prom/prometheus`](https://hub.docker.com/r/prom/prometheus) image, with the ConfigMap mounted at `/etc/prometheus/`. The container listens on port `9090`.
3. Expose it with a `Service` named `prometheus-svc` on port `9090`.


#### Grafana

5. Create a `Deployment` named `grafana` based on the [`grafana/grafana`](https://hub.docker.com/r/grafana/grafana) image. The container listens on port `3000`. Set the following environment variables:
   - `GF_AUTH_BASIC_ENABLED=true`
   - `GF_SECURITY_ADMIN_USER` - your choice
   - `GF_SECURITY_ADMIN_PASSWORD` - your choice
6. Expose it with a `Service` named `grafana-svc` on port `3000`.
7. Forward the Grafana service and open it in your browser:
   ```bash
   kubectl port-forward service/grafana-svc 3000:3000 --address 0.0.0.0
   ```
8. Log in with the admin credentials you set above, then add Prometheus as a data source:
   - Go to **Connections** → **Data sources** → **Add data source** → **Prometheus**.
   - Set the URL to `http://prometheus-svc:9090`.
   - Click **Save & Test**.
9. Send a few prediction requests through the frontend and explore the yolo service metrics in Grafana's **Explore** panel.



### :pencil2: Expand the cluster

In this exercise you will grow your cluster by adding a second worker node and a second control plane node, simulating a production-grade, highly-available setup.

> [!TIP]
> To save time, use the prepared AMI `kubeadm-cluster-node-base-img` when launching new instances - it already has `kubeadm`, `kubelet`, `kubectl`, and `cri-o` installed from the setup script above. You only need to launch the instance and run the join command.

#### Add a second worker node

1. Launch a new `t3.medium` Ubuntu instance from the prepared AMI, naming it `<your-name>-worker-2`. Attach the `kubeadm-cluster-node-role` IAM role and the `kubeadm-cluster-node-sg` security group, with a `30 GiB` root volume.

2. On the **control plane node**, generate a fresh join command (valid for 24 hours):
   ```bash
   kubeadm token create --print-join-command
   ```

3. SSH into `worker-2` and run the printed `kubeadm join ...` command as root, adding the `--cri-socket` flag:
   ```bash
   sudo <paste the join command here> --cri-socket unix:///var/run/crio/crio.sock
   ```

4. Back on the control plane, confirm the new node appears and eventually reaches `Ready`:
   ```bash
   kubectl get nodes -o wide
   ```


#### Add a second control plane node

A single control plane is a single point of failure - if it goes down, the entire cluster API becomes unreachable.
Adding a second control plane node makes the cluster highly available.

1. Launch another `t3.medium` Ubuntu instance from the prepared AMI, naming it `<your-name>-control-plane-2`. Attach the same IAM role and security group as above.

2. On the **existing control plane**, upload the certificates so the new control plane node can share them, and generate a join command with the `--control-plane` flag:
   ```bash
   sudo kubeadm init phase upload-certs --upload-certs
   ```
   This prints a `--certificate-key`. Use it together with the regular join command:
   ```bash
   kubeadm token create --print-join-command
   ```


3. SSH into `control-plane-2` and run the combined join command as root: 

 ```bash
   sudo <join command> --control-plane --certificate-key <certificate-key> --cri-socket unix:///var/run/crio/crio.sock
   ```

4. Once it completes, set up `kubectl` on the new node as well:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

5. From either control plane node, verify all nodes are present:
   ```bash
   kubectl get nodes -o wide
   ```
   Both control plane nodes should show the `control-plane` role.

6. Stop (do **not** terminate) your original control plane instance from the AWS console. Can you still run `kubectl` commands from `control-plane-2`? What does this tell you about HA control planes?




[k8s_core_objects]:  https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_core_objects.png