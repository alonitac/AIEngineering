
# Preparing Kubernetes Migration, Observability, MCP Tooling

## Overview

The PolyAI stack (Yolo, Agent, Frontend, img-proc-mcp, Prometheus, Grafana, Node Exporter) runs on EC2 via Docker Compose. This task migrates it to Kubernetes and adds observability tooling.

> [!IMPORTANT]
> **Keep the EC2 deployment running. Don't delete it even if your cluster is up and running**. The Kubernetes cluster will be the main deployment in future tasks. Both deployments run simultaneously throughout this task.


## Part I: Deploy the Full Stack to Kubernetes

Deploy every service from your Docker Compose stack to Kubernetes in both `dev` and `prod` namespaces. Put your Kubernetes manifests under `infra/k8s/` in your project repo.

> [!IMPORTANT]
> **Use plain `Deployment` objects for every service - including Prometheus and Grafana.** Do not use Helm charts or operators for this task. The goal is to understand how Kubernetes objects connect to each other and to AWS storage.


Add all three to every service Deployment:

1. **Liveness & Readiness probes** - HTTP probes on each service's `/health` endpoint. Reference: `k8s/liveness-demo.yaml`, `k8s/readiness-demo.yaml`.
2. **Resource requests & limits** - CPU and memory bounds per container. Reference: `k8s/resources-demo.yaml`.
3. **Horizontal Pod Autoscaler** - HPA for `yolo` targeting 50% CPU, `minReplicas: 1`, `maxReplicas: 5`. Reference: `k8s/hpa-demo.yaml`. you must test the HPA by sending a load of requests to the yolo service and observing the number of replicas increase.


Prometheus needs durable storage so metrics survive pod restarts. This requires wiring four Kubernetes objects together - EBS CSI driver → StorageClass → PersistentVolume → PersistentVolumeClaim - and mounting the PVC into the Prometheus Deployment.


## Part II: Container Log Collection to S3 (Old Deployment)

We collect metrics using Prometheus and visualize them in Grafana, but we also want to collect logs from the running containers. This is done by shipping logs to S3 using a tool called **Fluent Bit**.


1. In the S3 console, create a bucket named `<your-name>-polyai-logs` in the same region as your EC2 instances.
2. Add a lifecycle rule to delete objects older than **90 days** (let's assume this is the minimum required for regulation compliance).

### Configure Fluent Bit

Create `fluent-bit.conf` in your project root (next to `docker-compose.yaml`):

```ini
[SERVICE]
    Flush         5
    Log_Level     info

[INPUT]
    Name          tail
    Tag           docker.*
    Path          /var/lib/docker/containers/*/*-json.log
    Parser        docker
    DB            /var/log/flb_docker.db
    Mem_Buf_Limit 5MB
    Skip_Long_Lines On

[FILTER]
    Name          record_modifier
    Match         docker.*
    Record        host ${HOSTNAME}

[OUTPUT]
    Name              s3
    Match             *
    bucket            <your-name>-polyai-logs
    region            us-east-1
    s3_key_format     /logs/%Y/%m/%d/$TAG[1]_%H%M%S.gz
    compression       gzip
    upload_timeout    60s
    use_put_object    On
    total_file_size   1M
```

### Add Fluent Bit to Docker Compose

Add this service to `docker-compose.yaml`:

```yaml
fluent-bit:
  image: fluent/fluent-bit:3.1
  volumes:
    - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
    - /var/log:/var/log
```

Briefly review the Fluent Bit configuration, make sure you understand the general flow: it tails the Docker container logs, adds a `host` field, and uploads them to S3 in compressed format.

## Local MCP server for logs & metrics

In your project repo under `services/observability-mcp`, build a local MCP server to query container logs from S3 and metrics from Prometheus, directly from Copilot Chat in agent mode.

Register with VS Code Copilot by creating `.vscode/mcp.json` in your project root:

```json
{
  "servers": {
    "observability": {
      "type": "stdio",
      "command": "python",
      "args": ["services/observability-mcp/app.py"],
      "env": {
        "DEV_PROMETHEUS_URL": "http://<your-service-name>:9090",
        "PROD_PROMETHEUS_URL": "http://<your-service-name>:9090",
        "DEV_S3_LOGS_BUCKET": "<your-name>-polyai-logs-dev",
        "PROD_S3_LOGS_BUCKET": "<your-name>-polyai-logs-prod",
        "AWS_REGION": "us-east-1"
      }
    }
  }
}
```


### Test it

Open Copilot Chat in agent mode and send:

- *"Show me the logs of the yolo service container for the last 5 minutes"*
- *"Show me the CPU usage of the prod instance for the last 10 minutes"*
- *"What containers are shipping logs to S3?"*
- *"What happened to the yolo service at 2026-07-01 12:00:00? The client got an internal server error"*



## Agent Observability

Instrument the agent with Prometheus metrics (`prometheus_client`) and expose a `/metrics` endpoint. Build a Grafana dashboard that shows:

- Chat requests per minute, split by status (success / error)
- Request latency - p50, p95, p99
- Error rate over time
- Input and output token counts over time


Export the dashboard JSON to `infra/grafana/dashboards/agent.json`.


# Good Luck!

