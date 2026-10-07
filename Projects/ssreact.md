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
- **Repo layout (2026-10-07):** a pnpm workspace. The web app is in `web/`, the shared logic is in `packages/core/` (`@ss/core`); `mobile/` and `desktop/` come later. Run commands from the repo root.
- **Live:** app.davidfruin.com (prod) and dev.davidfruin.com, both on the split layout ([[deploy-layout-plan]]): the build goes into `public_html/app/`, the backend is in `ssapi/`, one root `.htaccess` from ssapi's `deploy/`. ssreact ships no `.htaccess`. react.davidfruin.com is retired.
- **Frontend deploy:** `scripts/deploy-web.sh dev|app`. It builds `web/`, checks `web/dist/index.html` exists, and runs `rsync --delete` into `public_html/app/`. Prod asks for a typed "prod". Never run the old `dist/` command.
- **Backend deploy:** see [[ssapi]] and [[deploy-layout-plan]] section 2.
- **Rollback anchor:** tag `v2.0.0` in ssreact and ssapi. Prod's pre-migration backups are listed in [[ssapi]].
- **Host quirks:** files copied from this machine land owned by root; `chown -R davidfruin:davidfruin` the `app/` folder afterward. PHP runs under mod_fcgid on dev and prod.
- **GitHub Actions deploy: still on hold** until Dave says go. The existing workflow only builds.

## Testing
- **Test accounts:** test on **dev** (react.davidfruin.com and its test DB are retired). Create a throwaway account in dev's DB with `password_hash()` + `INSERT`, or register with an invite once invites exist.
  - Passwords are never recorded (this vault is public).
  - Leave data as you found it.
- **The old Playwright suites** (`simple-social/tests/front-end-test/` and [[sstests]]' E2E suite) target the **retired vanilla frontend**: hash routes, a different DOM. Verification has been by manual or headless-browser walkthrough against real data. A new suite is an open idea, not a plan.
- **The automated browser sandbox can't** grant camera, mic or notification permission, or do fullscreen and clipboard writes. To exercise capture, inject a synthetic `getUserMedia` stream; anything else needs a real device. **Build and deploy before verifying:** testing against stale deployed code looks exactly like a real failure.

## Decisions
- **2026-10-07 (Dave): React Native and Electron are both bridges** (toward true native apps later), **so the three apps stay separate:** `web/` (its own shadcn UI), `mobile/` (its own React Native UI) and `desktop/` (a thin Electron wrapper around the web build). They share **only plain logic** through `packages/core` (`@ss/core`). **No universal / react-native-web UI**: that was considered and rejected, because it would make React Native the long-term base.
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

## Phone app, Phase 1 done (2026-10-07)
- `@ss/core` (`packages/core/`) holds the logic that doesn't depend on the browser: types, format helpers, the toast store, post/feed caches, the media-limits loader, the post-text tokenizer, the injectable API client (`createApiClient` with endpoints, a token store and extra headers passed in), the session-expiry queue and `resolveMediaUrl`. `web/` imports it; `getMediaDuration` stays in web because it needs a `<video>` element.
- **Core stays portable by lint, not by types:** its tsconfig keeps the DOM lib (fetch/FormData types are needed and exist in React Native), and ESLint `no-restricted-globals` bans `window`, `document`, `localStorage` and friends, `no-restricted-imports` bans React and anything from the apps. Verified that the rule fires. Core has no runtime dependencies.
- 23 Vitest tests cover the tokenizer, formatting, the session-expiry queue and `resolveMediaUrl` (`pnpm test`).
- Verified the web app is unchanged with a headless browser pass against a local ssapi bench: login, reload with no 401s, posting a link (trailing dot stays outside the link), forcing session expiry then re-login via the modal, Settings version block, logout. Deployed to dev.davidfruin.com on 2026-10-07 (prod not yet). Dave wants every change deployed to dev and tested there as the work goes.
- Next: Phase 2 (create `mobile/` with Expo). Needs Dave's phone with Expo Go for the checks; see the plan's 4a.

## Phone app, Phase 2 done and checked on Dave's phone (2026-10-07)
- `mobile/` is an Expo SDK 57 app (expo-router, `src/app/`), package `@ss/mobile`, using `@ss/core` through the workspace. Config is `mobile/app.config.ts`: API base from `EXPO_PUBLIC_API_BASE`, default `https://dev.davidfruin.com`.
- Platform adapters in `mobile/src/platform/`: `tokenStore.ts` (in-memory map hydrated once at boot from expo-secure-store, writes go through in the background) and `api.ts` (client with the `X-Client: ssreact-mobile/<version> (<os>)` header). The root layout holds the splash screen until the store is hydrated. The index screen is a **temporary debug screen** (email + password, logs in against dev and shows `getMediaLimits`); Phase 3 replaces it. **Deviation from plan 2.5:** the Theme, Auth and Toast providers are not built yet; they arrive with Phase 3's screens.
- **React isolation checked:** mobile uses Expo's React 19.2.3 (hoisted at the root), web keeps its own 19.2.5 in `web/node_modules`. `@ss/core` has no React.
- Use `npx expo install <pkg>` in `mobile/` for Expo packages. `expo lint` auto-installed ESLint 9 + eslint-config-expo into `mobile/` on first run.
- Verified by the agent: typecheck, lint (core, web, mobile), 23 core tests, web build, and Android and iOS JS bundles export cleanly (Metro resolves `@ss/core`). **Dave confirmed in Expo Go (2026-10-07):** the app opens, and the debug login against dev returns the media limits. Not yet checked: live reload when editing core, and relaunch behaviour of the stored login. To run it: `cd mobile && npx expo start --tunnel` (needs `@expo/ngrok`, installed globally with npm, no sudo), then open the `exp://` address in Expo Go.

## Phone app, Phase 3 done and checked on Dave's phone (2026-10-07)
- **Screens:** Login, Register (with an invite-code field; `sendRegisterOTP` now takes an optional `inviteCode`, ignored by a server without the invite system) and Reset Password (shared `OtpAuthFlow`, same three steps and wording as web). `app/(auth)` redirects logged-in users to the feed; `app/(tabs)` redirects logged-out users to login.
- **Shell:** bottom tabs Feed, Post, Search, Notifications, Profile (placeholders until Phase 4+), plus a hidden Settings tab reached from Profile. Settings is minimal for now: theme picker, Log Out and (dev builds only) an "Expire my session" button. The rest of Settings is Phase 7.
- **Themes:** all six in `mobile/src/theme/tokens.ts`, generated from `web/src/index.css` (the status colours a theme doesn't override inherit from light, as in the CSS). The theme is the logged-in user's saved choice, saved with `api.updateTheme`; logged out is always light; the status bar follows the theme.
- **Session expiry:** a non-dismissible Modal driven by the shared `@ss/core` queue (`SessionExpiredHost` sits inside the ThemeProvider so it's themed).
- **[ssapi] deployed to dev only (2026-10-07):** `deviceNameFromUserAgent($ua, $client)` recognises `X-Client: ssreact-mobile/<v> (android|ios)` and names the session "Simple Social app (Android|iOS)"; anything else falls back to the User-Agent. Checked on a local bench (valid android, valid ios, malformed header, no header). Prod doesn't have it.
- `typedRoutes` is off in `app.config.ts` (the generated types got stale as routes were added).
- Verified by the agent: typecheck, lint, 23 core tests, web build, Android bundle export. **Needs Dave's phone:** see the Phase 3 checks in the port plan.

## Phone app, Phase 4 done and checked on Dave's phone (2026-10-07)
- **Read screens, all in `mobile/src/app/(tabs)/`:** Feed (FlatList, 25 per page, pull to refresh, loads the next page at the end), Post (`post/[id]`, uses the cached post when the feed already loaded it, comments newest first, delete your own), Profile (own tab and `profile/[id]`, followers/following sheets, Follow/Unfollow, their posts), Notifications (Today/Yesterday/Earlier, same wording and targets as web, Mark as Seen) and Search (one `getUsers()` fetch filtered on the device, Follow toggles).
- **Shared pieces:** `PostCard` (like, likes sheet, delete with an Alert, six-line clamp with Show more), `PostText` (draws `@ss/core`'s tokens: links via `Linking`, mentions open the profile), `MediaView` (images contain-fit, tap for full screen; **video and audio show a "coming soon" placeholder until Phase 5**), `Sheet`/`PersonListSheet`. The post and profile screens are hidden tabs so they sit under the auth guard; Tabs uses `backBehavior="history"` so Back returns to where you came from.
- Not in this phase on purpose: writing comments and posts (Phase 5), the notification badge count (Phase 6).
- `mobile/eslint.config.js` turns off `react-hooks/set-state-in-effect` (the standard fetch-on-mount pattern trips it; web's lint doesn't cover TS files so it never mattered there).
- Verified by the agent: typecheck, lint, core tests, Android bundle. **Needs Dave's phone:** every screen against real dev data, like/unlike, mention and link taps, the URL-with-`@[26]` case (unit-tested only).

## Phone app, Phase 5 built, waiting on Dave's phone check of part B (2026-10-07)
- **Part A (checked by Dave, after one fix):** the Post tab composer with @mention suggestions (`MentionInput`, same `@email` to `@[id]` resolve, printable-ASCII/Latin-1 filter and "Returns aren't allowed" as web), 5000-character counter, draft in AsyncStorage (`ss_post_draft`, same shape as web, includes attached media), one library photo or video (`expo-image-picker`), Rotate for photos, Remove, comment writing on the post screen, and a `ToastHost` (solid fills). Photos are shrunk to the server's `maxSide` and saved as JPEG 0.92 (`expo-image-manipulator`); rotation is previewed and applied once at submit; a library video over `maxSeconds` is refused before upload (picker duration is ms).
- **Upload gotcha (fixed 2026-10-07):** Expo's fetch (SDK 57) rejects React Native's old `{ uri, name, type }` FormData part with "Unsupported FormDataPart implementation", which our client reported as "Could not reach the server". Uploads must be a Blob: on the phone, `new File(uri)` from `expo-file-system`. `UploadFile` in core is now just `Blob`, and the "could not reach the server" error now includes the underlying reason.
- **Part B (built, not yet checked on a phone):** playback (`VideoCard`: thumbnail with a play button, `expo-video` player with native controls and fullscreen mounted on tap; `AudioBar`: `expo-audio` play/pause, tap-to-seek bar and elapsed time; `platform/active-media.ts` makes only one play at a time) and capture (`CaptureModal`: Photo / Video / Audio tabs, front/back flip, video stops at `maxSeconds` through the camera's own `maxDuration`, audio through a timer that calls the latest stop function via a ref, permission prompts, closing mid-recording discards the clip). Captured photos go through the same resize/rotate path as library photos.
- **Server side:** ssapi accepts any media format now (see [[ssapi]]), so phone recordings (M4A, MP4/MOV) need no special handling.
- **Web follow-up, not done:** web's file picker still has the old `accept` list (`MEDIA_ACCEPT` in `CreatePostPage.tsx`).

## Phone app, Phase 6 built (2026-10-07); real push needs Dave's setup
- **Works in Expo Go (checkable now):** the Notifications tab shows the unseen count as a badge, refreshed every 60 s while the app is open and whenever it returns to the foreground (never in the background), and "Mark as Seen" clears it at once (`notifications/UnseenProvider.tsx`). Settings has a "Push Notifications" switch that explains why it can't be turned on in Expo Go.
- **Needs a development build (not checkable in Expo Go):** registering the Expo push token (asked for only from that Settings switch), receiving a push, tapping it to open the post/profile (`NotificationTaps`, via `@ss/core`'s tested `notificationPath()`, which accepts only post/profile/feed/notifications paths), and the app-icon badge. `expo-notifications` is loaded lazily and never in Expo Go (it only logs errors there). Logging out forgets the stored token (the server also drops it).
- **Dave's setup for real push (credentials, not the agent):** an EAS project (`eas init`, which puts a `projectId` in the config; until then the switch says push isn't set up), a Firebase project with `google-services.json` plus the FCM V1 service-account key uploaded through `eas credentials` for Android, and for iOS the Apple Developer account and an APNs key (EAS can create it on the first iOS build). None of those files go in the repo. Then `eas build --profile development -p android` and install it. `eas.json` doesn't exist yet (Phase 8.3).
- **Server side:** see [[ssapi]] (Expo push, dev only).

## Next steps
- [x] ~~Restructure into a pnpm workspace~~ done 2026-10-07 (`web/` now; `mobile/`, `desktop/`, `packages/core/` to come): [[repo-consolidation-plan]] Step 2. **This must happen before the phone port starts.**
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
