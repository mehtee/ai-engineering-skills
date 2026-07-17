# Agent Skills for AI Engineering

A curated collection of **Agent Skills** for AI coding assistants (Claude Code, Antigravity CLI, and any tool that loads `SKILL.md` folders). Focus: **Python backends, clean architecture, async systems, and the LangChain / LangGraph / Deep Agents ecosystem**.

Each skill is progressive-disclosure docs an agent can load on demand: a short `SKILL.md` entry point plus deeper `references/` (and sometimes scripts) so the model gets the right patterns without stuffing the full context window.

## What’s inside

| Skill | What it covers |
| --- | --- |
| [`clean-coding`](./clean-coding/) | DRY, SOLID, FP-leaning style, strict typing, ruff / pyright / black |
| [`django-clean-drf`](./django-clean-drf/) | Django REST Framework + Clean Architecture (use cases, services, views) |
| [`python-asyncio`](./python-asyncio/) | Structured concurrency, TaskGroup, FastAPI & Django async boundaries |
| [`pydantic`](./pydantic/) | Pydantic v2 models, validators, settings, serialization, FastAPI |
| [`redis`](./redis/) | Cache, locks, rate limits, sessions; redis-py, django-redis, redis-om |
| [`taskiq`](./taskiq/) | Async-native task queues (brokers, results, scheduler, FastAPI/Django) |
| [`langfuse`](./langfuse/) | Observability: traces, prompts, scores, CLI/API access |
| [`langchain-langgraph`](./langchain-langgraph/) | Full skill pack for LangChain, LangGraph, Deep Agents (see below) |

### LangChain / LangGraph / Deep Agents pack

Route through [`langchain-langgraph/AGENTS.md`](./langchain-langgraph/AGENTS.md). Skills live under `langchain-langgraph/skills/`:

| Skill | When to load |
| --- | --- |
| `ecosystem-primer` | **First** — framework choice & setup |
| `langchain-dependencies` | Install / version pins |
| `langchain-fundamentals` | `create_agent`, tools, basics |
| `langchain-middleware` | HITL middleware, structured output |
| `langchain-rag` | Loaders, splitters, embeddings, vector stores |
| `langgraph-fundamentals` | `StateGraph`, edges, `Command`, streaming |
| `langgraph-persistence` | Checkpointers, memory, time travel |
| `langgraph-human-in-the-loop` | `interrupt()`, resume, approval flows |
| `langgraph-cli` | `langgraph` CLI & `langgraph.json` |
| `deep-agents-core` | `create_deep_agent()` harness |
| `deep-agents-memory` | Store / filesystem backends |
| `deep-agents-orchestration` | Subagents, todos, HITL |
| `managed-deep-agents` | Managed Deep Agents + `mda` CLI |
| `swarm` | Parallel fan-out over many work items |

## Install

### Claude Code

**Project skills** (recommended — version with the repo):

```bash
# from your project root
mkdir -p .claude/skills
git clone https://github.com/mehtee/ai-engineering-skills.git /tmp/ai-engineering-skills

# top-level skills
for s in clean-coding django-clean-drf python-asyncio pydantic redis taskiq langfuse; do
  cp -a /tmp/ai-engineering-skills/$s .claude/skills/
done

# LangChain pack (flat install so each skill is discoverable)
cp -a /tmp/ai-engineering-skills/langchain-langgraph/skills/* .claude/skills/
# optional: keep the routing guide next to them
cp /tmp/ai-engineering-skills/langchain-langgraph/AGENTS.md .claude/skills/langchain-langgraph-AGENTS.md
```

**Personal skills** (available in every project):

```bash
mkdir -p ~/.claude/skills
# same copy steps as above, targeting ~/.claude/skills instead of .claude/skills
```

Claude Code discovers skills from:

- `~/.claude/skills/<name>/SKILL.md` — personal
- `.claude/skills/<name>/SKILL.md` — project
- plugin-provided skills

After install, skills are invoked automatically when their `description` matches the task, or explicitly via `/skill-name` (e.g. `/pydantic`, `/django-clean-drf`).

### Antigravity CLI / Gemini CLI–style loaders

Point the skills root at this repo (or a clone):

```bash
# example: copy into the Antigravity skills directory
cp -a ./* ~/.gemini/antigravity-cli/skills/
```

Or symlink:

```bash
ln -s "$(pwd)" ~/.gemini/antigravity-cli/skills/ai-engineering-skills
# then ensure your CLI walks nested skill folders, or flatten as in the Claude Code install
```

### Other agents (Cursor, Codex, custom harnesses)

Any runner that implements the [Agent Skills](https://agentskills.io) layout can use these folders: each skill is a directory with a YAML-frontmatter `SKILL.md` plus optional `references/`, `scripts/`, and `assets/`.

## Skill layout

```text
skill-name/
├── SKILL.md              # required — YAML frontmatter + concise instructions
└── references/           # optional — deep docs loaded on demand
    ├── *.md
    └── ...
```

Frontmatter (minimum):

```yaml
---
name: skill-name
description: >
  What it does and WHEN to use it (trigger phrases matter for auto-discovery).
---
```

## Design principles

1. **Progressive disclosure** — short entrypoint; pull references only when needed.
2. **Actionable patterns** — real code shapes for FastAPI, Django/DRF, and agent stacks — not generic essay docs.
3. **Opinionated defaults** — clean architecture, structured concurrency, explicit types, production failure modes.
4. **Portable** — plain Markdown + folders; no proprietary binary format.

## Contributing

- One skill = one directory named like its `name:` field (lowercase, hyphens).
- Keep `SKILL.md` focused; push long tables/examples into `references/`.
- Write `description` with clear **trigger conditions** so auto-invocation works.
- Prefer Python 3.12+ / modern library APIs (Pydantic v2, Django 5, current LangChain).

## License

MIT — use freely in personal and commercial agent setups. Attribution appreciated but not required.

## Credits

Collected and authored for day-to-day AI-assisted backend and agent engineering. Compatible with all coding agent harnesses.
