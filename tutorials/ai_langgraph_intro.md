# AI Agents with LangChain

We will start building the AI agent from the ground up.


## Step I: the LLMs

We start at the point we all know well and love: the LLM.

No matter if it's GPT, Gemini, Claude, or even [a model you run locally](https://docs.ollama.com/quickstart), models can interpret and generate text, understand intent, translate languages, summarize, and answer questions without needing specialized training for each task.

With LangChain, you can easily call hundreds of models from all major providers.

Let's install the LangChain package and some of the most popular model providers:

```bash 
pip install langchain[openai] langchain[anthropic] langchain[google-genai]
```

To initialize a model:

```python
import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."

model = init_chat_model("openai:gpt-5.4-mini")
```

Then you can call the model with a prompt:

```python
response = model.invoke("Why do parrots talk?")
```

## Step II: Tool calling 

Did you know that in addition to text generation, many modern models can **request** to **call tools** that perform tasks such as fetching data from a database, searching the web, or running code?

Let's define a tool that calls the Yolo API:

```python
from langchain.tools import tool
import requests

@tool
def detect_objects() -> str:
    """Detect and identify objects in the image provided by the user using YOLO object detection."""

    image_file = ... # get the image file from the user input

    return requests.post("http://localhost:8080/predict", files={"file": image_file}).text
```

The `@tool` decorator converts the function into a tool that LangChain can use.

To make this tool available to the model, we pass it in the `tools` argument when creating the agent:

```python
import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."

model = init_chat_model("openai:gpt-5.4-mini")
model_with_tools = model.bind_tools([detect_objects])
```

Now, upon the following example user prompt `hey what do you see in the image attached to my message?`, the LLM model (assuming supporting tool calling), based on the conversation context, can return a response containing a specific *request* to invoke this tool function, including the arguments it wants to pass.



```python
response = model_with_tools.invoke("hey what do you see in the image attached to my message? SOME_IMG_BASE64_STRING")
for tool_call in response.tool_calls:
    if tool_call['name'] == "detect_objects":
        tool_response = detect_objects.invoke(tool_call)
```

> [!IMPORTANT]
> Tools must be well-documented: informative and concise function’s docstring and typed argument names become part of the model's prompt.


## Step III: Agent = Model + Harness


An AI agent is a model calling tools in a loop until a given task is complete.

```mermaid
flowchart TD
    Request([request]) --> Model[model]

    Model -->|action| Tools[tools]
    Tools -->|observation| Model

    Model -.-> Result([result])
```

Your job as an agent developer is to create and manage the **harness**, i.e. get the right model, the right context at the right time for the given task.

A harness is everything around that loop:

- The model you choose to work with
- The prompts you provide
- The tools you provide
- Any middleware that shapes its behavior.


### The Agentic Loop

Here is a minimal, complete agentic loop:

```python
import os
import requests
from langchain.chat_models import init_chat_model
from langchain.tools import tool

os.environ["OPENAI_API_KEY"] = "sk-..."

@tool
def detect_objects() -> str:
    """Detect and identify objects in the image provided by the user using YOLO object detection."""
    with open("beatles.jpeg", "rb") as f:
        response = requests.post("http://localhost:8080/predict", files={"file": f})
    return response.text

llm = init_chat_model("openai:gpt-5.4-mini")
llm_with_tools = llm.bind_tools([detect_objects])

def run_agent(user_message: str) -> str:
    messages = [
        {"role": "system", "content": "You are a helpful vision assistant."},
        {"role": "user",   "content": user_message},
    ]

    while True:
        response = llm_with_tools.invoke(messages)
        messages.append(response)          # always record the model's reply

        # No tool calls → model has produced its final answer
        if not response.tool_calls:
            return response.content

        # Execute each tool the model requested
        for tool_call in response.tool_calls:
            if tool_call["name"] == "detect_objects":
                result = detect_objects.invoke(tool_call)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call["id"],
                    "content": str(result),
                })
        # Loop back: let the model see the tool results and decide what to do next

print(run_agent("How many people are in this image?"))
```

Two notes:

- Every model response goes into `messages`, even ones that only request tool calls. The model must see its own prior requests in the conversation history.
- The loop exits only when the model returns a reply with no tool calls - that is its final answer.

### Message Types

As can be seen, under the hood, LangChain sends the `messages` list to the LLM API as a plain JSON array. Each message has a **role** and a **content**.


Here is what the `messages` list actually looks like at each stage of the loop, in raw JSON form:

- After the first human message, before the first LLM call:

```json
[
  { "role": "system",    "content": "You are a helpful vision assistant." },
  { "role": "user",      "content": "How many people are in this image?" }
]
```

- After the model responds with a tool-call request (`AIMessage`):

```json
[
  { "role": "system",    "content": "You are a helpful vision assistant." },
  { "role": "user",      "content": "How many people are in this image?" },
  {
    "role": "assistant",
    "content": null,
    "tool_calls": [
      {
        "id": "call_abc123",
        "type": "function",
        "function": { "name": "detect_objects", "arguments": "{}" }
      }
    ]
  }
]
```

Note: `content` is `null` here - the model produced no text, only a tool-call request.

- After the tool result is appended:

```json
[
  { "role": "system",    "content": "You are a helpful vision assistant." },
  { "role": "user",      "content": "How many people are in this image?" },
  {
    "role": "assistant",
    "content": null,
    "tool_calls": [{ "id": "call_abc123", "function": { "name": "detect_objects", "arguments": "{}" } }]
  },
  {
    "role": "tool",
    "tool_call_id": "call_abc123",
    "content": "{\"detections\": [{\"label\": \"person\", \"score\": 0.95}, {\"label\": \"person\", \"score\": 0.88}]}"
  }
]
```

The `tool_call_id` ties the result back to the specific request - the model uses this to understand which tool call produced which result.

- Final model response:

```json
[
  ...,
  { "role": "assistant", "content": "There are 2 people in the image." }
]
```

This is when the loop exits and the agent returns the answer.


# Exercises

### :pencil2: Deploy the PolyAI System on Your EC2

PolyAI is a multi-service application: a `frontend` chat interface where you can upload images and ask questions about them, an LangChain `agent` that calls the `yolo` service you already built.

In this exercise you should deploy the whole system on your EC2.

#### Important implementation notes

- Use the same EC2 instance you used for the YOLO service. All services (`yolo`, `agent`, and `frontend`) should be run as Linux services.

- If you want, you can work **manually** on your dev instance, but the for prod instance, the `frontend` and `agent` (along with the existed `yolo`) services should be deployed **automatically, using your existed CD workflow**. For that, create a feature branch from `main` and modify your `.github/workflows/deploy.yml` workflow to deploy the `frontend` and `agent` services in addition to the `yolo` service. Then create a PR from your feature branch to `main` and merge it. If you've done everything correctly, the workflow should automatically deploy all three services on your prod EC2 instance.

- Better for your frontend app to be reachable at `<your-name>-dev.fursa.click:3000` instead of using the instance IP address.

- Don't forget to open required ports in your EC2 security group.

- For any required code changes, you must work according to our Git workflow: create a new branch from `main`, make your changes, merge to dev to test it there, create a PR to `main`. 

- To setup the frontend service locally: 

```bash
cd services/frontend
npm install   # only needed the first time
npm run dev
```

### :pencil2: Guard Against Infinite Loops

The current `run_agent` loop has no exit condition other than the model stopping tool calls. In theory, a confused model could call tools forever.

Add a `max_iterations` guard:

```python
def run_agent(history: list, max_iterations: int = 10) -> str:
    # your agentic loop code here
```

You can **manually** test it (in dev env only of course) by temporarily lowering `max_iterations` to `1` and sending a question that requires a tool call.

### :pencil2: Structured output

It's important to work with **validated**, **structured** data - in API responses, in service-to-service communication, and especially with LLMs, where the model can return an unstructured text reponse. 

**Pydantic** is a Python library that solves this. You define a schema as a Python class; Pydantic validates every value against it **at runtime**. FastAPI and LangChain both understand Pydantic natively - FastAPI uses it to type your HTTP responses, LangChain uses it to force the LLM to return valid JSON. [Python Docs](https://docs.pydantic.dev/latest/), visit there at your free time. 

Your task is to define **Pydantic models** for the below two outputs and make sure they are actually used (not just defined).

#### YOLO `/predict` response (`services/yolo/app.py`)

```json
{
  "uid": "a1b2c3d4-...",
  "timestamp": "2026-06-22T10:00:00Z",
  "original_image": "path/to/original/image.jpg",
  "predicted_image": "path/to/predicted/image.jpg",
  "detection_objects": [
    {
      "id": 0,
      "label": "person",
      "score": 0.95,
      "box": [x1, y1, x2, y2]
    }
  ]
}
```

> [!NOTE]
> Feel free to add any additional fields according to your YOLO service implementation.



#### Agent `/chat` response (`services/agent/app.py`)

```json
{
  "response": "I found 2 objects in the image.",
  "prediction_id": "a1b2c3...",
  "annotated_image": "<base64-string or null>",
  "agent_loop_time_s": 1.84,
  "iterations": 2,
  "tools_called": ["detect_objects"],
  "context_limit_exceeded": false
}
```

Some of these values (e.g. `annotated_image`, `prediction_id`) don't come from the LLM - you need to add them yourself as part of your agentic loop. Obviously, some of the values are optional.


### :pencil2: Return the Annotated Image

Right now the agent only responds with text. But the YOLO service can also return the **predicted image** (the original photo with bounding boxes)

Make the agent include the annotated image in its response, either always after a detection (simple), or only when the user explicitly asks for it (a bit more interesting).

#### Important implementation notes

- You guess right, you must follow the Git workflow of our course - create a new feature branch for this feature. 

- You will need to make changes in both the `agent` and the `frontend` services. As the `frontend` service is Next.js/TypeScript (out of the course scope), use AI to navigate and modify it. We strongly recommend you to use "Ask mode" first, let it to explain the relevant changes. Make sure you understand the changes. We recommend install the [official Next.js agent skills](https://github.com/vercel-labs/next-skills). 


### :pencil2: Model profiles

Model profiles are dictionaries of supported features for each model. For example, a profile for a model might look like this:

```python
model = init_chat_model("openai:gpt-5.4-mini")

print(model.profile)

# Output:
# {
#   "max_input_tokens": 400000,
#   "image_inputs": True,
#   "reasoning_output": True,
#   "tool_calling": True,
# }
```

Much of the model profile data is powered by the [models.dev](https://github.com/sst/models.dev) project, an open source initiative that provides model capability data. 

1. Use model profile to check that your selected model supports the features you need (structured output, tool calling) and raise an error if it doesn't.

2. Model profile has an important information about max input tokens. It's important to check this before sending large inputs to the model.

    You could try to count tokens *before* sending, but that requires the model's exact tokenizer - which is heavy, provider-specific, and sometimes not public. Instead, you can read usage metadata from the response after calling the LLM:

    ```python
    response = llm.invoke("Explain Docker in one paragraph")
    print(response.usage_metadata)
    # {'input_tokens': 12, 'output_tokens': 118, 'total_tokens': 130}
    ```

    Compare `input_tokens` against `model.profile["max_input_tokens"]` to detect when you're approaching the limit.

    Add a `tokens_used` field to your `/chat` endpoint response:

    ```json
    {
    "response": "There are 2 people in the image.",
    "tokens_used": { "input": 312, "output": 22, "total": 334 }
    }
    ```


### :pencil2: LLM API rate limits

Many LLM providers impose a limit on the number of request that can be made in a given time period.

Please carefully review [Antrhopic's rate limits](https://platform.claude.com/docs/en/api/rate-limits) and make sure you understand them. 

As seen, if you exceed the rate limit, the API will return a `429 Too Many Requests` error.

LangChain allows you to pass `rate_limiter` property to the `init_chat_model()` function. 

Please [read the docs](https://docs.langchain.com/oss/python/langchain/models#rate-limiting) and implement the `InMemoryRateLimiter` in your agent. Choose realistic values that fit all the models you use in your agent.


### :pencil2: Agent API and agentic loop testing


https://docs.langchain.com/oss/javascript/langchain/test/unit-testing