# langchain-lyrouter

LangChain integration for the **LyRouter** — one API key, one endpoint, every model in your catalog,
with routing, fallback, BYOK credentials and usage accounting handled by the gateway instead of your app.

---

## Why

A LangChain app that talks to several vendors ends up carrying several SDKs, several key formats and several
sets of retry/fallback logic. LyRouter already solves that on the server side: it speaks the OpenAI protocol,
resolves your virtual key to a workspace, picks an upstream by cost / latency / throughput, injects the right
platform or BYOK credential, and falls through the candidate chain when one upstream fails.

This package makes that gateway a native LangChain provider:

| Without | With `langchain-lyrouter` |
|---|---|
| `ChatOpenAI` + `ChatAnthropic` + `ChatDeepSeek` + … | one `ChatLyRouter` |
| One API key per vendor, stored per app | one gateway key, scoped to a workspace |
| Retry / fallback logic in your chain | fallback chain configured once in the console |
| Cost and token accounting scattered per vendor | one usage, log and trace view for every call |

Everything else stays LangChain: the same `Runnable` interface, the same `.stream()`, `.bind_tools()`,
`.with_structured_output()`, and the same LangGraph and LCEL composition.

---

## Install

```bash
pip install langchain-lyrouter
```

Requires Python 3.10+ and `langchain-core >= 0.3`.

---

## Quickstart

Create an API key in the LyRouter console (**Workspace → API Keys**), then:

```bash
export LYROUTER_API_KEY="sk-your-key"
export LYROUTER_BASE_URL="https://your-gateway-host/v1"
```

```python
from langchain_lyrouter import ChatLyRouter

llm = ChatLyRouter(model="deepseek/deepseek-v4-flash")

print(llm.invoke("Explain an AI gateway in one sentence.").content)
```

Model ids are the catalog slugs shown in the console (`author/model`, e.g. `anthropic/claude-sonnet-4`,
`openai/gpt-4o`, `deepseek/deepseek-v4-flash`). To list what the key can actually reach:

```python
from langchain_lyrouter import list_models

for m in list_models():          # GET /v1/models
    print(m.id)
```

### Streaming

```python
for chunk in llm.stream("Write a two-line poem about routing."):
    print(chunk.content, end="", flush=True)
```

### Tool calling

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Return the current weather for a city."""
    return f"It is sunny in {city}."

agent = ChatLyRouter(model="anthropic/claude-sonnet-4").bind_tools([get_weather])
result = agent.invoke("What's the weather in Shanghai?")
print(result.tool_calls)
```

### Structured output

```python
from pydantic import BaseModel

class Ticket(BaseModel):
    title: str
    severity: int

extractor = ChatLyRouter(model="openai/gpt-4o").with_structured_output(Ticket)
print(extractor.invoke("Login is broken for all enterprise users, this is urgent."))
```

### Embeddings

```python
from langchain_lyrouter import LyRouterEmbeddings

embeddings = LyRouterEmbeddings(model="text-embedding-3-small")
vector = embeddings.embed_query("text to embed")
```

### Async

Every component implements the async `Runnable` methods:

```python
result = await llm.ainvoke("Hello")
async for chunk in llm.astream("Hello"):
    ...
```

---

## Routing

The gateway, not the client, decides which upstream serves a request. Two ways to steer it:

**1. Let the router choose the model.** Use the intelligent routing id and the gateway picks a model from the
workspace's allowed set, balancing cost against quality according to the console slider:

```python
llm = ChatLyRouter(model="lyrouter/auto")
```

**2. Name a model and let the gateway rank its upstreams.** A single slug may be served by several channels —
the vendor's own endpoint, a compatible provider, your own BYOK credential. The gateway sorts those candidates
by the workspace's channel strategy (`cost`, `latency`, `throughput`, manual order) and walks down the chain on
failure — auth errors, exhausted quota, rate limits, timeouts — until one succeeds.

Nothing about the fallback chain lives in this package: configure it once under **Routing** and **BYOK** in the
console and it applies to every client. Which upstreams and credentials a given request actually tried is
visible per-request in the console **Logs**.

> **Note**
> If the workspace has **Prevent overrides** enabled, routing-related parameters sent by the client are ignored
> in favour of the workspace setting. That is expected — the constructor argument is a request, not a guarantee.

---

## Configuration

| Parameter | Env var | Default | Notes |
|---|---|---|---|
| `api_key` | `LYROUTER_API_KEY` | — | Gateway virtual key (`sk-…`), sent as `Authorization: Bearer`. Never forwarded upstream. |
| `base_url` | `LYROUTER_BASE_URL` | — | OpenAI-compatible base, i.e. your gateway host **plus `/v1`**. Deployment-specific, no default. |
| `model` | `LYROUTER_MODEL` | — | Catalog slug, or `lyrouter/auto`. Omit to use the workspace default model. |
| `temperature`, `max_tokens`, `top_p`, … | — | — | Standard OpenAI chat parameters, passed through. |
| `timeout` | — | `60` | Per-request timeout in seconds. |
| `max_retries` | — | `2` | Client-side retries; upstream fallback is separate and happens inside the gateway. |
| `default_headers` | — | `{}` | Extra headers, e.g. your own correlation id. |

```python
llm = ChatLyRouter(
    model="lyrouter/auto",
    api_key="sk-your-key",
    base_url="https://your-gateway-host/v1",
    temperature=0.7,
)
```

> **Warning**
> `base_url` for this package is the **OpenAI-compatible** base and ends in `/v1`. The gateway also exposes an
> Anthropic-compatible entrypoint, but that one takes the gateway **root** without `/v1` — mixing them up gives
> a request path of `/v1/v1/messages` and a 404. If you need the Anthropic protocol, use the official Anthropic
> SDK (or `langchain-anthropic`) pointed at the gateway root; use this package for everything OpenAI-shaped.

---

## Observability

Token counts come back on the message as standard LangChain `usage_metadata`, so LangSmith and any LCEL
callback see them unchanged:

```python
msg = llm.invoke("Hi")
print(msg.usage_metadata)     # {'input_tokens': …, 'output_tokens': …, 'total_tokens': …}
print(msg.response_metadata)  # model actually served, upstream, finish reason, request id
```

The authoritative record is still the gateway's: every call lands in the console **Logs** with the resolved
model, the upstream and credential that served it, latency, token usage and cost.

---

## Compatibility

| Item | Supported |
|---|---|
| Python | 3.10 – 3.13 |
| `langchain-core` | `>= 0.3` |
| Chat completions | ✅ `/v1/chat/completions`, sync + async, streaming + non-streaming |
| Tool calling | ✅ `bind_tools()`, parallel tool calls |
| Structured output | ✅ `with_structured_output()` (JSON schema / Pydantic) |
| Embeddings | ✅ `/v1/embeddings` |
| Multimodal input | ✅ image input via `image_url` content blocks, on vision-capable models |
| Responses API | 🚧 planned — the gateway exposes `/v1/responses`, the wrapper does not yet |
| Anthropic protocol | ❌ out of scope, use `langchain-anthropic` against the gateway root |

---

## Development

```bash
git clone https://github.com/lyrouter/langchain-lyrouter.git
cd langchain-lyrouter
uv sync --all-extras          # or: pip install -e ".[dev]"

make test                     # unit tests, no network
make integration              # requires LYROUTER_API_KEY + LYROUTER_BASE_URL
make lint format
```

Integration tests hit a real gateway and will consume credits; they are skipped unless both environment
variables are set.

---

## Links

- LyRouter console → **Docs** for gateway concepts (calling the gateway, routing, BYOK, logs)
- Issues and feature requests: <https://github.com/lyrouter/langchain-lyrouter/issues>

## License

MIT
