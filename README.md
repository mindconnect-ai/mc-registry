# mc-registry

The public registry of [Mindconnect](https://github.com/mindconnect-ai/mindconnect):
LLM configs, agents and workflows you can import into a Mindconnect installation
from its admin UI.

A registry is a plain GitHub repository — an index (`registry.json`) and the
entity files it points at. There is no server behind it: Mindconnect reads the
raw files over HTTPS.

## Using it

In the admin UI open **Registry → Add registry** and type

```
mindconnect-ai/mc-registry
```

Open the registry, pick an entry and press **Import**. An entry brings what it
needs along: an agent imports the LLM config it runs on and the sub-agents it
calls first. **Import** keeps what you already have under the same name;
**Import and overwrite** replaces it.

Pin a tag (`mindconnect-ai/mc-registry@v1.0.0`) once there is one, if you want
an installation to stay on a known state instead of following `main`.

## What is in it

| Directory | Kind | Contents |
|-----------|------|----------|
| `llm-configs/` | `llm-config` | One config per provider (OpenAI, Anthropic, Gemini, Azure OpenAI, LM Studio), embeddings, speech-to-text, and the `agent-default` alias every bundled agent runs on |
| `agents/` | `agent` | Assistants (`default-chat`, `coding-assistant`, `planner`, `research-lead`, …), their sub-agents, and the utility agents the runtime calls by name |
| `workflows/` | `workflow` | Small examples (`hello`, `approval`, `form-demo`, …) and document workflows (`file-ingestion`, `word-to-markdown`, `wordreport-gen`) |
| `packages/` | `package` | `mindconnect-defaults` (everything above), `runtime-utilities`, `document-kit` |

These are the entities a fresh Mindconnect installation seeds itself with, so
the registry is also the way to get a default back after you changed or deleted
it.

### API keys

No file here carries a key. LLM configs reference environment variables that
your installation resolves at call time:

| Variable | Used by |
|----------|---------|
| `OPENAI_API_KEY` | `openai-default`, `openai-embeddings`, `speech-to-text` |
| `ANTHROPIC_API_KEY` | `claude-default`, `claude-haiku-default` |
| `GEMINI_API_KEY` | `gemini-default` |
| `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT` | `azure-openai-default` |
| `LM_STUDIO_API_KEY` (optional, defaults to `lm-studio`) | the LM Studio configs |

Models can be overridden the same way (`OPENAI_MODEL`, `CLAUDE_MODEL`, …) — see
the `${VAR:default}` placeholders in each file. `agent-default` points at
`openai-default`; repoint the alias in your installation to move every bundled
agent to another model.

## Contributing

1. Add the entity file under the directory for its kind — the same JSON the
   Mindconnect admin UI stores, without an `id` (the importing installation
   assigns its own).
2. Add an entry to `registry.json`: `id`, `type`, `name`, `path`, a
   `description`, and in `requires` the ids of entries it depends on (the LLM
   config an agent names, the sub-agents it calls, the agents a workflow runs).
3. Never put a literal API key into an LLM config — use a `${ENV_VAR}`
   placeholder. A literal key is dropped on import.
4. Open a pull request.

The format is documented in the Mindconnect docs under
[Registry](https://github.com/mindconnect-ai/mindconnect/blob/main/website/docs/agents/registry.md).
