# Docker compose brief

Are you tired by executing `docker run`, `docker build`? so are we.

[Docker Compose](https://docs.docker.com/compose/) is a tool for defining and running multi-container Docker applications.
With Compose, you use a YAML file to configure your application's services. 
Then, with a single command, you create and start all the services from your configuration.

A `docker-compose.yaml` looks like this:

```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
volumes:
  grafana-data:
  prometheus-data:
```

The given Docker Compose file describes a multi-service application with two services: `prometheus` and `grafana`. 

## Compose benefits 

Using Docker Compose offers several benefits:

- **Simplified Container Orchestration**: Docker Compose allows for the definition and management of multi-container applications as a single unit.
- **Reproducible Environments**: Since compose is defined in a YAML file, it's easy to deploy the same environment in different machine without missing any `docker run` command. This ensures that the application runs consistently across different machines.
- **Automate Volumes and Networking**: Docker Compose automatically creates a network for the application and assigns a unique DNS name to each service. No need to create networks and volumes. 


# Exercises

## :pencil2: PolyAI Docker Compose

In this exercise you will compose the full PolyAI stack.

The final network architecture looks like the following - you will create two custom networks and connect the services accordingly:

```
┌──────────────────────────────────────────┐     ┌──────────────────────────────┐
│               polyai-net                 │     │        monitoring-net        │
│                                          │     │                              │
│  frontend ──► agent ──► yolo             │     │  prometheus ──► grafana      │
│                                          │◄────│  (prometheus also joins      │
└──────────────────────────────────────────┘     │   polyai-net to scrape       │
                                                 │   metrics from yolo)         │
                                                 └──────────────────────────────┘
```

#### Notes

- The agent should reach the Yolo service using the `yolo` hostname: `http://yolo:8080`.
- Both grafana and prometheus should persist data using named volumes, as done in the previous exercise.
- The Prometheus config file (`prometheus.yml`) should be bind-mounted from the host so you can edit it directly.

Start the stack by:

```bash
docker compose up -d
```

Now you should open `http://localhost:3000` in your browser and interact with the PolyAI frontend.

To stop the stack, run:

```bash
docker compose down        # stops and removes containers, but NOT volumes
```

