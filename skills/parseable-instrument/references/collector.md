# Parseable Collector Export

Use this reference when app telemetry must leave the process and arrive in Parseable.

Parseable agent observability uses two datasets:

- `genai-traces`: flattened OTel spans.
- `genai-logs`: flattened OTel log records.

The collector must route traces and logs separately. Do not configure one `PARSEABLE_STREAM` for both pipelines.

## Credentials And Environment

Before generating config, check whether these environment variables are set:

- `PARSEABLE_ENDPOINT`
- `PARSEABLE_USERNAME`
- `PARSEABLE_PASSWORD`
- `PARSEABLE_TRACES_STREAM`
- `PARSEABLE_LOGS_STREAM`

If values are missing, instruct the user to set them as environment variables or in `.env`. Do not ask for secrets in chat. Never commit `.env`.

Use this `.env.example` shape:

```bash
# App SDKs send OTLP to the local collector.
OTEL_SERVICE_NAME=my-genai-agent
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Collector sends to Parseable.
PARSEABLE_ENDPOINT=https://your-parseable-instance.example.com
PARSEABLE_TRACES_STREAM=genai-traces
PARSEABLE_LOGS_STREAM=genai-logs
PARSEABLE_USERNAME=your-username
PARSEABLE_PASSWORD=your-password
PARSEABLE_TENANT_ID=
```

Compute Basic auth at runtime or in local shell setup. Do not hardcode the result:

```bash
export PARSEABLE_BASIC_AUTH=$(printf "%s" "${PARSEABLE_USERNAME}:${PARSEABLE_PASSWORD}" | base64)
```

## Collector Configuration

Create `otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 100

exporters:
  otlphttp/traces:
    endpoint: "${PARSEABLE_ENDPOINT}"
    encoding: json
    headers:
      Authorization: "Basic ${PARSEABLE_BASIC_AUTH}"
      X-P-Stream: "${PARSEABLE_TRACES_STREAM}"
      X-P-Log-Source: "otel-traces"

  otlphttp/logs:
    endpoint: "${PARSEABLE_ENDPOINT}"
    encoding: json
    headers:
      Authorization: "Basic ${PARSEABLE_BASIC_AUTH}"
      X-P-Stream: "${PARSEABLE_LOGS_STREAM}"
      X-P-Log-Source: "otel-logs"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/traces]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/logs]
```

If the deployment requires tenant routing, add this header to both exporters after confirming the value exists:

```yaml
      X-P-Tenant-ID: "${PARSEABLE_TENANT_ID}"
```

## Key Rules

- One receiver is fine: SDKs send both OTLP traces and logs to the collector.
- Two pipelines are required: `traces` and `logs` are separate OTel signals.
- Two exporters are required: `otlphttp/traces` writes `genai-traces`; `otlphttp/logs` writes `genai-logs`.
- Use `encoding: json` so Parseable receives JSON OTLP data it can flatten.
- Use separate `X-P-Log-Source` values (`otel-traces`, `otel-logs`) for source clarity.
- Keep metrics disabled unless the project explicitly needs them.

## Docker Compose

Create `docker-compose.otel.yml`:

```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel/config.yaml"]
    env_file:
      - .env
    volumes:
      - ./otel-collector-config.yaml:/etc/otel/config.yaml:ro
    ports:
      - "4317:4317"
      - "4318:4318"
```

Run:

```bash
docker compose -f docker-compose.otel.yml up -d
```

## Direct Export

Prefer the collector for Parseable agent observability because it cleanly routes two OTel signals into two Parseable streams and keeps credentials out of app code.

Use direct SDK export only when:

- Runtime secret management is already solved.
- No batching, retry tuning, filtering, tenant headers, or routing are needed.
- The SDK can export traces and logs to separate Parseable streams without mixing signals.

## Kubernetes

For Kubernetes:

- Store Parseable credentials in a Secret.
- Store collector config in a ConfigMap.
- Run the collector as a Deployment or DaemonSet.
- Configure app pods with `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318`.
- Route collector `traces` and `logs` pipelines to separate Parseable exporters.

## Validation

After sending one agent run:

- Query `genai-traces` for `span_name` or `gen_ai.operation.name`; expect `invoke_agent`, `chat`, and `execute_tool` rows.
- Query `genai-logs` for `gen_ai.event.name`; expect message, choice, tool, and finish events.
- Verify log rows have non-empty `trace_id` and `span_id`.
- Verify trace and log rows do not land in the same Parseable stream.
