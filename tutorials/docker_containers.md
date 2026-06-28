# Intro to containers

You've already launched the Yolo service manually on an EC2 instance - installing Python, cloning the repo, installing dependencies, and running the app.

Now you'll do all of that with a single Docker command.

The image [`alonithuji/yolo-service:0.0.1`](https://hub.docker.com/r/alonithuji/yolo-service) packages the entire application and its dependencies into a ready-to-run container.

## Install Docker

To install Docker on your EC2 instance, SSH into the instance and run the following commands:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```

To run Docker commands without `sudo`, add your user to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

Then close the SSH session and log back in for the group change to take effect:


## Pulling the image from DockerHub

Before running the container, pull the image explicitly from DockerHub:

```bash
docker pull alonithuji/yolo-service:0.0.1
```

Docker downloads the image layers and stores them locally.
You can verify it was pulled by listing your local images:

```bash
docker images
```

## Running your first container

```bash
docker run alonithuji/yolo-service:0.0.1
```

You'll see the application logs appear in your terminal - the service is starting up.
Press `CTRL+C` to stop the container. Note that it is **stopped but not removed** - its filesystem still exists on your machine.

To run the container in the background (detached mode), so it's not tied to your terminal, use the `-d` flag:

```bash
docker run -d alonithuji/yolo-service:0.0.1
```

## Container lifecycle: stop and remove

List all containers, including stopped ones:

```console
$ docker ps -a
CONTAINER ID   IMAGE                            COMMAND            CREATED          STATUS                     PORTS     NAMES
a3f1c9e02b11   alonithuji/yolo-service:0.0.1   "python app.py"    2 minutes ago    Exited (0) 1 minute ago              a3f1c9e02b11
```

You can start the stopped container again:

```bash
docker start a3f1c9e02b11
```

Or remove it entirely (this deletes the container's filesystem):

```bash
docker rm a3f1c9e02b11
```

To stop a **running** container by name:

```bash
docker stop a3f1c9e02b11
```

> **Tip**: Use the `--rm` flag with `docker run` to automatically remove the container when it stops:
> ```bash
> docker run --rm alonithuji/yolo-service:0.0.1
> ```

## Publishing ports

The YoloService listens on port `8080` inside the container, but by default that port is not accessible from outside Docker.
Use the `-p` flag to map a host port to the container port:

```bash
docker run --name yolo -p 8080:8080 alonithuji/yolo-service:0.0.1
```

`-p 8080:8080` maps port `8080` on the **host machine** to port `8080` **inside the container**.

Now test it from your local machine (or the instance itself):

```bash
curl -X POST -F "file=@beatles.jpeg" http://localhost:8080/predict
```

Try running the container **without** `-p` and confirm the service is no longer reachable.

## Setting environment variables

The YoloService supports a `CONFIDENCE_THRESHOLD` environment variable that controls the minimum confidence score for a detection to be included in the results.

Pass it at runtime using the `-e` flag:

```bash
docker run --name yolo -p 8080:8080 -e CONFIDENCE_THRESHOLD=0.7 alonithuji/yolo-service:0.0.1
```

With this setting, only objects detected with >=70% confidence will be returned.
Try different values and compare the prediction results.

## Container logs

When a container is running in the background (detached mode with `-d`), you won't see its output directly in the terminal.
Use `docker logs` to inspect what the container has printed:

```bash
docker run -d --name yolo -p 8080:8080 alonithuji/yolo-service:0.0.1
docker logs yolo
```

To stream logs in real time (like `tail -f`):

```bash
docker logs -f yolo
```

Send a prediction request and watch the log output update live.

## Inspecting a container

The `docker inspect` command returns a detailed JSON representation of a container - its configuration, network settings, environment variables, mounts, and more.

```bash
docker inspect yolo
```

Look through the output and find:
- The container's IP address (`NetworkSettings.IPAddress`)
- The environment variables, including `CONFIDENCE_THRESHOLD`
- The exposed ports

## Overriding the default command

The default command defined in the Dockerfile is `python app.py`.
You can override it by appending a different command to `docker run`.

For example, to run the test suite instead of starting the server:

```bash
docker run --rm alonithuji/yolo-service:0.0.1 pytest tests/
```

Docker will start the container and execute `pytest tests/` instead of `python app.py`.
This is a common pattern for running tests in CI pipelines without any local setup.

## Executing commands inside a running container

Sometimes you need to peek inside a running container to debug an issue, inspect files, or run a one-off command.
Use `docker exec` to execute a command in a running container:

```bash
docker exec yolo ls /app
```

For a full interactive shell session inside the container:

```bash
docker exec -it yolo /bin/bash
```

- `-i` keeps STDIN open so you can type commands.
- `-t` allocates a pseudo-terminal (TTY) so it feels like a normal shell.

You're now inside the container's isolated filesystem. You can inspect the application files, check environment variables (`env`), or run any command as if you were logged into that machine.

Type `exit` to leave the shell - the container keeps running.


# Exercises 

## :pencil2: Monitor the Yolo Service with Prometheus and Grafana

In this exercise you will run three containers side by side:

- **Yolo Service** - the object detection app you already know.
- **[Prometheus](https://hub.docker.com/r/prom/prometheus)** - an open-source monitoring system that periodically *scrapes* (pulls) metrics from your services and stores them as time-series data. Think of it as a metrics database with a built-in collector.
- **[Grafana](https://hub.docker.com/r/grafana/grafana)** - an open-source visualization tool that connects to data sources like Prometheus and lets you build dashboards and graphs from the collected metrics.

#### 1. Run the Yolo Service

```bash
docker run -d --name yolo -p 8080:8080 alonithuji/yolo-service:0.0.1
```


#### 2. Configure and run Prometheus

First, take a look at the metrics the Yolo Service already exposes - open this URL in your browser (or use curl):

```
http://<your-ec2-ip>:8080/metrics
```

You'll see a long list of numbers and names. Each line is a metric - things like how many requests were made, how long they took, how much memory is in use. The app is already collecting all of this automatically.

Prometheus works by periodically visiting that `/metrics` URL and recording the values over time. To know *which* app to visit, it needs a small config file.

Create it on your host - replace `<yolo-container-ip>` with the container IP of your Yolo Service (get it from `docker inspect` as shown above):

```bash
cat > prometheus.yml << EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'yolo-service'
    static_configs:
      - targets: ['<yolo-container-ip>:8080']
EOF
```

This tells Prometheus: "every 15 seconds, go to `<yolo-container-ip>:8080/metrics` and collect the numbers."

Now run Prometheus, using the `-v` flag to make the file available inside the container:

```bash
docker run -d --name prometheus -p 9090:9090 -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

`-v <host-path>:<container-path>` is called a **bind mount** - it makes a file from your host machine visible inside the container. Here, Prometheus will read the config file you just created.

### 3. Run Grafana

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana
```

Open `http://<your-ec2-ip>:3000` in your browser (*don't forget to open port 3000 in your security group*). The default username and password are both `admin`.

### 4. Connect Grafana to Prometheus

Get the Prometheus container IP (again, from `docker inspect`) and use it to configure the data source in Grafana:

1. On the left panel, go to **Connections** → **Data sources** → **Add data source**.
2. Select **Prometheus**.
3. Set the URL to `http://<prometheus-container-ip>:9090`.
4. Click **Save & Test** - you should see a green confirmation.

Go to the **Explore** panel and try querying a metric from the Yolo Service. Can you build a graph showing the number of predictions over time? 

*Bonus*: Import Dashboard ID `22676` from the Grafana community to visualize Yolo Service metrics.
