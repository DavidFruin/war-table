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
- `skills/` — `war-table/`: the first personal skill checked in here. Reads and updates this vault itself (clone/pull/locate the right note/push), deferring to the vault's own `AGENTS.md` for what to actually write rather than duplicating those conventions into the skill. Symlinked, not copied, onto each machine — see that skill's own "Setting this skill up on a new machine" section. Dave's other current skills (`diagnose-crash`, `omarchy`) are symlinks to Omarchy's system-provided defaults at `/usr/share/omarchy/default/agents/skills/`, not personal files, so they don't live here.

## Decisions

- Keeping this as a "project" folder (not a single note) because the content is actual config files, not notes about them.
- Not symlinking `~/.claude/CLAUDE.md` to this file yet — that would make war-table a hard runtime dependency for every Claude Code session (broken if the vault isn't cloned on a given machine). Deferred until Dave decides he wants that.

## Next steps
- [ ] Decide: keep syncing `~/.claude/CLAUDE.md` here manually, or symlink `~/.claude/CLAUDE.md -> ~/war-table/Projects/claude-config/CLAUDE.md` for automatic sync (tradeoff: ~/.claude becomes dependent on the vault being cloned)
- [x] Personal skill written: `skills/war-table/` — symlink it into `~/.claude/skills/` on each machine (one command, see that skill's own setup section)
- [ ] Write `AGENTS.md` here if cross-project agent conventions emerge beyond what's in CLAUDE.md

## Links
- `~/.claude/CLAUDE.md` (live copy on this machine, hostname: omarchy)
