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
- **Not started:** Phase 5 (modals), Phase 6 (editor/filepicker boxes), post-detail/comment view redesign (not originally numbered — needs its own pass per the gap above), Phase 7 (polish pass + manual test sweep).

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
