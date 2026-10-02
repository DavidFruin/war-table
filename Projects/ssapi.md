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
- **What's here**: `api.php`, `media.php`, `auth.php`, `logging.php`, `schema.php`, `webpush.php`, `config.php`, the `src/{Auth,Comments,Follows,Media,Notifications,Posts,Users}/handlers.php` modules, `composer.json`/`.lock`, `.htaccess` (the Authorization-header passthrough fix + sensitive-file-blocking rules — the exact same class of fix [[ssreact]] had to independently rediscover as `CGIPassAuth On` for its own PHP-FPM vhost), and `.env.example`. (`clean-notifications.php`/`migrate-posts.php` were here too, deleted 2026-10-02 — see below, both one-off scripts already run on dev and prod.)
- **Tests removed 2026-09-30, moved to [[sstests]].** `tests/backend-tests/` briefly lived here (part of the initial copy from `simple-social`) but Dave asked directly for tests to never ship to the server — moved as-is into `sstests`' new `backend/` folder, nothing left here. Don't re-add a `tests/` folder to this repo without a specific reason to.
- **What's deliberately NOT here**: the vanilla-JS frontend (`js/`, `css/`, static `.html` pages, `manifest.json`/`sw.js`/`pwa-icons/`) — stays in `simple-social`, since that's the currently-deployed frontend, not backend. `ARCHITECTURE.md` and `notes.md` also stayed there since both describe the whole system, not just the backend. The frontend Playwright suite (`tests/front-end-test/`) stayed too — already known incompatible with `ssreact`, see that note.
- **No secrets in this repo.** `config.php` reads its JWT secret and other runtime config from a `private/.env` file that lives outside any repo, on the deployment host itself. `.env.example` only documents the variable names/shape. Verified directly (grepped for embedded-secret patterns) before the first push, since this repo is public.
- **Deployment target is unchanged for now.** `app.davidfruin.com`/`dev.davidfruin.com` still serve PHP from the same docroot they always have — splitting the source repo doesn't move where the code actually runs. Whoever eventually deploys from this repo needs to point at that same docroot (or wherever it lives by then).

## Status (2026-10-02): security/speed improvement plan implemented
An agent wrote a full security+speed review of this backend as used by
[[ssreact]] — the plan lives in the war-table Inbox as
`ssapi-improvement-plan.md` — and a second agent (Sonnet 5, this
session) carried it out task by task per the plan's own rules (local
bench only, never deployed, one commit per task, stopped after each
phase to report). **22 commits landed, all pushed to `master`,
everything verified against a from-scratch local PHP bench** (`php -S`
+ seeded SQLite), not just read through.

**What changed, by category:**
- **Security:** push-subscription SSRF closed (endpoints restricted to real
  browser push services, keys validated, capped per user); media uploads
  now sniffed from real file bytes instead of trusting the client's
  Content-Type, staged outside the docroot; OTP codes/media filenames
  switched to a CSPRNG; OTP send endpoints throttled per address/IP;
  logout now verifies the JWT and actually drops the session's push
  subscription; pagination/id-list inputs clamped; config fails closed
  (500) instead of falling back to creating the DB/logs inside the
  docroot when `private/` is missing; debug logging off by default with
  a single rotating logger; security headers + a Report-Only CSP.
- **Correctness/integrity:** comments/likes on a nonexistent post now
  404 instead of silently trusting the post ID's own prefix; registration
  is now atomic with a real unique index on email; post/account deletion
  wrapped in transactions (no more orphaned comments, no shared media
  between two posts, folder cleanup that doesn't short-circuit);
  unfollowing someone you never followed no longer notifies them.
- **Speed:** schema migrations now run once per database via
  `PRAGMA user_version` instead of ~15 statements on every request;
  mention/comment-count lookups batched instead of one query per row;
  posts carry an embedded `commentCount` now (saves ssreact a second
  round trip); session lookup carries the caller's email so five+
  handlers stopped re-querying it; push notifications deferred to run
  after the response is already sent instead of blocking the request.
- **Cleanup:** the two already-run one-off migration scripts deleted,
  ~80 lines of stale "moved to src/..." comment debris removed,
  `handle_getMyPosts`/`handle_getUserPosts` deduplicated, `.gitignore`/
  `composer.json`/`.env.example` tidied (including a real gap:
  `VAPID_SUBJECT` was read by `pushNotification()` but never actually
  loaded from the environment until now).

**Explicitly skipped, per Dave's instruction:** everything marked GATED
or DECISION in the plan — the structural Tier-5 items (D1–D8: real
follows table, HttpOnly-cookie refresh tokens, dropping dead columns,
likeCount-not-likes-array, the simple-social/ssapi backend-drift
question) and S9's unlike/unfollow-notification dedup *choice* (A vs B;
the bug-fix half of S9 — don't notify an unfollow that never followed —
did land). None of these were touched at all.

**Also explicitly not done, and why:** `ssapi/.htaccess`'s conversion to
a strict PHP-entry-point allowlist — the plan requires Dave to run
`ls *.php` on el1 (app/dev docroots) first to confirm nothing else is
needed, and no agent has el1 access. The *shared* `ssreact/public/.htaccess`
(the one actually enforced on react.davidfruin.com, where no vanilla PHP
frontend exists) got the full allowlist + CSP + long-cache treatment,
verified for real against a locally-installed Apache instance.

**Dave-on-el1 checklist, carried over from the plan, nothing done yet:**
1. Run `sqlite3 <dev private dir>/userdata.db '.schema users' '.schema pending_users'` and share the output — this session used a from-source reconstruction instead (confirmed correct against `api.php`'s own `ALTER TABLE`/`INSERT`, but never checked against the real dev DB).
2. Before `ssapi/.htaccess` can get the strict PHP allowlist: `ls *.php` in the app/dev docroots.
3. `SELECT LOWER(email), COUNT(*) FROM users GROUP BY 1 HAVING COUNT(*) > 1` on real dev/prod data, before the new unique email index matters there (the migration skips itself safely if duplicates exist, but worth knowing).
4. After deploying the S2/S12/P7 `.htaccess` changes to react.davidfruin.com: rerun the plan's curl checks for real (403s, cache headers, gzip), and separately confirm `fastcgi_finish_request()` actually releases the client early under its real PHP-FPM setup — this session's bench (`php -S`, no FastCGI pool) couldn't demonstrate that for the deferred-push change.
5. Click through every ssreact page with DevTools open once the CSP is live, watching for Report-Only violations, before renaming the header to enforce it.
6. Decide the two live DECISION items this session skipped: S9 (drop vs. dedupe unlike/unfollow notifications) and the much bigger D-series questions (D6 especially: how/when ssapi replaces simple-social's backend copy on app/dev — prod gets none of this session's fixes until that happens).

## Decisions
- **Copy, don't move, and don't rewrite git history.** Simplest correct choice for "make these independently cloneable" — a `git filter-repo`-style history-preserving extraction was not attempted; this repo's history starts fresh from the 2026-09-30 copy.
- **GitHub Actions deploy pipeline: on hold, same as [[ssreact]].** Dave asked (2026-09-30) whether deploying both repos on push via GitHub Actions is feasible once split — yes, it's a normal two-workflow setup (see [[simple-social]]'s Planning section for the full answer) — but **do not build either workflow without Dave's explicit go-ahead.** This is a standing hold, not a "get to it eventually."

## Next steps
- [ ] Dave-on-el1 checklist above (schema dump, `ls *.php`, duplicate-email check, post-deploy curl/CSP/FastCGI verification)
- [ ] Decide S9 (drop vs. dedupe unlike/unfollow notifications) and the D1–D8 structural items — all currently gated on Dave
- [ ] Decide whether `simple-social` should eventually stop tracking its own copy of the backend files now that `ssapi` exists (not decided — currently both repos have a copy; D6 above is the sharper version of this question)
- [ ] GitHub Actions deploy workflow — on hold, see Decisions
- [ ] Whoever picks this up should read [[simple-social]]'s own Decisions/Planning sections first — this repo's own history doesn't carry the "why" behind the backend module split that happened before this split existed

## Links
- Repo: https://github.com/DavidFruin/ssapi
- Related: [[simple-social]] (where this came from, and the original vanilla-JS frontend that's still deployed today), [[ssreact]] (the React frontend this backend is meant to eventually serve alongside), [[sstests]] (every test suite for the project, including this repo's own backend tests)
