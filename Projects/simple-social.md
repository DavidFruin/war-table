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
- Keep the CLI, TUI, API backend, web frontend, iOS frontend and Android frontend separate. Only the interactive CLI and the TUI are built on top of the CLI.

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
- [ ] Can't see all playback speed options — it's the browser's native video menu, clipped by the video box. Needs custom video controls.
- [ ] 4px horizontal overflow on phones from `.notif-badge` (low importance)
- [ ] Flash when expanding a post — doesn't reproduce. Possibly images shifting the page as they load (no reserved space). Confirm with Dave what he saw.
- [ ] Coming back from an expanded post lands a little below where you were — doesn't reproduce, scroll restore measured correct.
- [ ] Post IDs can collide when two posts are made in the same second. Needs a decision, since comments/media/likes all reference the post ID as a string.
- [ ] **Clean URLs (fix hash routing):** want `dev.davidfruin.com/feed` instead of `/app.html#/feed`. Needs an Apache rewrite (non-file paths serve `app.html`), router switched from `hashchange` to `pushState`/`popstate`, 28 `app.html` references updated (sw.js, manifest start_url, api.php push URLs, header, pwa.js, index.html) and 18 test files. Old `#/` links and already-sent push notifications must keep working. All-or-nothing change — do it with Dave watching on his phone, not unattended.

## Open — database
- [ ] **Biggest structural problem:** posts are one JSON blob in `users.posts` — can't be indexed or queried, and concurrent writes to the same user can overwrite each other. Real migration; not done unattended.
- [ ] No foreign keys — deleting a user leaves their media, comments and likes behind
- [ ] `pending_users` has no primary key
- [ ] `follows`/`followers` columns are typed NUMERIC but hold JSON text
- [ ] `users.followers` and `users.is_admin` are never read or written — dead columns
- [ ] Three date formats across tables (unix numbers, `YYYY-MM-DD HH:MM:SS`, ISO strings)

## Open — privacy
- [ ] Some internal docs in the repo are reachable from the live site's web root. Block them or move them out (needs an `.htaccess` change — only when Dave asks directly).
- [ ] Decide on a retention period for the API log.

## Done (recent)
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

## Links
- Live: https://app.davidfruin.com
- Related: [[simple-social-tui]], [[simple-social-cli]], [[simple-social-cli-interactive]]
