# dapplepot-sdk-skill

Drop-in agent instructions that teach any coding agent — Claude Code, Codex, Cursor, Windsurf, GitHub Copilot, and others — how to correctly integrate the [DapplePot Python SDK](https://pypi.org/project/dapplepot-sdk/) into your project.

One canonical playbook, packaged in each agent's native format. Ask your agent to install it, prompt normally, and it produces correct DapplePot integration code the first time.

## Why this exists

The DapplePot integration is small, but its correctness is in the details — the right identity kwargs on `dp.session()`, patching once at startup, the right catch site for `DapplePotBlockedError`, the right callback wiring for async LangGraph, and which surfaces are and aren't supported. This repo pins those details in the format each coding agent already reads, so you don't have to hand-hold the agent through the API surface.

## What's in the skill

Following steps, applied for each supported framework (**Anthropic, OpenAI, LangChain, LangGraph**), as end-to-end code snippets:

1. **Initialize** — construct `DapplePot(...)` once at startup.
2. **Instrument** — patch the framework in-place (`dp.instrument_anthropic()` / `dp.instrument_openai()`), or use `dp.callback_handler()` for LangChain/LangGraph.
3. **Session-wrap with identity** — every user request runs inside a session tagged with `user_context_id` and optionally `user_tenant_id`.
4. **Tool tracing** — how DapplePot picks up tool calls in each framework (off the wire for Anthropic/OpenAI; via `@tool` / `BaseTool` / `ToolNode` callbacks for LangChain/LangGraph).
5. **Handle `DapplePotBlockedError`** near the call site — fall back and keep the session alive.
6. **Handle `DapplePotSessionTerminatedError`** at the outer level — exit gracefully.
7. **`dp.node()`** — optional named trace steps that emit `node_start` / `node_end` / `node_error`.
8. **`RegexScrubber` + `redact_keys`** — optional PII scrubbing before events leave your process.
9. **`dp.shutdown()`** at process exit.

Plus an explicit **not-supported** list — OpenAI Responses/Assistants/Agents SDK, Anthropic Bedrock/Vertex, Gemini/Mistral/Cohere/Ollama/LiteLLM/OpenRouter, CrewAI/AutoGen/PydanticAI/LlamaIndex/smolagents, non-Python — so the agent doesn't hallucinate coverage.

**Not covered here** (see [docs.dapplepot.com](https://docs.dapplepot.com) for these):

- Sampling (`sample_rate`), buffer tuning, custom `ingest_url`.
- Custom `BaseScrubber` subclasses beyond the built-in `RegexScrubber`.
- Streaming detection internals.
- Framework wiring recipes (FastAPI lifespan, Flask `atexit`, etc.).
- Full event vocabulary and per-check dashboard configuration.

[`SKILL.md`](./SKILL.md) at the repo root is the source of truth. Every per-agent file is derived from it.

## Install the skill in your project

Give this line to your coding agent:

> Install the DapplePot skill from https://github.com/dapplepot/dapplepot-sdk-skill

The repo ships pre-formatted skill files for **Claude Code, Codex CLI, Cursor, Windsurf, and GitHub Copilot**. Your agent picks the right one, creates the directory if needed, and drops it in. Works for any coding agent with web-fetch + file-write.

Once installed, prompt normally — e.g. *"add DapplePot to this project, we're using Anthropic and LangGraph"* — and the agent will produce a correct integration.

### Other coding agents

Most modern coding agents — Aider, Amp, Continue, JetBrains AI Assistant, and newer CLI agents — read `AGENTS.md` at the repo root by default. The same install line works: your agent will fetch [`AGENTS.md`](./AGENTS.md) and drop it at your repo root.

## Contributing

Edit [`SKILL.md`](./SKILL.md) — it is the source of truth. Then mirror the change into the per-agent files. PRs welcome.

## Related

- **DapplePot SDK** source: <https://github.com/dapplepot/dapplepot-sdk>
- **DapplePot SDK** on PyPI: <https://pypi.org/project/dapplepot-sdk/>
- **DapplePot dashboard** (get your `sdk_key` and `agent_id`): <https://app.dapplepot.com>
- **DapplePot docs**: <https://docs.dapplepot.com>

## License

Apache License 2.0. See [LICENSE](./LICENSE).
