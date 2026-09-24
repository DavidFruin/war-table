---
status: active
repo: ~/.claude
---

# claude-config

Dave's personal Claude Code configuration: global rules, skills, and agent instructions — kept here so they're portable across machines and versioned like everything else.

## Summary

The canonical copies live in this folder. `~/.claude/` on each machine should match what's here. For now that's a manual copy (see Next steps for making it automatic via symlink).

## Contents

- `CLAUDE.md` — global rules file, normally installed at `~/.claude/CLAUDE.md`. Applies to every Claude Code session on the machine, in any project.
- `AGENTS.md` — not created yet. Would hold cross-project agent instructions if/when Dave writes one (distinct from `~/war-table/AGENTS.md`, which only covers how agents use this vault).
- `skills/` — empty. Dave's current skills (`diagnose-crash`, `omarchy`) are symlinks to Omarchy's system-provided defaults at `/usr/share/omarchy/default/agents/skills/`, not personal files, so there's nothing to check in yet. Any *personal* skill created later goes here.

## Decisions

- Keeping this as a "project" folder (not a single note) because the content is actual config files, not notes about them.
- Not symlinking `~/.claude/CLAUDE.md` to this file yet — that would make war-table a hard runtime dependency for every Claude Code session (broken if the vault isn't cloned on a given machine). Deferred until Dave decides he wants that.

## Next steps
- [ ] Decide: keep syncing `~/.claude/CLAUDE.md` here manually, or symlink `~/.claude/CLAUDE.md -> ~/war-table/Projects/claude-config/CLAUDE.md` for automatic sync (tradeoff: ~/.claude becomes dependent on the vault being cloned)
- [ ] If/when personal skills are written, add them under `skills/` here (and symlink or copy into `~/.claude/skills/`)
- [ ] Write `AGENTS.md` here if cross-project agent conventions emerge beyond what's in CLAUDE.md

## Links
- `~/.claude/CLAUDE.md` (live copy on this machine, hostname: omarchy)
