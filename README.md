# Kausate agents

Integration artifacts for AI coding assistants (Claude Code, Codex, Cursor, Copilot, Gemini CLI, Aider, ...) integrating the [Kausate](https://kausate.com) company-data API.

This repo is the **canonical source** for the Kausate integration playbook — `plugins/kausate-integration/skills/kausate-integration/SKILL.md` (the interactive orchestrator) plus `plugins/kausate-integration/skills/kausate-integration/references/*.md` (per-topic detail and competitor migration playbooks). Our docs site at [docs.kausate.com](https://docs.kausate.com) proxies the same files so customers can use either URL.

The skill is **interactive**: it inspects the user's codebase first, asks the few questions the code can't answer (use case mix, webhook reachability, jurisdictions), and then routes to only the references that match — including migration playbooks for replacing **Kyckr**, **Moody's** (kompany / Orbis / Maxsight / Bureau van Dijk), and **Topograph**.

## Install in Claude Code

```
/plugin marketplace add https://docs.kausate.com/.claude-plugin/marketplace.json
/plugin install kausate-integration@kausate
/reload-plugins
```

Equivalent (skips the docs domain):

```
/plugin marketplace add kausate/agents
/plugin install kausate-integration@kausate
/reload-plugins
```

## Use with Codex / Cursor / Copilot / Gemini CLI / Junie / Aider

Drop the AGENTS.md into your repo:

```bash
curl -o AGENTS.md https://docs.kausate.com/agents/kausate.md
```

For Claude Code, use the same AGENTS.md plus a one-line `CLAUDE.md` that imports it:

```bash
curl -o AGENTS.md https://docs.kausate.com/agents/kausate.md
echo "@AGENTS.md" > CLAUDE.md
```

## Layout

```
.claude-plugin/marketplace.json                              ← marketplace manifest
plugins/kausate-integration/
├── .claude-plugin/plugin.json                               ← plugin manifest
└── skills/kausate-integration/
    ├── SKILL.md                                             ← lean orchestrator + interactive opener
    └── references/
        ├── auth-versioning.md
        ├── endpoints.md
        ├── async-webhooks.md
        ├── customer-correlation.md
        ├── status-and-dedup.md
        ├── documents.md
        ├── monitors.md
        ├── identifiers.md
        ├── errors-and-anti-patterns.md
        ├── production-checklist.md
        └── migrations/
            ├── from-kyckr.md
            ├── from-moodys.md       ← covers kompany / Orbis / Maxsight / BvD
            └── from-topograph.md
```

## Editing the skill

Edit any file under `plugins/kausate-integration/skills/kausate-integration/`, push to `main`, and both surfaces update:

- `https://github.com/kausate/agents` (this repo) — direct fetch
- `https://docs.kausate.com/...` — proxied at the docs domain (SKILL.md, references, and migration playbooks all served as URLs for AGENTS.md curl users)

SKILL.md is the orchestrator (≈140 lines). Per-topic detail lives in `references/`, loaded on demand by progressive disclosure — keep each file self-contained and concise.

## Bug reports / suggestions

Please open an issue on this repo or contact us at [kausate.com/contact](https://kausate.com/contact).
