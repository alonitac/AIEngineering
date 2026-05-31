# HTTP Protocol

## Overview

The **Hypertext Transfer Protocol (HTTP)** is an application-level protocol that is being widely used over the Web.
HTTP is a **request/response** protocol, which means, the client sends a request to the server (request method, URI, protocol version, followed by headers, and possible body content). 
The server responds (status code line, a success or error code, followed by server headers information, and possible entity-body content).

![][http-req-res]

Under the hood, HTTP requests and responses are sent over a TCP socket with default port 80 (on the server side).
Servers should be able to handle thousands of simultaneous TCP connections.

## Endpoints

An **endpoint** is a specific URL path on a server that can be accessed by clients to perform a particular action or retrieve specific information.

For example, the Yolo service has an endpoint `/health` that clients can access to check if the server is running.

```python
@app.get('/health')
def health():
    return {'status': 'ok'}
```

This endpoint returns a simple JSON response indicating that the server is running. 
Start your application and verify it works:

```bash
python app.py
```

In another terminal, test the endpoint:

```bash
curl http://localhost:8080/health
```

You should see: `{"status":"ok"}`

## HTTP Request and Response

We can learn a lot by taking a closer look on a raw HTTP request and response that sent over the network:

```text
curl -v http://localhost:8080/health
```

Below is the actual raw HTTP request sent by `curl` to the server:

```text
GET /health HTTP/1.1
Host: localhost:8080
User-Agent: curl/7.68.0
Accept: */
```

The server response is:

```text
HTTP/1.1 200 OK
date: Sun, 04 May 2025 12:52:02 GMT
server: uvicorn
content-length: 15
content-type: application/json

{"status":"ok"}
```

The MDN web docs [specify the core components of request and response objects](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview#http_flow), review this resource.

## Status code

HTTP response status codes indicate how a specific HTTP request was completed.

Responses are grouped in five classes:

- Informational (100–199)
- Successful (200–299)
- Redirection (300–399)
- Client error (400–499)
- Server error (500–599)


## APIs

An **API (Application Programming Interface)** can be thought of as a collection of all server **endpoints** that together define the functionality that is exposed to the client.
Each endpoint typically accepts input parameters in a specific format and returns output data in a standard format such as JSON.

For example, a web API for a social media platform might include endpoints for retrieving a user's profile information, posting a new status update, or searching for other users.
Each endpoint would have a unique URL and a specific set of input parameters and output data.

Many platforms expose both API, and GUI. Like Spotify, OpenAI and GitHub.

## Introducing Postman

Postman is a powerful and user-friendly tool for testing, debugging, and documenting APIs.    

1. Download from: https://learning.postman.com/docs/getting-started/first-steps/get-postman/.
2. Create a Postman account to unlock some important features.

# Exercises

### :pencil2: Practicing the HTTP protocol 

#### The `Accept` header

The `Accept` header is used by HTTP clients to tell the server what content types they can process.
For example, if a client sends `Accept: application/json`, it indicates that the client expects the server to respond with JSON data.

1. Use `curl` to perform an HTTP `GET` request to `http://httpbin.org/image`.
   Add an `Accept` header to your requests with the `image/png` value to indicate that you anticipate a `png` image.
2. Read carefully the Warning message written by `curl` at the end of the server response, follow the instructions to save the image on the file system.
3. Execute another `curl` to save the image in the file system.

Which animal appears in the served image?

#### Status code

1. Perform an HTTP `GET` request to `google.com`
2. What does the server response status code mean? Follow the response headers and body to get the real Google's home page.
3. Which HTTP version does the server use in the above response?


### :pencil2: Return informational errors

In this exercise you'll modify your YOLO service to better handle unsuccessful requests.

When a user performs an unsuccessful request to your API, you'll raise an `HTTPException` with an informational `status_code` and `detail`:

```python
from fastapi import HTTPException

raise HTTPException(status_code=400, detail="Only image files are supported")
```

Modify your `/predict` endpoint to validate that the uploaded file is an image:

1. Check that the filename ends with a common image extension (`.jpg`, `.jpeg`, `.png`).
2. If the file is not an image, raise an `HTTPException` with status code `400` and an appropriate error message.
3. Test your changes using `curl`:

```bash
# This should fail with 400 Bad Request
curl -i -X POST -F "file=@document.pdf" http://localhost:8000/predict

# This should work
curl -i -X POST -F "file=@beatles.jpeg" http://localhost:8000/predict
```

Expected error response:

```json
{
  "detail": "Only image files are supported"
}
```
 

[http-req-res]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_http-req-res.png
[networking_cookies]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_cookies.png
