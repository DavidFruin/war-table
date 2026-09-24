---
status: active
repo: https://github.com/DavidFruin/code-conductor
---

# code-conductor

## Summary
Project-local, tmux-based multi-agent orchestration. Pi agents run in their own tmux windows/panes; a Python daemon (`agentd.py`) owns the SQLite lifecycle registry, JSONL communication, and a live monitor window. Two specs live in the repo's `docs/`; the plan below merges their phases.

## Plan

| # | Phase | Description | Status |
|---|-------|-------------|--------|
| P1 | tmux launcher | `start-agents.sh` creates the session (monitor + orchestrator panes) | done |
| P2 | Registry | SQLite agent registry, dynamic pane/window indexing, daemon supervision | done |
| P3 | Structured events | Full `agent.*` lifecycle taxonomy in `agent_events` | mostly done |
| P4 | Monitor window | Live per-agent panel in window 0 | done |
| P5 | Git/PR integration | `commits` + `pull_requests` tables, commit/PR events, branch/worktree isolation, notify on done → orchestrator review → merge | next |
| P6 | History & debugging | `agentd.py history <id>`: events, messages, PTY excerpts, commits, PRs | |
| P7 | Interactive sidebar | Select an agent in the monitor → jump to pane / task / events / PR | |
| P8 | Comms polish | Chat-style comms panes; agent→agent routing UX | |
| P9 | PTY hardening | ptylog auto-restart on pipe drop; input capture | |
| P10 | Fail-open + README | Degrade gracefully without `.agents`/DB; setup docs | |

## Decisions
- Non-goals (from the specs): no in-repo GitHub implementation — `git` and `gh` stay agent tools; no scraping Pi's visual TUI; `agentd.py` must never affect the agents themselves.
- Indexing is always dynamic, because Dave's tmux uses `base-index 1` / `pane-base-index 1`. Panes are assigned by `pane_top` (pi top, pty middle, comms bottom).
- `stop_agent` marks `stopped` in the DB *before* killing the window, so the daemon can't race it to `finished`.
- `start-agents.sh` attaches to an existing session rather than killing it.
- Agent IDs are `agent-<ts>-<uuid4[:6]>` (plain timestamps collided on same-second spawns).
- Push to GitHub after each completed phase.

## Status
- P1–P4 shipped (`564c811`, `eae4243`), plus the spawn-agent skill (`5f896e9`) that teaches the orchestrator when/how to delegate.
- P3 gaps: `agent.tool_started/finished`, `agent.test_*`, `agent.commit_created`, `agent.pr_created` are in the taxonomy but not emitted yet — they land with P5 and tool/test instrumentation.
- Spawn already creates an `agent-<id>-<name>` branch per agent; the rest of Git/PR integration is the biggest remaining chunk.

## Verification approach
- `python3 -m py_compile .agents/agentd.py` after every edit.
- Full-stack tests in real tmux (clean `agents-*` sessions, removed after each run): spawn → comms `status` JSON ingestion → registry updates → window kill → `finished` → daemon exit. Assert event trail and registry via direct SQLite queries.

## Next steps
- [ ] P5 — Git/PR integration
- [ ] P6–P10 per the plan table

## Links
- Specs: `docs/` in the repo
