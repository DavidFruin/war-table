---
status: active
repo: https://github.com/DavidFruin/ssapi
---

# ssapi

## Summary
The PHP backend for Simple Social — split out of [[simple-social]] on 2026-09-30 so the backend and frontend ([[ssreact]]) are independently cloneable, at Dave's direct request. Not a rewrite: the code is a straight copy of `simple-social`'s existing backend as of that date.

## Status (2026-09-30)
- **Repo created and populated.** Public, `github.com/DavidFruin/ssapi`, fresh git history (no attempt to preserve `simple-social`'s commit history for these files).
- **Copied, not moved.** `simple-social` keeps its own copy of everything — nothing was deleted there. This is purely an additive split; whether `simple-social` should eventually stop tracking its own copy of the backend is a separate decision, not made yet.
- **What's here**: `api.php`, `media.php`, `auth.php`, `logging.php`, `schema.php`, `webpush.php`, `clean-notifications.php`, `migrate-posts.php`, `config.php`, the `src/{Auth,Comments,Follows,Media,Notifications,Posts,Users}/handlers.php` modules, `composer.json`/`.lock`, `.htaccess` (the Authorization-header passthrough fix + sensitive-file-blocking rules — the exact same class of fix [[ssreact]] had to independently rediscover as `CGIPassAuth On` for its own PHP-FPM vhost), and `.env.example`.
- **Tests removed 2026-09-30, moved to [[sstests]].** `tests/backend-tests/` briefly lived here (part of the initial copy from `simple-social`) but Dave asked directly for tests to never ship to the server — moved as-is into `sstests`' new `backend/` folder, nothing left here. Don't re-add a `tests/` folder to this repo without a specific reason to.
- **What's deliberately NOT here**: the vanilla-JS frontend (`js/`, `css/`, static `.html` pages, `manifest.json`/`sw.js`/`pwa-icons/`) — stays in `simple-social`, since that's the currently-deployed frontend, not backend. `ARCHITECTURE.md` and `notes.md` also stayed there since both describe the whole system, not just the backend. The frontend Playwright suite (`tests/front-end-test/`) stayed too — already known incompatible with `ssreact`, see that note.
- **No secrets in this repo.** `config.php` reads its JWT secret and other runtime config from a `private/.env` file that lives outside any repo, on the deployment host itself. `.env.example` only documents the variable names/shape. Verified directly (grepped for embedded-secret patterns) before the first push, since this repo is public.
- **Deployment target is unchanged for now.** `app.davidfruin.com`/`dev.davidfruin.com` still serve PHP from the same docroot they always have — splitting the source repo doesn't move where the code actually runs. Whoever eventually deploys from this repo needs to point at that same docroot (or wherever it lives by then).

## Decisions
- **Copy, don't move, and don't rewrite git history.** Simplest correct choice for "make these independently cloneable" — a `git filter-repo`-style history-preserving extraction was not attempted; this repo's history starts fresh from the 2026-09-30 copy.
- **GitHub Actions deploy pipeline: on hold, same as [[ssreact]].** Dave asked (2026-09-30) whether deploying both repos on push via GitHub Actions is feasible once split — yes, it's a normal two-workflow setup (see [[simple-social]]'s Planning section for the full answer) — but **do not build either workflow without Dave's explicit go-ahead.** This is a standing hold, not a "get to it eventually."

## Next steps
- [ ] Decide whether `simple-social` should eventually stop tracking its own copy of the backend files now that `ssapi` exists (not decided — currently both repos have a copy)
- [ ] GitHub Actions deploy workflow — on hold, see Decisions
- [ ] Whoever picks this up should read [[simple-social]]'s own Decisions/Planning sections first — this repo's own history doesn't carry the "why" behind the backend module split that happened before this split existed

## Links
- Repo: https://github.com/DavidFruin/ssapi
- Related: [[simple-social]] (where this came from, and the original vanilla-JS frontend that's still deployed today), [[ssreact]] (the React frontend this backend is meant to eventually serve alongside), [[sstests]] (every test suite for the project, including this repo's own backend tests)
