# Docker images

So far you've been running containers from images that someone else built and pushed to DockerHub.
Now it's time to build your own.

You'll take the Yolo service source code and package it into a Docker image yourself - the same image you've been using all along.

## Building an image

To build an image out of the Yolo service source code, run this command *in the repository directory (`YoloService`)*:

```bash
docker build -t yolo-service:0.0.1 .
```

- `docker build` reads a file called `Dockerfile` and executes each instruction in order.
- `-t yolo-service:0.0.1` assigns the name `yolo-service` and the tag `0.0.1` to the resulting image.
- `.` is the **build context** - the directory whose files are made available to the build.

Watch the output as Docker works through each step. The first build will take a few minutes - it needs to download the base image and install all Python dependencies.

When it finishes, verify the image is available locally:

```bash
docker images
```

```console
REPOSITORY     TAG       IMAGE ID       CREATED          SIZE
yolo-service   0.0.1     3a8f2c1d4e01   10 seconds ago   1.02GB
```

Now let's look at what's inside that `Dockerfile` and understand what Docker actually did.

## The Dockerfile

A **Dockerfile** is a plain text file that contains a sequence of instructions telling Docker how to build an image.
Each instruction adds a layer on top of the previous one (more on layers shortly).

Here is the Dockerfile from the YoloService repository:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY . .

RUN pip install -r torch-requirements.txt
RUN pip install -r requirements.txt

EXPOSE 8080

CMD ["python", "app.py"]
```

Let's walk through each instruction:

| Instruction | What it does |
|---|---|
| `FROM python:3.11-slim` | Starts from an official Python 3.11 base image. The `slim` variant is a minimal Debian-based image - smaller than the full image, without unnecessary tools. Every image must begin with `FROM`. |
| `WORKDIR /app` | Sets `/app` as the working directory for all subsequent instructions. If the directory doesn't exist, Docker creates it. Commands like `RUN`, `COPY`, and `CMD` all operate relative to this path. |
| `COPY . .` | Copies everything from the **build context** (the directory you run `docker build` from) into `/app` inside the image. The first `.` is the source (host), the second `.` is the destination (image). |
| `RUN pip install -r torch-requirements.txt` | Executes a shell command during the build. Installs the PyTorch dependencies. This happens at build time, not at runtime. |
| `RUN pip install -r requirements.txt` | Installs the remaining Python dependencies. |
| `EXPOSE 8080` | Documents that the container listens on port 8080. This is informational only - it doesn't actually publish the port. You still need `-p` when running the container. |
| `CMD ["python", "app.py"]` | Defines the default command to run when a container starts from this image. Written in **exec form** (a JSON array), which is preferred because it doesn't invoke a shell and handles signals correctly. |

## Docker Image Layers

Every instruction in a Dockerfile that modifies the filesystem (`FROM`, `COPY`, `RUN`) creates a new **layer**.
Think of layers as a stack of filesystem snapshots:

```
┌─────────────────────────────────┐
│  CMD ["python", "app.py"]       │  ← Not a layer (metadata only)
├─────────────────────────────────┤
│  RUN pip install requirements   │  ← layer 5
├─────────────────────────────────┤
│  RUN pip install torch-req      │  ← layer 4
├─────────────────────────────────┤
│  COPY . .                       │  ← layer 3
├─────────────────────────────────┤
│  WORKDIR /app                   │  ← layer 2
├─────────────────────────────────┤
│  FROM python:3.11-slim          │  ← layer 1 (base image)
└─────────────────────────────────┘
```

Each layer only stores the **diff** - the files added or changed by that instruction relative to the layer below it.
The final image is the union of all layers stacked together.

You can inspect the layers of any image with:

```bash
docker inspect alonithuji/yolo-service:0.0.1
```

## Layer Caching

Docker is smart: when you rebuild an image, it doesn't re-execute every instruction from scratch.
Instead, it checks whether anything has changed since the last build. If a layer's instruction and its inputs are identical to a previous build, Docker **reuses the cached layer** and skips the step entirely.

This is called the **layer cache**, and it can make rebuilds dramatically faster.

**The catch**: once a layer is invalidated (because something changed), **all subsequent layers are also invalidated** and must be rebuilt - even if they haven't changed themselves.

### Caching in action

Clone the YoloService repository and build the image:

```bash
docker build -t yolo-service:my-build .
```

Watch the output - the first build downloads and installs everything from scratch. It will take a few minutes.

Now build again immediately, without changing anything:

```bash
docker build -t yolo-service:my-build .
```

```console
 => CACHED [2/6] WORKDIR /app                                              0.0s
 => CACHED [3/6] COPY . .                                                  0.0s
 => CACHED [4/6] RUN pip install -r torch-requirements.txt                 0.0s
 => CACHED [5/6] RUN pip install -r requirements.txt                       0.0s
```

Every step is served from cache. The rebuild completes in under a second.

Now make a small change to the application code - edit `app.py` and add a comment to the top:

```bash
echo "# my change" >> app.py
docker build -t yolo-service:my-build .
```

```console
 => CACHED [2/6] WORKDIR /app                                              0.0s
 => [3/6] COPY . .                                                         0.3s  ← cache miss!
 => [4/6] RUN pip install -r torch-requirements.txt                        ...   ← re-runs
 => [5/6] RUN pip install -r requirements.txt                              ...   ← re-runs
```

Layer 3 (`COPY . .`) detects that the source files changed, so its cache is invalidated.
Because layers 4 and 5 come *after* it in the stack, they are also invalidated and pip reinstalls everything - even though `requirements.txt` itself didn't change at all.

This is the core problem with the original Dockerfile.

## Multi-Stage Builds

When you compile code, the compiler itself doesn't belong in the final image. Consider a Go app:

```dockerfile
# Without multi-stage: the 800 MB Go toolchain ships to production
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
```

The `go build` command produces a single binary. Everything else - the Go compiler, standard library sources, build cache - is dead weight in the runtime image. Multi-stage builds solve this.

### The pattern

A **multi-stage build** uses multiple `FROM` instructions. Each starts a fresh filesystem. Use `COPY --from=<stage>` to pull only what you need into the final image:

```dockerfile
# Stage 1: compile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

# Stage 2: run - just the binary, nothing else
FROM debian:bookworm-slim
COPY --from=builder /app/server /server
CMD ["/server"]
```

The final image is `debian:bookworm-slim` (~90 MB) plus one binary. The Go toolchain never makes it in.


# Exercises

## :pencil2: Optimize the Dockerfile for Layer Caching

The `COPY . .` instruction in the original Dockerfile (Yolo service) copies all source files before installing dependencies.
This means any change to `app.py` (or any other file) invalidates the pip install layers and forces a full reinstall.

#### Reproduce the problem

- Make a trivial change to `app.py` (e.g., `echo "# my change" >> app.py`).
- Rebuild and observe that the `pip install` steps are re-executed even though the requirements didn't change.

#### Fix it

Optimize the Dockerfile by reordering the instructions so that:
- The requirement files (`torch-requirements.txt`, `requirements.txt`) are copied first.
- Dependencies are installed.
- The rest of the application code is copied last.

#### Bonus - Multi-stage build

Convert your optimized Dockerfile into a multi-stage build:

- In a `builder` stage, create a virtual environment in a **self-contained directory** (not inside `/app`) so it can be copied cleanly to the next stage. Install dependencies with `--no-cache-dir` to avoid storing pip's download cache in the layer.
- In a second stage, copy only that directory from the builder - no pip, no build tooling - then add the application code.

Compare image sizes before and after to verify the improvement:
```bash
docker images
```


## :pencil2: Build and push your project images

In this exercise you will build Docker images for each service in the project - Yolo, Agent, Frontend - and publish them to your personal DockerHub account.

- Go to [hub.docker.com](https://hub.docker.com) and click **Sign up**.
- You can sign up with your existing **GitHub account** using the "Continue with GitHub" option.
`
- Go to [hub.docker.com](https://hub.docker.com) → **Account Settings** → **Personal access tokens** → **Generate new token**.
- Give the token a name (e.g. `ec2`) and set the permissions to **Read & Write**.
- Copy the generated token - you won't be able to see it again.

  ```bash
  docker login --username <your-dockerhub-username> --password <your-token>
  ```

- The YoloService repository already has a Dockerfile. To build and push it:

  ```bash
  cd services/yolo
  docker build -t <your-dockerhub-username>/yolo-service:0.0.1 .
  docker push <your-dockerhub-username>/yolo-service:0.0.1
  ```

- Do the same for Agent and Frontend services. Each service has its own Dockerfile in its respective directory.


## :pencil2: Integrate Docker Scout into CI

[Docker Scout](https://docs.docker.com/scout/) scans your images for known security vulnerabilities (CVEs) in OS packages and Python dependencies, and shows you the fix version when one is available. In this exercise you will add a Scout scan step to your GitHub Actions workflow so every push reports the security posture of each image.


Docker Scout queries a continuously updated advisory database against the [Software Bill of Materials (SBOM)](https://github.com/resources/articles/what-is-an-sbom-software-bill-of-materials) it extracts from your image. It classifies findings by severity: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`.

Go to [hub.docker.com](https://hub.docker.com) → your repository → **Settings** → enable **Docker Scout**. This activates the advisory database for that repository (free for up to 3 repositories on the free plan).

### Add Scout to your GitHub Actions workflow

In your CI workflow, add a new **job** that builds and scans the image with Scout.

You can utilize the `docker/login-action`, `docker/build-push-action`, and `docker/scout-action` actions to log in, build, and scan the image.

Currently no need to push the image to DockerHub - you just scan it locally in the workflow. 

You should store your DockerHub username and personal access token as **GitHub secrets** so the workflow can log in to DockerHub.

Run the workflow and check the output. Did you find any CVEs? Try to fix them. 
