---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi @ c700a72, ssreact @ eb9ac21
---

# ssapi improvement plan (security + speed)

A full review of [[ssapi]] as used by [[ssreact]]. It covers security risks, speed, unused or poorly written code, and design problems. **It is written to be carried out task by task by an agent.** Every task has the files, the fix, a local check that proves it, and a commit message.

Line numbers refer to the commits above. If code has moved since then, use the **function name** to find it.

---

## 0. Read this first: rules for the implementing agent

1. **Repos you may change:**
   - `ssapi` for every backend task.
   - `ssreact` only for tasks marked **[ssreact]**.
   - Never edit [[simple-social]]. It holds a separate copy of the backend, which is what actually runs `app.davidfruin.com` today (see D6).
2. **Deploying:**
   - Never deploy anything, never SSH to any server, never point a tool or proxy at `app.davidfruin.com`.
   - Every check in this document runs against the **local bench** from Phase 0.
   - Dave deploys by hand when he is ready.
   - Some checks are marked **Dave on el1**. Leave those for Dave and list them in your hand-off.
3. **Commits and pauses:** make one task per commit. After each **phase**, stop and report to Dave before you start the next one. This is his standing preference, so don't chain phases together.
4. **Gated items:** items marked **GATED** or **DECISION** are skipped until Dave explicitly says go.
5. **Tests:** don't add a `tests/` folder to ssapi, because tests live in [[sstests]]. If you write automated tests, they go in `sstests/backend/`.
6. **Decisions already recorded in the war-table notes. Don't undo them:**
   - `getUsers` returning every user's email is by design. Email is the only handle, and search depends on it.
   - There is no CORS. The app stays same-origin.
   - Push URLs stay as `/app.html#/…`. ssreact's `sw.js` rewrites them on the client because the vanilla app still needs them.
   - The GitHub Actions deploy pipelines are **on hold**.
   - Post IDs keep the `"<userId>.<unixTime>"` shape.
7. **Many clients share this API:**
   - the vanilla web app in [[simple-social]]
   - ssreact
   - [[simple-social-cli]], [[simple-social-cli-interactive]] and [[simple-social-tui]]
   - ssreact-native and sselectron, which are planned

   So prefer additive changes: new fields and new optional parameters. Anything that changes an existing request or response shape is marked **BREAKING** with the clients it affects.
8. **Public repo:** this vault is public. Never paste secrets, tokens or real user data into commits, logs you share, or notes.
9. Confidence labels used below:
   - **LOCKED**: confirmed by reading the code.
   - **UNVERIFIED**: depends on server state that couldn't be checked from here.

---

## 1. How the system fits together (one-page model)

**Request lifecycle.** Every client sends `POST /api.php` with `action=<name>` as form-encoded fields. Uploads go to `POST /media.php`, which is a separate entry point with its own copies of the helpers.

`api.php` does the following in order:
1. Requires `config.php`, `logging.php`, `webpush.php`, `schema.php`, `auth.php` and `vendor/autoload.php`. The autoloader eagerly loads all seven `src/*/handlers.php` files through Composer's `files` list.
2. Merges the raw body into `$_POST`.
3. `db()` opens SQLite and runs about 15 `CREATE … IF NOT EXISTS` / `PRAGMA` statements. **This happens on every request.**
4. `requireAuth()` runs, unless the action is public.
5. It calls `handle_<action>($pdo, $user)`.

Every handler ends in `respond()`, which echoes JSON and calls `exit`.

**Auth model** (`auth.php`):
- Login creates a row in `sessions` and returns a 1-day HS256 JWT (`sub`, `sid`) plus a 30-day sliding refresh token. Only the refresh token's SHA-256 hash is stored.
- Every protected request checks the JWT signature **and** looks up the session row, so revoking a session takes effect immediately.

**Storage:**
- One SQLite file at `<domain>/private/userdata.db`.
- Media is stored as plain files under `<docroot>/media/<uid>/<type>/` and served directly by Apache.
- Follows are still a JSON array in `users.follows`. Posts and likes moved to real tables on 2026-09-24.

**Where it runs** (from the [[ssreact]] and [[simple-social]] notes):

| Host | Frontend | Backend code | PHP runtime |
|---|---|---|---|
| app.davidfruin.com (prod, real users) | vanilla JS | simple-social's copy | mod_fcgid |
| dev.davidfruin.com | vanilla JS | simple-social's copy | mod_fcgid |
| react.davidfruin.com (test) | ssreact `dist/` | a plain file copy of the backend in the same docroot | PHP-FPM in a chroot (no ffmpeg/ffprobe inside it) |

**ssreact's side** (`src/lib/api.ts`):
- A singleton `ApiClient` keeps the JWT and the refresh token in `localStorage` (`ss_jwt`, `ss_refresh`).
- On a 401 it runs one shared refresh request at a time, then retries the call.
- If the refresh fails, it shows a re-login modal that queues every waiting call.

---

## 2. Phase 0: local test bench (do this before any task)

Every later task is verified here. Nothing in Phase 0 changes ssapi.

### 0.1 Get the real `users` / `pending_users` schema
These two tables are created **in no source file**. They predate the repo, so a fresh install can't create them. That is itself a finding (see D1).

- Ask Dave to run this on dev and paste the output. It prints table definitions only, with no data:
  ```
  sqlite3 <dev private dir>/userdata.db '.schema users' '.schema pending_users'
  ```
- Until he does, use this reconstruction from the code (UNVERIFIED; the column types are guesses):
  ```sql
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL,
    password TEXT NOT NULL,
    posts TEXT, follows NUMERIC, followers NUMERIC, jwt TEXT,
    created_at TEXT,
    reset_otp TEXT, reset_expires INTEGER DEFAULT 0,
    last_notifications_seen_at TEXT,
    is_admin INTEGER DEFAULT 0
  );
  CREATE TABLE IF NOT EXISTS pending_users (email TEXT, password TEXT, otp TEXT, dateCreated INTEGER);
  ```
  `db()` adds `theme` and `hand` itself on the first request.

### 0.2 Build the bench
```bash
BENCH=/tmp/ssbench
rm -rf "$BENCH" && mkdir -p "$BENCH/private"
rsync -a --exclude .git <path-to-ssapi>/ "$BENCH/site/"
printf 'JWT_SECRET=%s\n' "$(openssl rand -hex 32)" > "$BENCH/private/.env"
cd "$BENCH/site" && (composer dump-autoload 2>/dev/null || php -r '
  $f = json_decode(file_get_contents("composer.json"), true)["autoload"]["files"];
  @mkdir("vendor");
  file_put_contents("vendor/autoload.php", "<?php\n" . implode("", array_map(fn($x) => "require_once __DIR__ . \"/../$x\";\n", $f)));')
```
`config.php` resolves `private/` as the docroot's **parent**, so `$BENCH/private` is picked up automatically.

Seed the bench with `$BENCH/seed.php`:
```php
<?php
$pdo = new PDO('sqlite:' . __DIR__ . '/private/userdata.db');
$pdo->exec(file_get_contents(__DIR__ . '/schema-users.sql')); // the SQL from 0.1
$ins = $pdo->prepare("INSERT INTO users (email, password, posts, follows, followers, jwt, created_at) VALUES (?, ?, '[]', '[]', '[]', '', datetime('now','localtime'))");
foreach (['alice@test.local', 'bob@test.local'] as $e) $ins->execute([$e, password_hash('Passw0rd!', PASSWORD_DEFAULT)]);
echo "seeded\n";
```
Then start the server:
```bash
php "$BENCH/seed.php"
# Mail is captured to a file instead of being sent:
php -d sendmail_path="tee -a $BENCH/mail.log >/dev/null" -S 127.0.0.1:8080 -t "$BENCH/site"
```
Note that `php -S` ignores `.htaccess`. Tasks that depend on Apache rules say so and are checked by Dave on the server.

### 0.3 curl helpers
```bash
api()   { local a=$1; shift; curl -s http://127.0.0.1:8080/api.php ${JWT:+-H "Authorization: Bearer $JWT"} -d "action=$a" "$@"; echo; }
login() { JWT=$(curl -s http://127.0.0.1:8080/api.php -d action=login -d "email=$1" --data-urlencode 'password=Passw0rd!' | php -r 'echo json_decode(stream_get_contents(STDIN))->jwt ?? "";'); echo "JWT set: ${JWT:0:12}…"; }
login alice@test.local && api getMyInfo
```

### 0.4 Run ssreact against the bench
Do task **C6** first. It is a 2-line change to `vite.config.ts`. Then run:
```bash
SS_API_TARGET=http://127.0.0.1:8080 pnpm dev
```
Check ssreact with `pnpm lint && pnpm build`. Both must pass clean after every ssreact task.

---

## 3. Tier 1: Security, fix first

### S1. The push endpoint accepts any URL, and the server then makes requests to it
**Severity:** High. **Where:**
- `src/Notifications/handlers.php` → `handle_savePushSubscription` (l.84)
- `webpush.php` → `sendWebPush` (l.118–151)

**Problem (LOCKED):**
- `savePushSubscription` stores whatever `endpoint` it is given, as long as it parses as a URL (`FILTER_VALIDATE_URL`). It doesn't restrict the scheme, the host or the port, and it has no per-user limit on rows.
- Later, every notification for that user makes the server `curl` that address. There are no protocol restrictions and a 10s timeout.
- Any logged-in user can trigger notifications to themselves at will, because self-mentions notify. That lets them make the server send requests to addresses of their choosing, including addresses that are only reachable from the server itself. This is a server-side request forgery (SSRF) weakness.
- It also makes their own requests slow (see P2).

**Fix:**
1. Accept only real browser push services.
2. Validate key lengths.
3. Cap each user at 10 subscriptions.
4. Lock curl down.

Add this to `webpush.php`:
```php
// Browser push services only. Anything else is either a bug or an attempt to
// make the server send requests somewhere it shouldn't.
const PUSH_HOST_SUFFIXES = [
    'fcm.googleapis.com',                // Chrome, Edge (Chromium), Android
    'updates.push.services.mozilla.com', // Firefox
    'notify.windows.com',                // WNS
    'push.apple.com',                    // Safari / iOS
];

function isAllowedPushEndpoint($endpoint) {
    $p = parse_url((string)$endpoint);
    if (!$p || ($p['scheme'] ?? '') !== 'https' || empty($p['host'])) return false;
    if (isset($p['user']) || isset($p['pass'])) return false;
    if (isset($p['port']) && (int)$p['port'] !== 443) return false;
    $host = strtolower($p['host']);
    foreach (PUSH_HOST_SUFFIXES as $suffix) {
        if ($host === $suffix || str_ends_with($host, '.' . $suffix)) return true;
    }
    return false;
}
```
In `sendWebPush`, before `curl_init`:
```php
if (!isAllowedPushEndpoint($endpoint)) return 410; // caller deletes the row
```
In the `curl_setopt_array` options:
```php
CURLOPT_PROTOCOLS => CURLPROTO_HTTPS,
CURLOPT_FOLLOWLOCATION => false,
CURLOPT_CONNECTTIMEOUT => 2,
CURLOPT_TIMEOUT => 4,
```
In `handle_savePushSubscription`, replace the `FILTER_VALIDATE_URL` line:
```php
if (!isAllowedPushEndpoint($endpoint)) bad('Invalid endpoint', 400);
if (strlen(webpush_b64url_decode($p256dh)) !== 65 || strlen(webpush_b64url_decode($auth)) !== 16) bad('Invalid subscription keys', 400);
```
and after the `INSERT OR REPLACE`:
```php
$pdo->prepare('DELETE FROM push_subscriptions WHERE user_id = ? AND id NOT IN
    (SELECT id FROM push_subscriptions WHERE user_id = ? ORDER BY id DESC LIMIT 10)')
    ->execute([$user['sub'], $user['sub']]);
```
The `410` return value makes `pushNotification()` delete any bad rows that were stored before this fix, with no separate cleanup step.

**Verify (bench):** `login alice@test.local`, then:
- `api savePushSubscription -d endpoint=http://127.0.0.1:9/x -d p256dh=… -d auth=…` → `400 Invalid endpoint`.
- Repeat with `https://fcm.googleapis.com/fcm/send/abc`, a valid 65-byte p256dh and a 16-byte auth (generate them with `openssl ecparam -genkey -name prime256v1` or copy them from a real browser subscription) → `200`.
- Insert 12 of them → `SELECT COUNT(*) FROM push_subscriptions WHERE user_id=1` returns 10.

**Clients:** browsers only ever send these hosts, so nothing breaks.

**Commit:** `security: restrict push endpoints to browser push services, cap per user`

---

### S2. The shared docroot is missing the backend's protective `.htaccess` (react.davidfruin.com; prod after migration)
**Severity:** High if confirmed. **UNVERIFIED** (Dave on el1). **Where:**
- `ssreact/public/.htaccess`, which ships in `dist/`
- `ssapi/.htaccess`
- `config.php` l.68–79 (log location)

**Problem:**
- On react.davidfruin.com the PHP backend and the React build share one docroot. The deploy is `rsync dist/ …`, and `dist/.htaccess` is ssreact's **SPA-only** file. The backend's `.htaccess` was never part of the file list copied there (see the [[ssreact]] note's "Backend files copied" list). That explains why `CGIPassAuth On` had to be added to the vhost.
- So on that host, nothing blocks direct web requests to `config.php`, `schema.php`, `src/**`, `vendor/**`, `composer.*` or `logs/`, and directory listing may be on for `/media/`.
- The most likely real exposure is the logs:
  - `config.php` falls back to `<docroot>/logs/` when `private/logs` doesn't exist.
  - `debug` is hard-coded `true` (see S10).
  - So `api.log` (emails, IPs, post snippets, session ids) can be sitting in a web-served folder with no deny rule.
  - The ssreact note already records `api.log` (about 10 MB) inside `app.davidfruin.com/public_html`. On prod the backend's `.htaccess` does deny `.log`, but the same overwrite will happen there **the day ssreact replaces the vanilla frontend on app.davidfruin.com**.

**Dave on el1, check first:**
```bash
for p in logs/api.log logs/media.log config.php composer.json src/Auth/handlers.php vendor/autoload.php media/; do
  printf '%-28s %s\n' "$p" "$(curl -s -o /dev/null -w '%{http_code}' https://react.davidfruin.com/$p)"; done
```
Anything other than `403`/`404` (and for `config.php`/handlers, anything other than `403`) confirms the problem.

**Fix:** keep **one** canonical `.htaccess` for a shared docroot in `ssreact/public/.htaccess`, since it is always deployed last. Mirror its backend rules into `ssapi/.htaccess` (leave out the SPA block there) and put a comment in both files saying they must stay in sync. Proposed `ssreact/public/.htaccess`:
```apache
# Shared docroot: ssreact's static build + the ssapi PHP backend (api.php, media.php).
# Backend rules here must stay in sync with ssapi/.htaccess.
Options -Indexes

# Authorization header passthrough (mod_fcgid hosts; PHP-FPM hosts also need CGIPassAuth On in the vhost).
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1

# Only the real entry points may execute. index.php = maintenance mode (see index.maintenance.php).
<FilesMatch "\.(php\d?|phtml|phar|inc)$">
    Require all denied
</FilesMatch>
<FilesMatch "^(api|media|index)\.php$">
    Require all granted
</FilesMatch>

# Never serve data, logs, env or tooling files.
<FilesMatch "(\.(db|db-wal|db-shm|sqlite|log|env|lock|md)|^\.(env.*|git.*|htaccess)|^composer\.json)$">
    Require all denied
</FilesMatch>

<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

  # Maintenance mode: active only while a file named index.php exists.
  RewriteCond %{DOCUMENT_ROOT}/index.php -f
  RewriteCond %{REQUEST_URI} !^/index\.php$
  RewriteRule ^ index.php [L]

  RewriteRule ^(vendor|src|logs|private|tmp)(/|$) - [F,L]
  # Uploaded media is data, never code or active content (see S3).
  RewriteRule ^media/.*\.(php\d?|phtml|phar|html?|svg|js|mjs)$ - [F,L]

  # SPA: real files are served as-is, everything else is the React app.
  RewriteCond %{REQUEST_FILENAME} -f [OR]
  RewriteCond %{REQUEST_FILENAME} -d
  RewriteRule ^ - [L]
  RewriteRule ^ index.html [L]
</IfModule>
```
Apache merges `<FilesMatch>` blocks in order, so the second block re-allows the three entry points after the first one denies all PHP.

**Before applying the allowlist to `ssapi/.htaccess`:** Dave lists the `*.php` files in the app and dev docroots on el1 (`ls *.php`). If the vanilla frontend relies on any other PHP file, add it to the allowlist.

Also do S11 so logs can never land in the docroot again.

**Verify:**
- Locally: `pnpm build` and confirm `dist/.htaccess` is the new file.
- **Dave on el1** after deploying: rerun the curl loop. It should print all 403/404, and `/feed`, `/api.php` (POST) and an existing `/media/...webp` should still work.

**Commit (ssreact):** `security: shared-docroot .htaccess with backend deny rules`
**Commit (ssapi):** `security: allowlist PHP entry points in .htaccess`

---

### S3. Uploads trust the type the client claims; the temp file sits in a web folder with the client's extension
**Severity:** High. **Where:** `src/Media/handlers.php`
- `handle_uploadMedia` (l.322–437; the type comes from `$file['type']` at l.356, the temp file at l.384–385)
- `processVideo` (l.265), `processAudio` (l.312)

**Problem (LOCKED):**
- The media type is decided by the client-supplied `Content-Type` alone.
- The upload is moved to `media/<uid>/<type>/temp_<base>.<client extension>`. That folder is **web-served**, and the extension is whatever the client's filename says. The file stays there while ffprobe and ffmpeg run (up to minutes).
- When ffmpeg fails, the **original bytes are kept** and served from `/media/`. On react.davidfruin.com ffmpeg *always* fails, because it isn't inside the chroot.
- Images are safe, because GD re-encodes them, which discards anything that isn't a real image. Video and audio are not.

**Fix:**
- Detect the real format from the file's own magic bytes. This needs no extension, because `fileinfo` and `mbstring` can't be assumed on this host.
- Pick the extension from the detected format.
- Stage the file outside the docroot.

Add this helper:
```php
// Real container/format from the file's first bytes; ignores whatever the client claimed.
// Returns ['family' => image|av|audio, 'ext' => ...] or null.
function sniffMedia($path) {
    $h = @file_get_contents($path, false, null, 0, 16);
    if ($h === false || strlen($h) < 12) return null;
    if (str_starts_with($h, "\x1A\x45\xDF\xA3")) return ['family' => 'av', 'ext' => 'webm'];
    if (substr($h, 4, 4) === 'ftyp') return ['family' => 'av', 'ext' => substr($h, 8, 4) === 'qt  ' ? 'mov' : 'mp4'];
    if (str_starts_with($h, 'RIFF') && substr($h, 8, 4) === 'WAVE') return ['family' => 'audio', 'ext' => 'wav'];
    if (str_starts_with($h, 'ID3') || (ord($h[0]) === 0xFF && (ord($h[1]) & 0xE0) === 0xE0)) return ['family' => 'audio', 'ext' => 'mp3'];
    $img = @getimagesize($path);
    $imgExt = [IMAGETYPE_JPEG => 'jpg', IMAGETYPE_PNG => 'png', IMAGETYPE_GIF => 'gif', IMAGETYPE_WEBP => 'webp'];
    if ($img && isset($imgExt[$img[2]])) return ['family' => 'image', 'ext' => $imgExt[$img[2]]];
    return null;
}
```
In `handle_uploadMedia`, after the size check:
```php
$sniffed = sniffMedia($tmpPath);
$compatible = $sniffed && (
    ($mediaType === 'image' && $sniffed['family'] === 'image') ||
    ($mediaType === 'video' && $sniffed['family'] === 'av') ||
    ($mediaType === 'audio' && in_array($sniffed['family'], ['av', 'audio'], true)) // MediaRecorder audio is WebM/MP4
);
if (!$compatible) bad("That file doesn't look like a valid $mediaType.", 400);

$stageDir = dirname($CONFIG['db_path']) . '/tmp';          // private/tmp, never web-served
if (!is_dir($stageDir)) mkdir($stageDir, 0700, true);
$tempInput = "$stageDir/{$base}.{$sniffed['ext']}";
```
Then:
- Use `$sniffed['ext']` wherever `originalExtension($mimeType, $fileName)` was used, and delete `originalExtension()`.
- Remove `$inputExt = pathinfo($fileName, …)`.
- Leave `processImage` alone. It already chooses its decoder from the temp file's extension, which is now trustworthy.
- `ensureMediaDir` still creates the final folder. Only the staging moved.

The `^media/…` RewriteRule in S2 is the second layer of protection. Also add `Header always set X-Content-Type-Options "nosniff"` (S12) so browsers never guess a different type for a kept original.

**Verify (bench):**
1. Upload a real PNG with `curl -F action=uploadMedia -F file=@x.png -H "Authorization: Bearer $JWT" http://127.0.0.1:8080/media.php` → `200` and a `.webp`.
2. Upload a text file renamed to `clip.mp4` with `-F 'file=@notes.txt;type=video/mp4'` → `400`.
3. Upload a real short mp4 → `200`.
4. During uploads, `ls $BENCH/site/media/1/*/` never shows a `temp_` file.
5. `ls $BENCH/private/tmp` is empty afterwards.

**Commit:** `security: detect upload type from content, stage uploads outside docroot`

---

### S4. Code-by-email endpoints can be used to flood any inbox
**Severity:** Medium-High (it also damages mail deliverability, which has already bitten once; see the TUI registration note in [[simple-social]]). **Where:** `src/Auth/handlers.php` → `handle_sendOTP` (l.210), `handle_sendRegisterOTP` (l.285).

**Problem (LOCKED):**
- Both endpoints are public and send an email on every call. There is no per-email, per-IP or cooldown limit.
- `sendOTP` also confirms whether an account exists ("No account found"). That is low impact, since `getUsers` exposes emails by design, but the throttle should count those misses too.

**Fix:** add a counter-style throttle that reuses `auth_attempts`, plus a 60-second cooldown per address:
```php
// Counts sends (not failures) per key per ATTEMPT_WINDOW.
function throttleSend($pdo, $key, $limit) {
    $now = time();
    $sel = $pdo->prepare('SELECT failures, window_start FROM auth_attempts WHERE attempt_key = ?');
    $sel->execute([$key]);
    $row = $sel->fetch(PDO::FETCH_ASSOC);
    $inWindow = $row && $now - $row['window_start'] < ATTEMPT_WINDOW;
    $count = $inWindow ? $row['failures'] + 1 : 1;
    if ($count > $limit) bad('Too many codes requested. Try again later.', 429);
    $pdo->prepare('INSERT OR REPLACE INTO auth_attempts (attempt_key, failures, window_start, locked_until) VALUES (?, ?, ?, 0)')
        ->execute([$key, $count, $inWindow ? $row['window_start'] : $now]);
}
```
In both handlers, right after the email validation and **before** the user lookup:
```php
$k = attemptKeys('otpsend', $email);
throttleSend($pdo, $k['email'], 3);   // 3 codes per address per 15 min
throttleSend($pdo, $k['ip'], 10);     // 10 per IP per 15 min
```
Cooldown:
- In `sendOTP`, after fetching the row: `if ($row['reset_expires'] - 600 > time() - 60) bad('Please wait a minute before requesting another code.', 429);` (this means a code was issued less than 60s ago). Add `reset_expires` to that `SELECT`.
- In `sendRegisterOTP`, before the `DELETE`: look up `pending_users.dateCreated` for this email, and if it is newer than 60s, return the same error.

Also make sure `auth_attempts` gets cleaned up: add `DELETE FROM auth_attempts WHERE window_start < ? AND locked_until < ?` (both set to `time() - ATTEMPT_WINDOW`) to `sessionSweep()`, or run it in the migration of P3.

**Verify (bench):** with the `tee` sendmail, call `api sendRegisterOTP -d email=new@test.local` four times quickly.
- The 1st call gives 200.
- The 2nd gives 429 (cooldown).
- After waiting more than 60s three times, the 4th within 15 minutes gives 429 (limit).
- `grep -c 'OTP code' $BENCH/mail.log` equals the number of 200 responses.

**Clients:** CLI, TUI and the web clients already show `message` on failure, so this needs no client change.

**Commit:** `security: throttle OTP emails per address and per IP`

---

### S5. Weak randomness for codes and filenames
**Severity:** Medium. **Where:** `src/Auth/handlers.php` l.219 and l.293; `src/Media/handlers.php` l.376.

**Problem (LOCKED):** `mt_rand()` isn't a cryptographic generator. It is used for the 6-digit login and registration codes, and for the random part of media filenames. Media URLs are public by design, but the random part shouldn't be guessable.

**Fix:**
- Codes: `$otp = sprintf('%06d', random_int(0, 999999));`
- Filenames: `$random = bin2hex(random_bytes(8));`

**Verify:** `grep -rn mt_rand` returns nothing. Then do one registration code round-trip on the bench (read the code from `mail.log`) and one upload.

**Commit:** `security: use CSPRNG for OTPs and media filenames`

---

## 4. Tier 2: Speed, biggest wins

### P1 [ssreact]. The saved JWT is never loaded, so every page load starts with a 401
**Severity:** High (speed), and Medium (security, see below). **Where:** `ssreact/src/lib/api.ts` → `call()` l.125 and `mediaRequest()` l.339.

**Problem (LOCKED):**
- Both functions read `this.jwt` directly. That field is only filled by `setJwt()` or `getJwt()`, and nothing calls `getJwt()` on startup.
- So after any page load or reload, the **first authenticated call of every burst goes out with no token**. It gets a 401, does a refresh round-trip (which writes to the DB in `sessionRefresh`), and then retries.
- The war-table note even records the resulting "harmless 401s" as expected behaviour.
- Security side effect: `logout()` after a reload sends **no token**, so the server never revokes the session. "Log out" then leaves a live session behind until it expires.

**Fix:** in both places:
```ts
const jwt = this.getJwt();
if (jwt) headers['Authorization'] = `Bearer ${jwt}`;
```

**Verify:**
- With `pnpm dev` against the bench, log in, reload `/feed`, and open the DevTools Network panel. There should be no 401 and no `refreshToken` call. Before the fix you'll see 401 → `refreshToken` → retry.
- Then Settings → Log out, log in again as the same user and open Devices. The old session must be gone.

**Commit:** `perf: send stored JWT on first request after reload`

---

### P2. Push notifications are sent while the user waits
**Severity:** High. **Where:** `api.php` → `createNotification` (l.149), `pushNotification` (l.180), `respond` (l.65); `webpush.php` → `sendWebPush`.

**Problem (LOCKED):**
- Likes, comments, follows, unfollows and mentions each send push messages to **every** device of the recipient. Each send does a fresh ECDH key generation, encryption and a blocking `curl` with a timeout of up to 10s, one after another.
- That all happens **before** the response is sent.
- One slow push service, or a post mentioning 10 people with several devices each, stalls the request for seconds.
- SQLite stays open the whole time, so other requests can hit `database is locked`.

**Fix:** queue the push work and run it after the response has been flushed.
```php
// api.php, near the helpers
$DEFERRED = [];
function defer(callable $fn) { global $DEFERRED; $DEFERRED[] = $fn; }

function runDeferred() {
    global $DEFERRED;
    $jobs = $DEFERRED; $DEFERRED = [];
    foreach ($jobs as $fn) {
        try { $fn(); } catch (Throwable $e) { logMsg('deferred failed: ' . $e->getMessage()); }
    }
}
```
In `respond()`, replace the echo/exit section:
```php
$body = json_encode($data, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
ob_clean();
http_response_code($code);
header('Content-Type: application/json; charset=utf-8');
header('Content-Length: ' . strlen($body));
echo $body;
global $DEFERRED;
if (!empty($DEFERRED)) {
    while (ob_get_level() > 0) ob_end_flush();
    flush();
    if (function_exists('fastcgi_finish_request')) fastcgi_finish_request(); // PHP-FPM: client is released here
    ignore_user_abort(true);
    runDeferred();
}
exit;
```
In `createNotification`, replace the `try { pushNotification(...) } catch…` block with:
```php
defer(fn() => pushNotification($pdo, $recipientId, $actorEmail, $type, $postId, $actorId));
```
Caveats:
- `fastcgi_finish_request` only exists under PHP-FPM, which react.davidfruin.com uses. Under mod_fcgid (app and dev), the `Content-Length` header plus `flush()` lets the client finish reading the response, but the PHP worker stays busy until the pushes are done.
- The shorter curl timeouts from S1 (2s connect, 4s total) cap the damage there.
- Optional follow-up: send to all of one user's devices in parallel with `curl_multi_*`.

**Verify (bench):**
- Temporarily add `defer(fn() => sleep(3));` at the top of `handle_getMyInfo`.
- `curl -o /dev/null -w '%{time_total}\n' …getMyInfo` should be well under 1s, even though the PHP process keeps running for 3s.
- Remove the temporary line before committing.
- Then run `api likePost -d postId=<bob's post>` as alice with one subscription row for bob that points at `https://fcm.googleapis.com/fcm/send/fake`. The response is immediate, and afterwards the row is deleted (404/410 → cleanup still runs).

**Commit:** `perf: send push notifications after the response is flushed`

---

### P3. About 15 schema statements run on every request; no busy timeout
**Severity:** High. **Where:** `api.php` → `db()` l.77–116; `schema.php`; `media.php` → `db()` l.46. `media.php` calls `db()` **twice** per upload (once in `requireAuth`, once in the handler).

**Problem (LOCKED):**
- Every request runs every `CREATE TABLE/INDEX IF NOT EXISTS`, two `PRAGMA table_info` scans, and the column-existence checks.
- No `busy_timeout` is set, so concurrent writes fail straight away with "database is locked" instead of waiting a few milliseconds.
- The schema is also spread across three places, and two core tables aren't defined anywhere (see 0.1).

**Fix:** run migrations once, tracked by `PRAGMA user_version`, through one shared connection function. Move this into `schema.php` and **delete both `db()` copies** from `api.php` and `media.php`. Both files already `require schema.php`, and no autoloaded file defines `db()`, so there is no redeclare problem.
```php
const SCHEMA_VERSION = 1;

function db() {
    static $pdo = null;
    if ($pdo) return $pdo;
    global $CONFIG;
    $pdo = new PDO('sqlite:' . $CONFIG['db_path'], null, null, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_TIMEOUT => 5,
    ]);
    $pdo->exec('PRAGMA busy_timeout = 5000');
    ensureSchema($pdo);
    return $pdo;
}

function ensureSchema(PDO $pdo) {
    if ((int)$pdo->query('PRAGMA user_version')->fetchColumn() >= SCHEMA_VERSION) return;
    $pdo->exec('BEGIN IMMEDIATE');
    try {
        $v = (int)$pdo->query('PRAGMA user_version')->fetchColumn(); // re-check under the write lock
        if ($v < 1) migration1($pdo);
        $pdo->exec('PRAGMA user_version = ' . SCHEMA_VERSION);
        $pdo->exec('COMMIT');
    } catch (Throwable $e) {
        $pdo->exec('ROLLBACK');
        throw $e;
    }
}

// Everything that used to run per request, verbatim and still idempotent, plus the P4 indexes
// and the users/pending_users definitions from Phase 0.1 (CREATE IF NOT EXISTS, so harmless on live DBs).
function migration1(PDO $pdo) { /* move the existing statements here */ }
```
Rules for future changes: add `migration2()` and bump the version. Never edit a migration that has already shipped.

**Optional P3b (GATED, Dave):** `PRAGMA journal_mode=WAL`, run once outside the transaction. It lets reads proceed during writes. **But** it creates `userdata.db-wal` and `-shm` files next to the DB, which means:
- copying `userdata.db` with `cp`, as was done for dev → react, can silently lose recent writes;
- backups must use `sqlite3 userdata.db ".backup <file>"`.

Dave decides whether that change to his copy and backup workflow is worth it.

**Verify (bench):**
1. Start with a DB that has the **old** schema (the seed from 0.2, before any request) and make one request. `php -r '$p=new PDO("sqlite:/tmp/ssbench/private/userdata.db"); echo $p->query("PRAGMA user_version")->fetchColumn();'` prints `1`, and all tables exist.
2. Time 50 requests before and after: `time (for i in $(seq 50); do api getMyInfo >/dev/null; done)`. Expect a clear drop.
3. Do one upload through `media.php`, confirm it works, and check that `media.php` opens only one PDO handle (`db()` is now static).

**Commit:** `perf: run schema migrations once via user_version; shared db() with busy_timeout`

---

### P4. Missing indexes and per-row queries on every list
**Severity:** High. **Where:**
- `src/Comments/handlers.php` → `handle_getPostCommentCounts` l.76 (one COUNT per post), `handle_getPostComments` l.41
- `api.php` → `hydrateMentions` l.253, which every post list and comment list calls **once per row**
- `src/Notifications/handlers.php` → `handle_getNotifications`

**Problem (LOCKED):**
- `comments.post_id` has no index, so every comment count and comment list scans the whole table.
- A 25-post feed page runs about 25 mention lookups, and the feed page in ssreact then asks for 25 comment counts.
- Notifications are filtered by `recipient_id` and sorted by `created_at`, but the existing index covers only `recipient_id`.

**Fix:**
1. In `migration1` (P3):
   ```sql
   CREATE INDEX IF NOT EXISTS idx_comments_post_created ON comments(post_id, created_at);
   CREATE INDEX IF NOT EXISTS idx_notifications_recipient_created ON notifications(recipient_id, created_at);
   ```
2. Batch the mentions:
   ```php
   // [key => [{id,email},...]] for every text, using a single users query.
   function hydrateMentionsBatch($pdo, array $texts) {
       $idsByKey = []; $all = [];
       foreach ($texts as $k => $t) {
           preg_match_all('/@\[(\d+)\]/', (string)$t, $m);
           $ids = array_values(array_unique(array_map('intval', $m[1])));
           $idsByKey[$k] = $ids;
           foreach ($ids as $id) $all[$id] = true;
       }
       $emails = [];
       if ($all) {
           $ids = array_keys($all);
           $ph = implode(',', array_fill(0, count($ids), '?'));
           $stmt = $pdo->prepare("SELECT id, email FROM users WHERE id IN ($ph)");
           $stmt->execute($ids);
           foreach ($stmt->fetchAll(PDO::FETCH_ASSOC) as $r) $emails[(int)$r['id']] = $r['email'];
       }
       $out = [];
       foreach ($idsByKey as $k => $ids) {
           $out[$k] = array_map(fn($id) => ['id' => $id, 'email' => $emails[$id] ?? null], $ids);
       }
       return $out;
   }
   function hydrateMentions($pdo, $text) { return hydrateMentionsBatch($pdo, [$text])[0]; }
   ```
   In `handle_getMyPosts`, `handle_getUserPosts`, `handle_fetchFollowedPosts` and `handle_getPostComments`, build the text array before the loop:
   ```php
   $mentions = hydrateMentionsBatch($pdo, array_column($rows, 'text'));
   ```
   Then use `$mentions[$i]` inside the loop, iterating with `foreach ($rows as $i => $row)`.
3. Batch the comment counts:
   ```php
   function getCommentCountsForPostIds($pdo, array $postIds) {
       if (!$postIds) return [];
       $ph = implode(',', array_fill(0, count($postIds), '?'));
       $stmt = $pdo->prepare("SELECT post_id, COUNT(*) AS n FROM comments WHERE post_id IN ($ph) GROUP BY post_id");
       $stmt->execute(array_values($postIds));
       $counts = array_fill_keys($postIds, 0);
       foreach ($stmt->fetchAll(PDO::FETCH_ASSOC) as $r) $counts[$r['post_id']] = (int)$r['n'];
       return $counts;
   }
   ```
   `handle_getPostCommentCounts` becomes `respond(good(['counts' => getCommentCountsForPostIds($pdo, $postIds)]))`, with the input list from S8. The response shape is unchanged.

**Verify (bench):**
- Seed 30 posts with mentions and comments using a small PHP loop. Responses must be **byte-identical** before and after: `api fetchFollowedPosts > before.json` on the old code, then `diff` against the new code's output.
- `EXPLAIN QUERY PLAN SELECT COUNT(*) FROM comments WHERE post_id='1.1'` shows `USING INDEX`.

**Commit:** `perf: index comments/notifications, batch mention and comment-count lookups`

---

### P5 [ssreact] + API. Feed and profile wait for posts, then make a second request for comment counts
**Severity:** Medium. **Where:**
- API: the three post-list handlers in `src/Posts/handlers.php`
- ssreact: `src/pages/FeedPage.tsx` l.34–44, `src/pages/ProfilePage.tsx` l.63–68 and l.92–97, `src/lib/types.ts` (`Post`)

**Fix (additive, non-breaking):**
- In each post-list handler, after `$rows`, compute `$counts = getCommentCountsForPostIds($pdo, array_column($rows, 'id'));` and set `$post['commentCount'] = $counts[$row['id']] ?? 0;`. Do the same in `handle_getPostById`.
- In ssreact:
  - add `commentCount?: number` to `Post`;
  - in `FeedPage.fetchPage` and the two Profile spots, use the embedded counts when **every** post has one, and fall back to the existing `getPostCommentCounts` call otherwise. The fallback matters because older backend copies, like the one on react.davidfruin.com today, won't send the field.
  ```ts
  const embedded = result.posts.every((p) => typeof p.commentCount === 'number');
  if (embedded) {
    setCommentCounts((prev) => ({ ...prev, ...Object.fromEntries(result.posts.map((p) => [p.id, p.commentCount!])) }));
  } else if (postIds.length > 0) { /* existing getPostCommentCounts block */ }
  ```

**Verify:** on the bench, the feed loads with **one** API call per page (Network panel), and the counts are correct. Pointing the dev proxy at an unchanged backend still shows the counts (fallback path).

**Commits:** `api: include commentCount in post objects` / `perf(ssreact): use embedded comment counts`

---

### P6 [ssreact]. Two notification pollers per tab, and they keep running in hidden tabs
**Severity:** Medium. **Where:** `src/lib/use-unseen-notifications.ts`, used by `Header.tsx` and `ThumbNav.tsx`, which are both always mounted by `Layout.tsx`.

**Problem (LOCKED):**
- The comment says "one poller", but each hook instance starts its own interval, so every tab makes **2 requests a minute** forever.
- That includes background tabs. Each request costs a JWT check, a session lookup and a COUNT query.

**Fix:** keep one shared poller at module level with a subscriber count, pause it while the tab is hidden, and check immediately when the tab becomes visible again.
```ts
const POLL_MS = 60000;
const subscribers = new Set<(n: number) => void>();
let lastCount: number | null = null;
let timer: ReturnType<typeof setInterval> | null = null;

function applyBadge(count: number) {
  if ('setAppBadge' in navigator) {
    if (count > 0) navigator.setAppBadge(count).catch(() => {});
    else navigator.clearAppBadge().catch(() => {});
  }
  navigator.serviceWorker?.controller?.postMessage({ type: 'SET_BADGE_COUNT', count });
}

async function check() {
  if (document.visibilityState === 'hidden' || subscribers.size === 0) return;
  try {
    const { count } = await api.getUnseenNotificationCount();
    lastCount = count;
    applyBadge(count);
    subscribers.forEach((fn) => fn(count));
  } catch { /* decoration only */ }
}
const onVisible = () => { if (document.visibilityState === 'visible') check(); };

function start() {
  timer = setInterval(check, POLL_MS);
  document.addEventListener('visibilitychange', onVisible);
  check();
}
function stop() {
  if (timer) clearInterval(timer);
  timer = null;
  lastCount = null;
  document.removeEventListener('visibilitychange', onVisible);
}

export function refreshUnseenNotificationCount() { check(); }

export function useUnseenNotificationCount(enabled: boolean) {
  const [count, setCount] = useState<number | null>(null);
  useEffect(() => {
    if (!enabled) { setCount(null); return; }
    subscribers.add(setCount);
    if (subscribers.size === 1) start();
    else if (lastCount !== null) setCount(lastCount);
    return () => {
      subscribers.delete(setCount);
      if (subscribers.size === 0) stop();
    };
  }, [enabled]);
  return count;
}
```

**Verify:**
- In `pnpm dev`, logged in, the Network panel shows **one** `getUnseenNotificationCount` per minute (it was two).
- Switch to another tab for 2 minutes: no calls. Switch back: one call immediately.
- "Mark as Seen" still clears both badges instantly.
- `pnpm lint` is clean.

**Commit:** `perf(ssreact): single visibility-aware notification poller`

---

## 5. Tier 3: Security and integrity, medium

### S6. Logout leaves push notifications running, and relies on an unverified token
**Where:** `src/Auth/handlers.php` → `handle_logout` l.118; `auth.php` → `jwtVerify` l.46, `jwtClaimsUnverified` l.68.

**Problem (LOCKED):**
- Logout revokes the session with its own `UPDATE`, which skips `sessionRevoke()`, so the device's push subscription isn't deleted. A signed-out phone keeps receiving notifications.
- It also decodes the token **without** checking the signature. This is limited, because the session id is 128-bit random, but there's no reason to skip the check.

**Fix:** let `jwtVerify` optionally ignore expiry, and use the normal revoke path.
```php
function jwtVerify($jwt, $allowExpired = false) {
    // ...unchanged...
    if (!$allowExpired && isset($payload['exp']) && time() > $payload['exp']) return false;
    return $payload;
}

function handle_logout($pdo) {
    $claims = jwtVerify(bearerToken(), true);
    if ($claims && !empty($claims['sid'])) {
        $stmt = $pdo->prepare('SELECT id FROM sessions WHERE id = ? AND user_id = ? AND revoked_at IS NULL');
        $stmt->execute([$claims['sid'], $claims['sub']]);
        if ($stmt->fetchColumn()) sessionRevoke($pdo, $claims['sid']);
        logMsg("LOGOUT: session={$claims['sid']}");
    }
    respond(good(['message' => 'Logged out']));
}
```
Then delete `jwtClaimsUnverified()`, which is now unused.

**[ssreact], optional:** in `SettingsPage`'s log-out handler, call `disablePush().catch(() => {})` before `logout()` so the browser drops its subscription too.

**Verify (bench):**
1. Log in and save a push subscription (see S1).
2. `api logout`.
3. `SELECT COUNT(*) FROM push_subscriptions` is 0, and `sessions.revoked_at` is set.
4. A token signed with the wrong secret (edit one character of the signature) leaves the session untouched.

**Commit:** `security: verified logout that also drops the session's push subscription`

---

### S7. Comments are accepted on posts that don't exist, and notify whoever the ID names
**Where:** `src/Comments/handlers.php` → `handle_createComment` l.14–39.

**Problem (LOCKED):**
- No check that the post exists.
- The owner to notify is taken from the ID's prefix (`explode('.', $postId)[0]`), so a comment on a made-up post ID still creates a "commented on your post" notification, and a push, for any user ID.

**Fix:** before the `INSERT`:
```php
$stmt = $pdo->prepare('SELECT user_id FROM posts WHERE id = ?');
$stmt->execute([$postId]);
$ownerId = $stmt->fetchColumn();
if ($ownerId === false) bad('Post not found', 404);
$ownerId = (int)$ownerId;
```
Delete the later `$ownerId = (int)explode(...)` line. Apply the same idea in `handle_likePost` (l.319–320): check self-likes against `$realOwnerId`, not the ID prefix.

**Verify (bench):** `api createComment -d postId=2.123 -d text=hi` → 404, and no new notifications row. A real post → 200.

**Commit:** `fix: reject comments on missing posts; derive owner from DB`

---

### S8. List inputs have no limits
**Where:**
- `limit`/`offset` in `handle_getMyPosts`, `handle_getUserPosts`, `handle_fetchFollowedPosts` (Posts l.209–310) and `handle_getPostComments`
- JSON ID lists in `handle_getUserEmails` (Users l.30), `handle_getPostPreviews` (Posts l.128) and `handle_getPostCommentCounts`

**Problem (LOCKED):**
- `limit` is cast to an int but never clamped. In SQLite `LIMIT -1` means "no limit", so a single call can return every post with every like and mention hydrated.
- The ID lists have no size cap and accept any JSON values. Very large lists can also go past SQLite's bound-variable limit and cause 500 errors.

**Fix:** add shared helpers in `api.php`, next to `good()`:
```php
function pageParams($default = 25, $max = 50) {
    $limit = isset($_POST['limit']) ? (int)$_POST['limit'] : $default;
    $offset = isset($_POST['offset']) ? (int)$_POST['offset'] : 0;
    return [max(1, min($max, $limit)), max(0, $offset)];
}

function jsonIdList($key, $max = 100, $ints = false) {
    $list = json_decode($_POST[$key] ?? '[]', true);
    if (!is_array($list)) return [];
    $list = array_values(array_unique(array_filter($list, 'is_scalar')));
    if (count($list) > $max) bad("Too many ids (max $max)", 400);
    if ($ints) return array_values(array_filter(array_map('intval', $list), fn($i) => $i > 0));
    return array_map('strval', $list);
}
```
Use them as follows:
- `[$limit, $offset] = pageParams();` in the four list handlers, plus `getNotifications`' offset.
- `jsonIdList('userIds', 500, true)` for `getUserEmails`. A popular post's likers list can be long, hence 500.
- `jsonIdList('postIds', 100)` for previews and counts.

ssreact always asks for 25, so it is unaffected.

**Verify (bench):**
- `api fetchFollowedPosts -d limit=-1` returns at most 50 posts.
- `api getUserEmails --data-urlencode 'userIds=["1","x",{"a":1}]'` → `{"1": …}` with no error.
- 101 post IDs → 400.

**Commit:** `security: clamp pagination and validate id lists`

---

### S9 (DECISION, Dave picks). Unlike and unfollow notifications can be used to spam
**Where:** `src/Posts/handlers.php` → `handle_unlikePost` l.357–359; `src/Follows/handlers.php` → `handle_followUser` l.78, `handle_unfollowUser` l.101.

**Problem (LOCKED):**
- Toggling like/unlike or follow/unfollow creates a new notification **and phone push** every time.
- `unfollowUser` notifies even when you weren't following the person.

**Do now, regardless of the decision (bug fix):** in `handle_unfollowUser`, only write the update and call `createNotification` when the filter actually removed an entry (compare `count()` before and after).

**Option A, drop them:** remove the `'unlike'` and `'unfollow'` `createNotification` calls.
- In `unlikePost`, also delete the earlier like notification:
  `DELETE FROM notifications WHERE recipient_id = ? AND actor_id = ? AND type = 'like' AND post_id = ?`.
- Every client already renders these types defensively, so removing them breaks nothing.

**Option B, deduplicate:** at the top of `createNotification`, skip if an identical (recipient, actor, type, post) notification exists from the last 10 minutes:
```php
$dup = $pdo->prepare('SELECT 1 FROM notifications WHERE recipient_id = ? AND actor_id = ? AND type = ?
    AND COALESCE(post_id, \'\') = COALESCE(?, \'\') AND created_at > ?');
$dup->execute([$recipientId, $actorId, $type, $postId, date('Y-m-d H:i:s', time() - 600)]);
if ($dup->fetchColumn()) return;
```

**Verify (bench):** as alice, like and unlike bob's post 5 times. Bob's `getNotifications` shows A: at most one "like" or nothing; B: at most one "like" and one "unlike" in 10 minutes.

**Commit:** `fix: don't notify unfollow without a prior follow`, plus the chosen option.

---

### S10. Debug logging is on in source, the log grows without limit, and there are two logging systems
**Where:**
- `config.php` l.5–6 (`'debug' => true, 'test_mode' => true`)
- `api.php` → `logMsg` l.28 (no rotation), `logRequest` l.39
- `media.php` → `logMsg` l.18 (always on, no rotation), `handle_uploadMedia` l.328 (logs the full `$_FILES`)
- `logging.php` (rotating logger; `logFrontendError`, `logFrontendInfo`, `logApiAccess` and `logPhpError` are never called)
- `handle_log_request` l.94 (lets clients write into the `api`, `access` and `php` categories)

**Problem (LOCKED):**
- Every request is logged, with emails, IPs, post text and session IDs, into `api.log`, which never rotates. About 10 MB on prod, per the notes.
- It sits in the docroot whenever `private/logs` is missing (see S2 and S11).

**Fix:**
1. In `config.php`, **after** the `.env` loading, set:
   ```php
   $CONFIG['debug'] = filter_var(getenv('APP_DEBUG') ?: 'false', FILTER_VALIDATE_BOOLEAN);
   ```
   Delete `debug` and `test_mode` from the array literal. Add `APP_DEBUG=false` to `.env.example`.
2. In `logging.php`, change the `writeLog` level check to `$minLevel = !empty($CONFIG['debug']) ? 'DEBUG' : 'WARN';`.
3. Make both `logMsg()` bodies one line: `writeLog('DEBUG', 'api', $msg);` (`'media'` in `media.php`). `media.php` must `require_once __DIR__ . '/logging.php';`. Now both logs rotate (5 MB × 3) and are off by default.
4. In `handle_uploadMedia`, replace `json_encode($_FILES)` with `"size=$fileSize declared=$mimeType"`.
5. In `handle_log_request`, force `$category = 'frontend'` and drop the `$user` check (`requireAuth` already guarantees it). Delete the four unused `log*` helpers and `getLogFile` if nothing else uses it.

**Verify (bench):**
- With no `APP_DEBUG`: 20 requests produce no `api.log` growth, and errors still land as WARN/ERROR.
- With `APP_DEBUG=true` in `private/.env`: logging comes back.
- `grep -rn "test_mode\|\['debug'\]" .` finds only the new line.

**Commit:** `security: debug logging off by default, single rotating logger`

---

### S11. Config falls back to the public docroot when `private/` is missing
**Where:** `config.php` l.54–79.

**Problem (LOCKED):** `config.php` loads `.env` from the docroot and from its parent, and **if `private/` doesn't exist** it creates the SQLite DB and the logs inside the docroot. A missing folder silently turns into a public data file.

**Fix:**
```php
$privateDir = dirname(__DIR__) . '/private';
loadDotEnv($privateDir);
$CONFIG['db_path'] = $privateDir . '/userdata.db';
$CONFIG['log_dir'] = $privateDir . '/logs';
```
Delete the other `loadDotEnv` calls and both fallbacks. In `db()` (P3), before `new PDO`:
```php
if (!is_dir(dirname($CONFIG['db_path']))) {
    error_log('ssapi: private/ directory missing');
    respond(['valid' => false, 'error' => 'Server misconfigured'], 500);
}
```
**Dave on el1, before deploying:**
- Confirm every host keeps `JWT_SECRET` and the VAPID keys in `<domain>/private/.env`, not in the docroot.
- After deploying, old `logs/` folders inside docroots can be deleted (his call).

**Verify (bench):**
- Rename `$BENCH/private` → any request returns `500 Server misconfigured`, and **no** `userdata.db` or `logs/` appears under `$BENCH/site`. Rename it back.
- Logs now appear in `$BENCH/private/logs` (with `APP_DEBUG=true`).

**Commit:** `security: config fails closed outside private/`

---

### S12. No security headers, while tokens sit in localStorage
**Where:** `api.php` and `media.php` → `respond()`; `ssreact/public/.htaccess`.

**Problem (LOCKED):**
- Both tokens live in `localStorage`, so any script that ever ran on the origin could read them.
- React escaping and the http/https-only link rendering make that unlikely today (see §9), but there's no second line of defence. There's no CSP, no `nosniff` and no frame protection.

**Fix:**
1. In both `respond()` functions, add:
   ```php
   header('X-Content-Type-Options: nosniff');
   header('Cache-Control: no-store');
   ```
2. In `ssreact/public/.htaccess`, add (start with **Report-Only**):
   ```apache
   <IfModule mod_headers.c>
     Header always set X-Content-Type-Options "nosniff"
     Header always set X-Frame-Options "DENY"
     Header always set Referrer-Policy "strict-origin-when-cross-origin"
     Header always set Permissions-Policy "camera=(self), microphone=(self), geolocation=()"
     Header always set Content-Security-Policy-Report-Only "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; media-src 'self' blob:; font-src 'self' data:; connect-src 'self'; worker-src 'self'; manifest-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'"
   </IfModule>
   ```
   `'unsafe-inline'` for styles is needed because React `style={}` props and the shadcn components set inline styles. Scripts stay strict.
3. The real fix for token theft is D4 (gated).

**Verify:**
- Temporarily copy the CSP into `vite.config.ts` under `preview: { headers: { 'Content-Security-Policy': '…' } }`, then `pnpm build && pnpm preview`.
- Click through every page (feed, post, create with image and capture modal, settings, API docs, download). The DevTools console must show **no** CSP violations.
- Remove the preview header before committing.
- **Dave:** after a week on react.davidfruin.com with no reports, rename the header to `Content-Security-Policy` to enforce it.

**Commit:** `security: security headers + report-only CSP`

---

### S13. Deletes aren't atomic, orphan data is left behind, and media can be attached to two posts
**Where:** `src/Posts/handlers.php` → `handle_deletePost` l.376, `handle_post` l.169–174; `api.php` → `handle_deleteAccount` l.280–334 (the `rmdir` chain at l.328).

**Problem (LOCKED):**
- Multi-step deletes run without a transaction, so a failure halfway through leaves partial state.
- `deletePost` leaves the post's **comments** behind.
- `handle_post` accepts a `mediaUrl` that is already attached to another post. Deleting either post then deletes the shared file.
- The `@rmdir(a) && @rmdir(b) && …` chain stops at the first type folder that doesn't exist, so a user's media folder is never removed.

**Fix:**
- In `handle_post`, add the condition `AND post_id IS NULL` to the media ownership `SELECT`.
- In `handle_deletePost`, wrap the DB deletes in `$pdo->beginTransaction()` / `commit()` (and `rollBack()` in a catch that rethrows). Add `DELETE FROM comments WHERE post_id = ?`. Run the `unlink()` calls **after** the commit.
- In `handle_deleteAccount`:
  - wrap all DB statements in one transaction;
  - also delete comments **on** the user's posts and notifications that reference those posts;
  - collect the file paths during the transaction and unlink them after the commit;
  - replace the `rmdir` chain with:
    ```php
    foreach (['image', 'video', 'audio'] as $t) if (is_dir("$mediaDir/$t")) @rmdir("$mediaDir/$t");
    if (is_dir($mediaDir)) @rmdir($mediaDir);
    ```

**Verify (bench):**
1. Bob comments on alice's post, and alice deletes the post → `SELECT COUNT(*) FROM comments WHERE post_id=…` is 0.
2. Post twice with the same `mediaUrl` → the second returns 400.
3. Delete an account that uploaded only an image → `media/<uid>` is gone.
4. Throw an exception halfway through (temporary `throw new Exception('x');`) → the row counts are unchanged.

**Commit:** `fix: transactional post/account deletion, no shared media, folder cleanup`

---

### S14. Registration: SQL that depends on a legacy setting, and a race that can create duplicates
**Where:** `src/Auth/handlers.php` → `handle_finishRegister` l.334–356.

**Problem (LOCKED):**
- The `INSERT` uses `"[]"` inside SQL. Those are **double-quoted identifiers** that only work because of SQLite's legacy fallback to treating them as strings, and builds compiled without that fallback reject them.
- Two concurrent `finishRegister` calls for the same email can both pass the checks and create duplicate users. There's no unique index on email.

**Fix:**
```php
$pdo->exec('BEGIN IMMEDIATE');
try {
    $s = $pdo->prepare('SELECT 1 FROM users WHERE LOWER(email) = LOWER(?)');
    $s->execute([$email]);
    if ($s->fetchColumn()) { $pdo->exec('ROLLBACK'); bad('Email already registered', 400); }
    $pdo->prepare("INSERT INTO users (email, password, posts, follows, followers, jwt, created_at)
        VALUES (?, ?, '[]', '[]', '[]', '', ?)")->execute([$email, $hashed, $created_at]);
    $pdo->prepare('DELETE FROM pending_users WHERE email = ?')->execute([$email]);
    $pdo->exec('COMMIT');
} catch (Throwable $e) { $pdo->exec('ROLLBACK'); throw $e; }
```
`bad()` exits after rolling back, so the catch isn't reached on that path.

Also add `CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email_nocase ON users(email COLLATE NOCASE)` to `migration1`. **First** check for existing duplicates:
```sql
SELECT LOWER(email), COUNT(*) FROM users GROUP BY 1 HAVING COUNT(*) > 1
```
If any exist, skip the index, record that, and tell Dave (he has a copy of dev's DB to check against).

**Verify (bench):**
- A full register flow (read the code from `mail.log`) → 200.
- Repeat `finishRegister` → 400.
- Run two concurrent finishRegister calls on a fresh pending email, each with a valid verified OTP, using `&`. Exactly one user is created.

**Commit:** `fix: atomic registration, standard SQL string literals, unique email`

---

### S15 [ssreact]. 15 UI components import `cn` from an unrelated npm package
**Where:** `src/components/ui/*.tsx` (15 files: `import { cn } from "cn"`); `package.json` (`"cn": "^0.4.0"`). `components.json` maps `utils` to `@/lib/utils`.

**Problem (LOCKED):**
- shadcn's generator was supposed to import `cn` from `@/lib/utils` (clsx plus tailwind-merge).
- Instead, 15 components import from the npm package `cn@0.4.0`. Its lock entry marks it `hasBin: true` and requires Node 20 or later, which suggests a CLI tool, not this helper.
- So: either class merging behaves differently from what shadcn expects (conflicting Tailwind classes aren't resolved), or the app is pulling in an unrelated package.

**Fix:**
1. First look at what it is: `cat node_modules/cn/package.json` and its entry file. Record the result in the commit message.
2. Replace `from "cn"` with `from "@/lib/utils"` in all 15 files: `grep -rl 'from "cn"' src | xargs sed -i 's#from "cn"#from "@/lib/utils"#'`
3. `pnpm remove cn`.

**Verify:**
- `pnpm lint && pnpm build` pass.
- `grep -rn 'from "cn"' src` returns nothing.
- Visually spot-check Button, Card, Dialog, Select and Switch in two themes. Expect no regressions, or *fixed* class conflicts.

**Commit:** `fix(ssreact): use local cn() helper, drop unrelated cn package`

---

### S16. Password maximum is 25 characters
**Where:** `src/Auth/handlers.php` → `validatePasswordRules` l.75.

**Problem:** a 25-character cap blocks password-manager passwords. bcrypt only reads the first 72 bytes, so cap at 72 instead.

**Fix:** change the length check to `strlen($password) < 8 || strlen($password) > 72` and the message to `'Password must be 8-72 characters'`. Check whether ssreact (`OtpAuthFlow.tsx`) or the C clients hard-code 25 for validation or `maxLength`, and update [ssreact] if so. Note the C clients in the hand-off; their repos are out of scope.

**Verify:** register with a 40-character password, then log in with it → 200.

**Commit:** `fix: allow passwords up to bcrypt's 72-byte limit`

---

## 6. Tier 4: Payload size and frontend load

### P7 [ssreact]. Caching and compression; stop revalidating hashed assets
**Where:** `ssreact/public/.htaccess` (add after S2's content); `ssreact/public/sw.js` → `isAppCode` l.34–39.

**Problem:**
- There are no cache headers and no compression.
- `sw.js` forces `no-cache` revalidation on **every** same-origin `.js`/`.css` request, including Vite's content-hashed `/assets/*` files, which never change under the same URL.
- Media filenames are unique per upload, so they never change either.

**Fix (`.htaccess`):**
```apache
<IfModule mod_headers.c>
  <If "%{REQUEST_URI} =~ m#^/(assets|media|pwa-icons)/#">
    Header set Cache-Control "public, max-age=31536000, immutable"
  </If>
  <FilesMatch "^(index\.html|sw\.js|manifest\.json)$">
    Header set Cache-Control "no-cache"
  </FilesMatch>
</IfModule>
<IfModule mod_deflate.c>
  AddOutputFilterByType DEFLATE text/html text/css application/javascript application/json image/svg+xml
</IfModule>
```
**Fix (`sw.js`):**
```js
return /\.(js|css)$/i.test(url.pathname) && !url.pathname.startsWith('/assets/');
```
Navigations, and therefore `index.html`, stay network-first. That keeps the stale-JS incident fix from [[simple-social]]'s ARCHITECTURE.md intact.

**Verify:**
- `pnpm build`, then confirm `dist/.htaccess` and `dist/sw.js` contain the changes.
- **Dave on el1** after deploying: `curl -sI https://react.davidfruin.com/assets/<a js file>` shows `immutable`. `curl -sI -H 'Accept-Encoding: gzip' …` shows `Content-Encoding: gzip`.
- A reload shows `/assets/*` served from the browser cache with no 304 round-trips.

**Commit:** `perf(ssreact): long-cache hashed assets/media, compress, skip SW revalidation of hashed files`

---

### P8 [ssreact]. Everything ships in one bundle
**Where:** `src/App.tsx` l.10–25 (all pages imported eagerly).

**Fix:**
- Load rarely used or heavy routes with `React.lazy` and one `<Suspense>` around `<Routes>`. Candidates: `ApiDocsPage` (it inlines about 800 lines of HTML through `?raw`), `RoadmapPage`, `DownloadPage`, `AboutPage`, `ConductPage`, `SettingsPage`, and `CreatePostPage` (which pulls in `CaptureModal`).
  ```tsx
  const ApiDocsPage = lazy(() => import('@/pages/ApiDocsPage').then((m) => ({ default: m.ApiDocsPage })));
  ```
- A chunk can 404 after a deploy if a tab is still running the old bundle. Add one recovery line to `main.tsx`:
  ```ts
  window.addEventListener('vite:preloadError', () => window.location.reload());
  ```

**Verify:**
- Record the `pnpm build` output (JS sizes) before and after, and put both in the commit message. The main chunk should shrink noticeably.
- Each lazy page still loads, including on a direct visit and after a refresh.
- `pnpm lint` is clean.

**Commit:** `perf(ssreact): lazy-load rare routes`

---

### P9. Redundant queries on hot paths
**Where:**
- `api.php` → `requireAuth` l.128–130, which looks up the email once more per request after `verifyUser`
- `handle_likePost`, `handle_unlikePost`, `handle_createComment`, `handle_followUser`, `handle_unfollowUser`, which each fetch the caller's email again
- `handle_getPostById`, which looks up the owner in a separate query

**Fix:**
- In `sessionLookup` (`auth.php` l.181), change the query to `SELECT s.*, u.email AS user_email FROM sessions s JOIN users u ON u.id = s.user_id WHERE s.id = ?`. In `verifyUser`, set `$payload['email'] = $session['user_email'];`.
- Delete the extra `SELECT` from both `requireAuth()` functions.
- Use `$user['email']` in the five handlers.
- In `getPostById`, `JOIN users` for the owner email.

This is roughly one query fewer on every authenticated request, and two fewer on likes and follows.

**Verify (bench):** the same `diff` of before/after JSON as P4 for `getMyInfo`, `fetchFollowedPosts`, `getPostById`, a like, a comment and a follow. Notifications still show the right `actor_email`.

**Commit:** `perf: carry email through session lookup, drop duplicate queries`

---

## 7. Tier 5: Structural (each GATED, needs Dave's OK)

- **D1. One versioned schema for everything.** P3 creates the mechanism and adds `users`/`pending_users` to `migration1` (not gated). The gated part is fixing the problems listed in [[simple-social]]'s "Open — database" section:
  - `pending_users` has no primary key;
  - `follows`/`followers` are typed `NUMERIC` but hold JSON;
  - three different date formats;
  - no foreign keys.

  Do each one as its own numbered migration with a dry run on a copy of dev's DB.
- **D2. Move follows from JSON into a real table.** Today `getMyFollowers` (Follows l.13) **loads every user and decodes every JSON array** on each profile view. Follow and unfollow are also read-modify-write with lost-update races, and `deleteAccount` rewrites every user row.
  - Use the same playbook as the posts migration: a `follows(follower_id, followee_id, created_at, PRIMARY KEY(follower_id, followee_id))` table with an index on `followee_id`, and a `migrate-follows.php --dry-run` script run once on dev, then prod.
  - **Dual-write the JSON** until D6 is resolved, because simple-social's backend copy still reads it.
  - The API is unchanged.
- **D3. One bootstrap for both entry points.** Move `respond`/`bad`/`good`/`logMsg`/`requireAuth` into `bootstrap.php` and have `api.php` and `media.php` require it. That removes the "same names, different implementations" trap described in `src/Media/handlers.php`'s header. P3 already moves `db()`.
- **D4 (BREAKING, web only). Move the refresh token into an HttpOnly cookie, with rotation.**
  - When the web client asks for it, login sets `ss_refresh` as a `HttpOnly; Secure; SameSite=Strict; Path=/api.php` cookie.
  - `refreshToken` reads the cookie when there's no body parameter, rotates the token on every use, and revokes the whole session if an old token is presented again.
  - The access token lives only in memory in ssreact.
  - The CLI/TUI keep the body-parameter flow unchanged.
  - **Affects:** ssreact, the vanilla web app (only if it opts in), and ssreact-native (later).
- **D5. DECIDED 2026-10-02, superseded by `Inbox/deploy-layout-plan.md`** (backend in `<domain>/ssapi/`, frontend in `public_html/app/`, one root `.htaccess` from `ssapi/deploy/`). Original text: **Move non-entry PHP, `vendor/` and logs out of the docroot.** The docroot keeps only `api.php`, `media.php`, `index.maintenance.php` and the frontend, and those `require` `../app/…`. This is a deploy-layout change on el1, and it makes S2's deny rules a second line of defence instead of the only one.
- **D6. The duplicate backend in simple-social.** Prod gets **none** of these fixes until either:
  - (a) ssapi is deployed to app and dev in place of simple-social's copy, or
  - (b) the fixes are mirrored there.

  To see the current drift, run `diff -r --exclude=.git --exclude=README.md <ssapi> <simple-social>` on the backend file list.
- **D7. Drop the dead columns** `users.posts`, `users.followers`, `users.jwt` and `users.is_admin`.
  - This needs SQLite ≥ 3.35 for `DROP COLUMN`. Check with `SELECT sqlite_version()` on each host.
  - Take a `.backup` first.
  - `finishRegister`'s `INSERT` must drop those columns in the same change.
- **D8 (BREAKING). Replace the per-post `likes` arrays with `likeCount` + `likedByMe`.** A post with 200 likes currently sends 200 objects on every feed load.
  - Make it additive first (send both). Remove `likes` only after every client has switched.
  - **Affects:** every client, and the Likes popover would need `getPostLikes`, which it already calls.

---

## 8. Tier 6: Cleanup (low risk, do last)

- **C1.** Delete `clean-notifications.php` and `migrate-posts.php`. Both are one-off scripts that have already been run on dev and prod (see [[simple-social]] Done). Remove their lines from `.htaccess` and the README. Git history keeps them.
- **C2.** Remove the comment debris:
  - about 80 lines of "X moved to src/…" breadcrumbs in `api.php` (l.118, l.214–217, l.247–248, l.273–277, l.336–372);
  - the stale "Not read or written by api.php yet" paragraph in `schema.php` (l.73–78);
  - module header comments that point to `.claude/commands/split-backend-modules.md`, which isn't in this repo;
  - the `// api.php - … (max 3 levels indentation)` header.
- **C3.** Remove the unused logging functions (done as part of S10) and `jwtClaimsUnverified` (S6).
- **C4.** `handle_getMyPosts` duplicates `handle_getUserPosts`. Make it `$_POST['userId'] = $user['sub']; handle_getUserPosts($pdo, $user);`, or share one helper. The response is identical.
- **C5.** Tidy the config files:
  - in `.gitignore`, drop the `node_modules/`, `tests/front-end-test/…` and `docs/` leftovers from simple-social;
  - in `composer.json`, set `"name"` to `davidfruin/ssapi`;
  - in `.env.example`, add `VAPID_PUBLIC_KEY=`, `VAPID_PRIVATE_KEY=`, `VAPID_SUBJECT=` and `APP_DEBUG=false`. Note that `vapid_subject` is read in `pushNotification` but never loaded from the environment, so also add `$CONFIG['vapid_subject'] = getenv('VAPID_SUBJECT') ?: null;` in `config.php`.
- **C6 [ssreact].** Make the dev proxy target configurable. In `vite.config.ts`:
  ```ts
  const target = process.env.SS_API_TARGET ?? 'https://dev.davidfruin.com';
  ```
  Use `target` for both proxy entries. The default is unchanged, and the "never prod" comment stays.
- **Leave alone, and know about:**
  - All timestamps are server-local strings in three formats (D1).
  - `api.php` re-parses the raw body into `$_POST` (l.19–26). That is redundant for form-encoded requests, but harmless, and some client may depend on it.
  - The GD image pipeline turns animated GIFs into a still WebP. That's a product decision, not a bug.

---

## 9. Checked and fine (don't "fix" these)

- **SQL injection:** every query uses bound parameters. The interpolated `LIMIT`/`OFFSET` values are integer-cast first, and the `IN (…)` lists are built from placeholders only.
- **JWT:** HS256 is computed server-side, compared with `hash_equals`, and the `alg` header is never trusted. Expiry is enforced. Every request also checks the session row, so revocation is immediate.
- **Passwords:** `password_hash` / `password_verify` (bcrypt). Login and code checks are rate limited per email and per IP. The refresh token is stored only as a SHA-256 hash.
- **Ownership checks** exist on `deletePost`, `deleteComment`, `deleteMedia`, `revokeSession`, `deletePushSubscription`, and media attachment in `post` (tightened by S13).
- **XSS in ssreact:**
  - Post and comment text renders through React text nodes.
  - Auto-links only match `https?://`, so `javascript:` URLs can't become links.
  - Mentions link to internal routes only.
  - `dangerouslySetInnerHTML` is used once, on the bundled, static `api-docs.html`.
- **CSRF:** not applicable today, because auth is a `Bearer` header, not a cookie. Revisit if D4 lands; `SameSite=Strict` covers it.
- **Maintenance mode** (`index.maintenance.php` plus the rewrite rule) is fine. Keep the rule first in any `.htaccess` rewrite.
- **The single-flight refresh and the re-login waiter queue** in ssreact are correct and deliberate. Leave them alone.

---

## 10. Open decisions for Dave

1. **S9:** drop unlike/unfollow notifications (A) or deduplicate them (B).
2. **D6:** how and when ssapi replaces simple-social's backend copy on app and dev. Until then, prod has none of these fixes.
3. **D4:** move the web client's refresh token to an HttpOnly cookie (breaking for web clients).
4. **D2:** timing of the follows-table migration. **D7/D8:** dropping dead columns, and the shape of the likes payload.
5. **P3b:** whether WAL mode is worth switching DB copies and backups to `.backup`.
6. **Infra:** ffmpeg/ffprobe inside react.davidfruin.com's chroot. Without them, videos have no thumbnail, no transcode, and no server-side duration limit. S3 makes the kept originals safe but doesn't fix that.
7. **Existing open item:** a retention period for the API log. S10 makes it rotate and defaults it off, but doesn't settle retention.
8. **S2 and the `.htaccess` allowlist:** the server checks marked "Dave on el1".

---

## 11. Suggested order (stop after each phase)

| Phase | Tasks | Why this order |
|---|---|---|
| 0 | Bench, C6 | Everything else is verified on it |
| 1 | P1, S1, S5, S7, S14, S16 | Small, high-value, independent |
| 2 | S3, S4, S6, S8, S9 (bug part only) | The rest of the input and auth hardening |
| 3 | P3 → P4 → P5 → P9 → P2, P6 | Schema first, then the queries that rely on it; push deferral last because it touches `respond()` |
| 4 | S10, S11, S13, S15 | Config, logging and integrity |
| 5 | S2 + S12 + P7 (one combined `.htaccess` pass), P8 | All of these touch the deploy-facing ssreact files, so Dave reviews them together and deploys once |
| 6 | C1–C5 | Cleanup |
| Gated | S9 option, P3b, D1–D8 | Only on Dave's word |

When a phase is done, give Dave in the hand-off: the commit list, which checks you ran, and every "Dave on el1" check still outstanding.
