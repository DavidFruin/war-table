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
- **2026-10-02 (Dave): top priority, and must ship on both the Apple App Store and Google Play** (iPhone is required). Individual (not organization) developer accounts. Expo push approved. Full plan: [[ssreact-native-port-plan]]. Store-mandated moderation (report/block/terms/privacy): [[store-readiness-plan]]. Proposed in the plan, not yet confirmed: Expo + logic-only sharing with [[ssreact]] via a copied `src/core` (no cross-platform UI kit, since RN is a bridge).
- **A bridge toward eventual true-native (Swift/Kotlin), not the end state** — decided in [[simple-social]]'s Planning section. Don't over-invest in this as if it's permanent.
- **Code-sharing approach with [[ssreact]] is deliberately undecided** — just the logic/API-client layer vs. UI-level sharing via a cross-platform kit (Tamagui, NativeWind, etc.). Given RN itself is a stepping stone, paying setup cost for a cross-platform UI kit here means paying it twice (once discarding shadcn's web-only choice, again when RN gets replaced by true native). Decide this when the project actually starts, not before — see [[simple-social]]'s Planning section for the full reasoning already worked out.
- Old rule superseded 2026-09-24: web and mobile no longer have to be kept separate (was: "keep CLI, TUI, API backend, web, iOS, Android frontends separate"). This project is explicitly meant to reuse from [[ssreact]] where it makes sense.

## Next steps
- [ ] Not started — ssreact is usable now; begins after the ssapi improvement plan's ssreact fixes land (see [[ssreact-native-port-plan]] §0)
- [ ] Dave: sign up for an individual Apple Developer account ($99/yr) and a Google Play Console account ($25); Play personal accounts need a 12-tester, 14-day closed test before production
- [ ] Blocker for the store release: prod must run [[ssapi]] (with moderation + Expo push) instead of [[simple-social]]'s backend copy
- [ ] Decide the code-sharing approach (see Decisions) once started

## Links
- Repo: https://github.com/DavidFruin/ssreact-native
- Related: [[simple-social]] (backend/API, and the Planning section this was scoped in), [[ssreact]] (web rewrite this follows)
