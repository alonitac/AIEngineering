# Model Context Protocol (MCP)

So far, your agent defines its own tools locally:

```python
@tool
def detect_objects() -> str:
    """Detect objects in an image."""
    return requests.post("http://localhost:8080/predict", ...).text

llm_with_tools = llm.bind_tools([detect_objects])
```

This works great when **you own the tools**. But what if:

- A third-party platform (GitHub, AWS, Slack) wants to expose tools to your agent
- You want to share your tools with other agents or platforms
- Your tools live in a different process, machine, or service

You need a standard **communication protocol** between your agent (the client) and the tool server - so any agent can **discover** and **invoke** tools from any provider.

That's exactly what **MCP** is.


## What is MCP?

**Model Context Protocol (MCP)** is an open standard (by Anthropic) that defines how AI agents communicate with tool servers.


![][ai_mcp]






The protocol covers:
- **Tool discovery** - the client asks "what tools do you have?"
- **Tool invocation** - the client calls a tool with arguments
- **Results** - the server returns the tool output

### MCP server 

Here's a simple MCP server, built using FastMCP, that exposes a single tool `blur`:

```python
from fastmcp import FastMCP

mcp = FastMCP("img-proc")

@mcp.tool()
def blur(image_b64: str) -> str:
    """Apply Gaussian blur to an image. Returns base64-encoded PNG."""
    
    # blur the image and return the result
    blurred = image_b64  # ... image processing logic ...
    return blurred


if __name__ == "__main__":
    mcp.run(transport="http", port=9000)
```

### MCP Transports

In MCP, the communication between the client and the server uses JSON-RPC to define the request/response format. It means that the syntax is the regular beloved JSON, and the content structure is defined by the JSON-RPC 2.0 specification.

For example: 

```json
{
  "jsonrpc": "2.0",
  "id": 17,
  "method": "...",
  "params": {}
}
```

MCP supports multiple **transports** protocols for sending this JSONs: 

- **Streamable HTTP** - The regular HTTP protocol, but the server can optionally make use of Server-Sent Events (SSE) to stream multiple server messages (single client request - multiple responses). This is useful for long-running tools that produce output incrementally.
- **stdio** - The **client** launches the MCP server as a subprocess and talks to it using the Linux standard input/output streams. This is useful for local tools that don't need to be exposed over the network.

We'll use MCP over HTTP. Let's use `curl` to see the raw protocol in action.


```bash
# Step 1 - You must initialize the session with the server
curl -i -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": { "name": "curl", "version": "1.0" }
    }
  }'
```

The response headers include `mcp-session-id: <some-id>`. Copy that value - every subsequent request must include it.

```bash
# Step 2 - Confirm the handshake (no response body expected).
curl -s -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: <SESSION_ID>" \
  -d '{
    "jsonrpc": "2.0",
    "method": "notifications/initialized",
    "params": {}
  }'

# Step 3 - Discover tools.
curl -s -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: <SESSION_ID>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
  }'

# Step 4 - Call a tool.
curl -s -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: <SESSION_ID>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "blur",
      "arguments": { "image_b64": "<BASE64_ENCODED_IMAGE>" }
    }
  }'
```


## LangChain Agent as an MCP Client

Let's connect a LangChain agent to an MCP server.

```bash
pip install langchain-mcp-adapters langgraph
```

`langchain-mcp-adapters` enables agents to use tools defined across one or more MCP servers.



```python
async with MultiServerMCPClient({
    "img-proc": {
        "url": "http://localhost:9000/mcp",
        "transport": "http",
    }
}) as client:
    tools = await client.get_tools()        # discover tools from the MCP server
    llm_with_tools = llm.bind_tools(tools)  # use them just like local tools
```


## Use MCP server in VSCode Copilot Chat

VSCode Copilot is itself an MCP client. Create a `.vscode/mcp.json` file in your project to register MCP servers — Copilot picks it up automatically.

### GitHub

GitHub exposes a hosted MCP server over HTTP — no installation needed.

Create `.vscode/mcp.json`:

```json
{
  "servers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  }
}
```

Copilot is already authenticated with GitHub, so no token is required. You can now ask things like *"list my open pull requests"* or *"create an issue"* directly in Copilot Chat.

### AWS

The AWS MCP server uses `mcp-proxy-for-aws`, a lightweight proxy that forwards MCP requests to the AWS-hosted MCP endpoint.

- Install `uv` (like `pip` - a package manager)

  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```

- Install the proxy package

  ```bash
  uv tool install mcp-proxy-for-aws==1.6.2
  ```

- Add to `.vscode/mcp.json`

```json
{
  "servers": {
    "aws": {
      "command": "uvx",
      "args": [
        "mcp-proxy-for-aws@1.6.2",
        "https://aws-mcp.us-east-1.api.aws/mcp",
        "--metadata", "AWS_REGION=us-east-1"
      ]
    }
  }
}
```


<!-- ## Exercises

### :pencil2: Resources

MCP servers can also expose **resources** - read-only data the client fetches and injects into the model's context. Think of them as virtual files: the model doesn't call them like a tool; the client reads them upfront and includes them in the system prompt or conversation.

Example use cases: project files, database schemas, open issues, config files.

Resources use `resources/list` and `resources/read`:

```json
// list
{ "jsonrpc": "2.0", "id": 3, "method": "resources/list", "params": {} }

// read
{ "jsonrpc": "2.0", "id": 4, "method": "resources/read", "params": { "uri": "file:///README.md" } }
```

With `fastmcp`, expose a resource like this:

```python
@mcp.resource("config://app")
def get_app_config() -> str:
    """Application configuration."""
    return open("config.yaml").read()
```

**Exercise:** Add a resource to your `server.py` that returns the contents of a local text file (e.g. a `cities.txt` list). Connect to the server from a Python client, read the resource, and print its content.

---

### :pencil2: Prompts

MCP servers can also expose **prompts** - reusable, parameterized prompt templates. The client fetches a prompt by name, fills in arguments, and gets back a ready-to-use message list.

```json
// list prompts
{ "jsonrpc": "2.0", "id": 5, "method": "prompts/list", "params": {} }

// get a prompt
{ "jsonrpc": "2.0", "id": 6, "method": "prompts/get",
  "params": { "name": "weather_report", "arguments": { "city": "Tel Aviv" } } }
```

With `fastmcp`:

```python
from fastmcp.prompts import Message

@mcp.prompt()
def weather_report(city: str) -> list[Message]:
    """Generate a weather report prompt for a city."""
    return [
        Message(role="user", content=f"Give me a detailed weather report for {city}.")
    ]
```

**Exercise:** Add a prompt to your `server.py` that takes a `topic` argument and returns a message asking the model to explain that topic in simple terms. Call it from a client and pass the returned messages directly to your LangChain agent.

---

### :pencil2: Local MCP Server + VSCode Integration

1. Create a new file `my_mcp_server.py` with at least two tools of your choice (e.g. read a local file, return system info, call a public API).

2. Run it locally to verify it works:

   ```bash
   python my_mcp_server.py
   ```

3. Register it in VSCode's `settings.json`:

   ```json
   {
     "mcp": {
       "servers": {
         "my-server": {
           "command": "python",
           "args": ["/absolute/path/to/my_mcp_server.py"]
         }
       }
     }
   }
   ```

4. Open VSCode Copilot Chat and ask it to use one of your tools. Verify in the chat that Copilot invoked the tool and returned a result. -->


[ai_mcp]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/ai_agent_mcp_architecture.svg