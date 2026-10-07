---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent
repos: ssreact (workspace after Inbox/repo-consolidation-plan.md Step 2: web/ = source, mobile/ = target, packages/core/ = shared), ssapi (small additive changes)
---

# Port plan: ssreact web → phone app (`ssreact/mobile/`)

A plan for building the phone app in [[ssreact]]'s `mobile/` folder, from the React web app in `web/`.

> **Updated 2026-10-03 (repo consolidation):** the phone app no longer has its own repo. `ssreact` is a pnpm workspace (`web/`, `mobile/`, `desktop/`, `packages/core/`), per `Inbox/repo-consolidation-plan.md`. Shared logic is the workspace package `@ss/core` in `packages/core/`, imported directly by web and mobile. There's **no copy/sync script**. **Prerequisite:** that plan's Step 2 is done (it is, as of `f974931`, 2026-10-07).

It is written to be carried out phase by phase by an agent. Each step names the files and the approach, and says how to check it.

> **Reviewed 2026-10-07 for Sonnet 5 (medium).** Fixes:
> - Phase 1.0: the type-checking setup for `@ss/core`. "No DOM lib" would have broken `fetch`/`FormData` types; it's now an ESLint restricted-globals rule.
> - Phase 2: React version isolation in the monorepo.
> - A new **§4a: how to verify without a Mac or an emulator**.
> - Paths updated to `web/…`; stale react.davidfruin.com/simple-social references removed.
> - The app version shown in Settings.

> **Updated 2026-10-02, Dave's decisions:**
> - **The phone app is the top priority**, ahead of [[sselectron]].
> - **Step 1 (now): family only.**
>   - **Android:** an **APK file** family install directly. No Play Store.
>   - **iPhone:** an **Unlisted App Store** listing (Apple's official route for limited audiences), with **TestFlight internal testing** in the meantime.
>   - Registration is **invite-only with free codes** (`Inbox/access-and-public-launch-plan.md` Step 1).
>   - Unlisted apps get full App Store review, so that plan's **Step 1B** (report/block/terms/privacy) comes before Phase 9.
> - **End goal (Step 2): a normal public app on both the App Store and Google Play**, invite-only and paid (Phase 10 + the access plan's Step 2). It's the planned final stage after the Unlisted listing; only its money and legal choices wait on Dave.
> - **Individual** Apple Developer account. **Expo push is approved.**
>
> Phase 8 (family release) depends on prod (which has run ssapi since 2026-10-06) also having the invite system and the Expo push additions deployed, because the family builds talk to `app.davidfruin.com`.

---

## 0. Read this first: rules for the implementing agent

1. **Sequencing:**
   - Don't start until `Inbox/repo-consolidation-plan.md` **Step 2** (ssreact becomes a workspace) has landed. The ssapi improvement plan's ssreact tasks have already landed (2026-10-02).
   - Check `Areas/active-work.md` and `git log` in ssreact before you begin.
2. **This app is a bridge, not the end state.** That was decided in [[simple-social]]'s Planning section: React Native leads toward true native (Swift/Kotlin) later. Don't pay for anything permanent: no cross-platform UI kit and no design system. **Share logic, not UI.** Dave confirmed this on 2026-10-07 (N2, N2b); it's recorded in the [[ssreact]] note.
3. **Backends:**
   - Development and testing point at `dev.davidfruin.com` (react.davidfruin.com is retired) or at the ssapi local bench.
   - **Never point at `app.davidfruin.com` (prod)** except in a deliberate, Dave-confirmed release test. Same rule as everywhere else in this project.
4. **Pause between phases.** Make one commit per step, and stop and report to Dave after each phase. Items marked **DECISION** or **GATED** wait for Dave.
5. **Where code goes:**
   - The phone app lives in `ssreact/mobile/` and shared logic in `ssreact/packages/core/`.
   - Changes in `ssreact/web/` are limited to Phase 1 (moving logic into `packages/core` and importing it from there). The web app must behave identically afterwards.
   - ssapi changes are limited to the small, additive items marked **[ssapi]**. Never deploy ssapi; Dave does that.
6. **Verify on a real device or emulator, not just "it compiles".** That bar is set in the [[ssreact]] note. When something can't be checked from where you are (camera, push, store install), say so explicitly in the hand-off; don't claim it.
7. **Public repo:** this vault is public. Never paste keys, signing credentials, Expo tokens or `google-services.json` contents into it.

---

## 1. What exists today (inputs)

- **ssreact `web/`** (the workspace's web app, `@ss/web`): about 7.5k lines of TS/TSX.
  - Routes in `web/src/App.tsx`, including the public `/history` page (release list in `web/src/lib/versions.ts`).
  - Logic lives in `web/src/lib/*`.
  - UI is shadcn/Base UI plus Tailwind 4 with six themes (the `.theme-*` blocks in `web/src/index.css`).
  - All paths below that start `lib/`, `components/` or `pages/` are under `web/src/`.
- **Browser-only code, which can't move over as-is:**

| ssreact file | Browser dependency | Native replacement |
|---|---|---|
| `lib/api.ts` | `localStorage`, relative `/api.php` URLs, `File` | Injected token store + absolute base URL + upload descriptor (Phase 1) |
| `lib/auth-context.tsx` | `localStorage`, `document.documentElement` themes, `useNavigate` | Core session-expiry queue + a native provider |
| `lib/media-image.ts` | `<canvas>`, `Image`, `URL.createObjectURL` | `expo-image-manipulator` (resize + rotate) |
| `lib/media-limits.ts` → `getMediaDuration` | `<video>`/`<audio>` element | Duration returned by the picker or recorder (`expo-image-picker` asset `.duration`) |
| `lib/push.ts`, `register-sw.ts`, `public/sw.js` | Web Push, service worker | `expo-notifications` plus a server-side Expo push path (Phase 6) |
| `lib/use-unseen-notifications.ts` | `navigator.setAppBadge`, SW `postMessage`, `document.visibilityState` | `Notifications.setBadgeCountAsync`, `AppState` |
| `components/post/CaptureModal.tsx` | `getUserMedia`, `MediaRecorder`, canvas, `AudioContext` | `expo-camera` (photo/video), `expo-audio` (record) |
| `components/post/VideoPlayer.tsx`, `AudioPlayer.tsx` | `<video>`/`<audio>`, Fullscreen API | `expo-video`, `expo-audio` |
| `components/post/MentionTextarea.tsx` | `<textarea>` caret and selection | `TextInput` with `onSelectionChange` |
| `pages/CreatePostPage.tsx` | `localStorage` draft, `<input type=file>` | AsyncStorage draft, `expo-image-picker` |
| `components/layout/ThumbNav.tsx`, `Header.tsx`, `ScrollTopButton.tsx` | DOM layout, media queries | Bottom tab bar + stack navigation |
| `components/ui/*` (shadcn) | DOM | Not ported. Use a few plain RN primitives instead (see §5) |

- **Pure logic that moves over unchanged:**
  - `lib/types.ts`, `lib/format.ts`, `lib/toast.ts` (pub-sub store)
  - `lib/post-cache.ts`, `lib/feed-cache.ts`
  - `lib/roadmap-data.ts` and `lib/versions.ts` stay in `web/` (Roadmap and History aren't ported; see §5). The phone app shows its **own** version in Settings, plus a link to the website's `/history` (Phase 7).
  - the tokenizing half of `lib/post-text.tsx`
  - the waiter-queue logic inside `auth-context.tsx`

---

## 2. Decisions this plan makes (Dave can override)

| # | Decision | Why |
|---|---|---|
| N1 | **Expo** (managed workflow, current SDK) with **expo-router** (confirmed by Dave 2026-10-07: Expo is React Native's standard toolkit, not just for multi-platform apps) | No Xcode or Mac needed: EAS Build does iOS builds in the cloud. Day-to-day testing runs on Dave's real phones through **Expo Go**, and later a development build (see §4a), so an Android emulator is optional. File-based routes map 1:1 to the web app's route table. |
| N2 | **Share logic only**, through the workspace package **`@ss/core`** (`ssreact/packages/core/`), imported by `web/` and `mobile/` (and later `desktop/`) | Fits "RN is a bridge": no UI kit, only plain TypeScript logic. The repo consolidation (2026-10-03) makes the shared package free, so there's no copying, sync script or drift check. A lint rule keeps the package free of browser APIs (Phase 1). |
| N2b | **No universal UI** (decided by Dave, 2026-10-07) | React Native and Electron are both bridges. A react-native-web "one UI everywhere" app was considered and rejected, because it would make React Native the long-term base. Web keeps its own UI; mobile gets its own; only `@ss/core` logic is shared. |
| N3 | **Plain `StyleSheet` + a theme-token object** ported from `index.css`, with no NativeWind or Tamagui | Keeps all six themes with zero dependencies. A UI kit would be thrown away when native arrives. |
| N4 | **Tokens in `expo-secure-store`**, not AsyncStorage | Keychain/Keystore is the platform-correct place for a 30-day refresh token. |
| N5 | **iOS and Android together** (decided 2026-10-02) | Family members use both. Android ships as an APK and iPhone as an Unlisted App Store app (TestFlight first). Every phase is checked on both platforms, using an Android emulator plus a real Android phone, and iOS through EAS builds on TestFlight. No Mac is needed: EAS builds and submits iOS in the cloud. |
| N7 | **Individual** Apple Developer account ($99/yr); no Google Play account for Step 1 (decided 2026-10-02) | iPhone needs the Apple account for TestFlight and the Unlisted listing. Android uses a sideloaded APK; a Play account only comes in at Step 2, unless Android developer verification needs one (Phase 8.1). Step 2 revisits individual vs. organization before charging money (access plan §2.3). |
| N6 | **Leave out the marketing and static pages** (Landing, About, API docs, Conduct, Roadmap, Download). Settings links out to the website for them. | Nobody reads API docs in a phone app, and keeping the content in one place avoids drift. |

---

## 3. Target structure (inside the ssreact workspace)

```
ssreact/
  packages/core/            @ss/core: api-client, types, format, toast, caches, media-limits, post-text-tokens,
    package.json            session-expiry, media-url. NO browser or React Native APIs (lint:core enforces it)
    src/index.ts
  web/                      the existing web app; now imports @ss/core
  mobile/
    app/                    expo-router routes (mirror web's App.tsx)
      _layout.tsx           providers: Theme, Auth, Toast; stack
      (auth)/login.tsx  register.tsx  reset-password.tsx
      (tabs)/_layout.tsx    bottom tabs: Feed, Post, Search, Notifications(badge), Profile
      (tabs)/feed.tsx  create-post.tsx  search.tsx  notifications.tsx  profile.tsx
      profile/[id].tsx  post/[id].tsx  settings.tsx
    src/
      platform/             native adapters: secure-store token store, config (API base), push, badge, upload
      theme/                tokens.ts (6 themes ported from web/src/index.css), ThemeProvider
      components/           PostCard, CommentItem, MentionInput, MediaView, VideoPlayer, AudioPlayer, Capture, ...
    app.config.ts           reads EXPO_PUBLIC_API_BASE (default: https://dev.davidfruin.com)
    metro.config.js         Expo's monorepo config (see 2.1)
  desktop/                  later (desktop plan)
```

---

## 4a. How verification works (read before Phase 2)
The implementing agent usually runs somewhere **without** a phone, a Mac or (often) an Android emulator. Be explicit about which checks you ran and which Dave must do; never claim a device check you couldn't run.

| Check | Who / how |
|---|---|
| Types, lint, `@ss/core` unit tests, web build | Agent: `pnpm lint`, `pnpm --filter @ss/core test`, `pnpm build`, `pnpm --filter @ss/mobile exec tsc --noEmit` |
| Backend changes ([ssapi]) | Agent: the ssapi local bench (`php -S`, see `Inbox/ssapi-improvement-plan.md` Phase 0) |
| Screens and flows, Phases 2–5 | **Dave's phone in Expo Go.** The agent runs `pnpm mobile -- --tunnel` (or `--lan` on the same network) and gives Dave the QR code/URL. Dave reports back. An Android emulator is fine too, if the machine has one. |
| Push (Phase 6), anything Expo Go can't run | A **development build** (`eas build --profile development -p android`, and `-p ios` once the Apple account exists), installed on Dave's phone(s) |
| Release builds (Phase 8+) | EAS builds, installed by Dave |

When a phase ends, the hand-off lists: what the agent verified itself, and a short **"Dave, please check on your phone"** list with exact steps.

## 4. Phases

### Phase 1: create `packages/core` and move the platform-free logic into it (the only `web/` work)
The goal is that the web app behaves **identically** afterwards. This is pure refactoring.

**1.0 Create the package:** `packages/core/package.json` with:
- `"name": "@ss/core"`, `"private": true`;
- `"type": "module"`;
- `"main"` and `"types"` pointing at `src/index.ts`. It's consumed as TypeScript source, with no build step: Vite and Metro both compile it.

Also add a `tsconfig.json` (strict, `"lib": ["ES2022", "DOM"]`, `noEmit`).
- **Why keep the DOM lib:** `fetch`, `FormData`, `URLSearchParams`, `Blob` and `Response` are typed there, and the API client needs them. They also exist in React Native, so using them is fine.
- **What actually keeps core portable:** an ESLint rule in `packages/core/eslint.config.js`, `no-restricted-globals` for `window`, `document`, `localStorage`, `sessionStorage`, `navigator`, `location` and `history`, plus `no-restricted-imports` for `react`, `react-dom`, `react-native` and anything under `web/`.
- **`@ss/core` has no dependencies at all.** It's plain TypeScript, with no React. That keeps it safe for both apps, which use different React versions (Phase 2.1).

Add `"@ss/core": "workspace:*"` to `web/package.json` and run `pnpm install`.

**1.1 Move the pure modules** from `web/src/lib/` to `packages/core/src/`:
- `types.ts`, `format.ts`, `toast.ts`, `post-cache.ts`, `feed-cache.ts`
- `media-limits.ts` (the `getMediaLimits` + fallback part only; `getMediaDuration` stays in `web/src/lib`)

Re-export them from `src/index.ts`, and update web's imports to `import { … } from '@ss/core'`. Don't leave shims at the old paths.

**1.2 Split the post-text renderer** into:
- `packages/core/src/post-text-tokens.ts`: a pure `tokenizePostText(text, mentions): Token[]`, where a token is `{kind:'text',value} | {kind:'url',href,trail} | {kind:'mention',id,email|null}`. Move `TOKEN_RE` and `splitTrailingPunctuation` here unchanged.
- `web/src/lib/post-text.tsx` keeps rendering tokens into `<a>`/`<Link>`/`<span>`.

**1.3 Make the API client injectable.** Turn `ApiClient` into `packages/core/src/api-client.ts`:
```ts
export interface TokenStore {            // synchronous on purpose: RN hydrates it once at boot
  get(key: string): string | null;
  set(key: string, value: string | null): void;
}
export type UploadFile = Blob | { uri: string; name: string; type: string }; // web File | RN asset
export interface ApiConfig {
  apiUrl: string;          // '/api.php' on web, `${base}/api.php` on native
  mediaUrl: string;        // '/media.php' or `${base}/media.php`
  store: TokenStore;
  extraHeaders?: Record<string, string>;   // mobile sends X-Client: ssreact-mobile/<version> (see 3.3)
}
export function createApiClient(config: ApiConfig) { return new ApiClient(config); }
```
- Replace every `localStorage.*` call with `this.store.get/set`, and every `BASE_URL`/`MEDIA_URL` with `this.config.*`. Change `uploadMedia(file: File)` to `uploadMedia(file: UploadFile)`; `FormData.append('file', file as Blob)` works on both platforms.
- `web/src/lib/api.ts` becomes:
  ```ts
  export const api = createApiClient({ apiUrl: '/api.php', mediaUrl: '/media.php', store: localStorageStore });
  ```
  where `localStorageStore` wraps `localStorage` in try/catch.

**1.4 Extract the session-expiry waiter queue** from `web/src/lib/auth-context.tsx` into `packages/core/src/session-expiry.ts`:
```ts
createSessionExpiryQueue({ onShow, onHide })
```
It returns `{ handler, relogin(retryAll), logout() }`. Move the `waitersRef`/`showingRef` logic and its comments over unchanged; that design is deliberate (see the [[ssreact]] note). `auth-context.tsx` then consumes it.

**1.5 Add a media URL helper:** `packages/core/src/media-url.ts` with `resolveMediaUrl(base, path)`. Web passes `''`; native passes the API base. Every place that renders `post.mediaUrl` or a `thumbnailUrl` goes through it.

**Verify Phase 1:**
- `pnpm lint && pnpm build` (workspace root) are clean.
- `pnpm --filter @ss/core exec tsc --noEmit` and `pnpm --filter @ss/core lint` pass, and the restricted-globals rule demonstrably fires: temporarily add `localStorage.getItem('x')` to a core file, see lint fail, then remove it. Make sure `pnpm lint` at the root runs core's lint.
- **Unit tests for core:** add **Vitest** to `packages/core` with tests for `tokenizePostText` (URL vs. mention ordering, trailing punctuation, `@[26]` inside a URL, deleted mention), `format` (timestamps, relative time), the session-expiry queue (two concurrent waiters both resolve after re-login; logout resolves both with undefined) and `resolveMediaUrl`. These are the only checks of shared logic that run without a device. `pnpm --filter @ss/core test` must pass.
- Run a manual pass against the dev proxy: login, reload (no 401 burst, which is P1), feed, like, comment with a mention, upload an image, session-expired modal (corrupt `ss_jwt` in DevTools to trigger it), logout.

**Commits:** one per step: `refactor(core): …`.

### Phase 2: scaffold `mobile/`
**2.1 Create the app inside the workspace:**
- From the ssreact root: `pnpm create expo-app mobile --template` (TypeScript, expo-router template). Set `"name": "@ss/mobile"` in `mobile/package.json`, and add `"@ss/core": "workspace:*"`.
- Add `expo-router`, `expo-secure-store`, `@react-native-async-storage/async-storage`, `expo-image`, and `react-native-safe-area-context` (Expo includes it).
- **Monorepo setup:**
  - Follow Expo's current "Work with monorepos" guide. Recent Expo SDKs configure Metro for workspaces automatically, so `mobile/metro.config.js` usually only needs `getDefaultConfig(__dirname)`.
  - Confirm that Metro resolves `@ss/core` from `packages/core` and picks up edits to it live.
  - The root `.npmrc` already has `node-linker=hoisted` (consolidation Step 2.2). Keep it; React Native's tooling expects it with pnpm.
  - **React versions:** Expo pins exact `react`/`react-native` versions, and `web/` uses its own `react` (`^19.2.x`). They may differ. With a hoisted layout, check `pnpm why react` and confirm that `mobile/` resolves **Expo's** React, from `mobile/node_modules` if versions differ, and web resolves its own. Two copies of React inside one app cause "Invalid hook call" errors. `@ss/core` must not import React (Phase 1.0), so it can't cause this. If Metro picks the wrong copy, follow the Expo monorepo guide's `resolver` settings rather than inventing a workaround.
  - **Use `npx expo install <pkg>`** (not `pnpm add`) for every Expo or React Native package, so versions match the SDK.
- Root scripts: add `"mobile": "pnpm --filter @ss/mobile start"`.

**2.2** (Removed 2026-10-03: there's no sync script any more. `@ss/core` is imported directly.)

**2.3 Add `app.config.ts`:**
- `extra.apiBase = process.env.EXPO_PUBLIC_API_BASE ?? 'https://dev.davidfruin.com'`.
- `scheme: 'simplesocial'` for deep links.
- App name "Simple Social".
- Icon and adaptive icon from `web/public/pwa-icons/` (the black-and-white heart).

**2.4 Platform adapters (`mobile/src/platform/`):**
- `tokenStore.ts`: an in-memory `Map` that implements `TokenStore`. `hydrate()` calls `SecureStore.getItemAsync` for `ss_jwt`, `ss_refresh` and `ss_user` once, before the first render. `set()` updates the map and calls `SecureStore.setItemAsync`/`deleteItemAsync` without awaiting.
- `api.ts`:
  ```ts
  createApiClient({ apiUrl: `${base}/api.php`, mediaUrl: `${base}/media.php`, store: tokenStore,
    extraHeaders: { 'X-Client': `ssreact-mobile/${version} (${Platform.OS})` } })
  ```

**2.5 Root layout:**
- Hydrate the token store and then render, using `expo-splash-screen` to hold the splash until then.
- Add providers: Theme, Auth (using `@ss/core`'s session-expiry queue), and a Toast host.

**Verify Phase 2** (see §4a for how):
- `pnpm mobile` (`expo start`). Dave opens it on his phone in **Expo Go**, or an emulator is used if this machine has one.
- A temporary debug screen calls `api.getMediaLimits()` after a hard-coded test login against **dev** and shows the JSON.
- `pnpm --filter @ss/mobile exec tsc --noEmit` and `pnpm lint` (including `lint:core`) are clean. Editing a file in `packages/core` hot-reloads both `pnpm dev` (web) and the running Expo app.

### Phase 3: auth and app shell
- **3.1 Screens:** Login, Register and Reset Password. Register's first step has an **Invite code** field (access plan §1.3/1.4: case-insensitive, sent as `inviteCode` with `sendRegisterOTP`, with the hint "Simple Social is invite-only. Ask the person who invited you for a code."). Until the backend invite system (access plan Step 1) is deployed, the server simply ignores the extra field, so build it now anyway. Port `OtpAuthFlow`'s three steps exactly, in the same order as `simple-social-tui`'s `auth.c` (noted in [[ssreact]]).
  - Use `TextInput` with `secureTextEntry`, `autoComplete="email"` / `"password"` / `"one-time-code"`, and `textContentType` for iOS autofill.
- **3.2 Navigation:**
  - `RequireAuth` / `RequireGuest` become redirect logic in the `(auth)` and `(tabs)` layouts, based on `user`.
  - The tabs are Feed, Post, Search, Notifications and Profile. Settings is reached from Profile's header button.
  - This **replaces** ThumbNav and Header. The hand preference only mirrors a floating "new post" button if one is added later; otherwise it is ignored on native. Note that in the hand-off.
- **3.3 [ssapi], additive:** in `auth.php` → `deviceNameFromUserAgent()`, recognise the native client so the Devices list shows a sensible name.
  - RN's default UA looks like `okhttp/…` or `CFNetwork…`.
  - Prefer the `X-Client` header when present: `$_SERVER['HTTP_X_CLIENT']`. Map `ssreact-mobile/<v> (android)` → "Simple Social app (Android)" and `(ios)` → "… (iOS)".
  - Pass it in from `sessionCreate()`.
  - Verify on the ssapi local bench (see the ssapi plan's Phase 0): `curl -H 'X-Client: ssreact-mobile/1.0 (android)' … login`, then `getSessions` shows the new name.
- **3.4 Session-expired modal:** a non-dismissible RN `Modal` driven by the core queue. Back press is ignored (`onRequestClose={() => {}}`).
- **3.5 Theme:**
  - Port the six themes from `web/src/index.css` to `mobile/src/theme/tokens.ts` as `{ background, foreground, card, primary, primaryForeground, muted, mutedForeground, border, destructive, success, warning }` per theme.
    - Stored values and their labels: `light` (Light), `dark` (Dark Blue), `red` (Red), `blue` (Light Blue), `hacker` (Hacker), `gray` (Dark; grayscale except the red/green/amber status colours).
    - **Copy the values exactly from `web/src/index.css`**, which is now the source of truth (simple-social is archived). Convert any non-hex colour formats to hex.
  - Theme changes call `api.updateTheme` and re-render through context. Set the status bar style per theme.

**Verify Phase 3 (emulator against dev):**
- Login with a wrong password shows the server error and clears the field.
- Login succeeds; kill the app and relaunch; you're still logged in, with no 401 and no refresh call on the first request.
- Force expiry (a dev-only button that corrupts the stored JWT) → the modal appears; re-login retries the waiting request; logout returns to Login.
- Devices in Settings (once built) shows the native device name.
- All six themes switch live.

### Phase 4: read paths (feed, post, profile, notifications, search)
- **Feed:**
  - `FlatList`, page size 25, "Load more" at the end (`onEndReached`, matching ssreact's offset logic).
  - Pull-to-refresh reloads offset 0.
  - Use the embedded `commentCount` from ssapi P5, falling back to `getPostCommentCounts`.
- **PostCard:**
  - Render text from `@ss/core`'s post-text tokens. URLs open with `Linking.openURL` (http/https only, which the tokenizer already guarantees). Mentions use `router.push('/profile/'+id)`. Deleted mentions show italic "@deleted user".
  - Like toggle, likes list in a bottom-sheet `Modal`, owner delete through `Alert.alert` confirm.
  - Timestamps go through `@ss/core`'s format helpers.
- **Post detail:**
  - Use `@ss/core`'s post cache the same way the web app does (skip `getPostById` on a cache hit).
  - Comments are paginated. Comment delete uses `Alert.alert`.
- **Profile:** own profile and someone else's, follow/unfollow, followers/following sheets.
- **Notifications:** Today / Yesterday / Earlier grouping, and copy and link targets **exactly** as ssreact's `NotificationsPage` (which itself matches the original). Mark as seen refreshes the badge (Phase 6).
- **Search:** one `getUsers()` fetch, filtered on the client, with Follow toggles. This matches the recorded design decision.
- **Media display:**
  - `expo-image` for images, with tap → full-screen viewer.
  - Video and audio come in Phase 5. Show a placeholder until then.
  - Every media URL goes through `resolveMediaUrl(base, …)`.

**Verify Phase 4 (on a phone via Expo Go, against dev, using a dev test account per the [[ssreact]] note):**
- Every screen renders real data.
- Like/unlike round-trips (count changes on the server).
- A mention opens the right profile.
- A post containing a URL with a literal `@[26]` inside it renders as one link, not a link plus a mention (the tokenizer's ordering guarantee).
- Leave the data as you found it.

### Phase 5: write paths and media
- **5.1 Create Post, text only first:**
  - `MentionInput`: `TextInput` + `onSelectionChange` + an `@word` trigger, a suggestion list from `getUsers()`, and the same `resolve(text)` → `@[id]` conversion before submit as ssreact's `MentionTextarea`.
  - Port its character filter (printable ASCII + Latin-1, no newlines) and the "Returns aren't allowed" toast.
  - Draft persistence in AsyncStorage under `ss_post_draft`, with the same shape as web.
- **5.2 Media from the library:**
  - `expo-image-picker` (images + videos).
  - Images are re-encoded with `expo-image-manipulator`: resize the longest side to 1920, apply rotation, save as JPEG 0.92 (PNG for alpha). This replaces `renderImageFile`.
  - Duration check uses the asset's `duration` against `getMediaLimits().maxSeconds` before upload. This replaces `getMediaDuration`.
  - Upload with `api.uploadMedia({ uri, name, type })`. Remove calls `api.deleteMedia` for orphans, as on web.
- **5.3 Capture:**
  - `expo-camera`: photo, and video with `maxDuration = maxSeconds`, front/back flip.
  - `expo-audio` recording with a hard stop at `maxSeconds`.
  - The "stale closure" bug ssreact hit (see the [[ssreact]] note, Seventh polish round) doesn't apply when the native `maxDuration` does the stopping. Still guard any timer with a ref, not state.
- **5.4 Playback:**
  - `expo-video` with native controls (fullscreen included) and `expo-audio` for audio.
  - Only one item plays at a time: port `active-media.ts`'s idea as `core`-free platform code.
- **5.5 Known cross-platform media risk (flag it, don't work around it silently):**
  - Recordings made in browsers are WebM. **iOS can't play WebM.** Prod and dev transcode to MP4/MP3 because ffmpeg exists there, but any server **without** ffmpeg (e.g. the retired react.davidfruin.com chroot) keeps WebM originals as uploaded, and those won't play on iOS.
  - Test iOS playback against dev, and check that ffmpeg works on any new host before pointing the phone app at it.
  - Native recordings come out as MP4/M4A (audio) and MP4/MOV (video), which are fine everywhere. ssapi's `ALLOWED_AUDIO_TYPES` (`src/Media/handlers.php`) is currently `audio/wav`, `audio/mpeg`, `audio/mp3`, `audio/webm`, so M4A is **rejected today**. **[ssapi], additive:** add `audio/mp4`, `audio/x-m4a` and `audio/aac` to the allowed audio types, and add the `ftyp` M4A case to S3's `sniffMedia` (it already maps `ftyp` to the `av` family, which the audio check accepts). Verify on the ssapi bench with a real `.m4a` file.

**Verify Phase 5 (a real device is required for capture):**
- Text post with a mention → the server stores `@[id]`.
- Library image → rotate → post, and it renders upright.
- An over-long library video is rejected before upload.
- Capture photo, video and audio on a **real phone** and confirm playback on the other platform.
- Delete the test posts and confirm no orphan media is left.
- If no device is available, say so in the hand-off.

### Phase 6: notifications, badge, deep links (backend work, approved 2026-10-02)
Web Push (VAPID) doesn't exist in React Native. Native push needs FCM (Android) and APNs (iOS). The simplest correct route for an Expo app is the **Expo Push Service**:
- The app gets an `ExponentPushToken[...]` from `expo-notifications`.
- The server POSTs JSON to Expo's push API.
- Expo forwards the message to FCM/APNs.

**[ssapi] changes (additive, approved; they add a new outbound service and a new table column):**
1. Add a migration (through ssapi plan P3's mechanism): `ALTER TABLE push_subscriptions ADD COLUMN kind TEXT NOT NULL DEFAULT 'webpush'`. Expo rows use `kind='expo'`, `endpoint = <ExponentPushToken>`, and empty `p256dh`/`auth`.
2. Add `handle_saveExpoPushToken` / `handle_deleteExpoPushToken`:
   - validate the token with the regex `^ExponentPushToken\[[A-Za-z0-9_-]+\]$`;
   - store the session id the same way web push does, so logout and revoke remove it (ssapi S6);
   - cap 10 per user (ssapi S1).
3. In `pushNotification()`, branch on `kind`:
   - Web push stays as it is.
   - Expo rows are batched into one POST to `https://exp.host/--/api/v2/push/send` with `[{to, title, body, data:{url}, badge: count}]`.
   - Use the same short curl timeouts (ssapi S1) and deferred sending (`defer()`, ssapi P2).
   - On a `DeviceNotRegistered` receipt, delete the row.
   - **Keep `exp.host` as the only allowed host for this kind**, consistent with S1. Branch on `kind` *before* `sendWebPush()`, whose `isAllowedPushEndpoint()` check would otherwise reject Expo rows.
4. Keep the push URL as it is (`/app.html#/post/…`). The app normalises it exactly like `web/public/sw.js` → `normalizeNotificationUrl` (strip through `#`).

**App side:**
- Ask for permission only from a Settings toggle (a user action), like web.
- Register the token, then call `saveExpoPushToken`.
- A tap goes through `Notifications.addNotificationResponseReceivedListener` → normalise the URL → `router.push`.
- Badge: `Notifications.setBadgeCountAsync(count)` from the unseen-count poller. Run the poller only while the app is foregrounded (`AppState`). Mark-as-seen clears the badge immediately.

**Platform setup (Dave, not the agent, because it involves credentials):**
- A Firebase project with `google-services.json` for Android FCM, plus the FCM V1 service-account key uploaded to EAS (`eas credentials`).
- The Apple Developer account (individual) and an APNs key. EAS can create and store the key itself during `eas credentials` / the first iOS build when Dave signs in.
- None of these files are committed: `.gitignore` them and use EAS secrets.

**Verify:**
- **Push needs a development build, not Expo Go.** Recent Expo Go versions don't deliver remote push. Build one with `eas build --profile development` and install it on Dave's phone(s); see §4a.
- ssapi bench: insert an Expo token row and trigger a like; the deferred send runs. The real delivery check is on a device.
- Real device: like a test post from a second account → the notification arrives, the tap opens the post, the badge count is right, and mark-as-seen clears it. Logout → no further pushes.

### Phase 7: settings and remaining parity
- **Settings:**
  - Theme picker, hand preference (stored, mostly unused on native; see 3.2), push toggle.
  - Devices list with revoke and revoke-all, using `Alert.alert` confirms.
  - Log out.
  - Delete Account (password + double confirm).
  - Links that open the website: About, Conduct, Roadmap, API, History (`/history`).
  - **App Version** block (matches the web Settings): the app's own version and build number (`expo-application`/`expo-constants`) and the product release name. Read the release list from `@ss/core` if Phase 1 moved `versions.ts` there; otherwise show the app's version and link to `/history`.
- **Toasts:** one host component subscribed to `@ss/core`'s toast store, with solid colours (the lesson from the Red-theme toast bug in the [[ssreact]] note).
- **Empty, loading and error states** on every screen.
- **Accessibility:** `accessibilityLabel` on icon-only buttons, and Dynamic Type / font scaling left enabled.
- **Not in Phase 7:**
  - the report/block UI, blocked-users list and terms gate come in **Phase 9.0** (needed for the Unlisted App Store review);
  - the paywall and Restore Purchases are **Step 2 only** (access plan §2.2).

### Phase 8: family release: Android APK + iPhone via TestFlight (interim)
Steps marked **Dave** need his accounts or credentials. The agent prepares everything else, never committing secrets.

**8.1 Accounts (Dave, week 1):**
- **Apple Developer Program, individual:** $99/yr; identity verification can take a day or two. Needed for TestFlight now and the Unlisted App Store listing (Phase 9).
- **No Google Play account is needed.** Android ships as an APK file.
- **Check one thing early:** Google has announced **developer verification for sideloaded apps on certified Android devices**, rolling out by country from 2026.
  - If it applies where the family lives, Dave must register as a verified Android developer and register the package name, or Android will block the install.
  - Check Google's current Android developer-verification page before the first release. It may need a small fee or the Play Console account after all.

**8.2 App identity and config (agent):**
- In `app.config.ts`:
  - `name: 'Simple Social'`
  - `ios.bundleIdentifier` / `android.package`: `com.davidfruin.simplesocial`. **DECISION:** Dave confirms; it is permanent, and the App Store listing and every APK update reuse it.
  - auto-incremented build numbers / `versionCode`
  - `ios.config.usesNonExemptEncryption: false`
- **Permission strings** (camera, microphone, photo library), written in plain language.
- **Icons:** a 1024×1024 iOS icon (no transparency) and an Android adaptive icon, from the black-and-white heart in `web/public/pwa-icons/`.

**8.3 Build profiles (`eas.json`):**
- `development`: dev client, dev backend.
- `preview`: dev backend, for Dave's own testing.
- `family`: **the `app.davidfruin.com` backend**.
  - Android: `"android": { "buildType": "apk" }`, `distribution: internal`.
  - iOS: `distribution: store`, for TestFlight and later the App Store.
- The `family` profile needs prod running ssapi with the invite system (access plan §1.6) and Expo push (Phase 6). **That's the release blocker.**

**8.4 Android: APK file:**
- Build with `eas build -p android --profile family`. The result is a signed `.apk`.
- **Signing key, critical:**
  - EAS creates and stores the Android keystore.
  - **Dave downloads a backup right away** with `eas credentials` → Android → download keystore, and keeps it somewhere safe, outside every repo and outside this vault.
  - Every update **must** be signed with the same key. If it's lost, every family member has to uninstall (losing nothing server-side, but it's a hassle) and reinstall.
- **Hosting:** put the APK at a stable URL Dave controls, e.g. `https://app.davidfruin.com/downloads/simple-social.apk`.
  - Serve it with `Content-Type: application/vnd.android.package-archive`.
  - Put a SHA-256 checksum next to it.
  - The app is useless without an invite code, so a public URL is acceptable. A secret-path URL works too.
- **Install (per family member):**
  1. Open the link in Chrome on the phone.
  2. Allow "Install unknown apps" for Chrome when Android asks.
  3. Install. Google Play Protect may show an "unrecognised developer" warning; tap "Install anyway". Developer verification (8.1) is what removes that warning over time.
  4. Register with the invite code and turn on notifications in Settings. Push works on any phone with Google Play services, through FCM.
- **Updates (sideloaded apps don't update themselves):**
  - **Recommended: EAS Update** (over-the-air JS updates) for normal changes. The app downloads new JS on launch, with no reinstall. Changes to native modules or permissions still need a new APK.
  - For those, add a small **"New version available"** check:
    - The app fetches `https://app.davidfruin.com/downloads/android-version.json` (`{versionCode, url, sha256, notes}`) on launch.
    - If `versionCode` is higher than its own, it shows a banner that opens the APK URL. Android installs it over the old one, because the signing key is the same.
  - The same mechanism lets Dave require an update if a security fix ever needs it (`minVersionCode`).

**8.5 iPhone, interim: TestFlight internal testing:**
This gets family onto iPhones quickly while Phase 9 (the Unlisted App Store listing) is prepared.
- `eas build -p ios --profile family`, then `eas submit -p ios`.
- **Dave:** add each family member in **App Store Connect → Users and Access** with the **most limited role**, restrict their app access to Simple Social, and add them to an **internal testing group**. They install the TestFlight app, then Simple Social from it. There's no review.
- **Builds expire after 90 days.** Upload a new one before then. That stops mattering once Phase 9 is live; family members then move to the App Store version through the unlisted link, after which TestFlight can be stopped.

**8.6 Onboarding a family member:**
1. Dave creates an invite code on the web `/admin/invites` page (access plan §1.3).
2. He sends the APK link (Android) or the TestFlight invite (iPhone; the unlisted App Store link after Phase 9), plus their code.
3. They register with the code and turn on notifications.

**8.7 Verify (on real phones, because this is the release):**
- One Android phone (APK) and one iPhone (TestFlight), with family builds against prod:
  - register with a fresh invite code (then delete that test account);
  - feed, post with a photo and a capture;
  - push arrives and opens the post;
  - badge clears;
  - logout stops pushes.
- **Android update path:**
  - Publish an EAS Update (a visible text change) → it appears after a relaunch.
  - Build a second APK with a higher `versionCode` and update `android-version.json` → the banner appears and the update installs over the old version, keeping you logged in.
- Record the results, the iOS build's expiry date and where the keystore backup lives (*where*, not the key) in the [[ssreact]] note.

### Phase 9: iPhone: Unlisted App Store distribution (after the family release)
**Unlisted App Distribution** is Apple's official route for apps meant for a limited audience. The app is on the real App Store but **doesn't appear in search, charts or categories**; only people with the direct link can find it. It **goes through full App Store review**, so the content rules apply.

**9.0 Prerequisite: access plan Step 1B** (report, block, admin actions, terms gate, Terms and Privacy pages, contact email) must be built and **deployed to prod**. Apple's guideline 1.2 applies to unlisted apps that have user posts too. The phone UI for it is:
- a "…" action sheet on posts and comments → **Report** (reason + optional details);
- profile header "…" → **Report user** / **Block user**;
- reported or blocked content disappears straight away;
- Settings → **Blocked users** with Unblock;
- Settings links to **Terms**, **Privacy Policy** and the contact email;
- a **Terms gate** after login (and on register) when `termsVersionAccepted < termsVersionCurrent`; it can't be dismissed and Back is ignored;
- a suspended-account login shows the server's message as it is.

**Verify the prerequisite:** access plan Step 1B's A5 scenarios, run through the iPhone and Android apps against the bench or dev. Then ship the moderation UI to Android too: through EAS Update, or a new APK if native code changed.

**9.1 Config additions (agent):**
- **iOS privacy manifest:** declare the required-reason APIs used through Expo modules in `ios.privacyManifests`. Fix any warnings App Store Connect reports on upload.
- Keep the permission strings clear (Apple reviews them).

**9.2 App Store Connect content (agent drafts in `store/` in the repo, Dave approves and enters it):**
- `store/listing.md`: name, subtitle, description (say plainly that it's **invite-only**), keywords, category (Social Networking), support URL, privacy policy URL.
- Screenshots: simulator, **test data only, no real users' emails**, sizes 6.9" and 6.5". Even unlisted apps need a product page.
- **App Privacy ("nutrition label")** answers, worked out from the Privacy Policy:
  - collected: email, user content, user ID, diagnostics (server logs);
  - not used for tracking;
  - not sold.
- **Age rating questionnaire:** answer honestly (user-generated content, with reporting and blocking). Expect 12+ or 17+.
- **Review notes:**
  - that it's an invite-only family app requested as unlisted;
  - how moderation works (report → email → admin action, usually within 24h);
  - where Report and Block are;
  - that account deletion is in Settings.
- **Reviewer access:** a demo account on prod. Its credentials go **only** into App Store Connect's review fields, never into a repo or this vault. Also enter a spare invite code there, so the reviewer can test registration.

**9.3 Submit and request Unlisted:**
- `eas build -p ios --profile family` → `eas submit -p ios` → submit the version for **App Review** with **manual release** selected.
- **Dave:** fill in Apple's **Unlisted App Distribution request form** (linked from Apple's developer documentation on unlisted apps) for the app. Explain that it's a private, invite-only app for his family. Apple approves the unlisted request separately from App Review. Ask for it **before** releasing, so the app never appears publicly.
- Check beforehand for the common first-time rejection reasons:
  - **1.2 (UGC):** report, block, terms and contact must all be there and working.
  - **2.1:** crashes or broken links; every Settings link must open.
  - **5.1.1:** account deletion must be in the app.
  - Permission prompts must have clear purpose strings.
  - The demo login must work.
- If rejected: fix, bump the build number, resubmit. Record the reason in the [[ssreact]] note.
- Once it's approved and unlisted, release it. Dave shares the **unlisted App Store link** with family. Updates go through review like any App Store app, and there's **no 90-day expiry**. Retire the TestFlight group.

### Phase 10 (end goal): public launch on the App Store + Google Play
The planned final stage, after Phase 9 is live and stable: invite-only, **paid** (access plan Step 2). Parts that depend on Dave's money and legal decisions (price, organization accounts, payments stack) wait for those decisions.

**Avoid rework along the way (applies to Phases 8–9 too):**
- Keep the same bundle ID / package name.
- Keep the Android signing key backed up; it moves to Play App Signing in 10.3.
- Keep every App Store answer accurate, so going public is a listing change rather than a new app.

**10.1 Prerequisites:**
- The paywall, Restore Purchases and the subscription disclosures (access plan §2.2).
- The organization-account decision (§2.3).

**10.2 iOS:**
- Ask Apple to make the app **public** instead of unlisted. A new submission states the change; the same listing, privacy answers and age rating are updated for payments.
- In-app subscriptions are set up in App Store Connect; RevenueCat is recommended (§2.2).

**10.3 Android:** move from the sideloaded APK to **Google Play**.
- Create a Play Console account: personal $25, or organization.
- **Upload the APK signing key to Play App Signing**, so Play builds can update sideloaded installs. Otherwise family members must uninstall and reinstall once.
- Complete the listing, Data Safety form and IARC rating.
- **Personal accounts:** run a closed test with **at least 12 testers for 14 consecutive days** before production access. Check the current rule; family can be part of it.
- Then production, with a staged rollout.
- Retire the APK update banner once everyone has moved to Play.

**10.4 After launch:** add store badges on the web app's `DownloadPage` (`web/src/pages/DownloadPage.tsx`). Moderation duty scales with users (§2.6).

---

## 5. Screen and component map

| web (`web/`) | mobile (`mobile/`) | Notes |
|---|---|---|
| `LoginPage`, `RegisterPage`/`ResetPasswordPage` (`OtpAuthFlow`) | `(auth)/login`, `register`, `reset-password` | Same OTP step order |
| `FeedPage` | `(tabs)/feed` | FlatList, pull-to-refresh, load more, feed-cache keeps the scroll position |
| `PostPage` | `post/[id]` | post-cache hit skips the fetch |
| `CreatePostPage` + `CaptureModal` + `MediaPreview` | `(tabs)/create-post` + `Capture` | Picker/manipulator/camera/audio |
| `ProfilePage` + `FollowListPopover` | `(tabs)/profile`, `profile/[id]` + sheet | |
| `NotificationsPage` | `(tabs)/notifications` | Exact copy and link targets |
| `SearchPage` | `(tabs)/search` | Client-side filter |
| `SettingsPage` | `settings` | Push toggle → Expo push |
| `SessionExpiredModal` | Native `Modal` | Non-dismissible |
| `Toast` | Toast host | |
| `Header`, `ThumbNav`, `ScrollTopButton` | Tab bar + stack headers; tapping the active tab scrolls to top | Native convention replaces both |
| `LandingPage`, `AboutPage`, `ApiDocsPage`, `ConductPage`, `RoadmapPage`, `DownloadPage`, `HistoryPage` | Not ported; linked from Settings | N6 |
| shadcn `Card`/`Button`/`Input`/`Switch`/`Select`/`AlertDialog`/`Popover`/`Dialog` | `View`+tokens / `Pressable` / `TextInput` / RN `Switch` / action sheet / `Alert.alert` / bottom-sheet `Modal` / `Modal` | Thin local components only |

---

## 6. Website follow-ups
- Phase 8: an APK download page or link at a stable URL (`/downloads/simple-social.apk` + `android-version.json` + checksum). The ssreact `DownloadPage` can say "Phone app: invite-only, ask Dave for a link."
- Phase 9: `/terms` and `/privacy` live at public URLs (access plan Step 1B).
- Phase 10: store badges after the public launch.

---

## 7. Open decisions for Dave
1. ~~N2: share logic only~~: confirmed 2026-10-07 (N2, N2b).
2. **Bundle ID / package name** (`com.davidfruin.simplesocial` proposed). It is permanent.
3. ~~How ssapi reaches prod~~: done 2026-10-06. What's still needed on prod for Phase 8: the invite system (access plan Step 1) and the Expo push additions (Phase 6), each deployed when Dave says so.
4. Where the APK is hosted, and whether the link is public or a secret path.
5. Whether Android developer verification applies in the family's country (8.1). Check before the first release.
6. A contact email, plus approval of the Terms and Privacy texts (needed for Phase 9).
7. Already decided 2026-10-02:
   - Android = APK sideload; iPhone = Unlisted App Store, with TestFlight in the meantime;
   - invite-only registration;
   - public paid launch later (Step 2, gated);
   - individual Apple account;
   - Expo push approved;
   - EAS Update recommended for Android updates.

---

## 8. Suggested order, stopping points and timeline

| Week | Agent work | Dave |
|---|---|---|
| 1 | Repo consolidation Step 2 (if not done), Phase 1 (`@ss/core`), Phase 2 (scaffold `mobile/`); access plan Step 1 (invites) backend + web | Apple account sign-up; confirm the bundle ID; check Android developer verification; decide how ssapi gets to prod |
| 2 | Phases 3–4 (auth with invite field, shell, read screens) | Install the preview builds on his own phones |
| 3 | Phases 5–6 (media, capture, Expo push + ssapi additions) | Firebase and APNs setup; real-device capture and push checks |
| 4 | Phase 7; APK + update check; TestFlight upload; access plan Step 1B (moderation) starts | Deploy ssapi to prod; host the APK; create invite codes; **family is using it** |
| 5 | Step 1B backend + web + phone moderation UI | Approve the Terms and Privacy; choose a contact email |
| 6 | Phase 9 listing drafts; submit to App Review | Deploy Step 1B to prod; fill in the Unlisted request form; reviewer demo account |
| ~6–7 | Fixes if review asks for any | Unlisted link goes to the iPhone family members; retire TestFlight |

- **Family on Android (APK) and iPhone (TestFlight): about 4–5 weeks.**
- **iPhone on the Unlisted App Store: about 6–7 weeks.** The App Review and Unlisted approval times are Apple's, usually days. A rejection adds roughly a week.
- Stop after each phase, as before.
- **Then the end goal, Step 2 / Phase 10 (public on both stores, paid): about 4–6 more weeks** after the Unlisted listing. Most of the waiting is Dave's decisions and Google's 14-day test.
