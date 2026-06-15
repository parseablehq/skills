---
name: parseable-instrument
description: Add, audit, or repair Parseable observability for LLM applications and AI agents. Use when users ask to instrument agents, trace LLM calls, export OpenTelemetry logs/metrics/traces to Parseable, set up collector config, choose Parseable-supported AI integrations, validate token/cost/error visibility, or query/debug telemetry in Parseable.
---

# Parseable Instrumentation

Use this skill to add production-grade Parseable observability to AI agents and LLM applications. Prefer current Parseable documentation, preserve application behavior, and choose the least invasive integration that captures complete telemetry.

## Core Rules

1. Fetch current Parseable docs before implementing when the task depends on APIs, SDKs, ingestion paths, deployment, or integration details. Start with `https://www.parseable.com/llms.txt` or the specific docs page.
2. Prefer native framework or OpenTelemetry integrations when they capture model name, token usage, latency, errors, and nested operations correctly. Use manual spans only for missing context or unsupported frameworks.
3. Keep secrets out of source code. Put Parseable credentials in environment variables, deployment secrets, or collector configuration.
4. Preserve existing behavior. Wrap calls, do not replace business logic.
5. Record errors and rethrow. Do not swallow exceptions for tracing.
6. Treat prompts, completions, tool inputs, and user identifiers as sensitive unless the user explicitly wants full-content capture.

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
| Supported AI framework or provider SDK already exists | Read `references/integrations.md` and use the native integration or OTel bridge |
| Custom agent loop, missing spans, or unsupported framework | Read `references/manual-otel.md` and add targeted spans/log events |
| Need to ship telemetry to Parseable | Read `references/collector.md` |
| Need docs, API, dataset, or query guidance | Read `references/docs-api.md` |
| Need to verify or debug the result | Read `references/validation-analysis.md` |

### 3. Implement

Make the smallest set of edits that gives useful traces:

- One root span per agent/user invocation.
- Child spans for LLM calls, tool executions, retrieval, external API calls, and long-running steps.
- Attributes for provider, model, operation, token usage, request ids, session ids, user ids, tenant/customer tags, and error status where available.
- Logs or span events for messages only after privacy review.
- Collector, environment, and launch docs when telemetry must leave the process.

### 4. Validate

Run project tests when available. Then verify telemetry with the checklist in `references/validation-analysis.md`. If validation requires a live Parseable instance and credentials are not available, leave exact commands and expected checks without asking the user to paste secrets into chat.

## Done Checklist

Confirm what changed:

- Dependencies or integration packages added.
- Initialization/import order handled.
- Agent entry point traced.
- LLM calls traced with model and token usage where available.
- Tool calls traced.
- Errors recorded and rethrown.
- Sensitive data handling documented.
- Parseable export configured.
- README or runbook updated.
- Validation performed or explicitly blocked by missing credentials/runtime.
