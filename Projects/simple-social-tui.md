---
status: active
repo: https://github.com/DavidFruin/simple-social-tui
---

# simple-social-tui

## Summary
Full-screen ncurses terminal UI for [[simple-social]]. Third terminal front end on the same C library as [[simple-social-cli]] and [[simple-social-cli-interactive]]; all three share the `~/.simple-social-cli/` session state.

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
