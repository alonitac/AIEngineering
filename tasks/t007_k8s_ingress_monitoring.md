# Exposing the Cluster and Production-Grade Monitoring

## Overview

Your cluster from previous task is provisioned by Terraform and synced by ArgoCD, but the PolyAI stack is still reachable only via `kubectl port-forward`.

In this task you'll:

1. Expose the stack to the internet through an **Nginx Ingress Controller**, an **Application Load Balancer**, and a **Route 53** domain.
2. Replace the hand-written Prometheus and Grafana Deployments with the **kube-prometheus-stack** Helm chart.
3. Route **Alertmanager** alerts to your mailbox through an **SNS** topic, and simulate a firing alert.
4. **Bonus**: install the **Cluster Autoscaler** so the ASG scales nodes automatically.



> [!NOTE]
> Everything you do manually in this task must eventually be as code in the provisioning and bootstrap workflows.
> The acceptance test stays the same: `terraform destroy` → re-run provision & bootstrap workflow → everything comes back up.


## Part I: Expose the stack to the internet

Install the Nginx Ingress Controller. Use the `baremetal` provider manifest (or the `ingress-nginx` Helm chart), so the controller is exposed by a `NodePort` Service as taught in class.

Pin the HTTP and HTTPS node ports to fixed values (e.g. `30080` and `30443`) instead of letting Kubernetes allocate random ones - your Terraform target group must know the port in advance.

Extend `modules/k8s-cluster` (or add a `modules/ingress` module) with:

- An **Application Load Balancer** with an HTTPS listener on port `443`.
- A corresponding **Target Group**.
- Attachment of the target group to the existing **worker ASG**.
- A **Route 53** record in the shared `fursa.click` hosted zone (use a Terraform **data source** to look up the zone, don't manage it as part of the Terraform stack, so `terraform destroy` won't delete it by accident):

Add `Ingress` manifests to your repo under `infra/k8s/dev/` and `infra/k8s/prod/` for the following services:

- Dev and prod Frontend and Agent
- Grafana & Prometheus (from the kube-prometheus-stack Helm chart)
- ArgoCD


## Part II: Monitoring and alerting with the kube-prometheus-stack

Read [Helm - The Kubernetes Package Manager](../tutorials/k8s_helm.md).

1. **Delete the plain Prometheus and Grafana manifests** you wrote in the previous task (Deployments, Services, ConfigMaps, PV/PVCs) from `infra/k8s/`, so ArgoCD prunes them from the cluster.

2. Install `prometheus-community/kube-prometheus-stack`, using a values file committed to `infra/k8s/monitoring/values.yaml`. Install with:

   Your values file must configure at least:
   - Prometheus storage on the `ebs-sc` StorageClass (3Gi) and `retention: 30d`.
   - Grafana persistence on `ebs-sc` StorageClass (1Gi).

3. Scrape **your own services**: create a `ServiceMonitor` for the agent and yolo `/metrics` endpoint in both `dev` and `prod`, and verify the targets are `UP` in the Prometheus UI.

   Do the same for the **Nginx Ingress Controller** (it exposes Prometheus metrics, you only need to enable them), then import the community [**NGINX Ingress Controller** dashboard](https://grafana.com/grafana/dashboards/9614-nginx-ingress-controller/) into Grafana. You should see request rate, latency and response codes per Ingress host - a single place to watch all traffic entering your cluster.

4. **Get alerts by email.** Configure Alertmanager (through the chart values) to publish alerts to an **SNS topic** that fans out to your mailbox. The topic and the email subscription are provisioned by Terraform.

   > [!TIP]
   > Alertmanager has a native SNS receiver - see [`sns_config`](https://prometheus.io/docs/alerting/latest/configuration/#sns_config).

5. Write your own `PrometheusRule` with at least **two** meaningful alert rules based on the agent and yolo metrics (e.g. high error rate, high latency), with different severities.

6. **Simulate a failure** that makes one of your rules fire, and prove the whole chain works: `Pending` → `Firing` in Prometheus → Alertmanager → email in your inbox. Then fix the cause and make sure you also receive the `RESOLVED` notification.


## Part III (Bonus): Cluster Autoscaler

The [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws) watches for Pods that are `Pending` because no node has room for them, and increases the ASG's desired capacity. When nodes are underutilized for long enough, it scales back down.

Install it in your cluster and prove it works: deploy a workload with resource requests too big for the current nodes, and watch the ASG launch a new worker that joins the cluster and runs the Pods. Then delete the workload and watch the cluster scale back down.

> [!IMPORTANT]
> Set the ASG `desired_capacity` back to `0` when you finish working, to avoid unnecessary costs.

# Good Luck!
