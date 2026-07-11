# Kubernetes Deployment and Storage

In this tutorial, we will deploy **Prometheus** as a Deployment backed by an AWS EBS persistent volume, so that collected metrics data survives Pod restarts and rescheduling.

## Persistent Storage

So far, Pods in the cluster were ephemeral - when a Pod restarts, all files written during its lifetime are lost.

For a monitoring tool like Prometheus, this means losing all historical metrics data every time the Pod is rescheduled or updated.

Kubernetes solves this with [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/): a piece of durable storage bound to a Pod so that data lives beyond the Pod's lifetime.
In our case, the backing storage will be an **AWS EBS (Elastic Block Store)** volume.

## The AWS EBS CSI Driver

The [Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec) is a standard API between container orchestrators (like Kubernetes) and storage systems (like AWS EBS).

The **AWS EBS CSI Driver** is a plugin that allows Kubernetes to:

- Attach and detach EBS volumes to/from EC2 nodes.
- Mount and unmount EBS volumes inside Pods.
- (Optionally) create and delete EBS volumes dynamically on demand.

Without this driver, Kubernetes has no way to communicate with AWS to manage EBS volumes.
Your nodes already carry an IAM role, which grants the necessary EBS permissions.

### Install the EBS CSI Driver

Install the driver into your cluster:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.62"
```

Verify the driver pods are running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

### Create a StorageClass for EBS

A **StorageClass** describes the type of storage and how it is provisioned. Create one for `gp3` EBS volumes (see [Amazon EBS volume types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html) for a full comparison of available types):

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

`volumeBindingMode: WaitForFirstConsumer` tells Kubernetes to delay volume binding until a Pod is scheduled, **ensuring the EBS volume is created in the same Availability Zone as the node**.

```bash
kubectl apply -f k8s/ebs-storage-class.yaml
```

## Storage Concepts

Before provisioning storage for Prometheus, let's understand the three key objects involved:

- **PersistentVolume (PV)** - a piece of storage in the cluster, representing a physical disk (such as an AWS EBS volume). Pods cannot use a PV directly.
- **PersistentVolumeClaim (PVC)** - a *request* for storage. Pods bind to a PVC, which binds to a PV. This indirection **decouples** storage management from Pod specification: Pods can request storage without knowing the underlying infrastructure.
- **StorageClass (SC)** - describes a "class" of storage and its provisioner. For example, a StorageClass for AWS EBS `gp3` volumes backed by the EBS CSI driver.

![k8s_statefulset_and_storage_summary][k8s_statefulset_and_storage_summary]

We will use **static provisioning**: manually create an EBS volume, create a PV pointing to it, then create a PVC that claims it.

## Step 1 - Create an EBS Volume in AWS

First, find the Availability Zone of your worker node:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}{end}'
```

Then create a 5 GiB `gp3` EBS volume in the **same AZ** (replace `<az>` with your node's AZ, e.g. `eu-central-1a`):

```bash
aws ec2 create-volume \
  --volume-type gp3 \
  --size 5 \
  --availability-zone <az> \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=prometheus-data}]'
```

Note the `VolumeId` from the output (e.g. `vol-0a1b2c3d4e5f67890`). You will need it in the next step.

## Step 2 - Create a PersistentVolume (PV)

Create a PV that points to your EBS volume. Replace `<volume-id>` with the actual EBS volume ID:

```yaml
# k8s/prometheus-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: <volume-id>
    fsType: ext4
```

```bash
kubectl apply -f k8s/prometheus-pv.yaml
kubectl get pv
```

The PV status should be `Available`.

## Step 3 - Create a PersistentVolumeClaim (PVC)

A PVC claims the PV we just created:

```yaml
# k8s/prometheus-pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f k8s/prometheus-pvc.yaml
kubectl get pvc
```

The PVC status should transition to `Bound`, meaning it has been matched to the PV.

## Step 4 - Deploy Prometheus as a Deployment

Now deploy Prometheus as a **Deployment**, referencing the PVC for its data directory `/prometheus`:

```yaml
# k8s/prometheus-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
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
      securityContext:
        fsGroup: 65534   # prometheus runs as user nobody (65534); ensures the volume is writable
      containers:
        - name: prometheus
          image: prom/prometheus:v2.53.0
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-storage
              mountPath: /prometheus
      volumes:
        - name: prometheus-storage
          persistentVolumeClaim:
            claimName: prometheus-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
```

Apply it and verify:

```bash
kubectl apply -f k8s/prometheus-deployment.yaml
kubectl get pods -l app=prometheus
kubectl get pvc
```

To confirm that data is persisted, delete the Prometheus Pod and observe that the Deployment recreates it and remounts the same EBS volume:

```bash
kubectl delete pod -l app=prometheus
kubectl get pods -l app=prometheus -w
```

After the Pod restarts, all historical metrics data will still be available.

## Volumes and Data Protection

PVCs and PVs represent real infrastructure holding important data that you cannot afford to lose.

Kubernetes ships with a **Storage Object in Use Protection** feature:

- A PVC actively used by a Pod is not deleted until the Pod releases it.
- A PV bound to a PVC is not deleted until the PVC is removed.

**Reclaim Policy** controls what happens to the underlying EBS volume after the PV is released:

- `Retain` - The EBS volume is kept after the PV is deleted. You are responsible for cleanup. **Use this in production.**
- `Delete` - The EBS volume is deleted along with the PV.

Our `prometheus-pv` uses `Retain`, so deleting the PV will not destroy the EBS volume or its data.


# Exercises

### :pencil2: Persist Grafana data using a Deployment

Provision a Grafana server using a Deployment backed by an EBS persistent volume.
All Grafana data is stored under `/var/lib/grafana`. The server should run with 1 replica.

Follow the same steps as for Prometheus:

1. Create a 2 GiB EBS volume in AWS in the same AZ as your worker node.
2. Create a PV pointing to it.
3. Create a PVC.
4. Deploy Grafana (`grafana/grafana`) as a Deployment using the PVC.

Verify that your Grafana dashboards and data sources survive a Pod deletion.

### :pencil2: (optional) Dynamic volume provisioning

In the previous steps you created the EBS volume manually and referenced it in a PV.
Kubernetes also supports **dynamic provisioning**: you only create a PVC, and the EBS CSI driver automatically creates both the EBS volume in AWS and the corresponding PV object — no manual `aws ec2 create-volume` needed.

Repeat the Grafana exercise above using dynamic provisioning:

1. Instead of creating an EBS volume and a PV manually, create only a PVC that references the `ebs-sc` StorageClass.
2. Deploy Grafana as a Deployment using the PVC.
3. Observe that once the Pod is scheduled, the PVC transitions to `Bound` and a PV is automatically created by the EBS CSI driver.

> [!NOTE]
> Because the `ebs-sc` StorageClass uses `volumeBindingMode: WaitForFirstConsumer`, the PVC stays in `Pending` until a Pod that uses it is scheduled. At that point the EBS volume is created in the same Availability Zone as the node.

### :pencil2: (optional) Increase volume capacity

Assume the Prometheus volume is running low on disk space. This exercise guides you through properly resizing a Kubernetes volume.

Resizing is done by editing the **PersistentVolumeClaim** object. Never delete and recreate PV/PVC objects to increase size - this risks data loss.

The `ebs-sc` StorageClass already has `allowVolumeExpansion: true`, so you can increase the storage request in the PVC directly:

```bash
kubectl edit pvc prometheus-pvc
```

Kubernetes will interpret the change as a volume expansion request and trigger automatic resizing via the EBS CSI driver.

Once the PVC shows the condition `FileSystemResizePending`, restart the Pod to complete the file system resize on the node:

```bash
kubectl delete pod -l app=prometheus
```

Once complete, the PVC will automatically reflect the new size:

```bash
kubectl get pvc prometheus-pvc
```


[k8s_statefulset_and_storage_summary]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_statefulset_and_storage_summary.png
