---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent
repos: ssreact workspace (web/ = bundled UI, desktop/ = this app), ssapi (one small additive change)
---

# Port plan: ssreact web → desktop app (`ssreact/desktop/`)

A plan for building the Electron desktop app for [[simple-social]] in [[ssreact]]'s `desktop/` folder, **by packaging the existing web build from `web/`** rather than writing a second UI. It is written to be carried out phase by phase by an agent.

> **Deprioritized 2026-10-02 (Dave):** the phone apps (`Inbox/ssreact-native-port-plan.md`, App Store + Google Play) come first. Start this plan only after the phone app is submitted, or when Dave says so. It gets the store-readiness report/block/terms UI for free, because it bundles ssreact.
>
> **Updated 2026-10-03 (repo consolidation):** there's no separate `sselectron` repo any more (archived). The app lives in `ssreact/desktop/`, next to `web/`, in the same pnpm workspace (`Inbox/repo-consolidation-plan.md`). So there's no git submodule: the desktop build simply builds `web/` first.

---

## 0. Read this first: rules for the implementing agent

1. **Sequencing:**
   - Start after the ssreact tasks in `Inbox/ssapi-improvement-plan.md` have landed (P1, P5, P6, P7, P8, S12, S15, C6).
   - This app ships ssreact's build. Any ssreact bug ships with it, and the small ssreact changes in Phase 3 touch the same files.
   - Check `Areas/active-work.md` first.
2. **Backends:**
   - Development points at `dev.davidfruin.com` (react.davidfruin.com is retired) or at the ssapi local bench.
   - **Never point at `app.davidfruin.com` (prod)** except in a release build Dave explicitly approves.
3. **Pause between phases.** Make one commit per step, and stop and report to Dave after each phase. Items marked **DECISION** or **GATED** wait for Dave.
4. **Security comes first in Electron.** A desktop app that renders user content has a bigger attack surface than a browser tab. Every rule in Phase 2's checklist is required, not optional.
5. **Where code goes:**
   - The app lives in `ssreact/desktop/`.
   - Changes in `ssreact/web/` are limited to Phase 3 and must be no-ops in a normal browser.
   - ssapi changes are limited to the one **[ssapi]** item. Never deploy anything.
6. **Verify by running the packaged app**, not just `electron .`. When something can't be checked (macOS signing, Windows installer), say so in the hand-off.
7. **Public repo:** this vault is public. Never put signing certificates, passwords or tokens in it.

---

## 1. The key design choice

| Option | How it works | Pros | Cons |
|---|---|---|---|
| A. Thin shell | `BrowserWindow.loadURL('https://<site>')` | Nearly no code; UI updates arrive with every web deploy | Only works where ssreact is the live frontend. As of 2026-10-03 that's no live host (react.davidfruin.com is retired; dev is proposed in the consolidation plan, Step 2.6), and app.davidfruin.com still serves the vanilla app. Needs a network connection to show anything. |
| **B. Bundled build + proxy (recommended)** | ssreact's `dist/` is packaged inside the app and served from a custom `app://` origin. The main process **proxies** `/api.php`, `/media.php` and `/media/*` to a configured backend. | Works **today** against any backend, including prod's API, without waiting for the frontend migration. ssreact keeps its relative same-origin URLs with **zero** `api.ts` changes. **No CORS**, which matches the recorded "no CORS" decision. The UI starts instantly. | UI updates need an app release (auto-update covers that, Phase 6). The proxy has to be written carefully. |

**This plan uses B.** Option A becomes reasonable after ssreact replaces the vanilla frontend on app.davidfruin.com; revisit then (**DECISION** for later).

**How the web build gets into the app:** both live in the same workspace. `desktop`'s build runs `pnpm --filter @ss/web build` and copies `web/dist` into the app. The release is reproducible because the desktop app and the web app it ships come from the **same commit**.

---

## 2. Target structure (inside the ssreact workspace)

```
ssreact/
  web/                     the web app (bundled into the desktop app)
  packages/core/           @ss/core (shared logic; the desktop main process doesn't need it)
  desktop/
    src/main/
      main.ts              app lifecycle, single-instance lock, window, tray
      protocol.ts          app:// handler: static app-dist/ + SPA fallback + backend proxy
      security.ts          permission handler, navigation/window-open guards, CSP header
      notifications.ts     unseen-count -> native Notification + badge/tray
      config.ts            backend base URL per build profile
    src/preload/preload.ts contextBridge: window.ssDesktop (tiny, typed)
    build/                 icons (from web/public/pwa-icons), entitlements (macOS)
    electron-builder.yml
    package.json           "@ss/desktop"; scripts: dev, build:ui, build, dist, dist:linux
```

---

## 3. Phases

### Phase 1: scaffold
- **1.1** In `ssreact/desktop/`: `pnpm init` (name `@ss/desktop`, private), then add `electron`, `electron-builder`, `typescript`, and `tsx` or `esbuild` (to compile main and preload). Use the current Electron stable release; **keep Electron updated**, because each major version ships Chromium security fixes.
- **1.2** No submodule: `web/` is next door in the same workspace. Add a root script `"desktop": "pnpm --filter @ss/desktop dev"`.
- **1.3** `config.ts`:
  ```ts
  export const API_BASE = process.env.SS_API_BASE ?? 'https://dev.davidfruin.com';
  ```
  Release builds bake in a value at build time. **The prod value is only used in a build Dave approves.**
- **1.4** `scripts.build:ui` = `pnpm --filter @ss/web build && rm -rf app-dist && cp -r ../web/dist app-dist`.

**Verify:** `pnpm build:ui` produces `app-dist/index.html` and `app-dist/assets/*`.

### Phase 2: secure window and the `app://` protocol
**2.1 Register the scheme before `app.whenReady()`:**
```ts
protocol.registerSchemesAsPrivileged([{ scheme: 'app', privileges: {
  standard: true, secure: true, supportFetchAPI: true, stream: true, corsEnabled: false } }]);
```

**2.2 Handler (`protocol.handle('app', …)`) for origin `app://ssreact`:**
- **Proxy:** if the path is `/api.php`, `/media.php` or starts with `/media/`, forward to `${API_BASE}${path}${search}` with `net.fetch`. Copy the method and body (`duplex: 'half'` for streamed uploads), and forward only these headers: `authorization`, `content-type`, `content-length`, `accept`, `range`. Range is needed for video seeking. Return the upstream status, body and headers (strip `set-cookie`; there are none today).
- **Static files:** resolve the path inside `app-dist`. **Reject any path that resolves outside `app-dist`** using `path.resolve` plus a `startsWith(distRoot + path.sep)` check, which blocks `..` traversal. Serve the file with the right content type.
- **SPA fallback:** any other path → `app-dist/index.html`. This is the same rule as the web root `.htaccess` (`ssapi/deploy/root.htaccess`).
- **Headers:** add a CSP to HTML responses. Start from ssreact's policy (ssapi plan S12) and enforce it here, because this origin serves only known files:
  `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; media-src 'self' blob:; font-src 'self' data:; connect-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'`.

**2.3 `BrowserWindow` `webPreferences`:**
```ts
{ contextIsolation: true, nodeIntegration: false, sandbox: true,
  webSecurity: true, preload: path.join(__dirname, 'preload.js'), spellcheck: true }
```
Load `app://ssreact/feed`. Size about 1100×800, minimum 380 wide. The layout already handles 375px.

**2.4 `security.ts`. Each item is required:**
- `webContents.setWindowOpenHandler`: deny all new windows. If the URL is `http:` or `https:`, call `shell.openExternal(url)` (post links use `target="_blank"`). Never call `openExternal` with any other scheme.
- `will-navigate` / `will-redirect`: `preventDefault()` unless the target origin is `app://ssreact`. Open http/https targets externally.
- `session.setPermissionRequestHandler`: allow only `media` (camera/mic for the capture modal), `notifications`, `clipboard-sanitized-write` and `fullscreen`, and **only** when the requesting origin is `app://ssreact`. Deny everything else. Add a matching `setPermissionCheckHandler`.
- `app.on('web-contents-created')`: block `<webview>` attachment (`will-attach-webview` → `preventDefault`).
- Single-instance lock: a second launch focuses the existing window.
- **Electron Fuses**, set at package time with `@electron/fuses`: `RunAsNode` off, `EnableNodeOptionsEnvironmentVariable` off, `EnableNodeCliInspectArguments` off, `EnableEmbeddedAsarIntegrityValidation` on, `OnlyLoadAppFromAsar` on.

**2.5 [ssapi], additive:** in `auth.php` → `deviceNameFromUserAgent()`, recognise the desktop app so Devices shows "Simple Social desktop (Linux)" and so on.
- In Electron, append a token to the user agent: `app.userAgentFallback += ' ssreact-desktop/' + app.getVersion()`.
- In PHP, check `stripos($ua, 'ssreact-desktop') !== false` **before** the browser checks, and keep the OS detection.
- Verify on the ssapi local bench: `curl -A 'Mozilla/5.0 (X11; Linux x86_64) … Chrome/… Electron/… ssreact-desktop/0.1.0'` login → `getSessions` shows the new name.

**Verify Phase 2 (against dev, with a test account per the [[ssreact]] note):**
- Login works.
- Reload the window: still logged in, with no 401 burst (ssapi plan P1).
- Feed, post, upload image, capture photo (allow camera at the OS prompt), video playback and seeking (Range is proxied), logout.
- DevTools console: **no CSP violations**.
- Click a link inside a post → it opens in the system browser, not in the app.
- In DevTools: `window.require` and `process` are undefined.
- Navigating to `app://ssreact/../../etc/passwd` (via `location.href` in DevTools) returns index.html, not the file.

### Phase 3: small ssreact changes (desktop-aware, no-ops in browsers)
Each goes through ssreact's normal `pnpm lint && pnpm build` and is checked in a regular browser too.

- **3.1 `preload.ts` exposes one narrow object:**
  ```ts
  contextBridge.exposeInMainWorld('ssDesktop', {
    version: string,
    setBadge(count: number): void,                      // ipcRenderer.send('badge', count)
    notify(title: string, body: string, url: string): void,
  });
  ```
  - Main validates each IPC call: `event.senderFrame.url` must start with `app://ssreact/`, counts must be integers between 0 and 9999, strings must be under 500 characters.
  - Add `web/src/lib/desktop.ts` with `export const desktop = (window as any).ssDesktop as DesktopBridge | undefined;` and a type for it.
- **3.2 `register-sw.ts`:** return early `if (desktop)`. Service workers on custom schemes aren't reliable, and the app needs no install or stale-cache protection, since the bundle is fixed per release.
- **3.3 Push and Settings:**
  - Web Push doesn't work in Electron (there's no browser push service behind it), so `isPushSupported()` returns false when `desktop` is set.
  - Settings shows a "Desktop notifications" toggle in its place, stored in `localStorage` (`ss_desktop_notify`), with Phase 4 explaining the behaviour.
- **3.4 Unseen-notification poller (`use-unseen-notifications.ts`, after ssapi plan P6):**
  - When the count rises and `desktop` is set, call `desktop.setBadge(count)`.
  - When notifications are enabled, fetch the newest unseen items with `getNotifications()` and call `desktop.notify(...)` for each new id. Remember the last seen id in memory to avoid repeats.
  - Use the same notification text as the server's `notificationText()`.
- **3.5 `DownloadPage`:** hide the PWA "Install app" button when `desktop` is set. The Desktop section text is updated in Phase 6.

**Verify Phase 3:**
- In a normal browser on `pnpm dev`: no behaviour change; push UI and SW exactly as before.
- In the app: no service worker is registered (`navigator.serviceWorker.getRegistrations()` returns `[]`), and the Settings toggle appears.

### Phase 4: notifications, tray, badge
- **4.1 `notifications.ts`:**
  - On `notify`, show `new Notification({ title, body, icon })`.
  - On click, focus or show the window and navigate it to `app://ssreact` + the URL normalised like `sw.js` → `normalizeNotificationUrl` (strip through `#`).
- **4.2 Badge:**
  - Call `app.setBadgeCount(count)` on macOS and Linux (where supported).
  - On Windows, use `win.setOverlayIcon` with a small generated count image, or a dot plus a tooltip.
  - Mark-as-seen in the app clears it, through the immediate refresh from ssapi plan P6.
- **4.3 Tray:**
  - A tray icon (black-and-white heart) with a menu: Open, Notifications on/off, Quit.
  - Closing the window **hides to tray** by default so notifications keep arriving. Quit from the tray or the app menu really exits.
  - **DECISION:** hide-to-tray on or off by default, and start-on-login (off by default).
- **4.4 Limitation to record in the hand-off:** notifications only arrive **while the app is running** (even if only in the tray). True closed-app push would need a separate push service. That isn't worth it for a bridge client, and the server-side Web Push can't reach Electron.

**Verify:**
- From a second test account, like a post → a desktop notification appears within about 60s (the poll interval, or sooner when the window comes back into focus).
- Clicking it opens the post.
- The badge or tray count is right, and clears after Mark as Seen.
- Close the window → the app stays in the tray and still notifies. Quit → the process is gone (`pgrep -f 'Simple Social'` returns nothing).

### Phase 5: OS integration polish
- **Camera and microphone:**
  - macOS needs `NSCameraUsageDescription` / `NSMicrophoneUsageDescription` in `extendInfo`, plus `systemPreferences.askForMediaAccess('camera'|'microphone')` before `getUserMedia`. Hardened-runtime entitlements are `com.apple.security.device.camera` and `com.apple.security.device.audio-input`.
  - Linux and Windows need nothing extra beyond the permission handler.
- **App menu:**
  - Edit (copy/paste), View (reload is dev-only, zoom, toggle fullscreen), Window, and Help → "About Simple Social" (version plus pinned ssreact commit).
  - No DevTools in production builds unless `SS_DEBUG=1`.
- **Window state:** remember size and position in a small JSON file under `app.getPath('userData')`.
- **Deep links (optional):** register `simplesocial://post/<id>` with `app.setAsDefaultProtocolClient`. Validate the id against `^\d+\.\d+$` before navigating.
- **Token storage:** tokens stay in the `app://ssreact` origin's localStorage, which Electron persists under `userData`. Optional hardening: move `ss_refresh` into `safeStorage`-encrypted storage through the bridge. **DECISION**, low priority. Revisit together with ssapi plan D4.

### Phase 6: packaging, releases, updates (GATED)
- **6.1 Targets, Linux first** (Dave's machines run LMDE and Omarchy/Arch):
  - Linux: `AppImage` + `deb` + `pacman`.
  - Windows: `nsis`.
  - macOS: `dmg`.

  Use `electron-builder.yml`, with `appId: com.davidfruin.simplesocial`, `productName: Simple Social`, `files: [dist-electron/**, app-dist/**]`, and `asar: true`.
- **6.2 Signing (DECISION, costs money):**
  - macOS needs an Apple Developer ID ($99/yr) and notarization, or Gatekeeper blocks the app.
  - Windows needs a code-signing certificate, or SmartScreen warns on every install.
  - Linux needs nothing.
  - Recommendation: ship Linux first, unsigned Windows "with a warning" for testers, and skip macOS until there's an Apple account (the same one the phone app's iOS build needs).
- **6.3 Auto-update (DECISION):**
  - `electron-updater` reads GitHub Releases, but **ssreact is a private repo**, so anyone outside it can't download updates from its releases.
  - Option (a): make ssreact public. It contains no secrets, but it would publish the web, phone and desktop code together. That's Dave's call.
  - Option (b): host releases on Dave's own server with the `generic` provider (`https://<site>/desktop/latest.yml`).
  - Option (c): no auto-update; the Download page links the latest file.
  - Recommendation: (b) once there are real users, (c) for now.
  - Any update feed must be served over HTTPS. On macOS and Windows, `electron-updater` only applies updates whose signature matches, which is another reason signing matters.
- **6.4 Build pipeline:**
  - Build by hand with `pnpm dist:linux` on Dave's machine.
  - **GitHub Actions is on hold project-wide** (see the [[ssreact]] and [[simple-social]] notes). Don't add a release workflow without Dave's explicit go.
- **6.5 Download pages:**
  - Update ssreact `DownloadPage`'s Desktop section, and simple-social's `download.html` "coming soon" placeholder, with the per-OS files and install notes.
  - AppImage: `chmod +x`. Deb: `sudo apt install ./…deb`. Arch: `sudo pacman -U …`. The existing LMDE/Omarchy tab toggle fits this.

**Verify Phase 6 (Linux, real install, not `electron .`):**
- Install the `.deb` on LMDE and the pacman package or AppImage on Omarchy.
- The app launches from the menu with the right icon.
- Log in against **dev**, create and delete a test post, capture a photo, receive a notification.
- Uninstall cleanly.
- Note honestly which OS targets were *not* tested.

---

## 4. ssreact parity checklist for the desktop app
Everything ssreact does should work as it does on the web, with these intentional differences:

| Feature | Desktop behaviour |
|---|---|
| Push notifications | Replaced by notifications while the app is running (poller) + tray |
| PWA install button, service worker | Hidden / not registered |
| Home-screen badge | `app.setBadgeCount` / tray / overlay icon |
| Thumb-nav (touch only) | Unchanged: it only shows on touch devices, so desktop gets the header nav |
| External links | System browser |
| Capture modal | Works (getUserMedia), with OS permission prompts |
| Fullscreen video | Works, because the permission handler allows `fullscreen` |
| Clipboard copy (Download page) | Allowed by the permission handler |

---

## 5. Open decisions for Dave
1. Confirm **option B** (bundled + proxy) over the thin shell. Revisit after ssreact goes live on app.davidfruin.com.
2. Hide-to-tray default; start-on-login default.
3. **Signing:** an Apple Developer ID (shared with the phone app's iOS build) and a Windows certificate, or Linux-only for now.
4. **Auto-update:** (a) public repo, (b) self-hosted feed, or (c) manual downloads.
5. Which backend release builds point at. Dev/test by default; prod only with an explicit go.
6. `safeStorage` for the refresh token (low priority; tie it to ssapi plan D4).
7. Record the outcome in the [[ssreact]] note's Decisions (desktop section).

## 6. Suggested order and stopping points
Phase 1 → stop → Phase 2 (+ the [ssapi] UA item) → stop → Phase 3 → stop → Phase 4 → stop → Phase 5 → stop. Then Phase 6 only on Dave's go, starting with the Linux targets.
