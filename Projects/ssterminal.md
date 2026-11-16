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

## Caught up (2026-11-16, Citadel)
[[ssterminal-catch-up-plan]] Phases 0-4 done for all three apps, plus the November ssapi review's changes (verify token on reset/registration, UTC times with the writer's GMT offset). Pushed to `master` (`d9265d4`..`b4ede66`); ssreact `a22e168` (Download page, deployed to dev).
- **Library:** whole responses (no 64 KB cut-off, no 8 KB request cut-off), depth-aware JSON with decoded strings, error status/code, the verify token, `X-SS-TZ`, and every new action. Stays on API version 1 on purpose: `sscli --json` prints `totalCount`, which version 2 drops (so no refresh-token rotation for the terminal yet).
- **All three apps:** code-first registration with the age/terms question, then logged in with terms recorded; welcome once; terms prompt when they change; folded reported posts/comments; "Hidden by a moderator"; threads and replies; notifications for every type; role badges; report, block, blocked list; invites; devices; upload pre-checks; times in UTC.
- **The wizard now prints with `cli/output.c`** (its own copy was deleted).
- **Fixed on the way:** `sscli --json profile <other>` printed a non-JSON line after the object.
- **Tests never touch dev/prod now:** `tests/bench.sh` starts a local ssapi server; `make unit` (26), `tests/test_cli.sh` (61) and `wizard/tests/test_wizard.py` (39) all pass. The TUI was checked by hand in tmux against the bench (terms prompt, feed, reply, report/fold, notifications, badge, Settings, devices, invite).
- **Not done:** the optional staff markers in lists (`getStaff`), and the TUI has no automated test.

## Next steps
- [ ] Build and install the new binaries on the machines that use them (`make && sudo make install`, or `make PREFIX=$HOME/.local install`).
- [ ] Optional: move the terminal to API version 2 (refresh-token rotation) once `--json` users don't need `totalCount`.

## Links
- Repo: https://github.com/DavidFruin/ssterminal
- History of the TUI's design work: [[simple-social-tui]]
