---
name: dapplepot-sdk
description: Integrate the DapplePot Python SDK (dapplepot-sdk) for security observability of LLM agents. Auto-instruments the anthropic, openai, langchain, and langgraph packages so every LLM and tool call is traced and scanned. Use when adding DapplePot to a Python agent, when the user mentions "dapplepot", "dp_sk_", "dapplepot-sdk", or asks to instrument Anthropic/OpenAI/LangChain/LangGraph calls for security or tracing.
---

# DapplePot SDK Integration

DapplePot is a Python SDK that adds runtime security and session tracing to LLM agents. It patches the standard `anthropic` and `openai` packages in-place, and plugs into `langchain` / `langgraph` via their native callback system — no per-call wrapping, no framework rewrite — running security checks synchronously on every LLM / tool event.

- Sign up + dashboard: <https://app.dapplepot.com>
- Docs: <https://docs.dapplepot.com>
- License: Apache 2.0

## When to use

- Adding DapplePot to a Python project.
- Instrumenting an Anthropic, OpenAI, LangChain, or LangGraph agent.
- Mentions `dp_sk_...`, `DapplePot(`, `instrument_anthropic()`, `instrument_openai()`, `callback_handler()`, `DapplePotBlockedError`, `DapplePotSessionTerminatedError`.

## Prerequisites

- Python 3.10+.
- `sdk_key` (looks like `dp_sk_…`) and `agent_id` — from <https://app.dapplepot.com>.

Do **not** invent credentials. If the user hasn't provided them, use env-var placeholders (`os.environ["DAPPLEPOT_SDK_KEY"]`, `os.environ["DAPPLEPOT_AGENT_ID"]`) or ask.

## Install

```bash
pip install dapplepot-sdk                # core
pip install "dapplepot-sdk[anthropic]"   # + anthropic patch deps
pip install "dapplepot-sdk[openai]"      # + openai patch deps
pip install "dapplepot-sdk[langchain]"   # + langchain / langgraph deps
pip install "dapplepot-sdk[all]"         # everything
```

## The core loop

Every DapplePot integration is the same shape:

1. **Initialize** — construct `DapplePot(...)` **once** at startup.
2. **Instrument** — call `dp.instrument_anthropic()` / `dp.instrument_openai()`, or (for LangChain/LangGraph) skip this and create a `dp.callback_handler()` per invocation.
3. **Session-wrap** each user request with an identity — `with dp.session(user_context_id=…, user_tenant_id=…)` for Anthropic/OpenAI, or pass the same identity into `dp.callback_handler(...)` for LangChain/LangGraph.
4. **Tool tracing** — DapplePot auto-detects tool calls per framework (see each framework section for how).
5. **Handle `DapplePotBlockedError`** near the LLM/tool call site so a single blocked call can fall back and the session continues.
6. **Handle `DapplePotSessionTerminatedError`** at the outer level to exit the conversation gracefully.
7. **Shutdown** — call `dp.shutdown()` before the process exits.

## Identity — who is this session for?

Every session (or callback handler) takes two optional identity kwargs. These are how the dashboard groups events by end-user and by tenant:

| Kwarg              | Purpose                                                                     |
| ------------------ | --------------------------------------------------------------------------- |
| `user_context_id`  | Identifies the end-user of the agent (e.g. `"user_123"`, an internal UID).  |
| `user_tenant_id`   | For multi-tenant agents, identifies the customer/tenant (e.g. `"acme_corp"`). |

```python
# Anthropic / OpenAI
with dp.session(user_context_id="user_123", user_tenant_id="acme_corp"):
    ...

# LangChain / LangGraph
handler = dp.callback_handler(user_context_id="user_123", user_tenant_id="acme_corp")
chain.invoke(..., config={"callbacks": [handler]})
```

Both are optional. The session ID itself is generated automatically — **do not pass `session_id=`** in default examples. Do not pass `tenant_id=` to the `DapplePot(...)` constructor — the tenant is resolved server-side from `sdk_key`.

## Integration mechanisms — what we hook into, what we don't

DapplePot supports exactly two integration mechanisms, one per family of framework:

| Framework                     | Mechanism                                                            | Entry point                     |
| ----------------------------- | -------------------------------------------------------------------- | ------------------------------- |
| Anthropic (`anthropic`)       | Runtime monkey-patch of the SDK's create/stream surfaces             | `dp.instrument_anthropic()`     |
| OpenAI (`openai`)             | Runtime monkey-patch of `chat.completions` (sync/async/streaming)    | `dp.instrument_openai()`        |
| LangChain                     | Callback handler wired into LangChain's own callback system          | `dp.callback_handler()`         |
| LangGraph                     | Same callback handler — LangGraph inherits LangChain's callback bus  | `dp.callback_handler()`         |

**What we do not support** (do not generate code that assumes these are patched):

- OpenAI **Responses API** (`client.responses.create`), **Assistants API**, **OpenAI Agents SDK**.
- Anthropic **Bedrock** (`AnthropicBedrock`) and **Vertex** (`AnthropicVertex`) clients.
- Other LLM providers: Google Gemini, Mistral, Cohere, Ollama, **LiteLLM**, OpenRouter.
- Other agent frameworks: **CrewAI**, **AutoGen**, **PydanticAI**, **LlamaIndex**, **smolagents**.
- Non-Python languages — there is no JS/TS/Go SDK today.

If the user is on any of these, say so plainly and offer migration to a supported client.

---

## Anthropic

Use the standard `anthropic` package unchanged.

```python
import anthropic
from dapplepot_sdk import DapplePot, DapplePotBlockedError, DapplePotSessionTerminatedError

# 1. Initialize + 2. Instrument (once, at startup)
dp = DapplePot(sdk_key="dp_sk_...", agent_id="your-agent-id")
dp.instrument_anthropic()

client = anthropic.Anthropic(api_key="...")

# 3. Session-wrap each user request with identity
# 5. Blocked → catch near the call site
# 6. Terminated → catch at the outer level
try:
    with dp.session(user_context_id="user_123", user_tenant_id="acme_corp"):
        try:
            resp = client.messages.create(
                model="claude-opus-4-7",
                max_tokens=1024,
                messages=[{"role": "user", "content": user_input}],
            )
        except DapplePotBlockedError as exc:
            resp = "[Response blocked by security policy]"   # session continues
except DapplePotSessionTerminatedError:
    print("Session terminated by security policy")           # session ended

# 7. Shutdown before process exit
dp.shutdown()
```

Sync, async (`AsyncAnthropic`), `stream=True`, and `messages.stream()` are all covered by one `instrument_anthropic()` call.

**How tools are traced.** Nothing you do. When you use Anthropic tool-use the standard way, DapplePot reads it off the wire:

- The `tool_use` blocks in the model's response → `tool_start`.
- The matching `tool_result` blocks in the *next* `messages.create()` call → `tool_end`.
- Add `"is_error": True` to a `tool_result` block to emit `tool_error` instead of `tool_end`.

```python
{"type": "tool_result", "tool_use_id": "...", "content": "API returned 503", "is_error": True}
```

---

## OpenAI

Use the standard `openai` package unchanged.

```python
import openai
from dapplepot_sdk import DapplePot, DapplePotBlockedError, DapplePotSessionTerminatedError

# 1. Initialize + 2. Instrument
dp = DapplePot(sdk_key="dp_sk_...", agent_id="your-agent-id")
dp.instrument_openai()

client = openai.OpenAI(api_key="...")

# 3. Session-wrap + 5. Blocked + 6. Terminated
try:
    with dp.session(user_context_id="user_123", user_tenant_id="acme_corp"):
        try:
            resp = client.chat.completions.create(
                model="gpt-4o",
                messages=[{"role": "user", "content": user_input}],
            )
        except DapplePotBlockedError as exc:
            resp = "[Response blocked by security policy]"
except DapplePotSessionTerminatedError:
    print("Session terminated by security policy")

# 7. Shutdown
dp.shutdown()
```

Covers `chat.completions` sync, async, and streaming. Reminder: Responses/Assistants/Agents SDK are **not** covered — use `chat.completions`.

**How tools are traced.** Same pattern as Anthropic — off the wire:

- `tool_calls` on the assistant response → `tool_start`.
- The matching `role="tool"` message on the *next* `chat.completions.create()` call → `tool_end`.
- Add `"is_error": True` to the `role="tool"` message to emit `tool_error`. OpenAI's endpoint ignores the extra field; DapplePot reads it.

```python
{"role": "tool", "tool_call_id": "...", "content": "Weather API returned 503", "is_error": True}
```

---

## LangChain

LangChain has its own callback system, so DapplePot hooks in that way — no `instrument_*` call.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from dapplepot_sdk import DapplePot, DapplePotBlockedError, DapplePotSessionTerminatedError

# 1. Initialize (no separate instrument step for LangChain/LangGraph)
dp = DapplePot(sdk_key="dp_sk_...", agent_id="your-agent-id")

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"{city}: 18 °C"

prompt = ChatPromptTemplate.from_messages([("human", "{input}")])
chain  = prompt | ChatOpenAI(model="gpt-4o").bind_tools([get_weather])

# 2. Fresh callback handler per invocation (== one session), with identity
handler = dp.callback_handler(user_context_id="user_123", user_tenant_id="acme_corp")

# 5. Blocked → catch inside each pipeline step
# 6. Terminated → catch at the root, around invoke()
try:
    result = chain.invoke({"input": user_input}, config={"callbacks": [handler]})
except DapplePotSessionTerminatedError:
    print("Session terminated by security policy")

# 7. Shutdown
dp.shutdown()
```

**How tools are traced.** DapplePot listens on LangChain's own tool callbacks:

- Tools declared via the `@tool` decorator or by subclassing `BaseTool` fire LangChain's `on_tool_start` / `on_tool_end` / `on_tool_error` → DapplePot maps those to `tool_start` / `tool_end` / `tool_error`. No extra code.
- If you parse `tool_calls` off an `AIMessage` and invoke the tool yourself (bypassing `@tool` / `BaseTool`), LangChain does not fire tool callbacks. Wrap the manual dispatch in `dp.node(...)` (see the Nodes section) so the trace still shows it.

**Async pitfall.** In `ainvoke` / `astream`, LangChain does **not** auto-propagate callbacks to child runs. Always pass callbacks via the run config, not just top-level:

```python
result = await chain.ainvoke({"input": "..."}, config={"callbacks": [handler]})
```

Symptom if you miss this: trace shows `session_start` then silence.

**Catching Blocked inside a step** (so the chain continues with a fallback):

```python
def classify(input, config):
    try:
        return llm.invoke(..., config=config)
    except DapplePotBlockedError:
        return {**input, "intent": "NONE"}
```

---

## LangGraph

Same callback model as LangChain — one `DapplePot(...)`, one `callback_handler()` per graph invocation. Named graph nodes → `node_start` / `node_end` automatically.

```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode
from langchain_core.messages import HumanMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from dapplepot_sdk import DapplePot, DapplePotBlockedError, DapplePotSessionTerminatedError

# 1. Initialize
dp = DapplePot(sdk_key="dp_sk_...", agent_id="your-agent-id")

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"{city}: 18 °C"

llm = ChatOpenAI(model="gpt-4o").bind_tools([get_weather])

def agent_node(state):
    try:
        return {"messages": [llm.invoke(state["messages"])]}
    except DapplePotBlockedError as exc:
        # 5. Blocked → recover inside the node so the graph continues
        return {"messages": [f"[Blocked: {exc.signal}]"]}

graph = StateGraph(dict)
graph.add_node("agent", agent_node)
# handle_tool_errors=False lets tool exceptions reach DapplePot as tool_error
graph.add_node("tools", ToolNode([get_weather], handle_tool_errors=False))
graph.add_edge(START, "agent")
graph.add_edge("tools", "agent")
app = graph.compile()

# 2. Fresh handler per graph run, with identity
handler = dp.callback_handler(user_context_id="user_123", user_tenant_id="acme_corp")

# 6. Terminated → catch at the root
try:
    result = app.invoke(
        {"messages": [HumanMessage(content=user_input)]},
        config={"callbacks": [handler]},
    )
except DapplePotSessionTerminatedError:
    print("Session terminated by security policy")

# 7. Shutdown
dp.shutdown()
```

**How tools are traced.**

- Tools defined with the `@tool` decorator (or `BaseTool` subclasses) and wrapped in `ToolNode(...)` fire LangChain's tool callbacks → DapplePot emits `tool_start` / `tool_end` / `tool_error`. No extra code.
- **Gotcha — `handle_tool_errors=True` (the default)** swallows tool exceptions inside `ToolNode` before the callback fires, so `tool_error` never gets emitted. Set `handle_tool_errors=False` in `ToolNode(...)` if you want failed tools to surface in DapplePot.
- If you invoke a tool inside a custom graph node (bypassing `ToolNode`), wrap it in `dp.node(...)` (see Nodes) so the manual dispatch still shows in the trace.

**Async pitfall.** Same as LangChain — thread `callbacks=[handler]` through `config=` in `ainvoke` / `astream`.

---

## MCP (Model Context Protocol)

DapplePot does not patch any MCP client library directly. Whether an MCP call shows up in the trace depends on how the framework surfaces it:

- **LangChain / LangGraph** — MCP tools wrapped as `BaseTool` (typically via `langchain-mcp-adapters`) fire the standard tool callbacks, so DapplePot emits `tool_start` / `tool_end` normally.
- **OpenAI `chat.completions`** — MCP tools bridged as regular `tools=[…]` function definitions come back as normal `tool_calls`, auto-detected.
- **Anthropic client-side MCP** — schemas passed into `messages.create(tools=[…])` produce regular `tool_use` blocks, auto-detected.
- **Anthropic server-side `mcp_servers=`** — returns `mcp_tool_use` / `mcp_tool_result` blocks, which the current interceptor does not match. **Not traced today** — use client-side MCP if you want these calls on the trace.

---

## Nodes — `dp.node()` for named steps

Optional. Wrap any named step inside a session (or callback handler) to emit `node_start` on entry and `node_end` / `node_error` on exit. Use it to give the trace named structure — retrieval, classification, tool dispatch, etc.

```python
with dp.session(user_context_id="user_123"):
    with dp.node("retrieve", input=query):
        docs = vector_store.search(query)

    with dp.node("respond"):
        response = client.messages.create(...)
```

Behavior:

- Enter → `node_start` (with `node_name` and optional `input`).
- Clean exit → `node_end`.
- Exception escapes the block → `node_error` fires, then the exception re-raises. Catch *outside* the `with dp.node()` block but *inside* `dp.session()` to keep the session alive.

When to use it:

- **Anthropic / OpenAI**: whenever you want the trace to show named steps (`retrieve`, `classify`, `respond`), or when you invoke a tool manually and want it to appear in the trace.
- **LangChain / LangGraph**: only when you're doing something *outside* the framework's own callbacks (e.g. manual tool dispatch on an `AIMessage`). Named LCEL components and graph nodes already emit `node_start` / `node_end` automatically — do not double-wrap them.

---

## PII scrubbing — `RegexScrubber`

DapplePot can scrub sensitive text from event payloads before they leave your process. Pass a scrubber to the constructor:

```python
from dapplepot_sdk import DapplePot
from dapplepot_sdk.scrubbers import RegexScrubber

dp = DapplePot(
    sdk_key      = "dp_sk_...",
    agent_id     = "your-agent-id",
    pii_scrubber = RegexScrubber(patterns=["email", "ssn", "aws_key"]),
    redact_keys  = ["api_key", "password"],
)
```

- **`pii_scrubber=RegexScrubber(patterns=[...])`** — matches by regex and replaces each match with a tag (e.g. `[EMAIL]`).
- **`pii_scrubber=RegexScrubber()`** (no arg) — enables *all* built-in patterns.
- **`redact_keys=[...]`** — any payload dict key whose name matches is replaced whole with `"[REDACTED]"`, no regex involved. Use for known-sensitive keys like `api_key`, `password`, `ssn`.

Built-in pattern names: `email`, `phone`, `ssn`, `credit_card`, `uk_nino`, `iban`, `ip_address`, `aws_key`, `jwt`.

**Important — scrubbing runs *after* online security checks.** Live checks always see the original, unscrubbed payload; scrubbing only affects what gets sent to the ingest API. Redacting first would hide the exact content the checks look for.

Custom scrubber? Subclass `BaseScrubber` and implement `scrub(text: str) -> str`. Full API in the docs.

---

## Error handling summary

| Exception                         | Meaning                              | Catch where                         |
| --------------------------------- | ------------------------------------ | ----------------------------------- |
| `DapplePotBlockedError`           | Single LLM/tool call blocked         | Near the call site — fall back, keep the session alive. Attrs: `.signal`, `.reason`, `.session_id`. |
| `DapplePotSessionTerminatedError` | Whole session terminated by policy   | At the outer level — exit the conversation. Attrs: `.signal`, `.session_id`. |

## Shutdown

```python
dp.shutdown()
```

Flushes any buffered events. Call before the process exits. Safe to call more than once.

---

## Anything not in this file — go to the docs

This skill covers the core loop and the most common gotchas. For everything else, point the user at <https://docs.dapplepot.com>:

- Sampling (`sample_rate`), buffer tuning (`flush_interval_ms`, `flush_batch_size`), custom `ingest_url`.
- Custom scrubbers (`BaseScrubber` subclass) beyond the built-in `RegexScrubber`.
- Streaming detection internals.
- Framework wiring recipes (FastAPI, Flask, etc.).
- The full event vocabulary and per-check configuration.

## Self-check before finishing

1. `DapplePot(...)` constructed **once** at startup, not per request.
2. `instrument_*()` called **once**, before the first LLM call — never inside a request handler. For LangChain/LangGraph, no `instrument_*` call; use `callback_handler()` instead.
3. Every user-facing request either wrapped in `with dp.session(user_context_id=…, user_tenant_id=…)` (Anthropic/OpenAI) or has a `callback_handler(user_context_id=…, user_tenant_id=…)` threaded via `config={"callbacks": [...]}` (LangChain/LangGraph).
4. Identity kwargs are spelled **`user_context_id`** and **`user_tenant_id`** — not `tenant_context_id`, not `tenant_id`.
5. No `session_id=` argument in `dp.session()` examples.
6. No `tenant_id=` argument in `DapplePot(...)`.
7. Async LangChain / LangGraph: `callbacks=[handler]` passed via `config=`, not just top-level.
8. LangGraph tool errors need `ToolNode(..., handle_tool_errors=False)`.
9. Manual tool dispatch (outside `@tool` / `BaseTool` / `ToolNode`) wrapped in `dp.node(...)`.
10. `DapplePotBlockedError` caught near the call site; `DapplePotSessionTerminatedError` caught at the outer level.
11. `dp.shutdown()` at process exit.
12. Unsupported stack (Bedrock, Vertex, Responses API, Gemini, CrewAI, LiteLLM, …)? Call it out — do not pretend it works.
