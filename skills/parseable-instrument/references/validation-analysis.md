# Validation And Analysis

Use this reference after instrumentation or when debugging missing telemetry.

## Local Validation

Run project tests and a representative agent invocation:

```bash
pytest
npm test
```

Use whichever commands exist in the project. If no tests exist, run the smallest CLI/API path that triggers one LLM call and one tool call.

## Collector Checks

```bash
docker compose -f docker-compose.otel.yml ps
docker compose -f docker-compose.otel.yml logs --tail=100 otel-collector
```

Look for:

- Collector started without config errors.
- OTLP receiver listening on `4317` and/or `4318`.
- No authentication failures from the Parseable exporter.
- No permanent exporter failures or dropped spans/logs.

## Trace Checks

In Parseable, confirm:

| Check | Expected |
|---|---|
| Dataset receives records | New records appear in the target dataset |
| Service name | `service.name` matches the app |
| Root span | One root agent/request span per invocation |
| LLM span | Provider, model, latency, status, and token usage where available |
| Tool spans | Tool name/type and error status where relevant |
| Hierarchy | Agent span contains LLM/tool/retrieval children |
| Errors | Failed calls are marked as errors and include exception metadata |
| Privacy | Message content is absent, redacted, sampled, or intentionally enabled |

## Common Failure Modes

| Symptom | Likely cause | Fix |
|---|---|---|
| No spans | Instrumentation imported too late, SDK not started, endpoint wrong | Initialize before provider clients and verify `OTEL_EXPORTER_OTLP_ENDPOINT` |
| Collector receives but Parseable empty | Bad endpoint, auth, stream, tenant, or dataset | Verify Parseable headers and target dataset |
| Flat traces | Context not propagated across async/task boundaries | Use active-span APIs and framework context propagation |
| Missing token usage | SDK response lacks usage or streaming aggregator missing | Capture usage after final response or count via provider metadata |
| Duplicate spans | Auto-instrumentation and manual wrapper both active | Disable one path or narrow manual spans |
| Secret leakage | Raw messages/tool inputs logged | Remove content logging, redact, or hash |
| Script exits before export | Batch processor not flushed | Flush/shutdown tracer provider before process exit |

## Runbook Text

Add a concise README section to instrumented projects:

```markdown
## Parseable Observability

1. Copy `.env.example` to `.env` and fill in Parseable values.
2. Start the OpenTelemetry Collector:
   ```bash
   docker compose -f docker-compose.otel.yml up -d
   ```
3. Run the app normally.
4. Open Parseable and inspect the configured dataset for traces/logs.
```

If the app requires a special launch command, include that command instead of "run the app normally."
