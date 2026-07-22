# Pods and containers design

In this tutorial we will take a closer look on Pods and containers characteristics, and how they are designed and scheduled on nodes properly.

Throughout this tutorial, we will work again with the **YoloService**.

Let's apply the following `yolo-service` Deployment (delete any previous Deployments if exist):

```yaml
# k8s/resources-demo.yaml 

apiVersion: apps/v1
kind: Deployment
metadata:
  name: yolo-service
  labels:
    app: yolo-service
spec:
  replicas: 1
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
        image: alonithuji/yolo-service:0.0.1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
---
apiVersion: v1
kind: Service
metadata:
  name: yolo-svc
spec:
  selector:
    app: yolo-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
```


## Resource management for Pods and containers

As can be seen from the above YAML manifest, when you specify a Pod, you can optionally specify how much of each resource (CPU and memory) a container needs: 

```text 
Limits:
  cpu:     200m
  memory:  200Mi
Requests:
  cpu:     100m
  memory:  100Mi
```

What do the **Limits** and **Requests** mean? 

### Resource limits

When you specify a resource **Limits** for a container, the **kubelet** enforces those limits, as follows:

- If the container tries to consume more than the allowed amount of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error.
- If the container tries to consume more than the allowed amount of CPU, it is just not allowed to do it.

Let's connect to the Pod and create an artificial CPU load, using he `stress` command:

> [!NOTE]
> Before starting, make sure the `metrics-server` is installed in your cluster. 
> 
> Install it by running:
> ```bash
> kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
> ```
> [Read more](https://github.com/kubernetes-sigs/metrics-server?tab=readme-ov-file#installation) about the metrics-server installation.


```console 
$ kubectl exec -it <yolo-service-pod-name> -- /bin/bash
root@yolo-service-698dcc74b9-7d56w:/ # apt update
...

root@yolo-service-698dcc74b9-7d56w:/ # apt install stress-ng
...

root@yolo-service-698dcc74b9-7d56w:/ # stress-ng --cpu 2 --vm 1 --vm-bytes 50M -v
stress-ng: info:  [723] defaulting to a 86400 second (1 day, 0.00 secs) run per stressor
stress-ng: info:  [723] dispatching hogs: 2 cpu, 1 vm
```

Watch the Pod's CPU and memory metrics (via `kubectl top pod`). Although the stress test tries to use 2 cpus, it's limited to 200m (200 mili-cpu, which is equivalent to 0.2 cpu), as specified in the `.resources.limits.cpu` entry.
In addition, the Pod **is** able to use a `50M` of memory as it's below the specified limit (200 megabytes). 

Let's try to use more than the memory limit:

```console
/app # stress-ng --cpu 2 --vm 1 --vm-bytes 500M -v
stress-ng: info:  [723] defaulting to a 86400 second (1 day, 0.00 secs) run per stressor
stress-ng: info:  [723] dispatching hogs: 2 cpu, 1 vm
```

Watch how the processed is killed by the kernel using a `SIGKILL` signal. 

### Resources requests 

While resources limits is quite straight forward, resources request has a completely different purpose. 

When you specify the resource **Requests** for containers in a Pod, the **kube-scheduler** uses this information to decide which node to place the Pod on. How?   

When a Pod is created, the kube-scheduler should select a Node for the Pod to run on. Each node has a maximum capacity for CPU and RAM. 
The scheduler ensures that the resource CPU and memory requests of the scheduled containers is less than the capacity of the node.

```console
$ kubectl get nodes
NAME                    STATUS   ROLES           AGE   VERSION
john-control-plane      Ready    control-plane   15d   v1.35.0
john-worker             Ready    <none>          15d   v1.35.0

$ kubectl describe nodes john-worker
Name:               john-worker
[ ... lines removed for clarity ...]
Capacity:
  cpu:                2
  ephemeral-storage:  30297152Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3956276Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  30297152Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3956276Ki
  pods:               110
[ ... lines removed for clarity ...]
Non-terminated Pods:          (13 in total)
  Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
  default                     adservice-746b758986-vh7zt                    100m (5%)     300m (15%)  180Mi (4%)       300Mi (7%)     15d
  default                     cartservice-5d844fc8b7-rl7fp                  200m (10%)    300m (15%)  64Mi (1%)        128Mi (3%)     15d
  default                     checkoutservice-5b8645f5f4-gbpwn              50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     currencyservice-79b446569d-dqq7l              50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     emailservice-55df5dcf48-5ksdz                 50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     frontend-66b6775756-bxhbq                     50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     loadgenerator-78964b9495-rphvs                50m (2%)      500m (25%)  256Mi (6%)       512Mi (13%)    15d
  default                     paymentservice-8f98685c6-cjcmx                50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     productcatalogservice-5b9df8d49b-ng6pz        100m (5%)     200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     recommendationservice-5b4bbc7cd4-rkdft        50m (2%)      200m (10%)  220Mi (5%)       450Mi (11%)    15d
  default                     redis-cart-76b9545755-nr785                   70m (3%)      125m (6%)   200Mi (5%)       256Mi (6%)     15d
  default                     shippingservice-648c56798-m8vjh               100m (5%)     200m (10%)  64Mi (1%)        128Mi (3%)     15d
  kube-system                 calico-node-xk2p4                             250m (12%)    0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 kube-proxy-kv6m6                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 metrics-server-7746886d4f-t5btd               100m (5%)     0 (0%)      200Mi (5%)       0 (0%)         15d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                1820m (91%)   2925m (146%)
  memory             1743Mi (45%)  2840Mi (73%)
```

By looking at the “Pods” section, you can see which Pods are taking up space on the node.

Note that although actual memory resource usage on nodes is low (`45%`), the scheduler still refuses to place a Pod on a node if the memory request is more than the available capacity of the node. 
This protects against a resource shortage on a node when resource usage later increases, for example, during a daily peak in request rate.

Test it yourself! Try to change the `.resources.request.cpu` to a larger value (e.g. `3000m` instead of `100m`), and re-apply the Pod. What happened? 

Since the total CPU requests on that node is `91%` (`1820m` out of `2000m` CPU), you can see that if a new Pod requests more than `180m` or more than `1.2Gi` of memory, that Pod will not fit on that node.

> [!NOTE]
> In Production, always specify resource request and limit. It allows an efficient resource utilization and ensures that containers have the necessary resources to run effectively and scale as needed within the cluster.

## Configure quality of service for Pods

What happened if a container exceeds its memory request and the node that it runs on becomes short of memory overall? it is likely that the Pod the container belongs to will be **evicted**.

When you create a Pod, Kubernetes assigns a **Quality of Service (QoS) class** to each Pod as a consequence of the resource constraints that you specify for the containers in that Pod.
QoS classes are used by Kubernetes to decide which Pods to evict from a Node experiencing [Node Pressure](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/). 

1. [Guaranteed](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#guaranteed) - The Pod specifies resources request and limit. CPU and Memory requests and limit are equal.
2. [Burstable](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#burstable) - The Pod specifies resources request and limit. But CPU and Memory requests and limit are not equal.
3. [BestEffort](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#besteffort) - The Pod does not specify resources request or limit for some of its containers.


When a Node runs out of resources, Kubernetes will first evict `BestEffort` Pods running on that Node, followed by `Burstable` and finally `Guaranteed` Pods.
When this eviction is due to resource pressure, **only Pods exceeding resource requests are candidates** for eviction.

Try to simulate Node pressure by overloading the node from a `Guaranteed` Pod, `Burstable` Pod and `BestEffort` Pod, and see the behaviour. 

## Container probes - Liveness and Readiness probes

A **probe** is a diagnostic performed periodically by the **kubelet** on a container, usually by an HTTP request, to check the container status. 

- **Liveness probes** are used to determine the health of a container (a.k.a. health check), by a periodic checks. The container is restarting if the probe fails.
- **Readiness probes** are used to determine whether a Pod is ready to receive traffic (when it is prepared to accept requests, preventing traffic from being sent to Pods that are not fully operational, mostly because the pod is still initializing, or is about to terminate as part of a rolling update).

> [!NOTE]
> In Production, always specify Liveness and Readiness probes, they are essential to ensure the availability and proper functioning of applications within Pods.

### Define a Liveness probe

The YoloService exposes a `/health` endpoint. Upon an HTTP GET request, if the endpoint returns a `200` status code, it means "the server is alive".

In the below example, the `livenessProbe` entry defines a liveness probe for our Pod: 

```yaml
# k8s/liveness-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: yolo-service
  labels:
    app: yolo-service
spec:
  replicas: 1
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
        image: alonithuji/yolo-service:0.0.1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/health"
            port: 8080
```

Let's make the liveness probe fail by connecting to the yolo-service pod and killing the server process:

```console
$ kubectl exec -it <yolo-service-pod> -- /bin/bash

root@yolo-service-678b7f6564-wzwrq:/app# kill 1
```

Killing PID 1 terminates the server process. The kubelet detects that nothing is listening on port 8080 anymore, the liveness probe HTTP requests fail, and the **kubelet** restarts the container.

This restarting mechanism helps the application to be more available.
For example, when the server is entering a deadlock (the application is running, but unable to make progress),
restarting the container in such a state can help to make the application more available despite bugs.

By default, the HTTP probe is done every **10 seconds**, and should be low-cost, it shouldn't affect the application performance. 

### Define a Readiness probe

Sometimes, applications are temporarily unable to serve traffic.
For example, an application might need to load large data or configuration files during startup, or depend on external services after startup. 
In such cases, you don't want to kill the application, but you don't want to send it requests either.

Kubernetes provides readiness probes to detect and mitigate these situations. 
A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

Readiness probes are configured similarly to liveness probes:

```yaml
# k8s/readiness-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: yolo-service
  labels:
    app: yolo-service
spec:
  replicas: 1
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
        image: alonithuji/yolo-service:0.0.1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/health"
            port: 8080
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/health"
            port: 8080
```

In the above example, we use the `/health` endpoint to perform both the liveness and readiness probe HTTP requests.
The YoloService exposes this endpoint - when the server is up and ready it returns `200 OK`.

Deploy the above manifest and observe that the `yolo-service` Deployment completes the rolling update only once the new Pod passes its readiness probe.


#### Readiness probes are critical to perform a successful Rolling Update 

In the context of a rolling update, readiness probes play a crucial role in ensuring a seamless transition.
When old replicaset Pods receive the SIGTERM signal from the kubelet, the Pod should start a graceful termination process. First of all it should stop receiving new requests.
By failing the readiness probes, the Pod indicates that it's not ready to receive new traffic. 

## Assign Pods to Nodes

There are some circumstances where you may want to control which Node the Pod is deployed on.
For example, we probably want to ensure that our `yolo-service` Pods are scheduled on nodes that have a GPU, since object detection workloads are computationally intensive and benefit greatly from GPU acceleration.

There are [several ways](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) to allow the Node selection, [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) is the simplest recommended form. 
You can add the `nodeSelector` field to your Pod specification and specify the desirable target node.

```yaml
# k8s/node-selector-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: yolo-service
  labels:
    app: yolo-service
spec:
  nodeSelector:
    gpu: "true"   # <------ Kubernetes only schedules the Pod onto nodes that have the label you specify.
  replicas: 1
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
        image: alonithuji/yolo-service:0.0.1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/health"
            port: 8080
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/health"
            port: 8080
```

When apply the above manifest, you'll discover that the Pod cannot be schedule on any of the cluster's nodes. 
This because the **kube-scheduler** didn't find any Node with the `gpu=true` label.

Like many other Kubernetes objects, Nodes have **labels**:

```console
$ kubectl get nodes --show-labels
john-control-plane   Ready    control-plane   22d   v1.35.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=john-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=
john-worker          Ready    <none>          22d   v1.35.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=john-worker,kubernetes.io/os=linux
```

Let's manually attach the label `gpu=true` to your cluster's worker node:

```console
$ kubectl label nodes <your-worker-node> gpu=true
node/<your-worker-node> labeled
```

Make sure the Pod has been successfully scheduled on the labelled Node.

### Taints and Tolerations

We've seen that Node Selector is a simple mechanism to attract a Pod on specific Node (or group of nodes) by a label.
This technique is useful, for example, to schedule Pods on Nodes with specific hardware specification.

But what if we want to ensure that **only** those specific Pods use the dedicated Node?
For example, Nodes with an expensive GPU, it is desirable to keep pods that don't need the GPU off of those nodes, thus leaving room for later-arriving pods that do need the GPU. 

**Taints and Tolerations** can help us. [Taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) allow a node to **repel** a set of pods.

Let's add a **taint** to a node using `kubectl taint`:

```bash 
kubectl taint nodes <your-worker-node> gpu=true:NoSchedule
```

The taint has key `gpu`, value `true`, and taint effect `NoSchedule`.
This means that **no pod will be able to schedule** onto the worker node, unless it has a matching **toleration**.

A **Tolerations** allow the scheduler to schedule pods with matching taints.
To tolerate the above taint, you can specify toleration in the pod specification.

```yaml
# k8s/taint-toleration-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: yolo-service
  labels:
    app: yolo-service
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  nodeSelector:
    gpu: "true"
  replicas: 1
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
        image: alonithuji/yolo-service:0.0.1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/health"
            port: 8080
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/health"
            port: 8080
```

The above defined toleration would allow to schedule the Pod onto the worker node.

**Note**: Taints and Tolerations **do not** guarantee that the pods only prefer these nodes. 
So the pods may be scheduled on the other untainted nodes. 
To guarantee that group of pods would be scheduled on a node, and **only** them, both **node selector** and **taint and tolerations** should be used.

To remove the taint: 

```bash 
kubectl taint nodes <your-worker-node> gpu=true:NoSchedule-
```




# Exercises 

### :pencil2: Readiness and Liveness to the YoloService stack

Define a `livenessProbe` and a `readinessProbe` for the following services in your stack:

   - YoloService
   - YoloFrontend
   - Prometheus
   - Grafana



### :pencil2: (optional) Zero downtime during scale

Your goal in this exercise is to achieve zero downtime during **manual** scale up/down events of the **YoloFrontend** service.

1. Make sure the YoloFrontend `Deployment` and its `Service` are deployed in your cluster. Make sure the `Deployment` has readiness and liveness probes defined.
2. Generate some incoming traffic (20 requests per second) from a dedicated pod: 
   ```bash
   kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.05; do (wget -q -O- http://SERVICE_URL &); done"
   ```
   Change `SERVICE_URL` to the YoloFrontend service address.

3. Manually scale up the Deployment replicas by: 

```bash
kubectl scale deployment yolo-frontend --replicas=3
```

Watch the `load-generator` pod logs. Did you lose requests?

4. Manually scale down the number of replicas. Lost requests? Why?
5. Remove the `liveness` and `readiness` probes. Repeat steps 2-4. What happened? Why?
