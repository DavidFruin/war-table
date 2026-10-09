---
status: active
repo: https://github.com/DavidFruin/ssterminal
---

# ssterminal

## Summary
Every Simple Social terminal client in one C repo: the shared library (`lib/`, libcurl plus a hand-rolled JSON reader), the scriptable CLI `sscli` (`cli/`), the interactive wizard (`wizard/`) and the full-screen ncurses TUI (`tui/`), built with one top-level `Makefile`. It was created 2026-10-06 from the simple-social-cli, simple-social-cli-interactive and [[simple-social-tui]] repos, with their history ([[repo-consolidation-plan]] Step 3).

## Status
- **2026-10-09:** the last feature work was 2026-09-24. The server has moved on a lot since, and prod has run the new version since 2026-10-09.
  - The library uses 33 of the server's 69 actions.
  - **Registration is broken.** It's invite-only and code-first now, and the terminal never sends a code.
  - Terms acceptance, comment replies, report/block, invites, badges, devices and media limits are all missing.
  - Two latent library bugs: responses over 64 KB are cut off, and the JSON reader isn't depth-aware.
- **The plan to catch up:** [[ssterminal-catch-up-plan]].
- The live message board for agents, `agent-board`, uses `sscli --json` against dev. Keep `--json` output backward-compatible.

## Decisions
- **The library's built-in default server is prod** (`lib/ss_config.c`). Tests (`tests/test_cli.sh`) default to dev, and agents use a local ssapi bench. Never point tests at prod.

## Next steps
- [ ] [[ssterminal-catch-up-plan]]: Phases 0–1 first (library robustness, code-first registration, terms).

## Links
- Repo: https://github.com/DavidFruin/ssterminal
- History of the TUI's design work: [[simple-social-tui]]
