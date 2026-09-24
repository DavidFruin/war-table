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

**Next steps**
- [ ] Backend module split (auth/users/posts/comments/follows/notifications/media)
- [ ] Land the posts-table migration to dev and prod (tracked in Open — database)
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
- Lock the site / log out all users for scheduled maintenance
- Admin panel for freezing or deleting users (abuse etc.)
- Chat, including group chats
- Complaints and ideas page: report a bug, report a person, suggest an idea
- Maybe require a referral code to register
- Language switcher

## Open — bugs
- [ ] **Dark mode: illegible blue link text** — not a themed color, it's the browser's unstyled default blue on three links with no CSS class: "Don't have an account? Register" / "Forgot password?" on the login page, "Already have an account? Login" on the register page. Everything else (nav, static pages, mentions, post links) already gets a theme-aware color. Narrow, cheap fix — just give those three a class.
- [ ] Can't see all playback speed options — it's the browser's native video menu, clipped by the video box. Needs custom video controls.
- [ ] 4px horizontal overflow on phones from `.notif-badge` (low importance)
- [ ] Flash when expanding a post — doesn't reproduce. Possibly images shifting the page as they load (no reserved space). Confirm with Dave what he saw.
- [ ] Coming back from an expanded post lands a little below where you were — doesn't reproduce, scroll restore measured correct.
- [ ] **Clean URLs (fix hash routing):** want `dev.davidfruin.com/feed` instead of `/app.html#/feed`. Needs an Apache rewrite (non-file paths serve `app.html`), router switched from `hashchange` to `pushState`/`popstate`, 28 `app.html` references updated (sw.js, manifest start_url, api.php push URLs, header, pwa.js, index.html) and 18 test files. Old `#/` links and already-sent push notifications must keep working. All-or-nothing change — do it with Dave watching on his phone, not unattended.

## Open — database
- [ ] **Posts table migration — code done, not deployed.** Posts were one JSON blob in `users.posts`: couldn't be indexed or queried, and concurrent writes to the same user could overwrite each other. `posts`/`post_likes` tables (`schema.php`), `migrate-posts.php` (hand-run, idempotent copy script — `php migrate-posts.php [--dry-run]`, not wired into any request path), and all 11 `api.php` handlers that used to read/write `users.posts` (post, deletePost, likePost, unlikePost, getPostLikes, getPostById, getPostPreviews, getMyPosts, getUserPosts, fetchFollowedPosts, deleteAccount) are done and committed to local master. Also simplified post-id collision handling: now that `posts.id` is a real PRIMARY KEY, a collision fails the INSERT itself, so the retry loop no longer needs the `BEGIN IMMEDIATE` transaction from the earlier fix (see Done). Verified locally against a copy of dev's real database (109 posts/130 likes/19 users): every read handler's output matched the old code byte-for-byte except display order of same-second-tied likes (cosmetic, never sorted/asserted anywhere); write paths (create/like/unlike/delete, deleteAccount cleanup, forced double-collision retry) behaved correctly. Caught and fixed one real bug in the process: reusing a `PDOStatement` after a constraint-violation exception threw `SQLSTATE HY000 general error 21` on the next `execute()` — fixed by re-preparing each retry. **Not yet done:** running `migrate-posts.php` on dev's actual database, pushing to GitHub, deploying to dev, running the Playwright suite there, and — only once confirmed — prod. `users.posts` is left in the schema, untouched and unread by these handlers, as a fallback until that's confirmed.
- [ ] No foreign keys — deleting a user leaves their media, comments and likes behind
- [ ] `pending_users` has no primary key
- [ ] `follows`/`followers` columns are typed NUMERIC but hold JSON text
- [ ] `users.followers` and `users.is_admin` are never read or written — dead columns
- [ ] Three date formats across tables (unix numbers, `YYYY-MM-DD HH:MM:SS`, ISO strings)

## Open — privacy
- [ ] Some internal docs in the repo are reachable from the live site's web root. Block them or move them out (needs an `.htaccess` change — only when Dave asks directly).
- [ ] Decide on a retention period for the API log.

## Done (recent)
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
