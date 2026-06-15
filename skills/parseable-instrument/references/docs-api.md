# Parseable Docs And API Access

Use this reference when a task needs current Parseable documentation, ingestion details, API usage, dataset setup, or query/debug instructions.

## Documentation Workflow

Prefer the user's available web/documentation tools. If no specialized tool exists, fetch with `curl`.

1. Start with the full LLM index:

```bash
curl -s https://www.parseable.com/llms.txt
```

2. Search within the returned text for the task area:

- `OpenTelemetry Ingestion`
- `LLM and AI Agent Observability`
- `Agent Observability`
- `API Documentation`
- `Traces Explorer`
- `Logs Explorer`
- `Metrics Explorer`
- `Auto Instrumentation`

3. Fetch the specific docs page from the URL listed in `llms.txt` when more detail is needed.

## Ingestion Basics

Parseable accepts telemetry over HTTP and OpenTelemetry protocols. Confirm exact endpoint paths against current docs for the user's Parseable version before hard-coding them.

Common ingestion headers:

| Header | Purpose |
|---|---|
| `X-P-Stream` | Target dataset/stream |
| `Authorization` | Basic auth or API key |
| `X-P-Tenant-ID` | Tenant selection for multi-tenant deployments |

Use environment variables for these values:

```bash
PARSEABLE_ENDPOINT=https://your-parseable-instance.example.com
PARSEABLE_TRACES_STREAM=genai-traces
PARSEABLE_LOGS_STREAM=genai-logs
PARSEABLE_USERNAME=your-username
PARSEABLE_PASSWORD=your-password
PARSEABLE_TENANT_ID=
```

Compute Basic auth at runtime: `echo -n "${PARSEABLE_USERNAME}:${PARSEABLE_PASSWORD}" | base64`

If `PARSEABLE_USERNAME` or `PARSEABLE_PASSWORD` are not set, ask the user for each value before proceeding. Do not ask users to paste credentials into chat beyond this initial prompt — instruct them to set as environment variables, deployment secrets, or a local `.env` file.

## Dataset Planning

Choose dataset names that match the telemetry signal and lifecycle:

| Dataset | Use |
|---|---|
| `genai-traces` | Agent spans, LLM calls, tool calls, trace waterfalls |
| `genai-logs` | Message events, application logs, structured audit events |
| `genai-metrics` | Token counts, latency histograms, rate limits, cost aggregates |

For agent observability, keep traces and logs in separate datasets. Spans go to `genai-traces`; log records with message, tool, and reasoning content go to `genai-logs`.

## Query And UI Guidance

When validating telemetry, guide the user to:

- Traces Explorer for waterfall and span hierarchy.
- Logs Explorer for message events and application logs.
- Metrics Explorer for token/cost/latency series if metrics are exported.
- SQL Editor for custom checks over a time range.
- Agent Observability views when using supported AI-agent telemetry.

Use SQL for concrete checks only after confirming the dataset and column names from Parseable, because OpenTelemetry field names can vary by SDK and collector version.
