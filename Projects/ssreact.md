---
status: active
repo: https://github.com/DavidFruin/ssreact
---

# ssreact

## Summary
React + Vite + TypeScript + shadcn/ui (Base UI, Tailwind 4, pnpm) rewrite of [[simple-social]]'s vanilla-JS web frontend. It talks to the PHP backend, now [[ssapi]].
- **Feature-complete since 2026-09-30:** every page the original has, plus the landing and static pages, PWA, push, the session-expired modal and the mobile thumb-nav.
- **2026-10-01:** eleven rounds of visual and real-device polish.
- **Live for testing at `react.davidfruin.com`.** It isn't the production frontend yet; `app.davidfruin.com` still serves the vanilla app.

**Full history** (dated status logs, every polish round, the detailed bug writeups, verification transcripts): [[ssreact-history]]. Read this note first, and go there only when you need the detail behind something summarized here.

## Status (2026-10-02)
- All pages built and verified live against real (isolated test) data on react.davidfruin.com.
- **Not yet confirmed on a real device or in a real setting:**
  - real push delivery and the native install prompt;
  - Delete Account end to end;
  - copy-to-clipboard;
  - the fullscreen video layout fix.

  Camera and mic capture *has* been used on Dave's real phone.
- **Done 2026-10-02: every ssreact task in `Inbox/ssapi-improvement-plan.md`** (P1, P5, P6, P7, P8, S2/S12/P7 combined `.htaccess` pass, S15, C6) — 8 commits, all local-bench/Apache-verified, pushed. `.htaccess` is now the canonical, enforced shared-docroot file for react.davidfruin.com (full PHP allowlist, security headers, Report-Only CSP, long-cache for hashed assets). Main bundle ~27% smaller (lazy-loaded rare routes). Full writeup in [[ssapi]]'s note (most of the plan was backend work).
  - **Fixed P1, the bug behind an earlier wrong claim:** the "harmless 401, then silent refresh" pattern seen after every reload was recorded as expected behaviour — it was actually `api.ts` never loading the stored JWT on startup. Also meant logout after a reload didn't end the session on the server. Both fixed.
  - **Not verified in a real browser (none was available that session):** the CSP's live effect on every page (Report-Only, so nothing breaks either way) and that each lazy-loaded page actually renders post-chunk-load. Click through once deployed, before enforcing the CSP.
- **Deployed live 2026-10-02 (Citadel, el1 access):** the build above (`865ff6f`) is now actually running on react.davidfruin.com, along with ssapi's full security/speed pass — see [[ssapi]]'s note for the full deploy writeup (backups, bench verification, what was checked before touching the live host). Confirmed live: real browser session, feed, posting/deleting a comment through the actual client, every sensitive-path curl check 403ing, gzip, long-cache on hashed assets. Lazy-loaded pages were specifically confirmed to render post-chunk-load this time (the previous session couldn't check this).
- **Next:** the phone app ([[ssreact-native]]). Its Phase 1 moves ssreact's browser-free logic from `src/lib` into `src/core` (`Inbox/ssreact-native-port-plan.md`).
- **Later:** the invite-code field, admin pages, and report/block/terms UI (`Inbox/access-and-public-launch-plan.md`).

## Hosting / deployment
*(Rewritten 2026-10-07; the old react.davidfruin.com details are in [[ssreact-history]].)*
- **Live on prod and dev since 2026-10-06**, release **2.0.0 "Elia"**: `app.davidfruin.com` and `dev.davidfruin.com` both serve this build from `public_html/app/`, with [[ssapi]] as the backend, in the split layout ([[deploy-layout-plan]]). The vanilla frontend is retired. **react.davidfruin.com is retired**; Dave is deleting that vhost.
- **Layout per domain:**
  - `private/` (`.env`, DB, logs);
  - `ssapi/` (backend code, outside the web root);
  - `public_html/`: the `api.php`/`media.php` stubs, one root `.htaccess` from `ssapi/deploy/root.htaccess`, `app/` (this build), `media/`, `downloads/`.

  ssreact ships **no** `.htaccess`. The root file does the SPA fallback, security headers and caching.
- **Deploy the frontend (current, before the workspace restructure):**
  ```
  pnpm run build
  rsync -rltz --no-owner --no-group --delete dist/ el1:/home/davidfruin/domains/<dev|app>.davidfruin.com/public_html/app/
  ```
  - Prod only on Dave's explicit go.
  - `--no-owner --no-group` still matters: `-a` resets group ownership that PHP needs.
  - Never rsync `--delete` from an empty or wrong folder.
  - **After the workspace restructure** ([[repo-consolidation-plan]] Step 2), the output is `web/dist/`; use `scripts/deploy-web.sh dev|app` (Step 2.7) instead of this command.
- **Rollback anchor:** tag `v2.0.0` (ssreact `466367f`, ssapi `236a338`). On `master` since then: `0379b5b` (version display + `/history`), not recorded as deployed.
- **Backups from the 2026-10-06 switch:** dev `app.vanilla.bak-2026-10-06`; prod `public_html.bak-2026-10-06` and `private/userdata.db.pre-ssapi-migration-20261006`. All are outside the web root. Delete them after a few days of clean running.
- **Not yet checked on prod with real use:** a browser login, a post with a photo, push delivery, and an old home-screen icon (`/app.html#/feed` should land on `/feed`).
- **Hosts:** app and dev run PHP through mod_fcgid, and have ffmpeg (thumbnails and transcoding work). The PHP-FPM `CGIPassAuth` and chroot-without-ffmpeg quirks were react-only.
- **GitHub Actions deploy: decided, but ON HOLD until Dave says go.**

## Testing
- **Test accounts:** test on **dev** (react.davidfruin.com and its test DB are retired). Create a throwaway account in dev's DB with `password_hash()` + `INSERT`, or register with an invite once invites exist.
  - Passwords are never recorded (this vault is public).
  - Leave data as you found it.
- **The old Playwright suites** (`simple-social/tests/front-end-test/` and [[sstests]]' E2E suite) target the **retired vanilla frontend**: hash routes, a different DOM. Verification has been by manual or headless-browser walkthrough against real data. A new suite is an open idea, not a plan.
- **The automated browser sandbox can't** grant camera, mic or notification permission, or do fullscreen and clipboard writes. To exercise capture, inject a synthetic `getUserMedia` stream; anything else needs a real device. **Build and deploy before verifying:** testing against stale deployed code looks exactly like a real failure.

## Decisions
- Reuse the `learn-react-site` scaffold (the stack Dave already chose). No need to keep the retro-BBS look; shadcn's design is fine.
- **The dev backend is reached through Vite's proxy** (`/api.php` and `/media.php` → `dev.davidfruin.com`), not CORS. The app always uses relative paths. **Never point it at `app.davidfruin.com` (prod).**
- **Clean URLs with React Router**, not hash routing.
- **Themes are `.theme-<name>` classes on `<html>`, mapped onto shadcn tokens.** Light (default), Dark Blue (`dark`), Red, Light Blue (`blue`), Hacker, and grayscale Dark (`gray`), which keeps red/green/amber for status colours. `simple-social/css/main.css` is the source of truth for the hex values. The backend's `$allowedThemes` includes `gray`.
- **API inconsistencies are documented, not "fixed" on the wire.** They're normalized at the `api.ts` boundary (see Gotchas).
- **Static API docs** are the original `api.html` injected with `?raw` + `dangerouslySetInnerHTML`. That's safe because the content is static and authored here.
- **Kept deliberately from the original:**
  - Search filters the full `getUsers()` list on the client, with no per-keystroke endpoint.
  - The session-expired modal uses a **queue of waiters**, not a single slot. A single slot orphans concurrent callers, and pages hang on "Loading…".
  - `postCache` / `feedCache` are trusted without revalidation.
- **Not done on purpose:**
  - A rotate button for video: it can't actually rotate the pixels without ffmpeg, so it would only fake it.
  - The `--primary-hover` variable was dropped.
- The roadmap data in `roadmap-data.ts` is shared with the original; keep them in sync rather than forking it.

## Gotchas (learned the hard way; full writeups in [[ssreact-history]])
- **Posts arrive with `userID` (capital D).** Every post-returning method goes through `normalizePost()` in `api.ts`, so new endpoints must too.
- Posts use camelCase. Comments and notifications use snake_case (`user_id`, `created_at`).
- **Post IDs** are strings shaped `"<userId>.<unixTime>"`.
- **Timestamps** look like `"YYYY-MM-DD HH:MM:SS"`; always use `timestamp.replace(' ', 'T')` before `new Date()`.
- `likeCount` and `isLiked` are worked out from the `likes` array.
- **Mentions** are `@[id]` tokens plus a `mentions` array. URLs are matched first, so a mention inside a URL isn't linked. A `null` email renders as "@deleted user".
- **Push URLs from the backend** are `/app.html#/post/…` (for the vanilla app). `public/sw.js` strips everything up to the `#`. Don't change the backend's URLs.
- **The service worker** fetches navigations and same-origin JS/CSS network-first, to avoid the stale-JS incident from simple-social's ARCHITECTURE.md. It only registers in production builds.
- **Push permission:** `Notification.requestPermission()` must be the first `await` in the click handler.
- **React stale closures:** timers and intervals must read refs or native objects (e.g. `recorder.state`), never React state captured when the timer was set up. This was the real bug behind recordings not stopping at the limit.
- **Recorded WebM has `duration = Infinity`.** Both `VideoPlayer` and `AudioPlayer` seek to the end to force the real duration, then reset React's `currentTime` themselves.
- **Only one media player at a time** (`active-media.ts`), driven by the user's own play actions, not `play` events.
- **Layout traps:**
  - Flex columns stretch buttons, so standalone buttons need `self-start`.
  - Email text next to a fixed element needs `min-w-0` + `truncate`.
  - `EmailLabel` must be block-level `flex`, not `inline-flex`.
  - Range inputs need `min-w-0`.
- **Media display:**
  - Inline images and video use `object-contain` with a `bg-card` background, because `object-cover` crops by different amounts at different widths.
  - The selfie camera preview is mirrored with CSS only; the saved file isn't affected.
- **Base UI `Select`** needs an `items` prop, or the closed control shows the raw value instead of the label.
- **Toasts** use solid fills (`bg-destructive`/`bg-success`), because tinted backgrounds vanished in the Red theme.
- Newlines are blocked in posts and comments, matching the server's character filter, and the user gets a toast explaining why.
- Recording limits and the frame-rate cap come from the server's `getMediaLimits`, with a fallback.
- The client-side duration check before upload exists because the chroot can't check on the server. A direct API call still gets around it.
- The notification badge refreshes straight away after "Mark as Seen" (`refreshUnseenNotificationCount()`), not only on the 60s poll.

## Architecture reference
- `src/lib/api.ts`: the singleton API client.
  - Every action is a `POST` with `action=<name>` as form fields; uploads go to `/media.php` as `FormData`.
  - `Authorization: Bearer <jwt>`.
  - Responses look like `{valid, message?|error?}`.
  - A 401 leads to one shared refresh, then a retry, then the re-login modal.
- `src/lib/auth-context.tsx` handles user state, themes and the session-expiry queue.
- `src/lib/types.ts` holds the wire types.
- **Routes:**
  - Public: `/about`, `/api`, `/conduct`, `/roadmap`, `/download`.
  - Guest-only: `/`, `/login`, `/register`, `/reset-password`.
  - Logged-in: `/feed`, `/create-post`, `/profile[/:id]`, `/post/:id`, `/notifications`, `/settings`, `/search`.
  - Anything else redirects to `/feed`.
- **Layout:** Header (desktop nav + badge), ThumbNav (touch devices only, mirrors for the left-hand setting), ScrollTopButton, Toast.

## Next steps
- [ ] Restructure into a pnpm workspace (`web/`, `mobile/`, `desktop/`, `packages/core/`). The `deploy-layout` branch is already merged (2026-10-06, `466367f`; ssreact ships no `.htaccess` now): [[repo-consolidation-plan]] Step 2. **This must happen before the phone port starts.**
- [x] ~~ssreact tasks from `Inbox/ssapi-improvement-plan.md`~~: done 2026-10-02
- [ ] `packages/core` (`@ss/core`) for the phone app ([[ssreact-native-port-plan]] Phase 1, after the workspace restructure)
- [ ] **Prod real-use check after the 2.0.0 switch** (login, photo post, push, old home-screen icon)
- [ ] Real-device checks: push delivery, install prompt, Delete Account with a disposable account, clipboard, fullscreen
- [ ] Invite-code field + `/admin/invites`; later report/block/terms (`Inbox/access-and-public-launch-plan.md`)
- [ ] GitHub Actions deploy, **on hold until Dave says go**

## Links
- Repo: https://github.com/DavidFruin/ssreact
- History: [[ssreact-history]]
- Related: [[simple-social]], [[ssapi]], [[ssreact-native]], [[sselectron]], [[sstests]], [[simple-social-tui]]
