# agent-skills

Personal, harness-agnostic agent skills and global instructions.

## Layout

- `skills/` — canonical skill root. Each `<skill-name>/SKILL.md` holds one skill.
- `AGENTS.md` — copy of `~/AGENTS.md`, the personal global instructions.
- `CLAUDE.md` — copy of `~/CLAUDE.md`, which imports `AGENTS.md`.

The live instruction files stay at `~/AGENTS.md` and `~/CLAUDE.md`. The copies here exist for versioning.

## Harness links

Each harness skill root symlinks to `skills/`:

| Harness | Path |
| --- | --- |
| Cursor | `~/.cursor/skills` |
| Codex, Kimi Code | `~/.agents/skills` |
| Claude Code | `~/.claude/skills` |
| OpenCode | `~/.config/opencode/skills` |

Do not copy skills into another harness root. See `skills/README.md` for authoring rules.
