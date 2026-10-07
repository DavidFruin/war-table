---
status: active
repo: https://github.com/DavidFruin/sstests
---

# sstests

## Summary
All test suites for the Simple Social project, kept in one repo so none of it ships to the production docroot. Renamed from `simple-social-tests` on 2026-09-30, the same day [[ssapi]] split off from [[simple-social]] — when tests turned up briefly living inside `ssapi`, Dave pointed out the existing `simple-social-tests` repo already served this purpose under a different name, so it was renamed rather than replaced.

## Status (2026-09-30)
- **Renamed, not recreated.** `gh repo rename sstests --repo DavidFruin/simple-social-tests` — this is the same repo, same history, just a new name. Local clone dir and `origin` remote URL updated to match.
- **Pre-existing content untouched**: a mature 20-spec Playwright E2E suite (`post-by-id`, `session-expiry`, `feed-pagination`, `media-lifecycle`, `event-listeners`, `regression-smoke`, etc.), `helpers.js` (`apiCall()` — runs `fetch` inside the page using the browser's stored JWT), `fixtures/`, env-based credentials (`.env`, no defaults on purpose — see README), `playwright.config.js`. This suite runs against a live `TEST_BASE_URL` and writes real data, cleaning up in `finally` blocks.
- **New `backend/` folder added 2026-09-30**: the 9 shell/JS/PHP scripts that used to live in `simple-social`'s `tests/backend-tests/` and briefly in `ssapi`'s copy of the same — `lint.sh`, `package.json`, `test.js`, `test_api.sh`, `test_api_auth.sh`, `test_auth.js`, `test_db.php`, `test_endpoint.php`, `test_media.sh`. Moved as-is, no rewrites.
- **Two of those files are not actually portable, flagged honestly in the README rather than silently fixed**:
  - `test_db.php` — `require_once __DIR__.'/../../config.php'` and direct SQLite access; only works copied onto the server two directories above wherever `config.php` lives (i.e. dropped into `ssapi`'s own docroot temporarily), not run from this repo.
  - `test_endpoint.php` — a one-line `{"test":"ok"}` fixture meant to be placed somewhere web-accessible and curled, not run locally at all.
  - `lint.sh` — assumes `$DIR/../../api.php` and `$DIR/../../js` (the old combined `simple-social` layout). Already stale even before this move, since `ssapi` has no `js/` directory. Left as-is; whoever picks it up should decide whether to parameterize the target repo path or retire it now that `ssapi` has its own PHP tooling for the syntax-check part.
- **`simple-social`'s own `tests/front-end-test/`** (the older 6-spec suite, hash-routing/`#/login`/element-ID based) is a separate thing entirely — already known incompatible with [[ssreact]]'s DOM, and was never part of this consolidation. Don't conflate the two.

## Media harness (2026-10-07)
- `backend/media/`: `make-fixtures.sh` (every fixture made by ffmpeg, no real media), `harness.php` (runs one step of ssapi's real `src/Media/handlers.php` with a stub `$CONFIG`, filling handler arguments by parameter name so it survives signature changes), `run.sh` (PASS/FAIL per check, peak RSS sampled across php and its ffmpeg grandchildren). Run with `SSAPI=~/dev/ssapi backend/media/run.sh`.
- **Needs PHP with GD.** The admin machine's PHP has no GD and there's no sudo. The fix used there: `apt download php8.4-gd`, extract with `dpkg -x` into `~/.local/phpgd/root`, add an ini file loading that `gd.so`, and run with `PHP_INI_SCAN_DIR=/etc/php/8.4/cli/conf.d:$HOME/.local/phpgd/conf.d`. (The apt download comes with an `install` script that wants sudo; it isn't needed.)

## Decisions
- **One test repo for the whole project, frontend and backend both** — Dave's direct instruction: "Tests shouldn't be on the server so that is another repo on gh." Consolidating avoids splitting test code across `ssapi` and a separate test repo for no benefit.
- **Rename over recreate.** Nearly created a fresh empty `sstests` repo before Dave caught it — the mature suite already existed under `simple-social-tests`. Worth remembering: check for an existing repo under a different name before assuming one needs to be created.

## Next steps
- [ ] Decide whether `lint.sh` gets parameterized (env var for target repo path) or retired in favor of `ssapi`'s own composer-based syntax checking
- [ ] `test_db.php`/`test_endpoint.php` portability — no action needed unless someone actually wants to run them; documented as-is

## Links
- Repo: https://github.com/DavidFruin/sstests
- Related: [[ssapi]] (backend these tests exercise), [[simple-social]] (original combined repo, still hosts its own separate `tests/front-end-test/`), [[ssreact]] (React frontend — not yet covered by any suite here)
