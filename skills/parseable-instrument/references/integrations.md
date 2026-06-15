# Parseable AI Integration Selection

Use this reference when choosing how to instrument a specific provider, framework, or runtime.

## Selection Rules

1. Prefer a Parseable-supported integration or an OpenTelemetry-native integration over hand-written spans.
2. Use framework callbacks/middleware when the application is built around a framework.
3. Add manual spans only for application-specific workflow steps, missing context, or unsupported libraries.
4. For streaming responses, capture first-token latency, total latency, output token count when available, and stream errors.
5. For tool-using agents, ensure tool spans are children of the relevant agent invocation or LLM decision span.

## Supported AI And Agent Surfaces

Parseable documentation lists AI-agent observability support for:

| Surface | Typical path |
|---|---|
| OpenAI | SDK integration or OpenTelemetry instrumentation |
| Anthropic | SDK integration, wrapper, or manual OTel around `messages.create` |
| LiteLLM | Callback/proxy instrumentation or OTel export |
| OpenRouter | HTTP/provider wrapper or OTel spans |
| vLLM | Server-side metrics/traces plus client spans |
| LangChain | Callback handler and/or OTel tracing |
| LlamaIndex | Callback handler and/or OTel tracing |
| AutoGen | Agent lifecycle hooks plus provider spans |
| CrewAI | Task/agent lifecycle spans plus provider spans |
| DSPy | Module/program spans plus provider spans |
| n8n | Workflow telemetry and HTTP/OpenTelemetry export |
| Other OTel frameworks | Export OTLP to the collector or directly to Parseable |

Fetch current docs for the exact setup before adding packages or changing imports.

## Baseline Requirements

Every integration must produce:

| Requirement | Check |
|---|---|
| Service name | `service.name` is meaningful and environment-specific when useful |
| Trace hierarchy | Agent invocation contains LLM/tool/retrieval child spans |
| Model names | Request and response model names captured where SDK exposes them |
| Token usage | Input/output token attributes captured where SDK exposes them |
| Errors | Exceptions, HTTP failures, rate limits, and timeouts marked as errors |
| Latency | LLM call duration and streaming first-token latency when relevant |
| Sessions | Conversation/session id attached when the app is multi-turn |
| User/tenant context | Stable hashed or non-sensitive identifiers attached when useful |
| Privacy | Prompt/completion content masked, sampled, or intentionally enabled |

## Framework Notes

### Python

Common dependencies:

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp-proto-http
```

Use the framework's integration package when available. If the framework does not expose complete LLM telemetry, combine framework callbacks with manual spans from `manual-otel.md`.

### TypeScript / JavaScript

Common dependencies:

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http @opentelemetry/exporter-logs-otlp-http
```

Initialize OpenTelemetry before importing provider clients or framework modules that need patching.

### Kubernetes

If the app runs on Kubernetes and the user wants low-code rollout, check current Parseable Auto Instrumentation docs. Confirm Kubernetes version, OpenTelemetry Operator, namespaces, and credential secret names before adding manifests.

## When To Avoid Manual Instrumentation

Avoid hand-written wrappers when a maintained integration already captures:

- Model name and token usage.
- Request/response metadata.
- Error status.
- Nested framework spans.
- Streaming metadata.

Manual wrappers are still appropriate for custom business workflow spans, privacy filtering, user/session attributes, and tool execution boundaries.
