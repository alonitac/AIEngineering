# Helm - The Kubernetes Package Manager

## Motivation

**Helm** is a "package manager" for Kubernetes. Here are some of the main features of the tool:

- **Helm Charts**: Instead of dealing with numerous YAML manifests, which can be a complex task, Helm introduces the concept of a "Package" (known as **Chart**) - a cohesive group of related YAML manifests that collectively define a single application within the cluster.
  For example, an application might consist of a Deployment, Service, PVC, PV, ConfigMap, etc.
  These manifests are interdependent and essential for the seamless functioning of the application. 
  Helm encapsulates this collection, making it easier to manage, version, and deploy as a unit.

- **Sharing Charts**: Helm allows you to share your charts, or use other's charts. Want to deploy MongoDB server on your cluster? Someone already done it before, you can use her Helm Chart to with your own configuration values to deploy the MongoDB.  

- **Dynamic manifests**: Helm allows you to create reusable templates with placeholders for configuration values. For example:
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: {{ .Values.serviceName }}   # the service value will be dynamically placed when applying the manifest
  spec:
    selector:
      app: {{ .Values.Name }}   # the selector value will be dynamically placed when applying the manifest
  ...
  ```
  
  This becomes very useful when dealing with multiple clusters that share similar configurations (Dev, Prod, Test clusters). 
  Instead of duplicating YAML files for each cluster, Helm enables the creation of parameterized templates.

## Install Helm

To install Helm, run the following command from your control plane machine:

```bash 
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

## Helm Charts 

Helm uses a packaging format called charts. A chart is a collection of files that describe a related set of Kubernetes resources. 
A single chart might be used to deploy some single application in the cluster.

### Deploy the Prometheus monitoring stack using Helm

> [!IMPORTANT]
> Before starting, remove any existing Prometheus or Grafana deployments from your cluster, including all related resources (StatefulSets, Services, PVCs).

Like DockerHub, there is an open-source community Hub for Charts of famous applications.
It's called [Artifact Hub](https://artifacthub.io/packages/search?kind=0), check it out.

Rather than deploying Prometheus and Grafana as separate charts, we'll use the [**kube-prometheus-stack**](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack) - a single chart that installs a complete, pre-wired monitoring stack:

- **Prometheus** - scrapes and stores metrics from your cluster.
- **Grafana** - pre-built dashboards for nodes, workloads, the API server, and more.
- **Alertmanager** - routes and silences alerts.
- **Node Exporter** - exposes host-level metrics (CPU, memory, disk) from every node.
- **kube-state-metrics** - exposes metrics about Kubernetes objects (Pods, Deployments, PVCs…).

Add the Prometheus community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

The command syntax for installing a chart is: `helm install <release-name> <repo>/<chart-name>`.

Install the stack into a dedicated `monitoring` namespace:

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

A new **Release** named `kube-prometheus-stack` is created. Helm will print a summary of all the resources it created. Please **read the output carefully** - it contains important information about how to access Grafana and Prometheus, and how to port-forward to the services.

Verify that all components are running:

```bash
kubectl get pods -n monitoring
```

#### Try it yourself

Access Grafana using port-forward:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80 --address 0.0.0.0
```

The default user is `admin`. Open `http://<control-plane-node-ip>:3000` and explore:

- **Dashboards → Browse**: dozens of pre-built dashboards for Node metrics, Kubernetes workloads, the API server, persistent volumes, and more.
- **Explore**: run live PromQL queries against the Prometheus datasource.
- **Alerting → Alert rules**: browse the pre-configured alerting rules shipped with the chart.


### Customize the monitoring release

A typical Chart contains hundreds of configurable values. Usually, you can view all the default values for the kube-prometheus-stack in the chart documentation. 

Let's override the defaults that matter most for our cluster. Create a values file:

```yaml
# k8s/monitoring-values.yaml

prometheus:
  prometheusSpec:
    retention: 15d          # keep 15 days of metrics
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: ebs-sc
          resources:
            requests:
              storage: 10Gi

grafana:
  adminPassword: "your-secure-password"
  persistence:
    enabled: true
    storageClassName: ebs-sc
    size: 2Gi
```

These values back both Prometheus and Grafana with persistent EBS volumes so metrics and dashboards survive Pod restarts.

Apply the values with `upgrade`:

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring -f k8s/monitoring-values.yaml
```

An `upgrade` takes an existing Release and upgrades it according to the new values.
Helm performs the least invasive change possible - only resources that need to change are updated.

If something goes wrong, roll back to the previous revision:

```bash
helm rollback kube-prometheus-stack 1 --namespace monitoring
```

To uninstall the release entirely:

```bash
helm uninstall kube-prometheus-stack --namespace monitoring
```
