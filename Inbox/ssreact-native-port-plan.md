---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent
repos: ssreact @ eb9ac21 (source), ssreact-native (target, empty), ssapi (small additive changes)
---

# Port plan: ssreact → ssreact-native

A plan for building [[ssreact-native]], the phone app, from [[ssreact]], the React web app. It is written to be carried out phase by phase by an agent. Each step names the files and the approach, and says how to check it.

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
> Phase 8 (family release) depends on prod running ssapi with the invite and Expo push additions, because the family builds talk to `app.davidfruin.com`.
---

## 0. Read this first: rules for the implementing agent

1. **Sequencing:**
   - Don't start until the ssreact tasks in [[ssapi]]'s improvement plan (`Inbox/ssapi-improvement-plan.md`: P1, P5, P6, P8, S15, C6) have landed.
   - Phase 1 below refactors the same `src/lib/api.ts` those tasks change.
   - Check `Areas/active-work.md` and `git log` in ssreact before you begin.
2. **This app is a bridge, not the end state.** That was decided in [[simple-social]]'s Planning section: React Native leads toward true native (Swift/Kotlin) later. Don't pay for anything permanent: no cross-platform UI kit and no design system. **Share logic, not UI.** This settles the open "code-sharing approach" question in the [[ssreact-native]] note (see §2). Record it there when Dave confirms.
3. **Backends:**
   - Development and testing point at `dev.davidfruin.com`, or at `react.davidfruin.com`'s isolated test copy.
   - **Never point at `app.davidfruin.com` (prod)** except in a deliberate, Dave-confirmed release test. Same rule as everywhere else in this project.
4. **Pause between phases.** Make one commit per step, and stop and report to Dave after each phase. Items marked **DECISION** or **GATED** wait for Dave.
5. **Repos:**
   - ssreact changes are limited to Phase 1 (the core extraction).
   - ssapi changes are limited to the small, additive items marked **[ssapi]**. Never deploy ssapi; Dave does that.
6. **Verify on a real device or emulator, not just "it compiles".** That bar is set in the [[ssreact]] note. When something can't be checked from where you are (camera, push, store install), say so explicitly in the hand-off; don't claim it.
7. **Public repo:** this vault is public. Never paste keys, signing credentials, Expo tokens or `google-services.json` contents into it.

---

## 1. What exists today (inputs)

- **ssreact:** about 7.4k lines of TS/TSX.
  - 22 routes in `src/App.tsx`.
  - Logic lives in `src/lib/*`.
  - UI is shadcn/Base UI plus Tailwind 4 with six themes (the `.theme-*` blocks in `src/index.css`).
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
  - `lib/roadmap-data.ts` (only if Roadmap is ported, which it isn't; see §5)
  - the tokenizing half of `lib/post-text.tsx`
  - the waiter-queue logic inside `auth-context.tsx`

---

## 2. Decisions this plan makes (Dave can override)

| # | Decision | Why |
|---|---|---|
| N1 | **Expo** (managed workflow, current SDK) with **expo-router** | No Xcode or Android Studio needed on Dave's Linux machines. EAS Build does iOS builds in the cloud. File-based routes map 1:1 to ssreact's route table. |
| N2 | **Share logic only**, through a `src/core/` folder extracted inside ssreact and **copied** into ssreact-native by a sync script pinned to an ssreact commit | Fits "RN is a bridge". A shared npm package or monorepo costs setup time for an app that will be replaced. The sync script plus a drift check (§4, step 2.2) stops the copy from silently going stale. Upgrade to a shared package later only if drift actually hurts. |
| N3 | **Plain `StyleSheet` + a theme-token object** ported from `index.css`, with no NativeWind or Tamagui | Keeps all six themes with zero dependencies. A UI kit would be thrown away when native arrives. |
| N4 | **Tokens in `expo-secure-store`**, not AsyncStorage | Keychain/Keystore is the platform-correct place for a 30-day refresh token. |
| N5 | **iOS and Android together** (decided 2026-10-02) | Family members use both. Android ships as an APK and iPhone as an Unlisted App Store app (TestFlight first). Every phase is checked on both platforms, using an Android emulator plus a real Android phone, and iOS through EAS builds on TestFlight. No Mac is needed: EAS builds and submits iOS in the cloud. |
| N7 | **Individual** Apple Developer account ($99/yr); no Google Play account for Step 1 (decided 2026-10-02) | iPhone needs the Apple account for TestFlight and the Unlisted listing. Android uses a sideloaded APK; a Play account only comes in at Step 2, unless Android developer verification needs one (Phase 8.1). Step 2 revisits individual vs. organization before charging money (access plan §2.3). |
| N6 | **Leave out the marketing and static pages** (Landing, About, API docs, Conduct, Roadmap, Download). Settings links out to the website for them. | Nobody reads API docs in a phone app, and keeping the content in one place avoids drift. |

---

## 3. Target structure (ssreact-native)

```
app/                      expo-router routes (mirror ssreact's App.tsx)
  _layout.tsx             providers: Theme, Auth, Toast; stack
  (auth)/login.tsx  register.tsx  reset-password.tsx
  (tabs)/_layout.tsx      bottom tabs: Feed, Post, Search, Notifications(badge), Profile
  (tabs)/feed.tsx  create-post.tsx  search.tsx  notifications.tsx  profile.tsx
  profile/[id].tsx  post/[id].tsx  settings.tsx
src/
  core/                   COPIED from ssreact/src/core by scripts/sync-core.sh -- never edit here
  platform/               native adapters: secure-store token store, config (API base), push, badge, upload
  theme/                  tokens.ts (6 themes ported from index.css), ThemeProvider
  components/             PostCard, CommentItem, MentionInput, MediaView, VideoPlayer, AudioPlayer, Capture, ...
scripts/sync-core.sh
app.config.ts             reads EXPO_PUBLIC_API_BASE (default: https://dev.davidfruin.com)
```

---

## 4. Phases

### Phase 1: extract a platform-free core inside ssreact (the only ssreact work)
The goal is that ssreact behaves **identically** afterwards. This is pure refactoring.

**1.1 Create `ssreact/src/core/` and move in the pure modules:**
- `types.ts`, `format.ts`, `toast.ts`, `post-cache.ts`, `feed-cache.ts`
- `media-limits.ts` (the `getMediaLimits` + fallback part only; `getMediaDuration` stays in `src/lib`)

Leave re-export shims at the old `src/lib/*` paths, or update the imports. Prefer updating the imports and deleting the shims in the same commit.

**1.2 Split the post-text renderer** into:
- `core/post-text-tokens.ts`: a pure `tokenizePostText(text, mentions): Token[]`, where a token is `{kind:'text',value} | {kind:'url',href,trail} | {kind:'mention',id,email|null}`. Move `TOKEN_RE` and `splitTrailingPunctuation` here unchanged.
- `lib/post-text.tsx` keeps rendering tokens into `<a>`/`<Link>`/`<span>`.

**1.3 Make the API client injectable.** Turn `ApiClient` into `core/api-client.ts`:
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
  extraHeaders?: Record<string, string>;   // native sends X-Client: ssreact-native/<version> (see 3.3)
}
export function createApiClient(config: ApiConfig) { return new ApiClient(config); }
```
- Replace every `localStorage.*` call with `this.store.get/set`, and every `BASE_URL`/`MEDIA_URL` with `this.config.*`. Change `uploadMedia(file: File)` to `uploadMedia(file: UploadFile)`; `FormData.append('file', file as Blob)` works on both platforms.
- `ssreact/src/lib/api.ts` becomes:
  ```ts
  export const api = createApiClient({ apiUrl: '/api.php', mediaUrl: '/media.php', store: localStorageStore });
  ```
  where `localStorageStore` wraps `localStorage` in try/catch.

**1.4 Extract the session-expiry waiter queue** from `auth-context.tsx` into `core/session-expiry.ts`:
```ts
createSessionExpiryQueue({ onShow, onHide })
```
It returns `{ handler, relogin(retryAll), logout() }`. Move the `waitersRef`/`showingRef` logic and its comments over unchanged; that design is deliberate (see the [[ssreact]] note). `auth-context.tsx` then consumes it.

**1.5 Add a media URL helper:** `core/media-url.ts` with `resolveMediaUrl(base, path)`. Web passes `''`; native passes the API base. Every place that renders `post.mediaUrl` or a `thumbnailUrl` goes through it.

**Verify Phase 1:**
- `pnpm lint && pnpm build` are clean.
- `grep -rnE "localStorage|document\.|window\.|navigator\." src/core` returns **nothing**. That grep is the rule that keeps the core portable; add it as a `lint:core` script in `package.json`.
- Run a manual pass against the dev proxy: login, reload (no 401 burst, which is P1), feed, like, comment with a mention, upload an image, session-expired modal (corrupt `ss_jwt` in DevTools to trigger it), logout.

**Commits:** one per step: `refactor(core): …`.

### Phase 2: scaffold ssreact-native
**2.1 Create the app:**
- `npx create-expo-app@latest` with the TypeScript template.
- Add `expo-router`, `expo-secure-store`, `@react-native-async-storage/async-storage`, `expo-image`, and `react-native-safe-area-context` (Expo includes it).
- Set the package manager to pnpm, to match ssreact.

**2.2 Add `scripts/sync-core.sh`:**
```bash
#!/usr/bin/env bash
# Copies ssreact/src/core into src/core at a pinned commit. Never edit src/core by hand.
set -euo pipefail
REF="${1:?usage: sync-core.sh <ssreact-commit>}"
SRC="${SSREACT_DIR:-../ssreact}"
git -C "$SRC" diff --quiet "$REF" -- src/core || { echo "ssreact src/core has uncommitted or different changes vs $REF"; exit 1; }
rm -rf src/core && git -C "$SRC" archive "$REF" src/core | tar -x --strip-components=1 -C src
echo "$REF" > src/core/.synced-from
```
Add a `check:core` script that compares `src/core/.synced-from` with ssreact's latest commit touching `src/core` and warns when they differ.

**2.3 Add `app.config.ts`:**
- `extra.apiBase = process.env.EXPO_PUBLIC_API_BASE ?? 'https://dev.davidfruin.com'`.
- `scheme: 'simplesocial'` for deep links.
- App name "Simple Social".
- Icon and adaptive icon from `ssreact/public/pwa-icons/` (the black-and-white heart).

**2.4 Platform adapters (`src/platform/`):**
- `tokenStore.ts`: an in-memory `Map` that implements `TokenStore`. `hydrate()` calls `SecureStore.getItemAsync` for `ss_jwt`, `ss_refresh` and `ss_user` once, before the first render. `set()` updates the map and calls `SecureStore.setItemAsync`/`deleteItemAsync` without awaiting.
- `api.ts`:
  ```ts
  createApiClient({ apiUrl: `${base}/api.php`, mediaUrl: `${base}/media.php`, store: tokenStore,
    extraHeaders: { 'X-Client': `ssreact-native/${version} (${Platform.OS})` } })
  ```

**2.5 Root layout:**
- Hydrate the token store and then render, using `expo-splash-screen` to hold the splash until then.
- Add providers: Theme, Auth (using `core/session-expiry`), and a Toast host.

**Verify Phase 2:**
- `npx expo start`, open in Expo Go or an Android emulator.
- A temporary debug screen calls `api.getMediaLimits()` after a hard-coded test login against **dev** and shows the JSON.
- `npx tsc --noEmit` is clean. `pnpm check:core` reports "in sync".

### Phase 3: auth and app shell
- **3.1 Screens:** Login, Register and Reset Password. Register's first step has an **Invite code** field (access plan §1.3/1.4: case-insensitive, sent as `inviteCode` with `sendRegisterOTP`, with the hint "Simple Social is invite-only. Ask the person who invited you for a code."). Port `OtpAuthFlow`'s three steps exactly, in the same order as `simple-social-tui`'s `auth.c` (noted in [[ssreact]]).
  - Use `TextInput` with `secureTextEntry`, `autoComplete="email"` / `"password"` / `"one-time-code"`, and `textContentType` for iOS autofill.
- **3.2 Navigation:**
  - `RequireAuth` / `RequireGuest` become redirect logic in the `(auth)` and `(tabs)` layouts, based on `user`.
  - The tabs are Feed, Post, Search, Notifications and Profile. Settings is reached from Profile's header button.
  - This **replaces** ThumbNav and Header. The hand preference only mirrors a floating "new post" button if one is added later; otherwise it is ignored on native. Note that in the hand-off.
- **3.3 [ssapi], additive:** in `auth.php` → `deviceNameFromUserAgent()`, recognise the native client so the Devices list shows a sensible name.
  - RN's default UA looks like `okhttp/…` or `CFNetwork…`.
  - Prefer the `X-Client` header when present: `$_SERVER['HTTP_X_CLIENT']`. Map `ssreact-native/<v> (android)` → "Simple Social app (Android)" and `(ios)` → "… (iOS)".
  - Pass it in from `sessionCreate()`.
  - Verify on the ssapi local bench (see the ssapi plan's Phase 0): `curl -H 'X-Client: ssreact-native/1.0 (android)' … login`, then `getSessions` shows the new name.
- **3.4 Session-expired modal:** a non-dismissible RN `Modal` driven by the core queue. Back press is ignored (`onRequestClose={() => {}}`).
- **3.5 Theme:**
  - Port the six `.theme-*` blocks in `index.css` to `theme/tokens.ts` as `{ background, foreground, card, primary, primaryForeground, muted, mutedForeground, border, destructive, success, warning }` per theme.
  - **Copy the hex values exactly.** `simple-social/css/main.css` is the source of truth (per the [[ssreact]] note).
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
  - Render text from `core/post-text-tokens`. URLs open with `Linking.openURL` (http/https only, which the tokenizer already guarantees). Mentions use `router.push('/profile/'+id)`. Deleted mentions show italic "@deleted user".
  - Like toggle, likes list in a bottom-sheet `Modal`, owner delete through `Alert.alert` confirm.
  - Timestamps go through `core/format`.
- **Post detail:**
  - Use `core/post-cache` the same way ssreact does (skip `getPostById` on a cache hit).
  - Comments are paginated. Comment delete uses `Alert.alert`.
- **Profile:** own profile and someone else's, follow/unfollow, followers/following sheets.
- **Notifications:** Today / Yesterday / Earlier grouping, and copy and link targets **exactly** as ssreact's `NotificationsPage` (which itself matches the original). Mark as seen refreshes the badge (Phase 6).
- **Search:** one `getUsers()` fetch, filtered on the client, with Follow toggles. This matches the recorded design decision.
- **Media display:**
  - `expo-image` for images, with tap → full-screen viewer.
  - Video and audio come in Phase 5. Show a placeholder until then.
  - Every media URL goes through `resolveMediaUrl(base, …)`.

**Verify Phase 4 (emulator, against dev or react's test copy, using a test account per the [[ssreact]] note):**
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
  - Recordings made in browsers are WebM. **iOS can't play WebM.** Prod and dev transcode to MP4/MP3 because ffmpeg exists there. **react.davidfruin.com can't** (ffmpeg is missing from its chroot), so WebM originals from web users are kept as uploaded on that host and won't play on iOS.
  - Test iOS playback against dev, not react.
  - Native recordings come out as MP4/M4A, which is fine everywhere. Check that the server accepts `audio/mp4`/`audio/m4a`: ssapi's `ALLOWED_AUDIO_TYPES` currently lacks them. **[ssapi], additive:** add `audio/mp4`, `audio/x-m4a` and `audio/aac` to the allowed audio types, and add the `ftyp` M4A case to S3's `sniffMedia` (it already maps `ftyp` to the `av` family, which the audio check accepts). Verify on the ssapi bench with a real `.m4a` file.

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
   - Use the same short curl timeouts and deferred sending as ssapi P1 and P2.
   - On a `DeviceNotRegistered` receipt, delete the row.
   - **Keep `exp.host` as the only allowed host for this kind**, consistent with S1.
4. Keep the push URL as it is (`/app.html#/post/…`). The app normalises it exactly like `ssreact/public/sw.js` → `normalizeNotificationUrl` (strip through `#`).

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
- ssapi bench: insert an Expo token row and trigger a like; the deferred send runs. The real delivery check is on a device.
- Real device: like a test post from a second account → the notification arrives, the tap opens the post, the badge count is right, and mark-as-seen clears it. Logout → no further pushes.

### Phase 7: settings and remaining parity
- **Settings:**
  - Theme picker, hand preference (stored, mostly unused on native; see 3.2), push toggle.
  - Devices list with revoke and revoke-all, using `Alert.alert` confirms.
  - Log out.
  - Delete Account (password + double confirm).
  - Links that open the website: About, Conduct, Roadmap, API.
- **Toasts:** one host component subscribed to `core/toast`, with solid colours (the lesson from the Red-theme toast bug in the [[ssreact]] note).
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
- **Icons:** a 1024×1024 iOS icon (no transparency) and an Android adaptive icon, from the black-and-white heart in `ssreact/public/pwa-icons/`.

**8.3 Build profiles (`eas.json`):**
- `development`: dev client, dev backend.
- `preview`: dev or react backend, for Dave's own testing.
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
- Record the results, the iOS build's expiry date and where the keystore backup lives (*where*, not the key) in the [[ssreact-native]] note.

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
- If rejected: fix, bump the build number, resubmit. Record the reason in the [[ssreact-native]] note.
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

**10.4 After launch:** add store badges on ssreact's `DownloadPage` and simple-social's `download.html`. Moderation duty scales with users (§2.6).

---

## 5. Screen and component map

| ssreact | ssreact-native | Notes |
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
| `LandingPage`, `AboutPage`, `ApiDocsPage`, `ConductPage`, `RoadmapPage`, `DownloadPage` | Not ported; linked from Settings | N6 |
| shadcn `Card`/`Button`/`Input`/`Switch`/`Select`/`AlertDialog`/`Popover`/`Dialog` | `View`+tokens / `Pressable` / `TextInput` / RN `Switch` / action sheet / `Alert.alert` / bottom-sheet `Modal` / `Modal` | Thin local components only |

---


---

## 6. Website follow-ups
- Phase 8: an APK download page or link at a stable URL (`/downloads/simple-social.apk` + `android-version.json` + checksum). The ssreact `DownloadPage` can say "Phone app: invite-only, ask Dave for a link."
- Phase 9: `/terms` and `/privacy` live at public URLs (access plan Step 1B).
- Phase 10: store badges after the public launch.

---

## 7. Open decisions for Dave
1. **N2:** confirm "share logic only, copied core + sync script". Then record it in the [[ssreact-native]] note's Decisions.
2. **Bundle ID / package name** (`com.davidfruin.simplesocial` proposed). It is permanent.
3. **How ssapi reaches prod** (ssapi plan D6). **The family release can't ship without it.**
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
| 1 | Phase 1 (ssreact core), Phase 2 (scaffold); access plan Step 1 (invites) backend + web | Apple account sign-up; confirm the bundle ID; check Android developer verification; decide how ssapi gets to prod |
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
