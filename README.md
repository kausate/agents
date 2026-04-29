# Kausate agents

Integration artifacts for AI coding assistants (Claude Code, Codex, Cursor, Copilot, Gemini CLI, Aider, ...) integrating the [Kausate](https://kausate.com) company-data API.

This repo is the **canonical source** for the Kausate integration playbook — `plugins/kausate-integration/skills/kausate-integration/SKILL.md`. Our docs site at [docs.kausate.com](https://docs.kausate.com) proxies the same file so customers can use either URL.

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
└── skills/kausate-integration/SKILL.md                      ← canonical skill content
```

## Editing the skill

`plugins/kausate-integration/skills/kausate-integration/SKILL.md` is the only content file. Edit it here, push to `main`, and both surfaces update:

- `https://github.com/kausate/agents` (this repo) — direct fetch
- `https://docs.kausate.com/...` — proxied at the docs domain

## Bug reports / suggestions

Please open an issue on this repo or contact us at [kausate.com/contact](https://kausate.com/contact).
