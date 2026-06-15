# Parseable Collector Export

Use this reference when telemetry needs to leave the app process and arrive in Parseable.

## Credentials Prompt

Before generating any config, check whether these environment variables are set:

- `PARSEABLE_ENDPOINT`
- `PARSEABLE_USERNAME`
- `PARSEABLE_PASSWORD`
- `PARSEABLE_STREAM`

If any are missing, ask the user for each value. Do not ask users to paste values into chat beyond this prompt — once collected, instruct them to set as environment variables or add to `.env`. Never commit credentials to source.

## Environment

Create or update `.env.example`:

```bash
# OpenTelemetry SDKs send to the local collector.
OTEL_SERVICE_NAME=my-agent
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Collector sends to Parseable.
PARSEABLE_ENDPOINT=https://your-parseable-instance.example.com
PARSEABLE_STREAM=genai-traces
PARSEABLE_USERNAME=your-username
PARSEABLE_PASSWORD=your-password
PARSEABLE_TENANT_ID=
```

Do not commit `.env`.

The collector config uses Basic auth computed from username and password:

```bash
# Compute at runtime or in a setup script — never hardcode the result.
export PARSEABLE_BASIC_AUTH=$(echo -n "${PARSEABLE_USERNAME}:${PARSEABLE_PASSWORD}" | base64)
```

## Collector Configuration

Create `otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 5s
  resource:
    attributes:
      - key: service.namespace
        value: genai
        action: upsert

exporters:
  otlphttp/parseable:
    endpoint: "${PARSEABLE_ENDPOINT}"
    headers:
      Authorization: "Basic ${PARSEABLE_BASIC_AUTH}"
      X-P-Stream: "${PARSEABLE_STREAM}"
      X-P-Log-Source: "otel"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [otlphttp/parseable]
    logs:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [otlphttp/parseable]
```

If the deployment requires tenant routing, add this header after confirming the value exists:

```yaml
      X-P-Tenant-ID: "${PARSEABLE_TENANT_ID}"
```

Confirm exact endpoint and authentication format against current Parseable docs for the target deployment.

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

Use direct SDK export only when credentials can be managed safely in the runtime and the app does not need collector processing, batching, sampling, routing, or header injection. For most production apps, prefer a collector.

## Kubernetes

For Kubernetes, prefer a collector Deployment/DaemonSet and secrets:

- Store Parseable credentials in a Kubernetes Secret.
- Use ConfigMap for collector config.
- Keep app pods configured with `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318`.
- Use Parseable Auto Instrumentation when the user wants zero/low-code telemetry injection and the cluster prerequisites are met.
