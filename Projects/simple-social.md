---
status: active
repo: https://github.com/DavidFruin/simple-social
---

# simple-social

## Summary
Small social app: PHP API backend + web frontend (PWA), live at app.davidfruin.com (dev at dev.davidfruin.com). Terminal front ends are separate repos: [[simple-social-cli]], [[simple-social-cli-interactive]] (wizard), [[simple-social-tui]].

## Principles
Always faster, safer, less code, simpler, cleaner looking, easier to understand.

## Decisions
- Keep the CLI, TUI and API backend separate from the client UI layer. Only the interactive CLI and the TUI are built on top of the CLI. **Superseded 2026-09-24:** web and mobile no longer have to be separate — see Planning section below. The React Native app is explicitly meant to reuse from the web frontend.
- **`/users` returning every user's email is being left as-is — very likely by design, not a leak.** Investigated: `getUserInfo` already lets any authenticated user fetch any single user's email given only a `userId`, no follow/ownership check; `getUserPosts` has the same shape (public-profile app — following curates the home feed, it isn't a privacy boundary). `/users` returning the same data in bulk doesn't expose anything not already reachable one call at a time, and `search.js` is built directly on the bulk version (there's no separate username field — email is the only identifier people search/follow/mention by), so restricting it would break search for no real reduction in exposure. If this is worth revisiting, the actual question is whether the app should have a username distinct from email at all — that's a product decision, not a bug fix.

## Planning — future architecture (2026-09-24)

Long-term direction for the backend and client stack, worked out with `/grill-me`. This fleshes out ideas already sitting in the app's own `notes.md` wishlist (detach backend from frontend, split into services, eventual CI/CD) rather than replacing them.

**Backend — modular monolith, not microservices (yet)**
- Stay PHP. Refactor the existing `api.php`/`media.php` in place (strangler-style), not a from-scratch rewrite.
- Split into modules, each its own folder: `auth/`, `users/`, `posts/`, `comments/`, `follows/`, `notifications/` (includes push-subscription registration), `media/`. Can split further later; deliberately not going to separately-deployed microservices unless it's actually needed.
- Composer + PSR-4 autoloading (`App\` → `src/`) landed 2026-09-24 (commit `ef9ab2d`) — autoloader skeleton only, no dependencies yet, ready for the module split. `vendor/` is gitignored and blocked in `.htaccess` (docroot == repo root, so it'd otherwise be web-reachable).
- Known issues get fixed as part of this migration, not deferred — including the posts-JSON-blob-to-real-table migration (see Open — database), which lands *before* the folder reorg.
- Deploy stays manual `git pull` for now. Moving to GitHub Actions deploying to the same server is the agreed direction, but that pipeline is its own separate planning effort.

**Frontend — TypeScript + Vite + shadcn**
- Full recreation of the web frontend (currently vanilla JS, no build step, no TS) in TypeScript/Vite/shadcn.
- No need to preserve the current retro-BBS visual identity — that's covered by the terminal clients (CLI/wizard/TUI) instead.
- Backend goes first; frontend rewrite starts once the backend's structure/API surface has settled.

**Mobile — React Native as a bridge, not the destination**
- A React Native app, built from/sharing with the new web frontend, is the step after the web rewrite — explicitly a bridge toward eventual true-native (Swift/Kotlin) apps, not the end state.
- How much actually gets shared with web (just the logic/API-client layer vs. UI-level sharing via a cross-platform kit) is deliberately left open until the React migration itself starts.

- [x] **Backend module split — done 2026-09-24, all 7 modules deployed to dev and verified.** Ran via looped `/split-backend-modules` invocations, one module per iteration (move → verify locally → commit → push → deploy to dev → run test suite): auth (`0868f0b`), users (`543f515`), posts (`c6e4e9f`), comments (`be056e8`), follows (`f16bb33`), notifications (`5c93b9b`), media (`a3df6b1`). `api.php`/`media.php` are now thin — only `handle_deleteAccount` still lives directly in `api.php` (deliberately, cascades across every domain). Real bugs the move itself caught, not just reorganized code:
  - `__DIR__`-based file paths (post-deletion media cleanup, and media's own directory layout) meant the old file's directory and silently broke once the code moved to `src/<Module>/` — fixed with `__DIR__ . '/../../...'`, verified against `realpath()` and, for Media, a live upload+fetch+delete round-trip against dev.
  - `media.php` is a separate entry point with its own `respond`/`bad`/`good`/`db`/`logMsg`/`requireAuth` (different implementations than `api.php`'s, same names) — those stayed put rather than moving into the shared Composer autoload, which would have been a straight function-redeclaration fatal the moment both entry points loaded it.
  - Plain global variables (`$allowedImageTypes` etc., `$maxSizes`) don't survive Composer's `files` autoload (it runs each file inside its own loader function, so top-level `$var = ...` never reaches true global scope) — converted to `const`s or a small function as appropriate.
  - The original module-boundary list here never assigned `likePost`/`unlikePost`/`getPostLikes` anywhere — corrected to live in Posts (post-scoped data, shares `getLikesForPostIds()`).
  - Test suite showed some flakiness unrelated to any of this (`search.spec.js`, `session-expiry.spec.js`'s re-login test) — confirmed via direct API calls that the backend data was correct each time; logged in Open — bugs rather than chased mid-migration.
- [x] Land the posts-table migration to dev and prod (done 2026-09-24, see Done)
- [ ] Plan the GitHub Actions deploy pipeline
- [ ] TS/Vite/shadcn frontend rewrite
- [ ] React Native app — decide code-sharing approach then

## Open — features
- [ ] Spinner loading: home screen on first load, media uploading, posts loading on a page
- [ ] Better video controls (custom controls — also fixes the clipped playback-speed menu below)
- [ ] Notifications when someone comments on a post you've commented on
- [ ] Change the style and color of the theme selector
- [ ] Media limits set in a config file
- [ ] Page listing upcoming features with ETAs

## Future ideas
- **Make dev visibly distinct from prod for real users** — prompted by a real user mixing up dev and prod (2026-09-24, see Done). They look identical today; something as simple as a banner or a different accent color on dev would prevent this.
- Lock the site / log out all users for scheduled maintenance
- Admin panel for freezing or deleting users (abuse etc.)
- Chat, including group chats
- Complaints and ideas page: report a bug, report a person, suggest an idea
- Maybe require a referral code to register
- Language switcher
- **Universal install script** (Dave, 2026-09-24): one `.sh` that installs the terminal client(s) on any Linux distro, auto-detecting the package manager instead of the current manual apt-vs-pacman split. `download.html`'s TUI section currently handles this with an explicit LMDE 7 / Omarchy Quattro tab toggle (added 2026-09-24) rather than detection — this would replace/automate that.
- **Electron desktop app** (Dave, 2026-09-24): `download.html` already has a "Desktop" section with "Installation instructions are coming soon" as a placeholder — this is what fills it in.

## Open — bugs
- [x] **`search.spec.js` flakiness — fixed 2026-09-24 (`e1b0700`).** Real bug, not test timing: `search.js`'s `render()` never awaited the user-list fetch before wiring up the search input, so typing a query before that fetch resolved filtered an empty list and showed a false "no users matching" state. Fixed by re-running the filter once the list actually loads, if a query's already typed. Verified with 3 clean full-file runs against dev after the fix (previously failed intermittently even alone).
- [x] **`session-expiry.spec.js`'s re-login test — fixed 2026-09-24 (`e1b0700`).** Real concurrency bug: page load fires more than one authenticated request at once (main.js renders the feed while header.js's own separate `DOMContentLoaded` handler starts polling notifications) — with an expired session, both can 401 close enough together that both call `SessionExpiredModal.show()`, which only had room for one pending caller. The second silently overwrote the first's resolve/retry, so that caller's promise never resolved — exactly the "stuck on Loading forever" symptom. Modal now queues every concurrent caller instead of one slot. Verified with 4 clean full-file runs against dev after the fix.
- [ ] **`media-capture.spec.js` still fails (3 tests) — environment, not chased.** Consistent across every run this session regardless of any backend/frontend change, config already passes `--use-fake-device-for-media-stream`. Likely something missing in this machine's headless-Chromium setup for fake camera/mic devices, not an app bug — not investigated further.
- [ ] **Dark mode: illegible blue link text** — not a themed color, it's the browser's unstyled default blue on three links with no CSS class: "Don't have an account? Register" / "Forgot password?" on the login page, "Already have an account? Login" on the register page. Everything else (nav, static pages, mentions, post links) already gets a theme-aware color. Narrow, cheap fix — just give those three a class.
- [ ] Can't see all playback speed options — it's the browser's native video menu, clipped by the video box. Needs custom video controls.
- [ ] 4px horizontal overflow on phones from `.notif-badge` (low importance)
- [ ] Flash when expanding a post — doesn't reproduce. Possibly images shifting the page as they load (no reserved space). Confirm with Dave what he saw.
- [ ] Coming back from an expanded post lands a little below where you were — doesn't reproduce, scroll restore measured correct.
- [ ] **Clean URLs (fix hash routing):** want `dev.davidfruin.com/feed` instead of `/app.html#/feed`. Needs an Apache rewrite (non-file paths serve `app.html`), router switched from `hashchange` to `pushState`/`popstate`, 28 `app.html` references updated (sw.js, manifest start_url, api.php push URLs, header, pwa.js, index.html) and 18 test files. Old `#/` links and already-sent push notifications must keep working. All-or-nothing change — do it with Dave watching on his phone, not unattended.

## Open — database
- [ ] **`users.posts` JSON column can be dropped.** The posts-table migration (see Done) is live on both dev and prod now, and both are actually serving posts/likes from `posts`/`post_likes`, not the JSON. `users.posts` itself is still sitting there untouched on both, kept as a fallback. Dropping it (and the column-read code paths that never got removed, if any remain) is separate, later cleanup — no urgency, it's dead weight, not a liability.
- [ ] No foreign keys — deleting a user leaves their media, comments and likes behind
- [ ] `pending_users` has no primary key
- [ ] `follows`/`followers` columns are typed NUMERIC but hold JSON text
- [ ] `users.followers` and `users.is_admin` are never read or written — dead columns
- [ ] Three date formats across tables (unix numbers, `YYYY-MM-DD HH:MM:SS`, ISO strings)

## Open — privacy
- [ ] Some internal docs in the repo are reachable from the live site's web root. Block them or move them out (needs an `.htaccess` change — only when Dave asks directly).
- [ ] Decide on a retention period for the API log.

## Open — tooling
- [ ] **`agent-board` (see [[claude-config]]'s skill) currently shares a login with the test suite's `TEST_EMAIL_2`** (`davefruin@gmail.com`). Works fine for now, but agent messages and test-run noise end up in the same account's post history. Should get its own dedicated account eventually.

## Done (recent)
- **Migrated a real user's stranded dev-only activity to prod (2026-09-24).** A family member was using dev by mistake for part of a session, then switched to prod correctly partway through. Traced exactly what existed on dev but not prod (accounts on both sides predate this — same email/timestamp on both, likely from dev being seeded off a prod snapshot at some point) and migrated just the gap: one post ("Latest sketch") plus its photo, one like she'd received today that hadn't landed on prod, and 4 likes she gave on another user's posts (confirmed those target posts already existed on prod before inserting). All additive — nothing deleted from dev, nothing overwritten on prod. Confirmed working with a live fetch of the migrated photo (200, image/webp) after the insert.
- **Posts moved off the JSON blob and onto real tables — live on both dev and prod (2026-09-24).** 11 `api.php` handlers (post, deletePost, likePost, unlikePost, getPostLikes, getPostById, getPostPreviews, getMyPosts, getUserPosts, fetchFollowedPosts, deleteAccount) switched from reading/rewriting a user's whole `users.posts` JSON to real `posts`/`post_likes` tables. Also simplified post-id collision handling: `posts.id` is now a real PRIMARY KEY, so a collision fails the INSERT itself and the retry loop no longer needs the `BEGIN IMMEDIATE` transaction from the earlier fix. Caught and fixed a real bug along the way: retrying a `PDOStatement` after a constraint-violation exception threw `SQLSTATE HY000 general error 21` on the next `execute()` — fixed by re-preparing the statement on each retry.
  - Verified locally first against a copy of dev's real data: every read handler's output matched the old code byte-for-byte (only exception: display order of likes tied to the same second, never sorted/asserted anywhere — cosmetic); write paths (create/like/unlike/delete, deleteAccount cleanup, forced double-collision retry) all behaved correctly.
  - **Dev:** `userdata.db` backed up first, `migrate-posts.php` run (109 posts, 130 likes, 19 users, 0 collisions), dev's Playwright suite passed (54 passed; only the 3 known-flaky `media-capture.spec.js` fake-camera tests failed, unrelated).
  - **Prod:** `userdata.db` backed up first, `migrate-posts.php` run (81 posts, 126 likes, 0 collisions), site and `api.php` confirmed responding normally afterward. `users.posts` is untouched on both — see Open — database for dropping it later.
- Post ID collision fixed. Two-part: bump the second forward when taken, then found that alone didn't survive genuinely concurrent creates (both requests read the same stale posts array), so wrapped the read-check-write in a `BEGIN IMMEDIATE` transaction. Verified with concurrent (`Promise.all`) creates across 3 rounds.
- Wizard and plain CLI now check a `--media` path exists locally (`access(path, R_OK)`) before uploading, instead of round-tripping a bad path to the server for a generic error. Verified in both.
- Links in post text; tagging people in posts and comments
- Navigation-hand setting now explains what it actually does (it isn't phones-only)
- Phone create-post page: Post button moved into thumb reach
- Success/error messages sit higher on phones
- Own new post now shows in the feed right after posting
- Settings page no longer opens scrolled down (was a reload bug, not navigation)
- Leaving create-post no longer loses media (it had actually been silently staying attached and getting posted unseen)
- Post previews no longer cut off mid `@[id]` mention
- Install failures on two Mint machines: makefiles linked libcurl by a hardcoded version path (`libcurl.so.4.8.0`), and cloning from `/` failed with an undocumented permission error. Fixed in all three terminal repos (link by SONAME) and on the download page (`cd ~` first; libcurl4/pkg-config listed up front).
- Wizard CLI: commands typed with arguments (`login me@x.com`) said "Unknown command" — now tells you to type the command bare and answer prompts
- CLI: `likes --json` now matches the bare-object shape of every other `--json` command
- DB: missing `user_id` index on media added to source (existed on live DBs by hand only); media table now defined once in `schema.php`
- Docs/tests: ARCHITECTURE.md rewritten; test suite reads `.env` itself; second test account created (13 tests now pass); session-expiry tests split into the real cases plus a silent-refresh test; impossible like-own-post test fixed
- Refresh-token leak into the API log fixed

## Terminal client testing — 2026-09-23 (machine: omarchy, Arch Linux)

Cloned, built, and manually tested `sscli`, `sswiz`, and `sstui` end-to-end against dev.davidfruin.com: auth (login/logout/session persistence), feed, create/comment/like/delete post, follow/unfollow, notifications, and destructive-action confirm dialogs. Each tool keeps its own independent session, as documented. `download.html` only covers Debian/Ubuntu — on Arch the equivalent packages (`base-devel`, `ncurses`, `curl`, `pkgconf`) were already installed, so no distro-specific step was needed here.

Machine-specific notes (not project facts, just what this session did):
- No `sudo` available non-interactively, so binaries went to `~/.local/bin` instead of `/usr/local/bin`. Functionally identical.
- API target overridden via `~/.config/simple-social-cli/config.ini` (`base_url=...`), shared by all three clients since they link the same vendored `ss_config.c`. **Currently still set to dev.davidfruin.com on this machine** — switch back before using these builds against production.
- TUI Settings screen shows one config path (`~/.config/simple-social-tui/config.ini`) but the `api` line under it is actually read from the other config file above — minor display confusion, not a functional bug.
- Double-checked two things that looked like bugs during testing but weren't: `q` quits the whole TUI from any view by design (`Esc`/`Backspace` is "back"), and Esc-to-cancel on a compose/comment box with text goes through a "Discard what you have written?" confirm first.

## Links
- Live: https://app.davidfruin.com
- Related: [[simple-social-tui]], [[simple-social-cli]], [[simple-social-cli-interactive]]
