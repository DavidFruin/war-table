---
status: active
repo: https://github.com/DavidFruin/ssreact
---

# ssreact

## Summary
React + Vite + TypeScript + shadcn/ui rewrite of [[simple-social]]'s web frontend (currently vanilla JS, no build step, no TS). Repo renamed 2026-09-25 from `learn-react-site` — Dave's own React-learning scaffold — rather than starting a new repo from scratch, since it already had exactly the stack this needs.

## Status
- **Repo renamed and cloned** (2026-09-25): `learn-react-site` → `ssreact` on GitHub, cloned to `~/ssreact`. Description updated to match its new purpose.
- **Scaffold already in place** from the learn-react-site work, not yet touched for simple-social: React 19, Vite 8, TypeScript 5.8, Tailwind 4, shadcn CLI already initialized (`components.json` present), `src/components/ui/button.tsx` already added, Base UI primitives + `class-variance-authority`/`clsx`/`tailwind-merge` (shadcn's usual utility stack), `lucide-react` icons, pnpm as package manager.
- CI (`.github/workflows/deploy.yml`) currently only builds on push to `master` — no deploy step wired to any target yet. Naming it `deploy.yml` while only building is pre-existing from the learn-react-site days, not a decision made for this project.
- **No simple-social-specific code written yet.** This note is the plan; implementation hasn't started.
- **Blocked on sequencing, not on this repo:** per [[simple-social]]'s Planning section, the backend module split needs to finish and the API surface needs to settle before the frontend rewrite starts in earnest. Scaffold/tooling work here doesn't have to wait for that; wiring up real pages against the live API does.

## Decisions
- **Reuse the learn-react-site scaffold rather than starting clean** — it already has the exact stack (React/Vite/TS/shadcn) this project wants, and Dave already made the tooling choices (Base UI over plain Radix, pnpm, Tailwind 4) while learning React with it. Not revisiting those choices without a reason to.
- **No need to preserve the current retro-BBS visual identity** (already decided in [[simple-social]]'s Planning section) — that look is covered by the terminal clients ([[simple-social-tui]] etc.) instead. shadcn's own design language is fine to adopt as-is.
- **Component mapping worked out 2026-09-25** (see below) — grounded in the actual current shadcn/ui docs (fetched live, not from memory), not guessed.
- **Theme system ports directly**: the current app's 5 themes (light/dark/red/blue/hacker) via `data-theme` + CSS custom properties is the same mechanism shadcn itself uses for light/dark — no redesign needed, just more variable sets.
- **Completion bar is the existing Playwright suite passing + manual parity walkthrough**, matching how every other piece of this project (TUI redesign, backend module split, posts-table migration) has been verified — against real behavior, not just "compiles."

## Component mapping (shadcn/ui, verified against current docs 2026-09-25)

| Current web app | shadcn/ui component |
|---|---|
| `.form-group`/`.form-input`/labels | `Field` (+ `FieldGroup`, `FieldLabel`, `FieldDescription`, `FieldError`) — supersedes the older bare Label+Input pattern |
| `.form-textarea` (composer, comment box) | `Textarea` |
| Theme `<select>` | `Select` (or `Native Select` for native mobile dropdown behavior — low-stakes choice) |
| Hand-toggle checkbox | `Switch` |
| Search-by-email input + dropdown | `Combobox` — built for exactly this ("autocomplete input with a list of suggestions") |
| Media file upload | `Attachment` — has real idle/uploading/processing/error/done states with progress, more than the current plain file input |
| `.btn-primary/-secondary/-ghost/-danger` | `Button` variants `default`/`secondary`/`ghost`/`destructive` |
| `.btn-like` | `Toggle` |
| `.post-card` | `Card` |
| Notification rows, comment rows, search results, followers/following rows | `Item`/`ItemGroup` — "media + title + description + actions" list rows, better fit than a `Card` per row |
| Empty states ("No posts yet," "Nothing here yet," scattered across the app) | `Empty` — real structure (icon/media, title, description, action) instead of a plain sentence |
| Likes list, followers/following list popups | `Popover` or `HoverCard` |
| Delete confirmations (currently native `confirm()`) | `Alert Dialog` — real upgrade, native `confirm()` can't be styled at all |
| Session-expired re-login modal | `Dialog` |
| Media viewer/lightbox | `Dialog` |
| `.error-message`/`.success-message` | `Alert` |
| Notification badge, "following" tag | `Badge` |
| "Loading..." text | `Skeleton` or `Spinner` |
| **Group chat (future goal, not yet built anywhere)** | `Message` + `MessageGroup` + `Bubble` + `MessageScroller` + `Attachment` — shadcn ships an actual chat kit now; this is close to assemble-not-build when that feature starts |

**Stays fully custom, no shadcn equivalent:**
- Mention (`@user`) autocomplete inside a textarea — `Combobox` needs its own dedicated input, not an arbitrary caret position mid-textarea
- Media capture UI (record photo/video/audio with a live waveform) — custom `MediaRecorder`/`getUserMedia` work, `Dialog` only provides the chrome
- Mobile thumb-nav (floating corner menu)

## Next steps
- [ ] Wait on backend module split finishing / API surface settling (tracked in [[simple-social]])
- [ ] Scaffold: routing, API client layer, design tokens for the 5-theme system
- [ ] Auth pages (login/register/reset-password) + session handling (silent refresh, session-expiry modal) — has to match existing subtle behavior, not just look right
- [ ] Feed + post-card + create-post (mention autocomplete, media capture are the hard parts here)
- [ ] Post detail + comments
- [ ] Profile + search + notifications
- [ ] Settings (themes, sessions list, push opt-in, account deletion)
- [ ] PWA parity (manifest, service worker, install prompt, push) — flagged as highest-risk area; `simple-social/ARCHITECTURE.md` already documents a real stale-JS-after-deploy incident in this exact area under the current no-build-step setup, and Vite's build/hashing model changes how cache invalidation has to work
- [ ] Verification pass: run [[simple-social]]'s Playwright suite against it, manual walkthrough, fix gaps

## Links
- Repo: https://github.com/DavidFruin/ssreact
- Related: [[simple-social]] (backend/API this talks to, and the Planning section this rewrite was originally scoped in), [[simple-social-tui]] (why the retro-BBS look doesn't need preserving here)
