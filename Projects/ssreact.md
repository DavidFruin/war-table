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
- **Changing soon (decided 2026-10-02):** the build will deploy to `public_html/app/` with a plain `rsync --delete`, and the root `.htaccess` will come from ssapi's `deploy/`, so ssreact will ship no `.htaccess` (task L4, merged on migration day). See [[deploy-layout-plan]]. Until react is migrated, the command below is still the correct one.
- **Target:** `react.davidfruin.com` → `/home/davidfruin/domains/react.davidfruin.com/public_html` on `el1`.
  - It's a **static build**: only the *contents* of `dist/` go in `public_html`.
  - `public/.htaccess` does the SPA rewrite. If `/feed` 404s on refresh, check that this file was deployed first.
- **Same docroot as a copy of the PHP backend** (copied 2026-09-29, superseding the earlier reverse-proxy plan): `api.php`, `media.php`, `config.php`, the `src/` handlers, `vendor/` and so on, plus uploaded `media/`.
- **Isolated test data:**
  - `react.davidfruin.com/private/userdata.db` is a copy of dev's SQLite DB, **never prod's**.
  - `private/.env` holds its **own** generated JWT secret, so tokens only work on this host.
- **Manual deploy, the only correct command.** Two real bugs came from simplifying it:
  ```
  pnpm run build
  rsync -rltz --no-owner --no-group --delete \
    --exclude='api.php' --exclude='media.php' --exclude='config.php' --exclude='auth.php' \
    --exclude='logging.php' --exclude='schema.php' --exclude='webpush.php' --exclude='clean-notifications.php' \
    --exclude='composer.json' --exclude='composer.lock' --exclude='vendor/' --exclude='src/' \
    --exclude='media/' \
    dist/ el1:/home/davidfruin/domains/react.davidfruin.com/public_html/
  ```
  - `-a` would reset `public_html`'s group, which breaks PHP-FPM writes. The directory needs `chgrp davidfruin` + setgid, which is already set.
  - Without `--exclude='media/'`, `--delete` wipes every uploaded file.
  - **Known risk:** `dist/.htaccess` replaces the backend's `.htaccess` in this shared docroot. That's why the ssapi plan's S2 merges the two.
- **Host quirks:**
  - PHP runs through **PHP-FPM over a unix socket**, so the vhost needs `CGIPassAuth On`, which is already set. Without it, the `Authorization` header never reaches PHP and every authenticated call returns 401. app and dev use mod_fcgid and don't need it.
  - PHP-FPM runs in a **chroot without ffmpeg/ffprobe**. So there are no video thumbnails, no transcoding, no server-side duration check (it fails open), and no video rotate. This needs an infrastructure fix, not app code.
  - Files copied over with `scp` as root end up owned `root:root`. That's fine for read-only PHP files; watch out if anything ever needs write access.
- **GitHub Actions deploy: decided, but ON HOLD until Dave says go.** It must use the exact rsync command above. Whoever has `el1` access builds and verifies it; it must not be written blind.
- **The endgame:** `app.davidfruin.com` serves this build directly, same-origin with the backend. No migration date has been set.

## Testing
- **Test accounts on react's DB:**
  - `e2e-test@ssreact.local` (id 32) and `e2e-test-2@ssreact.local`, created directly with `password_hash()` + `INSERT`.
  - Passwords are not recorded (this vault is public). Make a new account the same way if you need one.
  - Leave data as you found it.
- **The old Playwright suite** (`simple-social/tests/front-end-test/`) is **incompatible**: wrong domain, hash routes, a different DOM. Verification has been by manual or headless-browser walkthrough against real data. A new suite is an open idea, not a plan.
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
- [ ] ssreact tasks from `Inbox/ssapi-improvement-plan.md` (in progress)
- [ ] `src/core` extraction for the phone app ([[ssreact-native]] plan, Phase 1)
- [ ] Real-device checks: push delivery, install prompt, Delete Account with a disposable account, clipboard, fullscreen
- [ ] Invite-code field + `/admin/invites`; later report/block/terms (`Inbox/access-and-public-launch-plan.md`)
- [ ] GitHub Actions deploy, **on hold until Dave says go**

## Links
- Repo: https://github.com/DavidFruin/ssreact
- History: [[ssreact-history]]
- Related: [[simple-social]], [[ssapi]], [[ssreact-native]], [[sselectron]], [[sstests]], [[simple-social-tui]]
