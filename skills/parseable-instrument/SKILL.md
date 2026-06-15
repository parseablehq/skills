---
name: parseable-instrument
description: Add, audit, or repair Parseable observability for LLM applications and AI agents. Use when users ask to instrument agents, trace LLM calls, export OpenTelemetry logs/metrics/traces to Parseable, set up collector config, choose Parseable-supported AI integrations, validate token/cost/error visibility, or query/debug telemetry in Parseable.
---

# Parseable Instrumentation

Use this skill to add production-grade Parseable observability to AI agents and LLM applications. Prefer current Parseable documentation, preserve application behavior, and choose the least invasive integration that captures complete telemetry.

Parseable agent observability uses a two-dataset architecture: OTel spans go to the Parseable `genai-traces` stream, and OTel log records go to the Parseable `genai-logs` stream. Keep metadata on spans, keep full conversation/tool/reasoning content in logs, and correlate both with OTel `trace_id` and `span_id`.

## Core Rules

1. Fetch current Parseable docs before implementing when the task depends on APIs, SDKs, ingestion paths, deployment, or integration details. Start with `https://www.parseable.com/llms.txt` or the specific docs page.
2. Prefer native framework or OpenTelemetry integrations when they capture model name, token usage, latency, errors, and nested operations correctly. Use manual spans and OTel logs when full message/tool/reasoning content, exact event names, or the two-dataset Parseable shape is missing.
3. Keep secrets out of source code. Put Parseable credentials in environment variables, deployment secrets, or collector configuration.
4. Preserve existing behavior. Wrap calls, do not replace business logic.
5. Record errors and rethrow. Do not swallow exceptions for tracing.
6. Treat prompts, completions, tool inputs, tool outputs, reasoning blocks, and user identifiers as sensitive unless the user explicitly wants full-content capture.
7. Do not send spans and logs to the same Parseable stream. Configure two collector exporters: traces to `genai-traces`, logs to `genai-logs`.
8. Do not put full prompt/completion/tool content on spans. Put content in OTel log bodies and repeat query-critical `gen_ai.*` fields as log attributes.
9. For manual instrumentation, provide language-specific code for the target runtime. `references/manual-otel.md` includes Python and TypeScript/Node patterns; adapt the same span/log model for Java, Go, .NET, or other runtimes.

## Workflow

### 1. Assess Current State

Inspect the project and report a short table before editing:

| Item | What to identify |
|---|---|
| Language/runtime | Python, TypeScript/JavaScript, Java, Go, .NET, n8n, or other |
| LLM providers | OpenAI, Anthropic, LiteLLM, OpenRouter, vLLM, Bedrock, VertexAI, local models, or other |
| Agent framework | LangChain, LlamaIndex, AutoGen, CrewAI, DSPy, custom loop, workflow runner, or none |
| Entry points | Main request handler, CLI command, worker, API route, or workflow start |
| LLM calls | Chat/completion/response calls and streaming paths |
| Tool calls | Function/tool dispatch, retrieval, web/API calls, code execution, or workflow nodes |
| Existing telemetry | OpenTelemetry SDK, collector, auto-instrumentation, logging, metrics, tracing |
| Deployment | Local, Docker Compose, Kubernetes, serverless, managed platform |

### 2. Choose The Instrumentation Path

Use the path that fits the codebase:

| Situation | Path |
|---|---|
| Supported AI framework or provider SDK already exists | Read `references/integrations.md` and use the native integration or OTel bridge, then verify it preserves the `genai-traces`/`genai-logs` split |
| Custom agent loop, missing spans/logs, unsupported framework, or missing message/tool/reasoning content | Read `references/manual-otel.md` and add targeted spans plus OTel log records in the target language |
| Need to ship telemetry to Parseable | Read `references/collector.md` and configure separate traces/logs exporters |
| Need docs, API, dataset, or query guidance | Read `references/docs-api.md` |
| Need to verify or debug the result | Read `references/validation-analysis.md` |

### 3. Implement

Make the smallest set of edits that gives useful agent observability:

- One `invoke-agent` root span per agent/user invocation.
- One `chat {model}` child span per LLM call.
- One `execute_tool {tool.name}` child span per tool/function execution.
- Span attributes for provider, model, operation, token usage, response ids, finish reasons, tool call ids, session/user/tenant tags, and error status where available.
- OTel log records for `gen_ai.user.message`, `gen_ai.system.message`, `gen_ai.assistant.message`, `gen_ai.tool.message`, `gen_ai.choice`, `gen_ai.tool.call`, `gen_ai.thinking`, `gen_ai.tool.input`, `gen_ai.tool.output`, and `gen_ai.agent.finish`.
- Collector, environment, and launch docs that route spans to `genai-traces` and logs to `genai-logs`.

### 4. Validate

Run project tests when available. Then verify telemetry with the checklist in `references/validation-analysis.md`. If validation requires a live Parseable instance and credentials are not available, leave exact commands and expected checks without asking the user to paste secrets into chat.

## Done Checklist

Confirm what changed:

- Dependencies or integration packages added.
- Initialization/import order handled.
- Agent entry point traced.
- LLM calls traced with model and token usage where available.
- Tool calls traced.
- Message, choice, tool-call, tool I/O, reasoning, and agent-finish logs emitted inside active span context.
- Errors recorded and rethrown.
- Sensitive data handling documented.
- Parseable export configured with separate traces/logs streams.
- README or runbook updated.
- Validation performed or explicitly blocked by missing credentials/runtime.
