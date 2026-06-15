# Manual OpenTelemetry Patterns

Use this reference for custom agent loops, missing spans, unsupported SDKs, or codebases where framework integration is insufficient.

## Span Model

Use this hierarchy unless the app has a better existing convention:

```text
invoke_agent
  chat <model>
    execute_tool <tool-name>
  retrieve <source>
  external_call <service>
```

Minimum span attributes:

| Span | Attributes |
|---|---|
| `invoke_agent` | `gen_ai.operation.name`, `gen_ai.agent.name`, `session.id`, `user.id` or hashed equivalent |
| `chat <model>` | `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`, token usage |
| `execute_tool <name>` | `gen_ai.operation.name`, `gen_ai.tool.name`, `gen_ai.tool.type` |
| retrieval/API spans | source/service name, query id, result count, status/error |

Use the semantic convention names already present in the codebase if they differ. Consistency inside a service is more important than renaming every attribute.

## Python Initialization

Create an instrumentation module when no existing OpenTelemetry setup exists:

```python
import os

from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor


def setup_observability(service_name: str | None = None) -> None:
    resource = Resource.create({
        "service.name": service_name or os.getenv("OTEL_SERVICE_NAME", "ai-agent"),
    })
    provider = TracerProvider(resource=resource)
    provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
    trace.set_tracer_provider(provider)


tracer = trace.get_tracer("ai-agent")
```

Set exporter configuration through environment variables:

```bash
OTEL_SERVICE_NAME=my-agent
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

Import and call `setup_observability()` before LLM clients are constructed.

## Python Agent And LLM Wrapping

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

from instrumentation import tracer


def run_agent(user_input: str, session_id: str | None = None):
    with tracer.start_as_current_span("invoke_agent", kind=trace.SpanKind.INTERNAL) as span:
        span.set_attribute("gen_ai.operation.name", "invoke_agent")
        if session_id:
            span.set_attribute("session.id", session_id)
        try:
            return _run_agent_logic(user_input)
        except Exception as exc:
            span.record_exception(exc)
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            raise


def call_openai(messages: list[dict], model: str):
    with tracer.start_as_current_span(f"chat {model}", kind=trace.SpanKind.CLIENT) as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.provider.name", "openai")
        span.set_attribute("gen_ai.request.model", model)
        try:
            response = client.chat.completions.create(model=model, messages=messages)
            span.set_attribute("gen_ai.response.model", response.model)
            if response.usage:
                span.set_attribute("gen_ai.usage.input_tokens", response.usage.prompt_tokens)
                span.set_attribute("gen_ai.usage.output_tokens", response.usage.completion_tokens)
            return response
        except Exception as exc:
            span.record_exception(exc)
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            raise
```

For Anthropic, use `response.usage.input_tokens` and `response.usage.output_tokens` when present.

## Python Tool Wrapping

```python
from opentelemetry.trace import Status, StatusCode


def execute_tool(tool_name: str, tool_input: dict):
    with tracer.start_as_current_span(f"execute_tool {tool_name}") as span:
        span.set_attribute("gen_ai.operation.name", "execute_tool")
        span.set_attribute("gen_ai.tool.name", tool_name)
        span.set_attribute("gen_ai.tool.type", "function")
        try:
            return dispatch_tool(tool_name, tool_input)
        except Exception as exc:
            span.record_exception(exc)
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            raise
```

Do not attach raw tool inputs if they may contain secrets, PII, documents, prompts, or credentials.

## TypeScript Initialization

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { trace } from '@opentelemetry/api';

const sdk = new NodeSDK({
  serviceName: process.env.OTEL_SERVICE_NAME ?? 'ai-agent',
  traceExporter: new OTLPTraceExporter(),
});

sdk.start();

export const tracer = trace.getTracer('ai-agent');
```

Import this module before constructing provider clients.

## TypeScript Agent And LLM Wrapping

```typescript
import { SpanKind, SpanStatusCode } from '@opentelemetry/api';
import { tracer } from './instrumentation';

export async function runAgent(userInput: string, sessionId?: string) {
  return tracer.startActiveSpan('invoke_agent', { kind: SpanKind.INTERNAL }, async (span) => {
    span.setAttribute('gen_ai.operation.name', 'invoke_agent');
    if (sessionId) span.setAttribute('session.id', sessionId);

    try {
      return await runAgentLogic(userInput);
    } catch (error) {
      span.recordException(error as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(error) });
      throw error;
    } finally {
      span.end();
    }
  });
}

export async function callModel(messages: unknown[], model: string) {
  return tracer.startActiveSpan(`chat ${model}`, { kind: SpanKind.CLIENT }, async (span) => {
    span.setAttribute('gen_ai.operation.name', 'chat');
    span.setAttribute('gen_ai.provider.name', 'openai');
    span.setAttribute('gen_ai.request.model', model);

    try {
      const response = await client.chat.completions.create({ model, messages });
      span.setAttribute('gen_ai.response.model', response.model);
      if (response.usage) {
        span.setAttribute('gen_ai.usage.input_tokens', response.usage.prompt_tokens);
        span.setAttribute('gen_ai.usage.output_tokens', response.usage.completion_tokens);
      }
      return response;
    } catch (error) {
      span.recordException(error as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(error) });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

## Message Events

Only emit message content when the user approves the privacy tradeoff. Prefer summaries, redacted content, or metadata for production.

If full content is approved, attach it as span events or structured logs with:

- `gen_ai.event.name`
- `gen_ai.message.role`
- `session.id`
- `user.id` or hashed equivalent
- message length and content hash

Avoid logging API keys, credentials, retrieved documents, payment data, health data, or personal identifiers.
