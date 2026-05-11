# Servel Skill

Agent skill for [Servel](https://servel.dev) — self-hosted deployment platform via Docker Swarm.

## Installation

```bash
npx skills add K-NRS/servel-skill
# or
bunx skills add K-NRS/servel-skill
```

### Manual Installation

| Agent | Location |
|-------|----------|
| **Claude Code** | `~/.claude/skills/servel/` |
| **Cursor** | `.cursor/rules/servel.md` |
| **Gemini CLI** | `~/.gemini/skills/servel/` |
| **Goose** | `~/.config/goose/skills/servel/` |
| **OpenCode** | `~/.opencode/skills/servel/` |
| **Codex** | `$CODEX_HOME/skills/servel/` |
| **GitHub Copilot** | `.github/copilot-instructions.md` |

```bash
# Claude Code
mkdir -p ~/.claude/skills/servel
curl -o ~/.claude/skills/servel/SKILL.md https://raw.githubusercontent.com/K-NRS/servel-skill/main/SKILL.md
mkdir -p ~/.claude/skills/servel/references
curl -o ~/.claude/skills/servel/references/TEMPLATES.md https://raw.githubusercontent.com/K-NRS/servel-skill/main/references/TEMPLATES.md
```

## What This Skill Does

Teaches agents to use Servel for deployment and infrastructure:

- **Deploy applications** with auto-detection (Bun, Node, Python, Go)
- **Manage 42+ infrastructure types** (databases, queues, caches, platforms)
- **Link infrastructure** to apps (auto-injects connection strings)
- **Preview deployments** with configurable TTL
- **Backup and restore** databases
- **Configure domains** with automatic SSL
- **Manage secrets** (encrypted with Age)
- **Dev mode** with file sync and team collaboration
- **Alerts** via Telegram, Slack, Discord
- **Create custom templates** for new infrastructure types

## Structure

```
servel-skill/
├── SKILL.md              # Main skill (loaded when triggered)
└── references/
    └── TEMPLATES.md      # Template building guide (loaded on-demand)
```

## Recommended: Pair with a SessionStart Hook (Claude Code)

The skill loads when the user's prompt matches its description (deploy / infra / production-debugging intents) or when they explicitly invoke `/servel`. To make the skill activate **the moment you `cd` into a servel-managed project** — even before you've typed anything — add a `SessionStart` hook that detects `.servel/` or `servel.yaml` in the working directory and emits a reminder.

Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "if [ -d .servel ] || [ -f servel.yaml ]; then echo 'Servel project detected (.servel/ or servel.yaml present). Invoke the /servel skill for deployment, infra, logs, exec, env, secrets, domains, and routing. Inside the project, servel commands auto-detect the deployment name from .servel/state.json — run servel logs / exec / inspect / env without arguments.'; fi"
          }
        ]
      }
    ]
  }
}
```

**Effect**

| Setup | Trigger path |
|---|---|
| Skill description alone | Activates when user prompt mentions deploy/infra/production/logs/etc. |
| **+ SessionStart hook** | Activates **on session start** whenever cwd contains `.servel/` or `servel.yaml`, before the user types anything. |

The hook is silent in non-servel projects (the `if` guards it), so adding it globally has zero noise.

**Why this matters:** the skill knows the live-introspection workflow — `servel logs -f`, `servel env vars`, `servel exec sh`, `servel inspect`, `servel logs --http`, all auto-resolving the project from `.servel/state.json`. Without the hook, agents may speculate about production behavior from source code instead of asking the running deployment. With the hook, the live-data path is the default.

Equivalents for other agents (Gemini CLI, OpenCode, Codex) — check the agent's session-start / pre-prompt hook docs.

## Links

- [Servel Website](https://servel.dev)
- [Documentation](https://servel.dev/docs)
- [Infrastructure Hub](https://hub.servel.dev)

## License

MIT
