# Parseable Skills

Agent skills for adding Parseable observability to LLM applications and AI agents.

## Skills

| Skill | Description |
|---|---|
| `parseable-instrument` | Add, audit, and validate Parseable tracing/logging for LLM apps and AI agents using OpenTelemetry, framework integrations, and collector-based export. |

## Installation

### npx (recommended)

```bash
npx skills add parseablehq/skills --skill "parseable-instrument" --global
```

### Manual symlink

```bash
git clone https://github.com/parseablehq/skills.git ~/parseable-skills
ln -s ~/parseable-skills/skills/parseable-instrument ~/.claude/skills/parseable-instrument
```

## Prerequisites

You need a Parseable instance and credentials:

```bash
export PARSEABLE_ENDPOINT=https://your-parseable-instance.example.com
export PARSEABLE_USERNAME=your-username
export PARSEABLE_PASSWORD=your-password
export PARSEABLE_STREAM=genai-traces
```

Use an API key or Basic auth according to your deployment's access model. Create the target dataset before exporting, or change `PARSEABLE_STREAM` to an existing dataset.

## Usage

Once installed, the agent will automatically use the skill when relevant, for example:

- Adding Parseable tracing to an existing AI agent.
- Auditing LLM instrumentation for model names, token usage, errors, and span hierarchy.
- Setting up OpenTelemetry Collector export to Parseable.
- Choosing framework integrations for OpenAI, Anthropic, LiteLLM, OpenRouter, vLLM, LangChain, LlamaIndex, AutoGen, CrewAI, DSPy, n8n, or other OpenTelemetry-instrumented systems.
- Validating traces, logs, datasets, and dashboards in Parseable.

## License

MIT
