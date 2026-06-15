# Manual OpenTelemetry Instrumentation

Use this reference for custom agent loops, unsupported frameworks, or codebases where native integrations do not produce Parseable's agent observability shape.

Parseable's manual model is intentionally small:

- OTel spans describe agent, chat, and tool operations.
- OTel logs carry full message, tool, and reasoning content.
- Parseable stores them in two datasets: `genai-traces` for spans and `genai-logs` for log records.
- Correlation comes from OTel context: every log emitted inside a span has the matching `trace_id` and `span_id`.

## Architecture

```text
Agent application
  |
  |-- OTel traces -> Collector traces pipeline -> Parseable stream: genai-traces
  |
  `-- OTel logs   -> Collector logs pipeline   -> Parseable stream: genai-logs
```

Use one OTLP receiver in the collector, but route traces and logs through separate pipelines and exporters. Do not send both signals to one Parseable stream.

## Span Model

Instrument only three span types unless the target application already has additional useful spans:

| Span | Span name | SpanKind | Parent | One per |
|---|---|---|---|---|
| Agent invocation | `invoke-agent` | `CLIENT` | Current request/root context | Agent run |
| Chat call | `chat {model}` | `CLIENT` | `invoke-agent` | LLM request |
| Tool execution | `execute_tool {tool.name}` | `INTERNAL` | `invoke-agent` | Tool/function execution |

All `chat {model}` and `execute_tool {tool.name}` spans should be created while the `invoke-agent` span is current. This creates one trace waterfall for the complete agent path.

## Signal Placement

| Data | Signal | Parseable stream | Reason |
|---|---|---|---|
| Run metadata, model, provider, token totals, finish status | Span attributes | `genai-traces` | Dashboards, aggregation, status, cost |
| Per-call metadata, latency, tokens, response id, finish reasons | Span attributes | `genai-traces` | Per-model and per-provider analysis |
| Tool name, type, call id, success/error | Span attributes | `genai-traces` | Tool latency and failure analysis |
| System/user/assistant/tool messages | Log records | `genai-logs` | Conversation reconstruction |
| Tool call arguments and tool outputs | Log records | `genai-logs` | Debugging and analytics |
| Thinking/reasoning blocks when exposed by provider | Log records | `genai-logs` | Reasoning analysis |

Put query-critical `gen_ai.*` fields on both the span and the log record. Logs should be queryable without joining back to spans.

## Python Bootstrap

### Packages

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

### Environment

```bash
export OTEL_SERVICE_NAME=my-genai-agent
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Required for manual instrumentation when auto-instrumentors would create duplicate/noisy spans.
export OTEL_PYTHON_DISABLED_INSTRUMENTATIONS=openai,openai_v2,requests,urllib3,urllib,httpx,aiohttp-client,aiohttp-server,jinja2,grpc_client,grpc_aio_client,grpc_server,grpc_aio_server,sqlite3,click,asyncio,threading,tortoiseorm,logging
```

### Launch

Prefer the CLI wrapper:

```bash
opentelemetry-instrument \
  --traces_exporter otlp \
  --logs_exporter otlp \
  --metrics_exporter none \
  python my_agent.py
```

The CLI configures `TracerProvider`, `LoggerProvider`, OTLP span export, and OTLP log export. When using it, do not call `trace.set_tracer_provider()` or `set_logger_provider()` in application code.

### Manual Provider Setup

Use this only when the CLI cannot wrap the process:

```python
from opentelemetry import trace
from opentelemetry._logs import set_logger_provider
from opentelemetry.exporter.otlp.proto.http._log_exporter import OTLPLogExporter
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk._logs import LoggerProvider
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

tracer_provider = TracerProvider()
tracer_provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(tracer_provider)

logger_provider = LoggerProvider()
logger_provider.add_log_record_processor(BatchLogRecordProcessor(OTLPLogExporter()))
set_logger_provider(logger_provider)
```

### Module Setup

Create tracers and loggers from the already configured providers:

```python
import json

from opentelemetry import trace
from opentelemetry._logs import SeverityNumber, get_logger_provider

_tracer = trace.get_tracer("my-agent", "1.0.0")
_otel_logger = get_logger_provider().get_logger("my-agent", "1.0.0")
```

Use matching scope names for tracer and logger in a module. Example: `my-agent.llm` for both LLM spans and LLM logs.

## TypeScript / Node Bootstrap

Use this path for TypeScript and JavaScript agents, Next.js API routes, Node workers, CLIs, and custom loops.

### Packages

```bash
npm install @opentelemetry/api @opentelemetry/api-logs @opentelemetry/sdk-node @opentelemetry/sdk-logs @opentelemetry/exporter-trace-otlp-http @opentelemetry/exporter-logs-otlp-http
```

### Environment

```bash
export OTEL_SERVICE_NAME=my-genai-agent
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

### SDK Setup

Initialize telemetry before importing or constructing LLM clients:

```typescript
// telemetry.ts
import { diag, DiagConsoleLogger, DiagLogLevel } from '@opentelemetry/api';
import { logs } from '@opentelemetry/api-logs';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-http';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { BatchLogRecordProcessor, LoggerProvider } from '@opentelemetry/sdk-logs';
import { NodeSDK } from '@opentelemetry/sdk-node';

diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.ERROR);

const endpoint = process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318';

const loggerProvider = new LoggerProvider();
loggerProvider.addLogRecordProcessor(
  new BatchLogRecordProcessor(new OTLPLogExporter({ url: `${endpoint}/v1/logs` })),
);
logs.setGlobalLoggerProvider(loggerProvider);

export const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: `${endpoint}/v1/traces` }),
});

export async function startTelemetry() {
  await sdk.start();
}
```

Call it at process startup:

```typescript
// main.ts
import { startTelemetry } from './telemetry';

await startTelemetry();
const { runAgent } = await import('./agent');

await runAgent({
  model: 'gpt-4o',
  provider: 'openai',
  problem: process.argv.slice(2).join(' '),
});
```

For CommonJS or frameworks without top-level `await`, start telemetry in the earliest server/bootstrap hook.

### Module Setup

```typescript
import { SpanKind, SpanStatusCode, trace } from '@opentelemetry/api';
import { logs, SeverityNumber } from '@opentelemetry/api-logs';

const tracer = trace.getTracer('my-agent', '1.0.0');
const otelLogger = logs.getLogger('my-agent', '1.0.0');

type Attributes = Record<string, string | number | boolean | string[] | number[] | boolean[]>;

function emitGenAIEvent(args: {
  body: string;
  eventName: string;
  operation: 'invoke_agent' | 'chat' | 'execute_tool';
  attributes?: Attributes;
  severityNumber?: SeverityNumber;
}) {
  otelLogger.emit({
    body: args.body,
    eventName: args.eventName,
    severityNumber: args.severityNumber ?? SeverityNumber.INFO,
    attributes: {
      'gen_ai.operation.name': args.operation,
      'gen_ai.event.name': args.eventName,
      ...(args.attributes ?? {}),
    },
  });
}
```

If the installed JS logs API lacks `eventName`, keep `gen_ai.event.name` as a log attribute. Parseable can still query the event name as a flattened field.

## TypeScript / Node Instrumentation

These examples mirror the Python model exactly: spans to `genai-traces`, log records to `genai-logs`, correlation by active OTel context.

### Agent Invocation

```typescript
export async function runAgent(input: { model: string; provider: string; problem: string }) {
  return tracer.startActiveSpan('invoke-agent', { kind: SpanKind.CLIENT }, async (span) => {
    span.setAttribute('gen_ai.operation.name', 'invoke_agent');
    span.setAttribute('gen_ai.agent.name', 'my-agent');
    span.setAttribute('gen_ai.provider.name', input.provider);
    span.setAttribute('gen_ai.request.model', input.model);

    emitGenAIEvent({
      body: input.problem,
      eventName: 'gen_ai.user.message',
      operation: 'invoke_agent',
      attributes: {
        'gen_ai.agent.name': 'my-agent',
        'gen_ai.provider.name': input.provider,
        'gen_ai.request.model': input.model,
        role: 'user',
      },
    });

    let inputTokens = 0;
    let outputTokens = 0;

    try {
      const result = await runAgentLoop({
        ...input,
        onUsage: (usage) => {
          inputTokens += usage?.prompt_tokens ?? usage?.input_tokens ?? 0;
          outputTokens += usage?.completion_tokens ?? usage?.output_tokens ?? 0;
        },
      });

      span.setAttribute('gen_ai.usage.input_tokens', inputTokens);
      span.setAttribute('gen_ai.usage.output_tokens', outputTokens);
      span.setAttribute('gen_ai.response.finish_reasons', JSON.stringify(['stop']));

      emitGenAIEvent({
        body: JSON.stringify({
          exit_status: 'success',
          total_input_tokens: inputTokens,
          total_output_tokens: outputTokens,
        }),
        eventName: 'gen_ai.agent.finish',
        operation: 'invoke_agent',
        attributes: {
          'gen_ai.agent.name': 'my-agent',
          'gen_ai.provider.name': input.provider,
          'gen_ai.request.model': input.model,
          'gen_ai.usage.input_tokens': inputTokens,
          'gen_ai.usage.output_tokens': outputTokens,
        },
      });

      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.recordException(error as Error);
      span.setAttribute('error.type', error instanceof Error ? error.name : 'Error');
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(error) });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Chat Call

```typescript
export async function callLLM(params: {
  model: string;
  provider: string;
  messages: Array<{ role: string; content: unknown }>;
  temperature?: number;
}) {
  return tracer.startActiveSpan(`chat ${params.model}`, { kind: SpanKind.CLIENT }, async (span) => {
    span.setAttribute('gen_ai.operation.name', 'chat');
    span.setAttribute('gen_ai.provider.name', params.provider);
    span.setAttribute('gen_ai.request.model', params.model);
    if (params.temperature !== undefined) {
      span.setAttribute('gen_ai.request.temperature', params.temperature);
    }

    for (const message of params.messages) {
      const body = typeof message.content === 'string'
        ? message.content
        : JSON.stringify(message.content);
      emitGenAIEvent({
        body,
        eventName: `gen_ai.${message.role}.message`,
        operation: 'chat',
        attributes: {
          'gen_ai.provider.name': params.provider,
          'gen_ai.request.model': params.model,
          role: message.role,
        },
      });
    }

    try {
      const response = await client.chat.completions.create({
        model: params.model,
        messages: params.messages as any,
        temperature: params.temperature,
      });

      if (response.model) span.setAttribute('gen_ai.response.model', response.model);
      if (response.id) span.setAttribute('gen_ai.response.id', response.id);
      if (response.usage) {
        span.setAttribute('gen_ai.usage.input_tokens', response.usage.prompt_tokens ?? 0);
        span.setAttribute('gen_ai.usage.output_tokens', response.usage.completion_tokens ?? 0);
      }

      const finishReasons: string[] = [];
      const toolCallIds: string[] = [];

      response.choices.forEach((choice, index) => {
        const finishReason = choice.finish_reason ?? '';
        if (finishReason) finishReasons.push(finishReason);

        emitGenAIEvent({
          body: choice.message.content ?? '',
          eventName: 'gen_ai.choice',
          operation: 'chat',
          attributes: {
            'gen_ai.provider.name': params.provider,
            'gen_ai.request.model': params.model,
            index,
            finish_reason: finishReason,
          },
        });

        for (const toolCall of choice.message.tool_calls ?? []) {
          const toolName = toolCall.function?.name ?? '';
          if (toolCall.id) toolCallIds.push(toolCall.id);
          emitGenAIEvent({
            body: JSON.stringify(toolCall.function ?? toolCall),
            eventName: 'gen_ai.tool.call',
            operation: 'chat',
            attributes: {
              'gen_ai.provider.name': params.provider,
              'gen_ai.request.model': params.model,
              'gen_ai.tool.name': toolName,
              'gen_ai.tool.call.id': toolCall.id ?? '',
            },
          });
        }

        const reasoning = (choice.message as any).reasoning_content;
        if (reasoning) {
          emitGenAIEvent({
            body: reasoning,
            eventName: 'gen_ai.thinking',
            operation: 'chat',
            attributes: {
              'gen_ai.provider.name': params.provider,
              'gen_ai.request.model': params.model,
            },
          });
        }
      });

      if (finishReasons.length) {
        span.setAttribute('gen_ai.response.finish_reasons', JSON.stringify(finishReasons));
      }
      if (toolCallIds.length) {
        span.setAttribute(
          'gen_ai.tool.call.id',
          toolCallIds.length === 1 ? toolCallIds[0] : JSON.stringify(toolCallIds),
        );
      }

      span.setStatus({ code: SpanStatusCode.OK });
      return response;
    } catch (error) {
      span.recordException(error as Error);
      span.setAttribute('error.type', error instanceof Error ? error.name : 'Error');
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(error) });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Tool Execution

```typescript
export async function executeTool(
  toolName: string,
  toolInput: unknown,
  toolCallId?: string,
) {
  return tracer.startActiveSpan(`execute_tool ${toolName}`, { kind: SpanKind.INTERNAL }, async (span) => {
    span.setAttribute('gen_ai.operation.name', 'execute_tool');
    span.setAttribute('gen_ai.tool.name', toolName);
    span.setAttribute('gen_ai.tool.type', 'function');
    if (toolCallId) span.setAttribute('gen_ai.tool.call.id', toolCallId);

    const attrs = {
      'gen_ai.tool.name': toolName,
      'gen_ai.tool.type': 'function',
      'gen_ai.tool.call.id': toolCallId ?? '',
    };

    emitGenAIEvent({
      body: JSON.stringify(toolInput),
      eventName: 'gen_ai.tool.input',
      operation: 'execute_tool',
      attributes: attrs,
    });

    try {
      const result = await dispatchTool(toolName, toolInput);
      emitGenAIEvent({
        body: typeof result === 'string' ? result : JSON.stringify(result),
        eventName: 'gen_ai.tool.output',
        operation: 'execute_tool',
        attributes: attrs,
      });
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.recordException(error as Error);
      span.setAttribute('error.type', error instanceof Error ? error.name : 'Error');
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(error) });
      emitGenAIEvent({
        body: String(error),
        eventName: 'gen_ai.tool.output',
        operation: 'execute_tool',
        severityNumber: SeverityNumber.ERROR,
        attributes: attrs,
      });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

## Python Helper: Emit GenAI Logs

Use a small helper so every log includes `gen_ai.event.name` and operation attributes:

```python
def emit_genai_log(
    *,
    body: str,
    event_name: str,
    operation: str,
    severity_number=SeverityNumber.INFO,
    attributes: dict | None = None,
) -> None:
    attrs = {
        "gen_ai.operation.name": operation,
        "gen_ai.event.name": event_name,
    }
    if attributes:
        attrs.update(attributes)

    _otel_logger.emit(
        body=body,
        severity_number=severity_number,
        event_name=event_name,
        attributes=attrs,
    )
```

Emit logs only while the matching span is current. Logs emitted outside span context will not have useful `trace_id` and `span_id`.

## Python Agent Invocation: `invoke-agent`

Create one parent span per agent/user invocation.

Required span attributes:

- `gen_ai.operation.name`: `invoke_agent`
- `gen_ai.agent.name`
- `gen_ai.provider.name`
- `gen_ai.request.model`

Completion span attributes:

- `gen_ai.usage.input_tokens`: total input tokens across chat calls
- `gen_ai.usage.output_tokens`: total output tokens across chat calls
- `gen_ai.response.finish_reasons`: JSON array string or backend-compatible array

Required logs:

- `gen_ai.user.message`: task, prompt, issue, or problem statement that started the run
- `gen_ai.agent.finish`: final status, total tokens, result metadata

```python
def run_agent(model_name: str, provider: str, problem: str):
    with _tracer.start_as_current_span("invoke-agent", kind=trace.SpanKind.CLIENT) as span:
        span.set_attribute("gen_ai.operation.name", "invoke_agent")
        span.set_attribute("gen_ai.agent.name", "my-agent")
        span.set_attribute("gen_ai.provider.name", provider)
        span.set_attribute("gen_ai.request.model", model_name)

        emit_genai_log(
            body=problem,
            event_name="gen_ai.user.message",
            operation="invoke_agent",
            attributes={
                "gen_ai.agent.name": "my-agent",
                "gen_ai.provider.name": provider,
                "gen_ai.request.model": model_name,
                "role": "user",
            },
        )

        total_input_tokens = 0
        total_output_tokens = 0
        finish_reasons = []

        def record_usage(usage) -> None:
            nonlocal total_input_tokens, total_output_tokens
            if usage:
                total_input_tokens += getattr(usage, "prompt_tokens", None) or getattr(usage, "input_tokens", 0) or 0
                total_output_tokens += getattr(usage, "completion_tokens", None) or getattr(usage, "output_tokens", 0) or 0

        try:
            result = run_agent_loop(
                model_name=model_name,
                provider=provider,
                on_usage=record_usage,
            )
        except Exception as exc:
            span.set_status(trace.StatusCode.ERROR, str(exc))
            span.set_attribute("error.type", type(exc).__name__)
            raise

        span.set_attribute("gen_ai.usage.input_tokens", total_input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", total_output_tokens)
        span.set_attribute("gen_ai.response.finish_reasons", json.dumps(finish_reasons or ["stop"]))

        emit_genai_log(
            body=json.dumps({
                "exit_status": "success",
                "total_input_tokens": total_input_tokens,
                "total_output_tokens": total_output_tokens,
            }),
            event_name="gen_ai.agent.finish",
            operation="invoke_agent",
            attributes={
                "gen_ai.agent.name": "my-agent",
                "gen_ai.provider.name": provider,
                "gen_ai.request.model": model_name,
                "gen_ai.usage.input_tokens": total_input_tokens,
                "gen_ai.usage.output_tokens": total_output_tokens,
            },
        )
        return result
```

The sample uses placeholders for the agent loop and usage aggregation. In real code, aggregate usage from each `chat {model}` span response.

## Python Chat Call: `chat {model}`

Create one child span for each LLM request.

Required span attributes:

- `gen_ai.operation.name`: `chat`
- `gen_ai.provider.name`
- `gen_ai.request.model`
- `gen_ai.request.temperature` when used
- `gen_ai.request.top_p` when used
- `gen_ai.request.max_tokens` when used
- `gen_ai.response.model` on success
- `gen_ai.response.id` on success
- `gen_ai.response.finish_reasons` on success
- `gen_ai.usage.input_tokens` on success
- `gen_ai.usage.output_tokens` on success
- `error.type` on error

Required logs inside the active chat span:

- `gen_ai.system.message`
- `gen_ai.user.message`
- `gen_ai.assistant.message`
- `gen_ai.tool.message`
- `gen_ai.choice`
- `gen_ai.tool.call`
- `gen_ai.thinking` when the provider exposes reasoning or thinking blocks

```python
def call_llm(model: str, messages: list[dict], provider: str = "openai", temperature: float = 0.0):
    with _tracer.start_as_current_span(f"chat {model}", kind=trace.SpanKind.CLIENT) as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.provider.name", provider)
        span.set_attribute("gen_ai.request.model", model)
        if temperature is not None:
            span.set_attribute("gen_ai.request.temperature", temperature)

        for msg in messages:
            role = msg.get("role", "user")
            event_name = f"gen_ai.{role}.message"
            content = msg.get("content", "")
            body = content if isinstance(content, str) else json.dumps(content)
            emit_genai_log(
                body=body,
                event_name=event_name,
                operation="chat",
                attributes={
                    "gen_ai.provider.name": provider,
                    "gen_ai.request.model": model,
                    "role": role,
                },
            )

        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=temperature,
            )
        except Exception as exc:
            span.set_status(trace.StatusCode.ERROR, str(exc))
            span.set_attribute("error.type", type(exc).__name__)
            raise

        if getattr(response, "model", None):
            span.set_attribute("gen_ai.response.model", response.model)
        if getattr(response, "id", None):
            span.set_attribute("gen_ai.response.id", response.id)

        usage = getattr(response, "usage", None)
        if usage:
            # OpenAI-compatible fields. Adapt for Anthropic and wrappers.
            input_tokens = getattr(usage, "prompt_tokens", None) or getattr(usage, "input_tokens", None)
            output_tokens = getattr(usage, "completion_tokens", None) or getattr(usage, "output_tokens", None)
            if input_tokens is not None:
                span.set_attribute("gen_ai.usage.input_tokens", input_tokens)
            if output_tokens is not None:
                span.set_attribute("gen_ai.usage.output_tokens", output_tokens)

        finish_reasons = []
        for index, choice in enumerate(response.choices):
            message = choice.message
            finish_reason = choice.finish_reason or ""
            if finish_reason:
                finish_reasons.append(finish_reason)

            emit_genai_log(
                body=getattr(message, "content", None) or "",
                event_name="gen_ai.choice",
                operation="chat",
                attributes={
                    "gen_ai.provider.name": provider,
                    "gen_ai.request.model": model,
                    "index": index,
                    "finish_reason": finish_reason,
                },
            )

            tool_calls = getattr(message, "tool_calls", None) or []
            tool_call_ids = []
            for tool_call in tool_calls:
                tool_call_dict = tool_call.model_dump() if hasattr(tool_call, "model_dump") else tool_call
                tool_call_id = tool_call_dict.get("id", "")
                function = tool_call_dict.get("function", tool_call_dict)
                tool_name = function.get("name", "")
                if tool_call_id:
                    tool_call_ids.append(tool_call_id)
                emit_genai_log(
                    body=json.dumps(function),
                    event_name="gen_ai.tool.call",
                    operation="chat",
                    attributes={
                        "gen_ai.provider.name": provider,
                        "gen_ai.request.model": model,
                        "gen_ai.tool.name": tool_name,
                        "gen_ai.tool.call.id": tool_call_id,
                    },
                )

            if tool_call_ids:
                span.set_attribute(
                    "gen_ai.tool.call.id",
                    tool_call_ids[0] if len(tool_call_ids) == 1 else json.dumps(tool_call_ids),
                )

            thinking_blocks = getattr(message, "thinking_blocks", None) or []
            reasoning_content = getattr(message, "reasoning_content", None)
            for block in thinking_blocks:
                thinking_text = block.get("thinking", "") if isinstance(block, dict) else str(block)
                emit_genai_log(
                    body=thinking_text,
                    event_name="gen_ai.thinking",
                    operation="chat",
                    attributes={
                        "gen_ai.provider.name": provider,
                        "gen_ai.request.model": model,
                    },
                )
            if reasoning_content:
                emit_genai_log(
                    body=reasoning_content,
                    event_name="gen_ai.thinking",
                    operation="chat",
                    attributes={
                        "gen_ai.provider.name": provider,
                        "gen_ai.request.model": model,
                    },
                )

        if finish_reasons:
            span.set_attribute("gen_ai.response.finish_reasons", json.dumps(finish_reasons))
        span.set_status(trace.StatusCode.OK)
        return response
```

Use provider usage fields. Do not compute output tokens from assistant text because tool-call responses can have empty content and non-zero completion tokens.

## Python Tool Execution: `execute_tool {tool.name}`

Create one child span for each tool execution.

Required span attributes:

- `gen_ai.operation.name`: `execute_tool`
- `gen_ai.tool.name`
- `gen_ai.tool.type`
- `gen_ai.tool.call.id` when available
- `error.type` on error

Required logs inside the active tool span:

- `gen_ai.tool.input`: command, function arguments, request body, or tool input
- `gen_ai.tool.output`: observation, result, response body, or failure output

```python
def execute_tool(tool_name: str, tool_input: dict, tool_call_id: str | None = None):
    with _tracer.start_as_current_span(f"execute_tool {tool_name}", kind=trace.SpanKind.INTERNAL) as span:
        span.set_attribute("gen_ai.operation.name", "execute_tool")
        span.set_attribute("gen_ai.tool.name", tool_name)
        span.set_attribute("gen_ai.tool.type", "function")
        if tool_call_id:
            span.set_attribute("gen_ai.tool.call.id", tool_call_id)

        attrs = {
            "gen_ai.tool.name": tool_name,
            "gen_ai.tool.type": "function",
            "gen_ai.tool.call.id": tool_call_id or "",
        }
        emit_genai_log(
            body=json.dumps(tool_input),
            event_name="gen_ai.tool.input",
            operation="execute_tool",
            attributes=attrs,
        )

        try:
            result = dispatch_tool(tool_name, tool_input)
        except Exception as exc:
            span.set_status(trace.StatusCode.ERROR, str(exc))
            span.set_attribute("error.type", type(exc).__name__)
            emit_genai_log(
                body=str(exc),
                event_name="gen_ai.tool.output",
                operation="execute_tool",
                severity_number=SeverityNumber.ERROR,
                attributes=attrs,
            )
            raise

        emit_genai_log(
            body=result if isinstance(result, str) else json.dumps(result),
            event_name="gen_ai.tool.output",
            operation="execute_tool",
            attributes=attrs,
        )
        span.set_status(trace.StatusCode.OK)
        return result
```

Use the same `gen_ai.tool.call.id` on chat logs and tool spans when function calling provides an ID.

## Event Reference

| Span | Log `event_name` | Body |
|---|---|---|
| `invoke-agent` | `gen_ai.user.message` | Problem statement that started the run |
| `invoke-agent` | `gen_ai.agent.finish` | JSON summary: status, total tokens, result metadata |
| `chat {model}` | `gen_ai.system.message` | Full system prompt |
| `chat {model}` | `gen_ai.user.message` | Full user message |
| `chat {model}` | `gen_ai.assistant.message` | Prior assistant response |
| `chat {model}` | `gen_ai.tool.message` | Tool result message included in chat input |
| `chat {model}` | `gen_ai.choice` | Full assistant response text for a choice |
| `chat {model}` | `gen_ai.tool.call` | JSON tool call with name and arguments |
| `chat {model}` | `gen_ai.thinking` | Full provider-exposed reasoning/thinking text |
| `execute_tool {tool.name}` | `gen_ai.tool.input` | Tool input, arguments, command, or request |
| `execute_tool {tool.name}` | `gen_ai.tool.output` | Tool result, observation, response, or error |

Every log record must include:

- `gen_ai.operation.name`
- `gen_ai.event.name`
- Any relevant provider/model/tool/token/error fields needed for direct queries in `genai-logs`

## Span Attribute Reference

### `invoke-agent`

| Attribute | Required | Notes |
|---|---|---|
| `gen_ai.operation.name` | Yes | Always `invoke_agent` |
| `gen_ai.agent.name` | Yes | Agent identifier |
| `gen_ai.provider.name` | Yes | Primary LLM provider |
| `gen_ai.request.model` | Yes | Primary model |
| `gen_ai.usage.input_tokens` | On completion | Total across chat calls |
| `gen_ai.usage.output_tokens` | On completion | Total across chat calls |
| `gen_ai.response.finish_reasons` | On completion | JSON array string or backend-compatible array |
| `error.type` | On error | Exception class or provider code |

### `chat {model}`

| Attribute | Required | Notes |
|---|---|---|
| `gen_ai.operation.name` | Yes | Always `chat` |
| `gen_ai.provider.name` | Yes | OpenAI, Anthropic, Google, local, etc. |
| `gen_ai.request.model` | Yes | Requested model |
| `gen_ai.request.temperature` | If used | Sampling temperature |
| `gen_ai.request.top_p` | If used | Top-p sampling |
| `gen_ai.request.max_tokens` | If used | Output cap |
| `gen_ai.response.model` | On success | Actual response model |
| `gen_ai.response.id` | On success | Provider response ID |
| `gen_ai.response.finish_reasons` | On success | JSON array string or backend-compatible array |
| `gen_ai.usage.input_tokens` | On success when available | Provider-reported tokens |
| `gen_ai.usage.output_tokens` | On success when available | Provider-reported tokens |
| `gen_ai.tool.call.id` | When tool calls exist | Single ID or JSON array string |
| `error.type` | On error | Exception class or provider code |

### `execute_tool {tool.name}`

| Attribute | Required | Notes |
|---|---|---|
| `gen_ai.operation.name` | Yes | Always `execute_tool` |
| `gen_ai.tool.name` | Yes | Tool/function name |
| `gen_ai.tool.type` | Yes | Usually `function` |
| `gen_ai.tool.call.id` | When available | Correlates to `gen_ai.tool.call` logs |
| `error.type` | On error | Exception class or tool error code |

## Scenarios

### Simple chat

Expected output:

- One `chat gpt-4o` span in `genai-traces`.
- Input message logs and one `gen_ai.choice` log in `genai-logs`.
- All logs share the chat span's `trace_id` and `span_id`.

### Multi-turn chat

Expected output:

- One `chat {model}` span.
- One log for every input message in order: system, user, assistant, tool, etc.
- One `gen_ai.choice` log for each response choice.

### Tool calling

Expected output:

- One `chat {model}` span with `gen_ai.response.finish_reasons` such as `["tool_calls"]`.
- One or more `gen_ai.tool.call` logs containing tool name and arguments.
- One `execute_tool {tool.name}` span per executed tool.
- `gen_ai.tool.call.id` shared across the chat log and the matching tool span when available.

### Streaming

Keep the chat span open for the full stream. Emit input logs before starting the stream, accumulate chunks, then set response attributes and emit `gen_ai.choice` after the stream completes. Token usage may be unavailable unless the provider sends usage chunks.

### Retries

Create one `chat {model}` span per attempt. The retry loop should be outside the chat span. Failed attempts have `ERROR`; the successful attempt has `OK`.

## Privacy

Manual instrumentation intentionally emits full log bodies so conversations and tool use can be reconstructed. Before enabling full content capture in production:

- Redact secrets, credentials, payment data, health data, personal identifiers, and confidential documents.
- Decide whether prompts, completions, tool arguments, tool outputs, and thinking blocks are allowed in `genai-logs`.
- Document any redaction, truncation, hashing, or sampling policy.

Do not put full message content on spans. Put content in log bodies so metadata and content stay separated across `genai-traces` and `genai-logs`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Spans appear in `genai-traces`, logs missing from `genai-logs` | CLI or collector logs pipeline missing | Use `--logs_exporter otlp`; configure collector `logs` pipeline and `otlphttp/logs` exporter |
| Logs and traces appear in same stream | Collector uses one exporter/`X-P-Stream` for both signals | Split exporters: `otlphttp/traces` -> `genai-traces`, `otlphttp/logs` -> `genai-logs` |
| Logs have empty `trace_id`/`span_id` | `emit()` called outside span context | Emit logs inside the active `start_as_current_span` block |
| Duplicate HTTP or SDK spans | Auto-instrumentors active | Set `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS` for manual mode |
| `"Overriding of current LoggerProvider is not allowed"` | App calls `set_logger_provider()` after CLI configured provider | Remove manual provider setup when using `opentelemetry-instrument` |
| Token columns are null | Provider did not return usage | Keep columns nullable; do not invent token counts from text length |
