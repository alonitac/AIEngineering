# LangGraph Memory

The PolyAI vision agent you built in the previous tutorial can hold a conversation within a single session. Restart the process and it forgets everything. A different user on a different conversation can't access their own history or preferences. This tutorial covers the two memory layers LangGraph provides to solve these problems.

## Two kinds of memory

| | Checkpointer | Store |
|---|---|---|
| Scope | One conversation (thread) | Across conversations (any thread) |
| Persists | Full graph state snapshots | Arbitrary key-value data |
| PolyAI use case | Remember the image and detections across turns | Remember a user's preferences across sessions |



## Short-term memory: Checkpointers

You already used this: compile with a checkpointer and every call sharing the same `thread_id` picks up where it left off.

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "user-42"}}

# Turn 1 - send image + message
app.invoke(
    {"messages": [...], "image_key": "uploads/img.jpg", "detections": None, "processed_keys": []},
    config=config,
)

# Turn 2 - image_key, detections, and processed_keys already in state
app.invoke(
    {"messages": [{"role": "user", "content": "Now rotate it 90°"}]},
    config=config,
)
```

### Checkpointer backends

`InMemorySaver` (RAM-only) is fine for development. Use a persistent backend in production:

| Backend | Package | When to use |
|---|---|---|
| SQLite | `langgraph-checkpoint-sqlite` | Local dev |
| PostgreSQL | `langgraph-checkpoint-postgres` | Self-hosted prod |
| DynamoDB | `langgraph-checkpoint-aws` | AWS / serverless |



## Long-term memory: Stores

A Store persists data **outside** the graph state so it's accessible from any conversation thread. In the PolyAI stack, use this to remember preferences that apply to all of a user's sessions - for example, always remove watermarks before applying any filter.

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
```

Items live under a **namespace tuple** (typically `(user_id, category)`):

```python
user_id = "user-42"
namespace = (user_id, "preferences")

store.put(namespace, "watermark", {"remove_watermark": True})

results = store.search(namespace)
# results[-1].value == {"remove_watermark": True}
```

### Using the store from a node

You can access the store from a node by passing a `Runtime` object to the node function. The runtime provides access to the current context and the store.

```python
from dataclasses import dataclass
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str

async def agent_node(state: VisionState, runtime: Runtime[Context]):
    namespace = (runtime.context.user_id, "preferences")
    prefs = await runtime.store.asearch(namespace)
    pref_text = "\n".join(p.value.get("note", "") for p in prefs)

    system = SystemMessage(content=f"You are a vision assistant.\nUser preferences:\n{pref_text}")
    return {"messages": [llm_with_tools.invoke([system] + state["messages"])]}
```

Compile with both and pass context on invocation:

```python
app = graph.compile(checkpointer=checkpointer, store=store)

config = {"configurable": {"thread_id": "thread-1"}}
await app.ainvoke(
    {"messages": [{"role": "user", "content": "Blur all people"}]},
    config,
    context=Context(user_id="user-42"),
)
```

A new thread with the same `user_id` loads the same store data - preferences survive across sessions.


## Handling long conversations

After many filter operations, the `messages` list grows and token costs grow with it. The simplest fix is to trim before passing messages to the LLM. The full history stays in the checkpointer; you only shrink what gets sent to the model.

```python
from langchain_core.messages import trim_messages

def agent_node(state: VisionState):
    trimmed = trim_messages(
        state["messages"],
        max_tokens=4096,
        token_counter=llm,     # uses the model's tokenizer
        strategy="last",        # keep the most recent messages
        include_system=True,    # always keep the system message
    )
    return {"messages": [llm_with_tools.invoke(trimmed)]}
```

> **Trim vs summarize:** Trimming is simple and stateless - older messages are dropped. Summarization compresses them into a synthetic message, preserving more context at higher implementation cost. Use trimming unless context loss causes real problems.


## Exercises

### :pencil2: DynamoDB Checkpointer

The PolyAI stack runs on AWS. Replace `SqliteSaver` with `DynamoDBSaver` so conversation state survives Lambda restarts.

1. Install the package:
   ```bash
   pip install langgraph-checkpoint-aws
   ```

2. Create a DynamoDB table named `<yourname>-polyai-checkpoints` with:
   - Partition key: `pk` (String)
   - Sort key: `sk` (String)

3. Update `build_graph()` to use the new checkpointer:
   ```python
   from langgraph.checkpoint.aws.dynamodb import DynamoDBSaver

   checkpointer = DynamoDBSaver(table_name="<yourname>-polyai-checkpoints")
   ```

4. **Verify persistence**: start the agent, apply a filter, kill the process, restart it, then send a follow-up message (`"rotate it 90°"`) on the same `thread_id` without re-uploading the image. The agent should remember the `image_key` and `processed_keys`.


### :pencil2: Trim Long Conversations

A user applies 15 sequential filters in one session. By the end the messages list is huge and every LLM call is slow.

1. Add `trim_messages` to `agent_node` in `build_graph()`. Keep the most recent 4096 tokens and always include the system message.

2. After trimming, confirm the agent still applies filters correctly - `image_key`, `detections`, and `processed_keys` live in graph state (restored by the checkpointer), not in the messages list. Trimming messages does not lose them.

3. Test with a thread that has 20+ turns. Measure the token count before and after trimming.
