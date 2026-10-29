# Personal skill layout

`~/agents/skills` is the **single canonical root** (the Core) for shared personal skills. All harnesses read from here — do not maintain duplicate copies elsewhere. Follows the Core/Shell pattern from https://hamy.xyz/blog/2026-05_harness-agnostic-ai-skills.

- `~/AGENTS.md` holds personal global instructions. `~/CLAUDE.md` imports it via `@AGENTS.md`.
- **Cursor** reads skills via `~/.cursor/skills`, a symlink here.
- `~/.agents/skills` symlinks here (Codex, Kimi Code).
- `~/.claude/skills` symlinks here (Claude Code).
- `~/.codex/skills` is reserved for Codex-managed system content only.
- `~/.cursor/skills-cursor` is Cursor-managed. Do not edit it or link its skills into the Core.
- Install plugins (e.g. superpowers) natively per harness. Plugin skills do not belong in the Core; plugin hooks, updates, and harness-specific workflows are harness-scoped.
- Never copy skills into another harness skill root.
- When a skill installer supports `--dest`, set it to `~/agents/skills`.
- When citing or editing a shared skill path in chat or docs, use `~/agents/skills/<skill-name>/`.
