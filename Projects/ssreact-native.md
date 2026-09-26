---
status: active
repo: https://github.com/DavidFruin/ssreact-native
---

# ssreact-native

## Summary
React Native app for [[simple-social]] — the phone client. Repo created 2026-09-25, not started yet.

## Status
- Repo created (private, empty except a README pointing back here).
- **Blocked on nothing structurally**, but sequenced deliberately last: per [[simple-social]]'s Planning section, this is meant to follow [[ssreact]] (the web rewrite), not run in parallel with it.

## Decisions
- **A bridge toward eventual true-native (Swift/Kotlin), not the end state** — decided in [[simple-social]]'s Planning section. Don't over-invest in this as if it's permanent.
- **Code-sharing approach with [[ssreact]] is deliberately undecided** — just the logic/API-client layer vs. UI-level sharing via a cross-platform kit (Tamagui, NativeWind, etc.). Given RN itself is a stepping stone, paying setup cost for a cross-platform UI kit here means paying it twice (once discarding shadcn's web-only choice, again when RN gets replaced by true native). Decide this when the project actually starts, not before — see [[simple-social]]'s Planning section for the full reasoning already worked out.
- Old rule superseded 2026-09-24: web and mobile no longer have to be kept separate (was: "keep CLI, TUI, API backend, web, iOS, Android frontends separate"). This project is explicitly meant to reuse from [[ssreact]] where it makes sense.

## Next steps
- [ ] Not started — wait for [[ssreact]] to reach a real, usable state first (per the agreed sequencing)
- [ ] Decide the code-sharing approach (see Decisions) once started

## Links
- Repo: https://github.com/DavidFruin/ssreact-native
- Related: [[simple-social]] (backend/API, and the Planning section this was scoped in), [[ssreact]] (web rewrite this follows)
