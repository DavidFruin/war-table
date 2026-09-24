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
  - `grill-me/` + `grilling/`: third-party skills vendored from [mattpocock/skills](https://github.com/mattpocock/skills) (`skills/productivity/grill-me` and `skills/productivity/grilling`), MIT licensed — see each folder's `LICENSE.txt`. `grill-me` is a thin `/grill-me`-only trigger (`disable-model-invocation: true`) that hands off to `grilling`, which holds the actual interview logic: relentlessly question a plan/idea in rounds until the "design tree" is fully resolved, no repo or code required. Also symlinked into `~/.claude/skills/` on each machine, unmodified from upstream.
  - `agent-board/`: uses [[simple-social]]'s dev site as a live message board between agents, via `sscli --json` — a shared agent account, one-line posts only (the server rejects newlines), tagged by machine/agent name since the account isn't per-agent. Deliberately kept separate from this skill (war-table is for durable notes; agent-board is for live coordination between running agents, not a record). Account creation is manual, by Dave, since it needs an OTP to a real inbox.

## Decisions

- Keeping this as a "project" folder (not a single note) because the content is actual config files, not notes about them.
- Not symlinking `~/.claude/CLAUDE.md` to this file yet — that would make war-table a hard runtime dependency for every Claude Code session (broken if the vault isn't cloned on a given machine). Deferred until Dave decides he wants that.

## Next steps
- [ ] Decide: keep syncing `~/.claude/CLAUDE.md` here manually, or symlink `~/.claude/CLAUDE.md -> ~/war-table/Projects/claude-config/CLAUDE.md` for automatic sync (tradeoff: ~/.claude becomes dependent on the vault being cloned)
- [x] Personal skill written: `skills/war-table/` — symlink it into `~/.claude/skills/` on each machine (one command, see that skill's own setup section)
- [x] Vendored `skills/grill-me/` and `skills/grilling/` from mattpocock/skills — symlink both into `~/.claude/skills/` on each machine:
      `ln -sfn ~/war-table/Projects/claude-config/skills/grill-me ~/.claude/skills/grill-me && ln -sfn ~/war-table/Projects/claude-config/skills/grilling ~/.claude/skills/grilling`
- [x] Personal skill written: `skills/agent-board/` — symlink it the same way (`ln -sfn ~/war-table/Projects/claude-config/skills/agent-board ~/.claude/skills/agent-board`). Still needs the shared agent account actually created before it's usable — that's on Dave, see the skill's own setup section.
- [ ] Write `AGENTS.md` here if cross-project agent conventions emerge beyond what's in CLAUDE.md

## Links
- `~/.claude/CLAUDE.md` (live copy on this machine, hostname: omarchy)
