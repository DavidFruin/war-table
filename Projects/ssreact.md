---
status: active
repo: https://github.com/DavidFruin/ssreact
---

# ssreact

## Summary
React + Vite + TypeScript + shadcn/ui rewrite of [[simple-social]]'s web frontend (currently vanilla JS, no build step, no TS). Repo renamed 2026-09-25 from `learn-react-site` — Dave's own React-learning scaffold — rather than starting a new repo from scratch, since it already had exactly the stack this needs.

**This note is written to be handed to a fresh agent with zero prior context on this project.** If that's you: read this whole note before touching code. The scaffold's own source files also carry inline comments explaining non-obvious decisions — this note is the durable, complete version; those comments are pointers back to it, not a substitute.

## Status (2026-09-25)
- **Repo renamed and cloned**: `learn-react-site` → `ssreact` on GitHub, cloned to `~/ssreact` on this machine. `pnpm` needed installing (`npm install -g pnpm`) — wasn't on this machine by default.
- **Scaffold built and pushed.** Routing, API client, auth context, and the full 5-theme system are done and verified (clean `tsc --noEmit`, clean `eslint`, clean production build, and an actual browser load via `playwright-core` against a real running dev server confirming the auth redirect and rendering — not just "it compiles"). All 10 pages exist as placeholders with inline notes; no real page UI built yet.
- **Blocked on sequencing, not on this repo:** per [[simple-social]]'s Planning section, the backend module split needed to finish and the API surface needed to settle before the frontend rewrite starts in earnest. **That's now done** (backend split deployed to prod 2026-09-25) — nothing is blocking real page work from starting.

## Decisions
- **Reuse the learn-react-site scaffold rather than starting clean** — it already had the exact stack (React/Vite/TS/shadcn) this wants, and Dave already made the tooling choices (Base UI over plain Radix, pnpm, Tailwind 4) while learning React with it.
- **No need to preserve the current retro-BBS visual identity** (decided in [[simple-social]]'s Planning section) — that look is covered by the terminal clients ([[simple-social-tui]] etc.) instead. shadcn's own design language is fine to adopt as-is.
- **Dev backend via Vite proxy, not CORS or an env var.** `vite.config.ts` proxies `/api.php` and `/media.php` to `dev.davidfruin.com`, so the app always uses the same relative-path convention as production and never needs to know which backend it's actually hitting. **Never point this at `app.davidfruin.com` (prod)** outside a deliberate, confirmed deploy test — same rule as the agent-board skill and every terminal client's config.
- **Clean URLs via React Router, not hash routing.** [[simple-social]]'s own web app has an open item ("Clean URLs (fix hash routing)") that's a whole migration there because of 28 file references and old `#/` links that must keep working. A fresh app has none of that baggage — just use React Router's normal paths from day one and that whole problem never exists here.
- **Themes as `.theme-<name>` classes on `<html>`, not shadcn's `.dark` convention.** simple-social's 5 themes (light/dark/red/blue/hacker) are equal siblings, not a light/dark binary with variants layered on — see the Theme system section below for the full mapping.
- **API type inconsistency is documented, not silently normalized.** The real backend uses camelCase for posts (`userId`, `timestamp`) and snake_case for comments/notifications (`user_id`, `created_at`) — confirmed against the actual `js/api.js` and page components in simple-social, not a guess or a typo to fix. `src/lib/types.ts` captures this as-is. If a single consistent shape in components is wanted, normalize it at the `src/lib/api.ts` boundary — but the wire format itself should stay documented accurately, not "corrected" based on assumption.
- **Component mapping worked out 2026-09-25** (see below) — grounded in the actual current shadcn/ui docs (fetched live, not from memory).
- **Completion bar is the existing Playwright suite passing + manual parity walkthrough**, matching how every other piece of this project (TUI redesign, backend module split, posts-table migration) has been verified — against real behavior, not just "compiles" or "looks right."

## Architecture reference

### Folder structure (as scaffolded)
```
src/
  lib/
    api.ts            API client (singleton, ported from js/api.js)
    auth-context.tsx  React auth/session state (replaces store.js)
    types.ts          TS types for API responses, incl. the casing quirk
  routes/
    RequireAuth.tsx   RequireAuth (redirect to /login) + RequireGuest (redirect to /feed) route guards
  components/
    layout/
      Header.tsx      Nav placeholder — real header needs the logged-in/
                       logged-out link sets + mobile thumb-nav, see
                       simple-social/js/header.js
      Layout.tsx       Header + <Outlet/> shell
    ui/                shadcn components land here (already has button.tsx)
  pages/               One file per route, all placeholders currently
  App.tsx              Route map
  main.tsx             Entry point
  index.css            Theme system (see below)
```

### API contract
Base URL is always a relative path (`/api.php` for most actions, `/media.php` specifically for `uploadMedia`/`deleteMedia`), proxied to dev in `vite.config.ts`. Every action is a `POST` with `action=<name>` plus its own params as `application/x-www-form-urlencoded` body (or `FormData` for media upload). Auth is `Authorization: Bearer <jwt>` once logged in. Response envelope: `{ valid: true, ...fields }` on success, `{ valid: false, message?, error? }` on failure (check both keys for the error text). A `401` on an authenticated call almost always means the access token aged out — `api.ts`'s `call()` already handles the silent-refresh-then-retry for this; don't re-implement it per-page.

Full action list (source of truth: `simple-social/js/api.js`, already ported into `src/lib/api.ts` — this table is for reference, not re-derivation):

| Domain | Actions |
|---|---|
| Auth | `login`, `logout`, `sendRegisterOTP`, `verifyRegisterOTP`, `finishRegister`, `sendOTP`, `verifyOTP`, `resetPassword`, `refreshToken` |
| User | `getMyInfo`, `getUserInfo`, `getUsers`, `getUserEmails`, `deleteAccount`, `updateTheme`, `updateHand` |
| Push | `getVapidPublicKey`, `savePushSubscription`, `deletePushSubscription` |
| Sessions | `getSessions`, `revokeSession`, `revokeAllOtherSessions` |
| Posts | `post` (create), `deletePost`, `getMyPosts`, `getUserPosts`, `fetchFollowedPosts` (the feed), `getPostById`, `getPostPreviews`, `likePost`, `unlikePost`, `getPostLikes` |
| Media | `uploadMedia`, `deleteMedia` (both via `/media.php`, not `/api.php`) |
| Comments | `createComment`, `getPostComments`, `deleteComment`, `getPostCommentCounts` |
| Follows | `followUser`, `unfollowUser`, `isFollowing`, `getMyFollows`, `getMyFollowers` |
| Notifications | `getNotifications`, `getUnseenNotificationCount`, `markNotificationsSeen` |

### Data models
See `src/lib/types.ts` for the actual TS interfaces. Key things a fresh agent will get wrong if they don't read this first:
- **Post uses camelCase** (`id`, `userId` — sometimes `userID` with a capital D on some responses, check both — `userEmail`, `text`, `mentions`, `mediaUrl`, `timestamp`, `likes: [{userId}]`).
- **Comment and Notification use snake_case** (`user_id`, `user_email`, `created_at`, `actor_id`, `actor_email`, `post_id`).
- **Post IDs are strings shaped `"<userId>.<unixTimestamp>"`** (e.g. `"26.1790287862"`), not plain integers.
- **Timestamps are `"YYYY-MM-DD HH:MM:SS"`, not ISO 8601.** `new Date(timestamp)` will misparse this in some browsers — always do `new Date(timestamp.replace(' ', 'T'))` first, exactly like the original does everywhere it touches a date.
- **`likeCount` and `isLiked` aren't separate fields** — they're derived: `likeCount = post.likes.length`, `isLiked = post.likes.some(l => l.userId === me.id)`.
- **Mentions live inside the text itself** as literal `@[id]` tokens, resolved via a parallel `mentions: [{id, email}]` array the API attaches — render pipeline is: escape HTML → linkify raw URLs → linkify `@[id]` tokens outside of any link the URL pass produced (that ordering matters, see `renderPostText` in `simple-social/js/components/post-card.js` for exactly why swapping the order breaks a URL containing a literal `@[26]`). A `null` email in a mention means that user was deleted — render as plain unlinked "@deleted user" text, not a broken link.

### Routing map
| Path | Auth | Notes |
|---|---|---|
| `/login`, `/register`, `/reset-password` | guest-only | `RequireGuest` bounces a logged-in visitor to `/feed` |
| `/feed` | required | the home feed |
| `/create-post` | required | |
| `/profile`, `/profile/:id` | required | same component, `:id` absent = own profile |
| `/post/:id` | required | single post + comments |
| `/notifications` | required | |
| `/settings` | required | |
| `/search` | required | |
| `/`, unmatched | — | redirect to `/feed` |

### Theme system
5 themes (light/dark/red/blue/hacker), applied as a class on `<html>` (`theme-dark`, `theme-red`, `theme-blue`, `theme-hacker`; light is the unclassed default) by `src/lib/auth-context.tsx`'s `applyTheme()`. Each theme block in `src/index.css` remaps simple-social's **original hex values** (ported directly from `simple-social/css/main.css`, not re-derived or approximated — if the two ever disagree, `simple-social/css/main.css` is the source of truth, not this note or the CSS file here) onto shadcn's own token names, so every shadcn component picks up the right theme with zero per-component theme logic:

| simple-social CSS var | shadcn token |
|---|---|
| `--color-primary` | `--primary` (`--primary-foreground` is white in every theme — the original's `.btn-primary`/`.btn-secondary`/`.btn-danger` set `color: white` unconditionally, never overridden per theme) |
| `--color-secondary` | `--secondary` |
| `--color-success` | `--success` *(custom addition — shadcn has no default success token)* |
| `--color-error` | `--destructive` |
| `--color-warning` | `--warning` *(custom addition, same reason)* |
| `--color-bg` | `--background`, `--card`, `--popover` (original has no distinct elevated-surface color) |
| `--color-bg-secondary` | `--muted`, `--accent` (original doesn't distinguish these two) |
| `--color-text` | `--foreground` |
| `--color-text-secondary` | `--muted-foreground`, `--accent-foreground` |
| `--color-border` | `--border`, `--input` |
| `--color-primary` (again) | `--ring` (focus rings use the brand color) |

`--primary-hover` is carried over as a custom addition too, but **nothing reads it yet** — shadcn's `Button` uses opacity for its hover state by default. Undecided: wire `--primary-hover` in via a custom class, or drop it and accept shadcn's own hover treatment. Flagged in `index.css`'s comment block; decide when actually building `Button` usage, not before.

## Component mapping (shadcn/ui, verified against current docs 2026-09-25)

| Current web app | shadcn/ui component |
|---|---|
| `.form-group`/`.form-input`/labels | `Field` (+ `FieldGroup`, `FieldLabel`, `FieldDescription`, `FieldError`) — supersedes the older bare Label+Input pattern |
| `.form-textarea` (composer, comment box) | `Textarea` |
| Theme `<select>` | `Select` (or `Native Select` for native mobile dropdown behavior — low-stakes choice) |
| Hand-toggle checkbox | `Switch` |
| Search-by-email input + dropdown | `Combobox` — built for exactly this ("autocomplete input with a list of suggestions"); note the original filters a client-side-fetched full user list, it does NOT call a search endpoint per keystroke (see `simple-social/js/pages/search.js`) |
| Media file upload | `Attachment` — has real idle/uploading/processing/error/done states with progress, more than the current plain file input |
| `.btn-primary/-secondary/-ghost/-danger` | `Button` variants `default`/`secondary`/`ghost`/`destructive` |
| `.btn-like` | `Toggle` |
| `.post-card` | `Card` |
| Notification rows, comment rows, search results, followers/following rows | `Item`/`ItemGroup` — "media + title + description + actions" list rows, better fit than a `Card` per row |
| Empty states ("No posts yet," "Nothing here yet," scattered across the app) | `Empty` — real structure (icon/media, title, description, action) instead of a plain sentence |
| Likes list, followers/following list popups | `Popover` or `HoverCard` |
| Delete confirmations (currently native `confirm()`) | `Alert Dialog` — real upgrade, native `confirm()` can't be styled at all |
| Session-expired re-login modal | `Dialog` (non-dismissible — no close button, matches current behavior of forcing re-auth) |
| Media viewer/lightbox | `Dialog` |
| `.error-message`/`.success-message` | `Alert` |
| Notification badge, "following" tag | `Badge` |
| "Loading..." text | `Skeleton` or `Spinner` |
| **Group chat (future goal, not yet built anywhere)** | `Message` + `MessageGroup` + `Bubble` + `MessageScroller` + `Attachment` — shadcn ships an actual chat kit now; this is close to assemble-not-build when that feature starts |

**Stays fully custom, no shadcn equivalent:**
- Mention (`@user`) autocomplete inside a textarea — `Combobox` needs its own dedicated input, not an arbitrary caret position mid-textarea
- Media capture UI (record photo/video/audio with a live waveform) — custom `MediaRecorder`/`getUserMedia` work, `Dialog` only provides the chrome
- Mobile thumb-nav (floating corner menu)

## Page-by-page build order

Recommended order (each unblocks testing the next; auth first since nothing else is reachable without it):

1. **Auth (Login/Register/ResetPassword)** — match `simple-social/js/pages/{login,register,reset-password}.js` field-for-field. Login: on success, `api.login()` then `api.getMyInfo()` to populate `useAuth().setUser()` (note `getMyInfo`'s response needs both `id` and `userId` set on the stored user — see `login.js`'s exact object shape). Register/reset-password: same 3-step OTP dance (send code → verify code → set password ×2), matching `simple-social-tui`'s `do_register()`/`do_reset()` in `auth.c` — keep the step sequence identical across every client, it's already been kept in sync twice.
2. **Feed** — `api.fetchFollowedPosts()`, offset/limit-25 pagination, build the `PostCard` component here (reused by Profile and Post pages after).
3. **Post detail + comments** — `api.getPostById` + `api.getPostComments`, comment composer reuses the mention-autocomplete `Textarea` built for Create Post.
4. **Create Post** — the hardest page: mention autocomplete, media upload (`Attachment`), and the camera/mic capture modal (fully custom). Consider building it in that order — text-only post first, media upload second, capture modal last — rather than all at once.
5. **Profile** — own + `:id` variants, followers/following popovers.
6. **Notifications** — grouped Today/Yesterday/Earlier (exact bucketing logic in `notifications.js`'s `groupNotificationsByDate`), per-type icon/text/link (`getNotificationIcon`/`getNotificationText`/`getNotificationLink` in the same file — match the copy and link targets exactly, don't rephrase).
7. **Search** — `Combobox` over a client-side-filtered full user list (see note above).
8. **Settings** — themes (already wired via `useAuth`, just needs the `Select` UI), hand preference (`Switch`), push notifications (`Notification.requestPermission()` **must be the first `await` in the click handler** — browsers only allow the permission prompt inside a user gesture, see `settings.js`'s `enablePush` comment for why), devices/sessions list with revoke, delete account (`AlertDialog`).
9. **PWA parity** (manifest, service worker, install prompt, push) — deliberately last, and flagged as the highest-risk single piece: `simple-social/ARCHITECTURE.md` already documents a real stale-JS-after-deploy incident in this exact area under the current no-build-step setup, and Vite's build/hashing model changes how cache invalidation has to work. Don't improvise this from scratch — read that incident writeup first.
10. **Verification pass** — run [[simple-social]]'s Playwright suite against this app, manual walkthrough, fix gaps. This is the actual finish line, not "every page has some UI."

## Known gaps / decisions still open
- **Session-expired modal not built.** `api.ts` has the hook (`onSessionExpired`) wired to a stub in `auth-context.tsx` that just logs the user out silently. The original shows a re-login modal that lets the user resume exactly where they were (their in-flight action retries automatically after re-auth) — this is real product behavior, not optional polish. Build before calling auth "done."
- **`postCache` not ported.** `store.js` caches whatever post a card just rendered so clicking into it on the Post page skips a loading flash. Not reproduced yet — if this project ends up using TanStack Query (or similar) for data fetching, its own cache likely provides this for free; if it's plain `fetch`-in-`useEffect`, it needs porting explicitly. Decide when building the Feed/Post pages, not before.
- **`--primary-hover` unwired** — see Theme system section above.
- **No favicon/PWA icons yet** — deferred to the PWA parity phase along with the rest of the manifest/service-worker work, not a scaffold bug.
- **GitHub Actions deploy pipeline** — tracked in [[simple-social]], not this repo; the existing `.github/workflows/deploy.yml` here only builds (leftover from the learn-react-site days), doesn't deploy anywhere yet.

## Next steps
- [x] Rename/clone repo
- [x] Scaffold: routing, API client, auth context, theme system
- [ ] Auth pages (Login/Register/ResetPassword)
- [ ] Feed + PostCard component
- [ ] Post detail + comments
- [ ] Create Post (mentions, media upload, capture modal)
- [ ] Profile
- [ ] Notifications
- [ ] Search
- [ ] Settings (incl. session-expired modal, push notifications)
- [ ] PWA parity
- [ ] Verification pass against the Playwright suite + manual walkthrough

## Links
- Repo: https://github.com/DavidFruin/ssreact
- Related: [[simple-social]] (backend/API this talks to, and the Planning section this rewrite was originally scoped in), [[simple-social-tui]] (why the retro-BBS look doesn't need preserving here, and why the register/reset-password OTP step order must match)
