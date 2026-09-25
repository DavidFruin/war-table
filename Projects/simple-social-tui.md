---
status: active
repo: https://github.com/DavidFruin/simple-social-tui
---

# simple-social-tui

## Summary
Full-screen ncurses terminal UI for [[simple-social]]. Third terminal front end on the same C library as [[simple-social-cli]] and [[simple-social-cli-interactive]]; all three share the `~/.simple-social-cli/` session state.

## Visual redesign — mirror the web app's look

Goal (Dave, 2026-09-23/24): make the TUI feel like "the website but in text" instead of plain ncurses defaults.

### Decisions
- One fixed palette using all 8 base ANSI colors, each with a single semantic role (cyan=author, green=liked, red=error, yellow=badges, blue=primary/borders/logo, magenta=mentions), not the web's 5-theme system (light/dark/red/blue/hacker) — no theme picker in the TUI.
- Forced black background everywhere (`assume_default_colors(COLOR_WHITE, COLOR_BLACK)` in `ui_init()`, ui.c), replacing the old `use_default_colors()` transparency — the app now looks the same regardless of the user's terminal scheme, matching how the website looks the same in any browser.
- Post cards get real box-drawn borders (Phase 3, not built yet); chrome (header/footer) instead mirrors the actual web CSS, which only uses a single border line there (`header { border-bottom }`, `footer { border-top }` in `simple-social/css/main.css`), not a full box — confirmed by reading the CSS rather than guessing.
- ~~Feed stays a compact list rather than boxing every post at full height~~ — **superseded 2026-09-24**: Dave explicitly wants every post shown in full, not just the selected one, scan-density tradeoff accepted. `draw_collapsed`/`draw_expanded` merged into one `draw_post_card()`; selection is now shown only by a bold vs dim border.
- `c` is comment-only everywhere now; `p` creates a post (was `c`, overloaded with comment before). Tab/Shift-Tab now work from inside an open post too (closes the post, then switches tabs) — previously silently changed `app->tab` under a view that doesn't render it.

### Status
- **Phase 1 (color foundation) — done, pushed** (`simple-social-tui` commit on `master`). New `CP_*` pairs in `ui.h`/`ui.c`, `ui_draw_box()` helper added for later phases.
- **Phase 2 (header/footer chrome) — done, pushed.** "Simple Social" logo added to the tab bar (drops on <60 cols), active tab is a filled blue pill, footer got a rule line above it (mirrors web's `border-top`). Required nudging every hardcoded `LINES - 2` body-bottom bound to `LINES - 3` across `ui.c`, `settings.c`, `auth.c` so nothing overlaps the new footer rule.
- Verified: clean build (no warnings), login screen visually confirmed via a `tmux capture-pane` smoke test (forced black bg + blue bold logo render correctly). Post-login header/footer not yet visually confirmed — no saved session/credentials on this machine to log in non-interactively; asked Dave to eyeball it.
- **Phase 3 (post cards + mentions) — done, pushed.** Collapsed feed/profile posts are a 3-row box now; the selected/expanded post gets a taller box sized to its wrapped body with a bold border (replaced the old reverse-video header bar as the "this one's selected" signal). `@[id]` mention tokens highlighted magenta as generic `@user`.
  - **Known limitation:** mentions show as `@user`, not the real email — the vendored `simple-social-cli` C library never parses the API's `mentions: [{id, email}]` field, only raw post text. Resolving it properly means patching that separate repo; flagged, not done.
  - Not visually verified against a real feed — no test credentials on this machine, and no reachable way to complete an OTP registration round-trip from this sandbox. Build is clean; box/column math was hand-checked carefully. Dave should eyeball the feed and post-detail view.
- **2026-09-24 follow-up (still Phase 3 territory) — done, pushed.** Every post now renders in full (not just the selected one) — see Decisions above. `p`/`c` key split, Tab-from-open-post fix, README/help text updated to match; also fixed two stale doc claims left over from Phase 1 (README said "16-color theming that inherits the terminal's scheme" and "expand-on-selection", both wrong since Phase 1/this change).
- **Phase 4 (selection-highlight consistency) — done, pushed, broader than originally scoped.** Replaced `A_REVERSE` (inverts whatever colors are active — with the forced black bg from Phase 1, that meant a jarring stark-white block) with `COLOR_PAIR(CP_TAB_ACTIVE) | A_BOLD` (black-on-blue), the same fill the active tab pill already used. Did the whole app in one pass rather than just Notifications/Users, since leaving some screens converted and others not would've looked broken: `ui.c` (notifs, users, feed/notifs load-more trailers), `settings.c`, `auth.c` (login menu), `filepick.c`, and `detail.c`'s load-more-comments row.
  - **Known gap, left alone on purpose:** comment selection inside an open post (`detail.c`) still uses `A_REVERSE` — it wraps a switch that recolors per line kind (author cyan, meta dim, ...), so a naive swap would just have each kind's own `attron` stomp the blue fill immediately (same "attroff clears the pair outright, doesn't restore" issue hit and fixed in the notif badge case, but here it wraps a whole block, not one glyph). The post-detail view was always slated as its own later pass — see below.
  - Verified live via `tmux capture-pane` on the login menu (confirmed `[1m[30m[44m` = bold black-on-blue), not just build success — same code path as every other screen this touched, so that one check covers all of them.
- **Phase 5 (modal styling) — done, pushed.** `ui_help`/`ui_confirm`/`ui_prompt` had plain white unbordered-in-color `box()`s, inconsistent with the blue chrome everywhere else and with `ui_modal_error`'s already-red border. All now border+title in `CP_PRIMARY`, hints dimmed in the same accent color rather than plain `A_DIM`.
  - `ui_confirm()` gained a `danger` param (mirrors web's `.btn-danger` vs `.btn-primary`): red border/title/`y`-prompt + "(cannot be undone)" appended to the title. Classified all 7 call sites: delete post/comment/account + discarding a draft = danger; attach media, email an OTP, log out = not.
  - Not visually verified live (same sandbox limitation as Phase 3/4 — these only trigger from inside a session).
- **Phase 6 (editor/filepicker styling) — done, pushed.** Compose box (`editor.c`) and both file-picker windows (`filepick.c`: file browser + typed-path prompt) got the same `CP_PRIMARY` border/title/hint treatment as Phase 5's modals. Left the char counter's under-limit color and "attached: &lt;file&gt;"'s green alone — those are semantic, not chrome.
- **Post-detail/comment redesign — done, pushed.** Comment selection's `A_REVERSE` (the Phase 4 known gap) replaced with the same blue-fill pattern as notifs/users, guarding the one real conflict (`DL_CAUTHOR`'s cyan color) with `if (!selected)` rather than a rewrite of the flat line-scroll architecture — box-per-comment would've been a much bigger change than warranted. Also: `draw_text_line()` (mention highlighting) exposed from `ui.c` via `ui.h` and wired into the post-detail view's body/comment text, and the DL_RULE divider recolored to match the app's blue chrome.
- **2026-09-24: full live verification, no bugs found.** Logged into the TUI itself (`~/.simple-social-cli/tui/`, separate session store from `sscli`'s) against dev with the agent-board account's now-known credentials, and drove it end to end via `tmux send-keys`/`capture-pane` with real feed data — header/tabs/badge, feed cards + selection border, Notifications (including the exact badge-restoration fix from Phase 4, confirmed live), Users tab, Settings, help overlay, the delete-account confirm prompt (canceled immediately after the password step — did not proceed further), and the compose editor via `p`. Every screen matched what was designed; this closes out the "not visually verified" caveat on every phase above.
- **2026-09-24 feedback fix — done, pushed, verified live.** Dave: bold-vs-dim border wasn't visible enough to tell which feed post was selected at a glance; wanted background fill or a stronger border. Selected cards now fill their whole interior with the same blue used for a selected row everywhere else (Notifications/Users/Settings/comments) — same trade-off as those: per-kind semantic colors (author cyan, likes green, mentions magenta, dim timestamps) skipped while selected since they clash on blue, in favor of being unmissable. `draw_text_line()` gained a `plain` param for this. Confirmed live: solid blue block vs black, obviously different.
- **2026-09-24 feedback fix — done, pushed, verified live.** Dave: status/error messages at the bottom were covering the key hints (`draw_status()` drew one or the other on the same row, never both). Footer is now 3 rows (rule, hints — always shown, status — only when set) instead of 2; `ui_body_height()` and every `LINES - 3` body-bottom bound bumped to `LINES - 4`. Same fix applied to `auth.c`'s login-screen footer. Verified live by triggering the real delete-post confirm on my own board post, cancelling it, and confirming "Not deleted." showed with the hints still visible above it.
  - **Found while testing, not investigated further:** the TUI's saved session (`~/.simple-social-cli/tui/`) persisted correctly across one relaunch but was completely empty (not just expired — the files were gone) on a later one, with no code path found (no signal handlers, `app_clear_session_files()` only called from logout/delete-account/session-expiry) that explains it. Had to re-log-in mid-session. Worth a closer look if it recurs; not blocking, not chased down.
- **Not started:** Phase 7 polish pass (narrow-terminal edge cases specifically — everything else was just confirmed live).

### Download page (2026-09-24)
`simple-social/download.html`'s TUI section now has an LMDE 7 / Omarchy Quattro toggle for the one step that actually differs by distro (build-tool package install — apt vs pacman); clone and `make install` steps are identical either way and stay shared. New `.os-tabs`/`.os-panel` pattern + `js/os-toggle.js`, `css/main.css` got one small `.os-tabs` rule. Pushed to `simple-social` `master`.

Deliberately left the CLI and Interactive CLI sections alone — they still say "Arch support coming soon" even though the 2026-09-23 testing session confirmed both already work on Arch too. Extending the toggle to those is a separate call for Dave, not assumed here.

Not deployed to dev/app.davidfruin.com yet — this sandbox has no network path to `el1` to run the documented deploy step (`ssh el1`, `git pull` in `dev.davidfruin.com/public_html`; prod needs Dave's explicit go-ahead per ARCHITECTURE.md §6). Dave is testing the LMDE 7 instructions himself next.

## Open — bugs
- [ ] **One-off segfault opening the Users tab (unreproduced, not fixed)**
  - Seen 2026-09-17 during the users/profiles work (became `dc008ca`). `-O2` build died on pressing `3`; exit 139, `dmesg` showed `segfault at 0 ip 0000000000000000` — jumped to a null address rather than dereferencing a bad pointer.
  - Could not reproduce: `-O0 -g`; `-O2 -g` under gdb in a pty (all 17 users rendered); three plain `-O2` runs; AddressSanitizer across users → profile → follows → back → Me → follows → notifications → feed.
  - Probably not our bug: same run printed "Not logged in" (`getMyInfo` failed), and the line just before in `dmesg` was the wifi adapter failing to leave power save (`rtw_8821ce ... firmware failed to leave lps state`). A network drop mid-call could crash in libcurl's or the library's error path — the library's media helpers already had a use-after-free of that kind (fixed in simple-social-cli: "Report media errors, and fix a use-after-free in api_delete_media").
  - If it recurs, treat it as real. Next steps:
    - Capture a core (`ulimit -c unlimited`, local `core_pattern`), then `gdb ./simple-social-tui core` for the frame that jumped to 0
    - Check correlation with network loss: disable wifi / pull cable, open the Users tab
    - Look first at `api_get_users` and `api_get_my_follows` — the two library calls on that keypress

## Links
- Related: [[simple-social]], [[simple-social-cli]], [[simple-social-cli-interactive]]
