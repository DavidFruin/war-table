---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent; server steps for Dave / an agent with el1 access
repos: ssapi @ 73e05fd, ssreact @ 865ff6f
supersedes: ssapi-improvement-plan D5 (now decided), and the "shared docroot .htaccess lives in ssreact" arrangement from S2
---

# Deploy layout plan: backend, frontend, private data and media in separate folders

**Decided by Dave, 2026-10-02.** Each domain gets this layout:

```
/home/davidfruin/domains/<domain>/
  private/              .env, userdata.db, logs/, tmp/        (exists today, unchanged)
  ssapi/                the whole backend: api.php, media.php, config.php, src/, vendor/ ...
                        NOT inside public_html, so none of it can ever be requested from the web
  public_html/
    .htaccess           ONE root file, owned by neither app deploy; canonical copy: ssapi/deploy/root.htaccess
    api.php             2-line stub -> ../ssapi/api.php
    media.php           2-line stub -> ../ssapi/media.php
    index.maintenance.php   maintenance page (unchanged behaviour)
    app/                the frontend build (ssreact dist/, or the vanilla frontend on app/dev until it's replaced)
    media/              user uploads (stays web-served)
    downloads/          static files, e.g. the Android APK (phone plan, Phase 8)
```

**Why:**
1. **Deploys stop stepping on each other.** Today's frontend deploy needs a long exclude list, and two real bugs came from getting it wrong: uploads wiped, and permissions reset. Each deploy now owns exactly one folder, so `rsync --delete` is safe.
2. **The `.htaccess` problem goes away.** Today ssreact's `.htaccess` replaces the backend's on a shared docroot (ssapi plan S2). One root file, deployed on purpose, ends that.
3. **Backend code can't be requested from the web at all**, not even if `.htaccess` breaks. That's the old D5.
4. **No change to any URL.** `/api.php`, `/media.php`, `/media/…` and every frontend path stay exactly the same. Clients (vanilla web, ssreact, CLI/TUI, and the phone and desktop apps) need no changes.

**Performance:** no meaningful change. Apache serves the frontend, media and downloads directly as static files. The internal rewrites cost microseconds.

**Unchanged trade-off, revisit in Step 2:** media in `public_html/media/` is public to anyone with the URL. That's fine for family use. For the paid public launch, consider members-only media (`Inbox/access-and-public-launch-plan.md` §2.6).

---

## 0. Rules for the implementing agent
- The usual rules apply: one task per commit, stop after each task group, never deploy, never touch a server. Server steps (§4) are for Dave or an agent he gives `el1` access to, using its runbook.
- **Verify with a real local Apache**, not just `php -S`, which ignores `.htaccess`. The ssreact `.htaccess` work was verified this way before. See §3.
- **Order matters on react.davidfruin.com.** Its current `public_html/.htaccess` comes from ssreact's `public/.htaccess`. Removing that file from ssreact (L4) must be deployed **at the same time as** that host's migration (§4), never before. Otherwise the next normal ssreact deploy leaves the shared docroot without its rules.

---

## 1. Code tasks

### L1 (ssapi). One configurable media folder instead of four hard-coded paths
The backend finds uploads relative to its own code folder in four places. These all break the moment the code moves out of `public_html`:
- `src/Media/handlers.php:69`: `getMediaDir()` → `__DIR__ . '/../../media/' . $userId`
- `src/Posts/handlers.php:399`: `deletePost` → `__DIR__ . '/../../' . ltrim($mediaRow['path'], '/')`
- `api.php:379`: `deleteAccount` file unlink → `__DIR__ . $r['path']`
- `api.php:401`: `deleteAccount` folder cleanup → `__DIR__ . '/media/' . $uid`

**Fix.** In `config.php`, after `$privateDir`:
```php
// Uploaded media lives in the web root (public_html/media), next to -- not inside -- this code.
// Default fits the target layout (domain/ssapi + domain/public_html); MEDIA_DIR in private/.env
// overrides it, e.g. for the old layout where the code sits in public_html itself.
$CONFIG['media_dir'] = rtrim(getenv('MEDIA_DIR') ?: dirname(__DIR__) . '/public_html/media', '/');
```
Add one helper in `api.php` (or `schema.php`, which both entry points load):
```php
// Maps a stored media URL path ("/media/<uid>/<type>/<file>") to its file on disk.
// Returns null for anything that isn't a plain path under /media/.
function mediaFilePath($urlPath) {
    global $CONFIG;
    if (!is_string($urlPath) || !str_starts_with($urlPath, '/media/') || str_contains($urlPath, '..')) return null;
    return $CONFIG['media_dir'] . substr($urlPath, strlen('/media'));
}
```
Then:
- `getMediaDir($uid)` returns `$CONFIG['media_dir'] . '/' . $userId`.
- `deletePost` and `deleteAccount` use `mediaFilePath(...)`, skipping `null`. Their thumbnail lookups work on the returned path exactly as before.
- `deleteAccount`'s folder cleanup uses `getMediaDir($uid)`.

Also fix two leftovers:
- `logging.php:7` still falls back to `__DIR__ . '/logs'`. Remove that fallback; `$CONFIG['log_dir']` is always set now (S11).
- `schema.php:30` has `?? __DIR__ . '/userdata.db'`. Remove it for the same reason.

Add to `.env.example`:
```
# Only needed if uploads aren't at <domain>/public_html/media (e.g. the old layout)
MEDIA_DIR=
```

**Verify:** use the bench from the ssapi plan's Phase 0, but laid out like the target: `$BENCH/private`, `$BENCH/ssapi` (the code), `$BENCH/public_html/media`. Run `php -S` with `-t $BENCH/public_html`, plus the stubs from L2.
- Upload an image → the file appears under `$BENCH/public_html/media/<uid>/image/`.
- Delete the post → the file is gone.
- Delete the account → `media/<uid>` is gone.
- `grep -rn "__DIR__" --include=*.php . | grep -v vendor` shows only `require` lines and `config.php`'s `dirname(__DIR__)`.

**Commit:** `deploy: configurable media dir; no code-relative data paths`

### L2 (ssapi). Public entry stubs and the canonical root `.htaccess`
Create a `deploy/` folder in ssapi, holding everything that goes **into** `public_html` (apart from the frontend and media):

```
deploy/
  root.htaccess              -> public_html/.htaccess
  public/api.php             -> public_html/api.php
  public/media.php           -> public_html/media.php
  public/index.maintenance.php   (git mv from the repo root)
```

`deploy/public/api.php`:
```php
<?php
// Public entry point. The real backend lives outside the web root, next to public_html.
require dirname(__DIR__) . '/ssapi/api.php';
```
`deploy/public/media.php`: the same, but requiring `media.php`. `dirname(__DIR__)` is the domain folder, so this works on every host without any configuration.

**`deploy/root.htaccess`** merges ssreact's current `public/.htaccess` (headers, CSP Report-Only, caching, deny rules) with the new routing:
```apache
# public_html/.htaccess -- canonical copy: ssapi/deploy/root.htaccess. Deployed on purpose,
# never by an app deploy. Layout: see war-table Inbox/deploy-layout-plan.md.
#   /api.php, /media.php   stubs into ../ssapi (backend code is outside the web root)
#   /app/                  frontend build, served at "/" via the rewrites below
#   /media/                uploads (data only, never executable)
#   /downloads/            static downloads (APK etc.)
Options -Indexes

# Authorization header passthrough (mod_fcgid hosts; PHP-FPM hosts also need CGIPassAuth On in the vhost).
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1

# Only the entry stubs may execute. index.php = maintenance mode (see index.maintenance.php).
<FilesMatch "\.(php\d?|phtml|phar|inc)$">
    Require all denied
</FilesMatch>
<FilesMatch "^(api|media|index)\.php$">
    Require all granted
</FilesMatch>

# Never serve data, logs, env or tooling files, wherever they end up by mistake.
<FilesMatch "(\.(db|db-wal|db-shm|sqlite|log|env|lock|md)|^\.(env.*|git.*|htaccess)|^composer\.json)$">
    Require all denied
</FilesMatch>

<IfModule mod_headers.c>
  Header always set X-Content-Type-Options "nosniff"
  Header always set X-Frame-Options "DENY"
  Header always set Referrer-Policy "strict-origin-when-cross-origin"
  Header always set Permissions-Policy "camera=(self), microphone=(self), geolocation=()"
  Header always set Content-Security-Policy-Report-Only "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; media-src 'self' blob:; font-src 'self' data:; connect-src 'self'; worker-src 'self'; manifest-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'"

  # Hashed build assets, icons and per-upload media never change under the same URL.
  # REQUEST_URI is checked with and without the internal /app/ prefix.
  <If "%{REQUEST_URI} =~ m#^/(app/)?(assets|pwa-icons)/|^/media/#">
    Header set Cache-Control "public, max-age=31536000, immutable"
  </If>
  # Must always revalidate, or a reload can load stale JS after a deploy.
  <FilesMatch "^(index\.html|sw\.js|manifest\.json|android-version\.json)$">
    Header set Cache-Control "no-cache"
  </FilesMatch>
</IfModule>

<IfModule mod_deflate.c>
  AddOutputFilterByType DEFLATE text/html text/css application/javascript application/json image/svg+xml
</IfModule>

<IfModule mod_mime.c>
  AddType application/vnd.android.package-archive .apk
</IfModule>

<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

  # Maintenance mode: active only while a file named index.php exists.
  RewriteCond %{DOCUMENT_ROOT}/index.php -f
  RewriteCond %{REQUEST_URI} !^/index\.php$
  RewriteRule ^ index.php [L]

  # Defence in depth: none of these should exist in the web root at all.
  RewriteRule ^(vendor|src|logs|private|tmp|ssapi)(/|$) - [F,L]
  # Uploaded media is data, never code or active content.
  RewriteRule ^media/.*\.(php\d?|phtml|phar|html?|svg|js|mjs)$ - [F,L]

  # Served as they are.
  RewriteRule ^(api|media|index)\.php$ - [L]
  RewriteRule ^(media|downloads)/ - [L]

  # Already rewritten into the frontend folder (second pass).
  RewriteRule ^app/ - [L]

  # A real frontend file: /assets/x.js -> app/assets/x.js, /sw.js -> app/sw.js (URL stays /sw.js,
  # so the service worker still controls the whole site).
  RewriteCond %{DOCUMENT_ROOT}/app/$1 -f
  RewriteRule ^(.+)$ app/$1 [L]

  # Everything else is a frontend page (React Router paths like /feed).
  RewriteRule ^ app/index.html [L]
</IfModule>
```
Replace the repo-root `ssapi/.htaccess` with a single seal. The backend folder isn't web-served any more, so this only matters if it is ever deployed into a web root by mistake:
```apache
# This folder is backend code and must never be web-served. See deploy/root.htaccess for the real web-root rules.
Require all denied
```
Update `README.md`'s "Deploying" section to show the layout and point to this plan.

**Verify:** §3 (local Apache).

**Commit:** `deploy: entry stubs + canonical root .htaccess for the split layout`

### L3 (ssreact). Nothing changes in the code
- Vite `base` stays `/`.
- The service worker stays at `/sw.js` (the rewrite keeps that URL).
- `api.ts` URLs stay as they are.

### L4 (ssreact, deploy on migration day only). Remove `public/.htaccess`
- The root `.htaccess` now handles the SPA fallback and every header.
- An `app/.htaccess` would run its own `RewriteBase /` rules **again** inside `app/` and could loop. So ssreact must ship **no** `.htaccess`.
- Make this commit on a branch named `deploy-layout`. **Dave merges it on the day react.davidfruin.com is migrated (§4)**, not before.

**Commit:** `deploy: drop public/.htaccess (root .htaccess owns routing in the split layout)`

---

## 2. New deploy commands (after migration)
Run these from Dave's machine with `el1` access. Each one touches only its own folder.
```bash
D=/home/davidfruin/domains/react.davidfruin.com     # or app./dev. once migrated

# Backend (code only; never touches private/ or public_html/)
rsync -rltz --no-owner --no-group --delete --exclude='.git' --exclude='deploy/' \
  ssapi/ el1:$D/ssapi/
ssh el1 "cd $D/ssapi && composer install --no-dev --optimize-autoloader"   # or rsync a locally built vendor/

# Root files (stubs + .htaccess); deliberately NO --delete
rsync -rltz --no-owner --no-group ssapi/deploy/public/ el1:$D/public_html/
rsync -tz ssapi/deploy/root.htaccess el1:$D/public_html/.htaccess

# Frontend (ssreact)
pnpm run build && rsync -rltz --no-owner --no-group --delete dist/ el1:$D/public_html/app/
```
- `--no-owner --no-group` still matters, for the same group/setgid reason as before.
- The long exclude list and the `media/` exclude are gone; neither deploy can reach those folders now.

---

## 3. Local verification with Apache (agent)
Build the target layout in a scratch folder and serve it with Apache + PHP, the same way ssreact's `.htaccess` was checked before:
```
/tmp/sslayout/private/        .env (JWT_SECRET, APP_DEBUG=false), userdata.db (bench seed)
/tmp/sslayout/ssapi/          ssapi checkout (with vendor/)
/tmp/sslayout/public_html/    deploy/public/* + deploy/root.htaccess as .htaccess, app/ = ssreact dist/, media/
```
Use a vhost with `DocumentRoot /tmp/sslayout/public_html`, `AllowOverride All`, and `mod_rewrite`, `mod_headers` and `mod_deflate` enabled. Use PHP-FPM (`CGIPassAuth On`) or mod_php.

Then check, recording the actual status codes in the commit message:

| Request | Expect |
|---|---|
| `GET /` and `GET /feed` | 200, the React app's `index.html` |
| `GET /assets/<hashed>.js` | 200 JS, `Cache-Control: … immutable` |
| `GET /sw.js`, `GET /manifest.json` | 200, `Cache-Control: no-cache` |
| `POST /api.php action=login` (bench user) | 200 JSON with a jwt; then `getMyInfo` with `Authorization` → 200 |
| `POST /media.php` upload | 200; file under `public_html/media/<uid>/…`; `GET` that URL → 200 + immutable |
| `GET /media/1/x.php`, `/media/1/x.html` | 403 |
| `GET /config.php`, `/ssapi/config.php`, `/src/Auth/handlers.php`, `/vendor/autoload.php`, `/composer.json`, `/.env` | 403 or 404 (never PHP output, never file contents) |
| `GET /app/` | 200 (duplicate path to the app, harmless) |
| `GET /downloads/test.apk` (put a dummy file there) | 200, `Content-Type: application/vnd.android.package-archive` |
| Create `public_html/index.php` (copy of `index.maintenance.php`) | Every URL → 503 page; delete it → normal again |
| `GET /media/` (folder) | 403 (no listing) |
| `GET /logs/api.log` | 403/404 |

Then do one end-to-end run of the React app on this Apache: login, feed, upload, post, logout.

---

## 4. Server migration runbook (Dave, or an agent with el1 access): react.davidfruin.com first
Do **react** (the test host) first. **dev** and **app** follow later, as part of the ssapi plan's D6 decision ("ssapi replaces simple-social's backend copy"), because those hosts still run simple-social's backend and the vanilla frontend.

1. **Pre-checks:**
   - `ls $D/private` → `.env` and `userdata.db` are there.
   - Confirm PHP can read a sibling folder of `public_html`. react's PHP-FPM runs **in a chroot**, so put a throwaway `public_html/t.php` containing `<?php require dirname(__DIR__) . '/ssapi/api.php';` after step 3, `curl` it once, then delete it. `private/` already works there, so a sibling is very likely fine.
2. **Backup:** `cp -a $D/public_html $D/public_html.bak-$(date +%F)`. Keep it until step 7 passes.
3. **Backend:** create `$D/ssapi/` with the backend deploy command from §2. Set ownership to match `private/` (`davidfruin:davidfruin`), readable by PHP-FPM.
4. **Frontend into `app/`:** `mkdir $D/public_html/app` and run the frontend deploy command from §2 (from ssreact **with L4 merged**).
5. **Switch over** (do it quickly, one step after another):
   - Run the root-files deploy, which overwrites `public_html/api.php` and `media.php` with the stubs and `.htaccess` with the root file.
   - Then remove the old files from `public_html`:
     - old frontend: `index.html`, `assets/`, `sw.js`, `manifest.json`, `pwa-icons/`, `site-icon.png`;
     - old backend: `config.php`, `auth.php`, `logging.php`, `schema.php`, `webpush.php`, `clean-notifications.php`, `composer.json`, `composer.lock`, `vendor/`, `src/`;
     - any `logs/`.
   - **Don't touch `media/`.**
6. **Ownership:** check that `public_html` still has group `davidfruin` + setgid, and that `media/` is writable by PHP-FPM (the earlier fix).
7. **Verify on the live host:**
   - Rerun §3's table with `curl https://react.davidfruin.com/...`.
   - Log in with the test account in a browser, view the feed, upload, post, delete, log out.
   - The Devices list still shows sessions.
   - Push still arrives, if a test device is subscribed.
8. **Rollback, if anything fails:** `mv public_html public_html.failed && mv public_html.bak-<date> public_html`. The data in `private/` was never touched.
9. **Afterwards:**
   - Delete the backup after a few days.
   - Update the [[ssreact]] note's Hosting section with the new deploy commands (§2) and remove the old exclude-list command.
   - Record the migration in the [[ssapi]] note.

**dev / app later (with D6):**
- Same steps. The **vanilla frontend** (simple-social's `.html`, `js/`, `css/`, `sw.js`, `manifest.json`, `pwa-icons/`) goes into `public_html/app/`.
- Its URLs keep working through the rewrites: `/app.html` → `app/app.html`, and push links `/app.html#/post/…` still resolve.
- **This ends the "simple-social repo is the docroot, deploy = `git pull`" model on those hosts.** The frontend is deployed with rsync into `app/` like ssreact. **DECISION for Dave when D6 happens.**

---

## 5. Effects on the other plans
- **Phone plan, Phase 8:** host the APK and `android-version.json` in `public_html/downloads/`. The root `.htaccess` already serves them with the right content type and `no-cache` for the version file.
- **Desktop plan:** unaffected (it proxies `/api.php`, `/media.php` and `/media/*`, which are unchanged).
- **ssapi plan S2:** the canonical web-root `.htaccess` now lives in `ssapi/deploy/root.htaccess`, not in ssreact. **D5:** decided and replaced by this plan.
- **ssapi el1 checklist:** its react `.htaccess` checks still apply after the migration; rerun them.

## 6. Estimate
- **Agent work:** L1 about half a day, L2 + the §3 Apache check about a day, L4 trivial. On the $20 plan, call it **2–4 days** of calendar time.
- **Migration:** about an hour per host for Dave or an `el1` agent, plus verification.
