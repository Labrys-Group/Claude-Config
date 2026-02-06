# Labrys Claude Config

Labrys' global coding standards and commands for Claude Code.

## Setup

```bash
git clone https://github.com/Labrys-Group/Claude-Config.git ~/.claude
```

## Structure

- `CLAUDE.md` - Entry point, lists standards and commands
- `rules/` - Coding standards (auto-loaded by Claude Code)
- `commands/` - Slash commands (`/pr`, `/check`, `/testing-plan`)
- `agents/` - Specialized agents for different domains (Frontend, Backend, etc.)
- `docs/` - Internal guides ([Subagents Guide](docs/subagents-guide.md))