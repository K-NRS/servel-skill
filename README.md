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

## Links

- [Servel Website](https://servel.dev)
- [Documentation](https://servel.dev/docs)
- [Infrastructure Hub](https://hub.servel.dev)

## License

MIT
