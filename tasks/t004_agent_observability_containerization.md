# Extend the agent, observability, containerization

## Overview

In this task you'll extend the PolyAI service with image processing capabilities, set up monitoring solution for you ec2 instances, and run the entire system with a single `docker-compose.yaml`.


## Part I: Image processing capabilities for the agent

### Digital image processing

Reference: https://ai.stanford.edu/~syyeung/cvweb/tutorial1.html

If we take a closer look on a digital image, we will notice it comprised of individual pixels, 
each pixel has its own value. For a grayscale image, each pixel would have an **intensity** value between 0 and 255, with 0 being black and 255 being white. 

![][python_project_pixel]

A grayscale image, then, can be represented as a matrix of pixel values:

![][python_project_imagematrix]

A color image is just a simple extension of this. The colors are constructed from a combination of Red, Green, and Blue (RGB). Instead of one matrix of pixel values, we use 3 different matrix, one for the Red (R) values, one for Green (G), and one Blue (B) values. 

<img src="https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/python_project_colorpixels.png" width="50%">

As can be seen, each pixel of the image has three channels, represent the red, green, blue values. 

Python-wise, a digital grayscale image is essentially a matrix (list of lists):

![][python_project_pythonimage]

Each element in the `image` list is a list represented a **row** of pixels. 

### Image filtering

Filtered images are ubiquitous in our social media feeds, news articles, books—everywhere!
Image filtering is a technique in image processing that involves modifying or enhancing an image by applying a filter to it.
Filters can be used to remove noise, sharpen edges, blur or smooth the image, or highlight specific features or details, among other effects.

Python-wise, image filtering is as simple as manipulate the pixel values. 


### Image processing MCP Server

Under `services/img-proc-mcp`, create an MCP server app that exposes image manipulation tools. The agent will use these tools to process images or specific regions of images on demand.

```python
# services/img-proc-mcp/app.py
import base64
import io
from fastmcp import FastMCP
from PIL import Image, ImageFilter

mcp = FastMCP("img-proc")

def _decode(b64: str) -> Image.Image:
    return Image.open(io.BytesIO(base64.b64decode(b64)))

def _encode(img: Image.Image) -> str:
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    return base64.b64encode(buf.getvalue()).decode()

@mcp.tool()
def blur(image_b64: str, radius: float = 2.0) -> str:
    """Apply Gaussian blur to an image. Returns base64-encoded PNG."""
    img = _decode(image_b64).filter(ImageFilter.GaussianBlur(radius))
    return _encode(img)


if __name__ == "__main__":
    mcp.run(transport="http", port=9000)
```

Run it locally to verify the tools are registered:

```bash
pip install "mcp[cli]" pillow
mcp dev app.py
```

#### Tools to implement

Below is the full list your MCP server should support:

| Tool | Description |
|---|---|
| `rotate` | Rotate the image by a given angle |
| `flip` | Flip horizontally or vertically |
| `blur` | Apply Gaussian blur with a given radius |
| `resize` | Resize to given width × height |
| `crop` | Crop a region by bounding box coordinates |
| `add_noise` | Add salt-and-pepper noise |

#### Connect to the agent

Register the MCP server with your agent. The agent should now be able to handle natural-language prompts such as:

- *"Blur the second dog in the image (from the right)"*
- *"Rotate the entire image 90 degrees"*
- *"Add salt and pepper noise to the detected car"*

For object-specific operations (e.g. "the second dog"), the agent should call the Yolo API to get bounding boxes, extract that region, apply the transformation, and return the result.

#### Notes

- You should think what's better approach for the agent to communicate with the MCP server - sending the image in the request or use S3 as a storage layer? Send the entire image or just the relevant bounding box area?
- The MCP server should be covered by tests. 
- You might need to add some local tools to the agent to call relevant endpoints in the Yolo service. 

## Part II: Observability

As said in class, Prometheus works by **scraping**, i.e. it periodically sends an HTTP GET to a `/metrics` endpoint on each target and reads the numbers. In such way, you can monitor with Prometheus many different targets - your own applications, the operating system, databases, cloud services, and more. All you need is a `/metrics` endpoint that exposes the data in Prometheus format.

But what if your target-to-be-monitored does not natively expose a `/metrics` endpoint?
**Exporters** are small programs that bridge this gap. They sit next to the thing you want to monitor, collect its data (via an API, a file, execute command or any oher way), convert it to the Prometheus text format, and serve it on a `/metrics` endpoint for Prometheus to scrape.

A great example is your Ubuntu EC2 - you want to monitor CPU, memory, disk, network, etc. But Ubuntu does not expose a `/metrics` endpoint. So you run [**Node Exporter**](https://prometheus.io/docs/guides/node-exporter/) - a tool someone wrote that reads data from your Linux and exposes it in Prometheus format.


Here you can find a list of [Prometheus Exporters](https://prometheus.io/docs/instrumenting/exporters/).


### Metrics with Prometheus and Grafana

- Run **Prometheus** and **Grafana** and [**Node Exporter**](https://hub.docker.com/r/prom/node-exporter) as Docker containers.

- Configure Prometheus to scrape Node Exporter and the PolyAI service.
- In Grafana, import the community **Node Exporter Full** dashboard (ID **1860**) to get host-level visibility. 


## Part III: Service containerization

### Docker Compose

Create a single `docker-compose.yaml` in the **root of the repository** that brings up the entire system:

- Yolo
- Agent
- Frontend
- Image processing MCP server
- Prometheus
- Grafana
- Node Exporter


Verify the full stack starts cleanly with:

```bash
docker compose up
```

### Github Actions workflows

Replace your existed deployment pipeline as follows:

- The pipeline first should build and push a Docker image to DockerHub.  
  Every code change (either in `dev` or `main`) should trigger a new build with **a new image tag**. You can use the Git commit SHA (`${{ github.sha }}`) or a timestamp (`$(date +%Y%m%d%H%M%S)`) to generate a unique tag per build. **Never tag images as `latest` or the same tag for different build runs**.

- You should no longer run apps as a Linux service or have any source code deployed directly. Stack runs the entirely with a single `docker compose up -d` command.

- A simple skeleton of the workflow file should look like this: 


  ```yaml
  # .github/worflows/deploy.yaml
  name: Deploy PolyAI Stack
  on:
    push:
      branches:
        - dev
        - main

  jobs:
    build:
      # Build and push the Docker image to DockerHub

    deploy:
      needs: build   # If the build fails, don't deploy
      # Connect to the server and run `docker compose up -d` to redeploy the stack
  ```

- Since the image tag is changed dynamically, you must reference it in `docker-compose.yaml` as an environment variable:

  ```yaml
  services:
    yolo:
      image: myregistry/yolo:${YOLO_IMAGE_TAG}
  ```

  Then, you can set the `YOLO_IMAGE_TAG` environment variable (or in a `.env` file) before running `docker compose up`. Compose automatically reads environment variables or `.env` files and substitutes the variable in the YAML.

- **Each service should be independently deployable**. If only the agent code changes, only the agent should be rebuilt and redeployed. There are multiple ways to achieve this, the simplest is to create a separate workflow YAML file for each service - and it's TOTALLY FINE.

# Good Luck!


[python_project_demo]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/python_project_demo.gif
[python_project_pixel]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/python_project_pixel.gif
[python_project_imagematrix]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/python_project_imagematrix.png
[python_project_pythonimage]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/python_project_pythonimage.png
