# Docker Networking

## Docker network sandbox and drivers

The core idea of containers is **isolation**, so how can a container communicate with other container over the network? 
How can a container communicate with the public internet? 
It must be using the host machine network resources. 
How can that be implemented while keeping the host machine secure enough? 

Docker implement a virtualized layer for container networking, enables communication and connectivity between Docker containers, as well as between containers and the external network.
It includes all the traditional stack we know - unique IP address for each container, virtual network interface that the container sees, default gateway and route table. 

Below is the virtualized model scheme. In Docker, this model is implemented by [`libnetworking`](https://github.com/moby/libnetwork) library: 

![][docker_sandbox]

A **sandbox** is an isolated network stack. It includes Ethernet interfaces, ports, routing tables, and DNS configurations.

**Network Interfaces** are virtual network interfaces (E.g. `veth` ).
Like normal network interfaces, they're responsible for making connections between the container and the rest of the world. 

Network interfaces are connected the sandbox to networks.

A **Network** is a group of network interfaces that are able to communicate with each-other directly. 
An implementation of a Network could be a Linux bridge, a VLAN, etc. 

This networking architecture is not exclusive to Docker. Docker is based on an open-source pluggable architecture called the [**Container Network Model** (CNM)](https://github.com/moby/libnetwork/blob/master/docs/design.md). 

The networks that containers are connecting to are pluggable, using **network drivers**.
This means that a given container can communicate to different kind of networks, depending on the driver. 
Here are a few common network drivers docker supports:

- [`bridge`](https://docs.docker.com/network/bridge/): This network driver connects containers running on the **same** host machine. If you don't specify a driver, this is the default network driver.

- [`host`](https://docs.docker.com/network/host/): This network driver connects the containers to the host machine network - there is no isolation between the
  container and the host machine, and use the host's networking directly.

- [`overlay`](https://docs.docker.com/network/overlay/): Overlay networks connect multiple container on **different machines**,
  as if they are running on the same machine and can talk locally. 

- [`none`](https://docs.docker.com/network/none/): This driver disables the networking functionality in a container.

## The Bridge network driver

The `bridge` network driver is the default network driver used by Docker.
It creates an internal network bridge on the host machine and assigns a unique IP address to each container connected to that bridge.

Containers connected to the `bridge` network driver can communicate with each other using these assigned IP addresses.

A container receives an IP address out of the IP pool of the network it attaches to. 
The Docker daemon effectively acts as a DHCP server for each container.
Each network also has a **default subnet** mask and **gateway**.

In the same way, a container's hostname defaults to be the container's ID or name in Docker. 
You can override the hostname using `--hostname`.

## DNS services

Containers that are connected to the default bridge network inherit the DNS settings of the host, as defined in the `/etc/resolv.conf` configuration file in the host machine (they receive a copy of this file). 

Containers that attach to a custom network use Docker's embedded DNS server. 
The embedded DNS server forwards external DNS lookups to the DNS servers configured on the host machine.


# Containers storage

By default, all files created inside a container are stored on a writable container layer.
This means that the data doesn't persist when that container no longer exists.

Docker has two options for containers to store files on the host machine, so that the files are persisted even after the container stops: **volumes**, and **bind mounts**.

## Bind mounts 

**Bind mounts** provide a way to mount a directory or file **from the host machine into a container**.
Bind mounts directly map a directory or file on the host machine to a directory in the container. 

Let's take the `alonithuji/yolo-service:0.0.1` image as a concrete example.
The YoloService stores every uploaded image in `/app/uploads/` and records every prediction in an SQLite database at `/app/predictions.db` inside the container.
By default those files vanish when the container is removed - unless we bind-mount host paths in their place:

```bash
cd YoloService
docker run --rm -p 8080:8080 \
  -v $(pwd)/predictions.db:/app/predictions.db \
  -v $(pwd)/uploads:/app/uploads/ \
  --name yolo alonithuji/yolo-service:0.0.1
```

In this example, the two `-v` flags specify the bind mounts:
- `-v $(pwd)/uploads:/app/uploads/` maps the `uploads/` directory in your current working directory to `/app/uploads/` inside the container, so every uploaded image is stored on the host.
- `-v $(pwd)/predictions.db:/app/predictions.db` maps a single file on the host to `/app/predictions.db` inside the container, so every prediction record written to the SQLite database is persisted on the host.

Whenever the `yolo` container writes to either path, the data is actually stored on the host machine.
Every change the container makes is immediately reflected on the host, and vice-versa.

With these bind mounts in place, both the uploaded images and the predictions database survive even after the container is stopped or removed - they remain on your host machine.

Bind mounts are commonly used for development workflows, where file changes on the host are immediately reflected in the container without the need to rebuild or restart the container.
They also allow for easy access to files on the host machine, making it convenient to provide configuration files, logs, or other resources to the container.

## Volumes 

**Docker volumes** is another way to persist data in containers. 
While bind mounts are dependent on the directory structure and OS of the host machine, volumes are logical space that completely managed by Docker.
Volumes offer a higher level of abstraction, allow us to work with different kind of storages, e.g. volumes stored on remote hosts or cloud providers.
Volumes can be shared among multiple containers.

### Create and manage volumes

Unlike a bind mount, you can create and manage volumes outside the scope of any container.

Grafana stores its dashboards, data sources, and settings in `/var/lib/grafana` inside the container. To persist this data, we can create a volume and mount it to that path:

```bash
 
docker volume create grafana-storage

docker run -d \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  --name grafana \
  grafana/grafana
```

Let's inspect the created volume:

```bash
$ docker volume inspect grafana-storage
[
    {
        "CreatedAt": "2023-05-10T14:28:15+03:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/grafana-storage/_data",
        "Name": "grafana-storage",
        "Options": {},
        "Scope": "local"
    }
]
```

Inspecting the volume discovers the real location that the data will be stored on the host machine: `/var/lib/docker/volumes/grafana-storage/_data`.

The `local` volume driver is the default built-in driver that stores data on the local host machine.
But docker offers [many more](https://docs.docker.com/engine/extend/legacy_plugins/#volume-plugins) drivers that allow you to create different types of volumes that can be mapped to your container.
For example, the [Azure File Storage plugin](https://github.com/Azure/azurefile-dockervolumedriver) lets you mount Microsoft Azure File Storage shares to Docker containers as volumes.

# Exercises

### :pencil2: PolyAI service stack

```
┌──────────────────────────────────────────┐     ┌──────────────────────────────┐
│               polyai-net                 │     │        monitoring-net        │
│                                          │     │                              │
│  frontend + agent + yolo                 │     │  prometheus ──► grafana      │
│                                          │◄────│  (prometheus also joins      │
└──────────────────────────────────────────┘     │   polyai-net to scrape       │
                                                 │   metrics from yolo)         │
                                                 └──────────────────────────────┘
```

Your goal is to bring up the full stack shown above. You've already worked with the `yolo`, `grafana`, and `prometheus` containers in previous exercises. Now you need to connect them together using Docker's `bridge` network driver, and add the `frontend` and `agent` containers to the stack as well.


**Networking requirements**

- Create two custom bridge networks: `polyai-net` and `monitoring-net`.
- `frontend`, `agent`, and `yolo` should all be attached to `polyai-net`. The agent should reach the Yolo service using the `http://yolo:8080` URL.
- `prometheus` and `grafana` should both be attached to `monitoring-net`.
- `prometheus` should also be attached to `polyai-net` so it can scrape metrics from `yolo`.


### :pencil2: Persist Grafana and Prometheus using Volumes 

**Grafana** stores all of its state - dashboards, data source configurations, users, and settings - inside the container at `/var/lib/grafana`. Without persistence, any dashboard you create or data source you configure will be lost the moment the container is removed.


**Prometheus** has two distinct paths worth thinking about:

- `/prometheus` - the database path where Prometheus writes all scraped metrics. This is runtime data that grows over time, and we want it to survive container restarts. A **named volume** is the right choice here.
- `/etc/prometheus/prometheus.yml` - the configuration file that tells Prometheus which targets to scrape, what intervals to use, and how to label metrics. This is a file *you* author and maintain on the host. Since you need to edit it directly and have it reflected in the container immediately, a **bind mount** is more appropriate than a volume - it maps your local config file directly into the container rather than copying it into Docker-managed storage.

For each of the above, run the relevant containers with the appropriate `-v` flags to apply the persistence strategy described. Verify your persistence works by stopping and removing a container, starting a fresh one with the same mounts, and confirming the data is still there.


### :pencil2: Understanding user file ownership in docker

Many popular images - including `nginx` - run their main process as `root` inside the container by default.
Inside an isolated container this is usually acceptable, but it becomes a real concern the moment you bind-mount a directory from the host: files written by the container's `root` process appear on the host machine as owned by `root` (UID 0), even if *you* are a regular unprivileged user on that host.

We will investigate this case in this exercise.

1. On your host machine, create a directory `mkdir ~/nginx_logs`.
2. Run an `nginx` container with a bind mount that maps `~/nginx_logs` on the host to `/var/log/nginx` inside the container (where nginx writes its access and error logs). Run it in detached mode and expose port 80 outside the container (e.g. `8080:80`).
3. Send a few HTTP requests to the nginx server (`curl localhost:8080`) so that nginx actually writes to its log files.
4. On your host machine, inspect `~/nginx_logs` path by `ls -l`. Who owns the files that nginx created?
   What UID and GID do they have? Why?
5. Try to delete or overwrite one of those log files from your host machine as your regular user. What happens?


**Rootless containers**

One modern mitigation for this class of problem is running Docker itself in [**rootless mode**](https://docs.docker.com/engine/security/rootless/).
In rootless mode, the Docker daemon and all containers run entirely within the user's own namespace - the `root` user *inside* the container is mapped to your unprivileged host UID, not to the real system root. This means files created by a container via a bind mount are owned by you on the host, not by root, significantly reducing the blast radius if a container is compromised.



[docker_sandbox]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_sandbox.png
[docker_cache]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_cache.png
[docker_nginx_frontend_catalog]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_nginx_frontend_catalog.png