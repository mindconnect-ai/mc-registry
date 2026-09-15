# mc-registry

The public registry of [Mindconnect](https://github.com/mindconnect-ai/mindconnect):
LLM configs, agents, workflows and skills you can import into a Mindconnect
installation from its admin UI.

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
| `skills/` | `skill` | One `SKILL.md` per skill: `docx-builder`, `pptx-builder` (with diagram slides whose arrows are real connectors) and `xlsx-builder` (Word, PowerPoint and Excel files through `code_execute`), `db-timetables` and `swiss-transport-ojp` (Deutsche Bahn and Swiss public-transport timetables through `bash`) |
| `packages/` | `package` | `mindconnect-defaults` (the defaults below), `runtime-utilities`, `document-kit`, `release-notes-kit`, `office-skills`, `transport-skills` |

Most of it is what a fresh Mindconnect installation seeds itself with, so the
registry is also the way to get a default back after you changed or deleted it.

The **skills** are not defaults either. A skill is know-how an agent loads
when it needs it — the three office skills carry a standard-library Python
generator that runs inside the `code_execute` container, the two transport
skills describe an API and read its key from the environment `bash` inherits.
Every agent whose skills mode is *all* is offered an imported skill; an agent
naming its skills has to name it.

The **Release notes kit** is not a default: an example of a package that brings
everything new — a model config (`release-notes-haiku`), two agents
(`changelog-writer`, `release-announcer`) and the workflow that runs them in
turn (`release-notes`, from `version` and `commits` to a changelog and an
announcement).

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
| `DB_CLIENT_ID`, `DB_API_KEY` | the `db-timetables` skill (DB API Marketplace, Timetables plan) |
| `OPENTRANSPORTDATA_API_KEY` | the `swiss-transport-ojp` skill (opentransportdata.swiss, OJP 2.0) |

Models can be overridden the same way (`OPENAI_MODEL`, `CLAUDE_MODEL`, …) — see
the `${VAR:default}` placeholders in each file. `agent-default` points at
`openai-default`; repoint the alias in your installation to move every bundled
agent to another model.

## Contributing

1. Add the entity file under the directory for its kind — the same JSON the
   Mindconnect admin UI stores, without an `id` (the importing installation
   assigns its own). A skill is a `skills/<name>/SKILL.md` — front matter
   with `name`, `description` and `tools`, then the instructions; only that
   file is imported, so anything the skill needs goes into its text.
2. Add an entry to `registry.json`: `id`, `type`, `name`, `path`, a
   `description`, and in `requires` the ids of entries it depends on (the LLM
   config an agent names, the sub-agents it calls, the agents a workflow runs).
3. Never put a literal API key into an LLM config — use a `${ENV_VAR}`
   placeholder. A literal key is dropped on import.
4. Open a pull request.

The format is documented in the Mindconnect docs under
[Registry](https://github.com/mindconnect-ai/mindconnect/blob/main/website/docs/agents/registry.md).
