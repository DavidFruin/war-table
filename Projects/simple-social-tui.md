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
- Feed stays a compact list rather than boxing every post at full height, to avoid tanking scan density; only the selected/expanded post gets the full box treatment (planned for Phase 3).

### Status
- **Phase 1 (color foundation) — done, pushed** (`simple-social-tui` commit on `master`). New `CP_*` pairs in `ui.h`/`ui.c`, `ui_draw_box()` helper added for later phases.
- **Phase 2 (header/footer chrome) — done, pushed.** "Simple Social" logo added to the tab bar (drops on <60 cols), active tab is a filled blue pill, footer got a rule line above it (mirrors web's `border-top`). Required nudging every hardcoded `LINES - 2` body-bottom bound to `LINES - 3` across `ui.c`, `settings.c`, `auth.c` so nothing overlaps the new footer rule.
- Verified: clean build (no warnings), login screen visually confirmed via a `tmux capture-pane` smoke test (forced black bg + blue bold logo render correctly). Post-login header/footer not yet visually confirmed — no saved session/credentials on this machine to log in non-interactively; asked Dave to eyeball it.
- **Phase 3 (post cards + mentions) — done, pushed.** Collapsed feed/profile posts are a 3-row box now; the selected/expanded post gets a taller box sized to its wrapped body with a bold border (replaced the old reverse-video header bar as the "this one's selected" signal). `@[id]` mention tokens highlighted magenta as generic `@user`.
  - **Known limitation:** mentions show as `@user`, not the real email — the vendored `simple-social-cli` C library never parses the API's `mentions: [{id, email}]` field, only raw post text. Resolving it properly means patching that separate repo; flagged, not done.
  - Collapsed cards deliberately stay 3 rows (not full multi-line boxes) to avoid tanking feed scan density — only the selected post expands.
  - Not visually verified against a real feed — no test credentials on this machine, and no reachable way to complete an OTP registration round-trip from this sandbox. Build is clean; box/column math was hand-checked carefully. Dave should eyeball the feed and post-detail view.
- **Not started:** Phase 4 (users/notifs/profile selected-row fill), Phase 5 (modals), Phase 6 (editor/filepicker), Phase 7 (polish pass + manual test sweep).

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
