---
status: approved (decisions made 2026-10-07)
written: 2026-10-07
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi @ e3304e2 (media handler), ssreact (web/ + mobile/ client tasks marked [ssreact]), sstests (media test harness)
---

# Media pipeline plan: safer, faster, more reliable uploads

A review of how [[ssapi]] handles uploaded images, video and audio (`src/Media/handlers.php`, `media.php`, GD and ffmpeg), as of `e3304e2` "Accept any image, video or audio format…". **Every finding marked TESTED was reproduced on 2026-10-07** by running the real handler functions with GD 2.3.3 and ffmpeg 6.1 (§2 shows how).

**What's already good (keep it):**
- Files are identified from their bytes (`sniffMedia` + ffprobe container allow-list), never from the client's Content-Type or filename.
- ffmpeg/ffprobe only read plain local files (`-protocol_whitelist file`, `file:` prefix), so crafted playlists can't make the server fetch URLs or read other files.
- Staging happens in `private/tmp` (0700), outside the web root.
- `timeout` wraps ffmpeg and ffprobe, and output is capped with `-t media_max_seconds`.
- GD re-encoding strips photo EXIF/GPS.
- No ImageMagick.
- `/media/` refuses script/HTML extensions and sends `nosniff`.
- Failed conversions reject the upload instead of keeping an unplayable original.

---

## 0. Rules for the implementing agent
- The usual rules apply: one task per commit, stop after each phase and report, never deploy, never touch prod. Dave deploys. Server checks are marked **Dave on el1** (or an agent he gives `el1` access to).
- Check `Areas/active-work.md` first, and claim a row for ssapi media work. **Phone-app work in ssreact `mobile/` is active**; the [ssreact] tasks here touch `mobile/` media code, so coordinate (do them last, or ask Dave).
- **Tests don't go in ssapi** ([[sstests]] rule). The media harness and fixtures (§2) go in `sstests/backend/media/`.
- Keep the TESTED reproductions as regression checks: after each fix, rerun the matching fixture and record the before/after in the commit message.
- The vault is public: never paste real users' media, names or locations anywhere.

---

## 1. Findings at a glance

| ID | Finding | Kind | Severity |
|---|---|---|---|
| M1 | **Decompression bomb.** A 777 KB PNG claiming 16000×16000 pixels made one PHP process use **1.7 GB RAM for 9 s**. PHP's `memory_limit` (128 MB) never triggered, because GD allocates outside PHP's accounting. A few at once could exhaust the server. ffmpeg-decoded images and video have the same exposure. TESTED. | Safety | **High** |
| M2 | **GPS and other metadata survive video conversion.** An iPhone-style `.mov` with `location=+37.33-122.00` and a comment produced a public MP4 that still contains both. TESTED. | Privacy | **High** |
| M3 | **MP3s are passed through untouched:** tags (artist, comment), an embedded cover image, and the full length. A 30 s MP3 came out 30 s long; it's only rejected earlier if its header reports the duration truthfully. TESTED. | Privacy/limits | Medium |
| M4 | **Broad decoder attack surface:** any stream inside an allowed container (ASF, FLV, MPEG-TS, AVI, …) goes to ffmpeg's decoders, which have a long CVE history. | Safety | Medium |
| M5 | **No resource limits:** ffmpeg uses every core, there's no concurrency cap, and conversion runs in the request (about 6 s for a 10 s 1080p60 clip on 4 cores; slower on the shared host). A few uploads at once can starve the site or hit FastCGI timeouts. | Safety/speed | Medium |
| M6 | **Half-written public files and leftovers:** ffmpeg writes straight into the public `media/` folder while encoding, and a fatal error leaves staged files in `private/tmp`. | Reliability | Low-medium |
| M7 | **No upload rate limit, quota or orphan cleanup:** media uploaded but never attached to a post stays forever (abandoned drafts, crashes), and nothing stops disk-filling. | Safety/ops | Medium |
| M8 | **`deleteMedia` deletes media still attached to a post**, leaving a broken post. | Reliability | Low |
| M9 | **Log injection:** the uploaded filename goes into the log unsanitised, so a filename containing a newline can fake log lines. | Safety | Low |
| F1 | ffprobe runs **three times** per upload (classify, duration, fps). | Speed | Low (about 0.1 s) |
| F2 | **Feed images are full 1920 px**, with no smaller copy and no stored width/height, so heavy phone data use and layout jumps while images load. | Speed/UX | Medium |
| F3 | **Video poster not exposed:** the server makes `thumb_*.webp`, but post data doesn't include its URL, so clients decode the video to show a frame. The poster is also the first frame, often black. | Speed/UX | Low-medium |
| R1 | **Transparent WebP/GIF become solid black** when resized (alpha only preserved for PNG). TESTED. | Reliability | Medium |
| R2 | **iPhone HEIC photos:** ffmpeg before 7.1 decodes only the first tile of a multi-tile HEIC, giving a cropped piece. Dev and prod run 7.1 (per the [[ssapi]] note), so the server is probably fine, but no real HEIC has been tested yet. The phone app picks library photos at `quality: 1`, possibly HEIC. | Reliability | Medium (test first) |
| R3 | **If ffmpeg is missing or broken, every video/audio upload now fails** (originals are no longer kept), with only a generic error. | Reliability | Medium |
| R4 | **HDR iPhone videos come out washed out:** no tone-mapping to standard colour. | Quality | Low-medium |
| R5 | **The web file picker still lists the old narrow MIME set**, while the server now accepts any media. | Consistency | Low |
| R6 | **Ops gaps:** `media/` backups; PHP upload limits and FastCGI request size/timeouts must fit the 100 MB / 10 s limits. | Ops | Check |

---

## 2. Phase 0: the media test harness (agent, about an hour)
**Done 2026-10-07, sstests commit 23f392e** (A5 fixtures added in e8b5776).

Create `sstests/backend/media/` with three files:

**`make-fixtures.sh`** generates every test file locally. It uses ffmpeg only, with no real photos:
```bash
#!/usr/bin/env bash
set -euo pipefail
D="${1:-./fixtures}"; mkdir -p "$D"
ffmpeg -loglevel error -y -f lavfi -i color=c=white:s=16000x16000 -frames:v 1 -compression_level 9 "$D/bomb.png"           # M1
ffmpeg -loglevel error -y -f lavfi -i testsrc=s=1280x720:r=30 -f lavfi -i sine=f=440 -t 3 -c:v libx264 -c:a aac \
  -metadata location="+37.3349-122.0090/" -metadata comment="secret-comment" -movflags use_metadata_tags "$D/phone.mov" # M2
ffmpeg -loglevel error -y -f lavfi -i "color=c=red@0.0:s=3000x2000,format=rgba" -frames:v 1 -c:v libwebp -lossless 1 "$D/alpha.webp" # R1
ffmpeg -loglevel error -y -f lavfi -i sine=f=300 -i "$D/alpha.webp" -t 30 -map 0:a -map 1:v -c:a libmp3lame -c:v mjpeg \
  -disposition:v attached_pic -metadata artist="Some Person" -metadata comment="private note" "$D/tagged.mp3"           # M3
ffmpeg -loglevel error -y -f lavfi -i testsrc2=s=1920x1080:r=60 -f lavfi -i sine=f=440 -t 10 -c:v libx264 -preset ultrafast \
  -b:v 20M -c:a aac "$D/clip.mp4"                                                                                         # M5/F1 timing
ffmpeg -loglevel error -y -f lavfi -i testsrc2=s=1280x720:r=30 -t 3 -c:v libx264 -color_trc arib-std-b67 \
  -color_primaries bt2020 -colorspace bt2020nc "$D/hdr-tagged.mp4"                                                         # R4 path selection
ffmpeg -loglevel error -y -f lavfi -i testsrc=s=320x240:r=10 -t 2 "$D/anim.gif"                                            # GIF path
```

**`harness.php`** loads the real handler functions with a stub environment and runs one step:
```php
<?php
// php harness.php <classify|image|video|audio> <input> [outputBase]
$CONFIG = ['media_max_side' => 1920, 'media_max_fps' => 60, 'media_max_seconds' => 10,
           'media_max_pixels' => 50000000, 'media_max_dimension' => 12000];
function logMsg($m) { fwrite(STDERR, "$m\n"); }
require getenv('SSAPI') . '/src/Media/handlers.php';
[$_, $what, $in, $out] = array_pad($argv, 4, '');
$t = microtime(true);
$r = match ($what) {
  'classify' => classifyMedia($in), 'image' => processImage($in, $out),
  'video' => processVideo($in, $out, "$out.thumb.webp"), 'audio' => processAudio($in, $out),
};
echo json_encode($r), sprintf("  %.2fs\n", microtime(true) - $t);
```
Adjust the calls if signatures change (e.g. M3 removes `processAudio`'s third argument).

**`run.sh`** generates the fixtures, then runs each check below and prints PASS/FAIL. It measures peak memory by sampling `/proc/<pid>/status` VmRSS while the PHP process runs (GNU `time` may be missing).
- **M1:** `bomb.png` is rejected; peak RSS stays under 200 MB.
- **M2:** `phone.mov` → MP4 whose `ffprobe -show_entries format_tags` has **no** `location` or `comment`.
- **M3:** `tagged.mp3` → output has no artist/comment tags, no video (cover) stream, and duration ≤ 10.1 s.
- **R1:** `alpha.webp` → output pixel (10,10) is transparent (alpha 127).
- **R4:** `hdr-tagged.mp4` → the code picks the HDR path (log line or return flag).
- **Timing:** report seconds for `clip.mp4` before/after (informational).

Run with `SSAPI=/path/to/ssapi ./run.sh`. Before any fixes, M1, M2, M3 and R1 must **FAIL**; that proves the checks detect the problems.

**Commit (sstests):** `media: fixture generator + harness reproducing M1/M2/M3/R1`

---

## 3. Phase A: urgent safety fixes (ssapi, about a day)
**Done 2026-10-07** (local bench only, not deployed): A1 54ea259, A5 4a0dee9, A2 43d6767, A3 021c236, A4 b382855. Notes and gaps in [[ssapi]].


### A1 (M1). Pixel limits before any decode
Add to `config.php`'s `$CONFIG`:
```php
'media_max_pixels' => 50_000_000,   // ~50 MP: covers 48 MP phone photos; refuses decompression bombs
'media_max_dimension' => 12000,     // longest side, px
```
1. **Images GD reads** (jpg/png/gif/webp): `sniffMedia` already calls `getimagesize()`, which only reads headers and doesn't decode pixels. Return the width and height with the result. In `classifyMedia`, reject if `w*h > media_max_pixels` or `max(w,h) > media_max_dimension`.
2. **Everything ffmpeg decodes** (video, and HEIC/AVIF/BMP/TIFF images): add `width,height` to `probeMedia`'s `-show_entries` (it's `stream=codec_type,codec_name,width,height,...`). Reject if any video stream is over the limits. For `img`-type images, also probe them (they currently skip ffprobe), using the same function.
3. **Belt and braces inside ffmpeg:** every ffmpeg **input** gets `-max_pixels <media_max_pixels>`. TESTED: ffmpeg 6.1 rejects the 16000×16000 PNG with "exceeds specified max pixel count". Restructure `runFfmpeg` so callers pass the input path and the output args separately, and the function always prepends the safe input options:
   ```php
   function runFfmpeg(string $inputPath, array $outputArgs): bool {
       global $CONFIG;
       $in = ['-max_pixels', (string)$CONFIG['media_max_pixels'], '-i', 'file:' . $inputPath];
       // ... build: nice/prlimit/timeout prefix (A5) + ffmpeg -hide_banner -loglevel error -nostdin -y
       //     -protocol_whitelist file + $in + $outputArgs
   }
   ```
4. User-facing error: "That image is too large (max about 50 megapixels)."

**Verify:** the harness M1 check passes (rejected quickly, peak RSS < 200 MB). A normal 4032×3024 JPEG still works.

### A2 (M2). Strip metadata from all ffmpeg output
Add `-map_metadata -1 -map_chapters -1` to the output args of `processVideo`, `processAudio`, `createVideoThumbnail` and `convertImageToPng`. TESTED: this removes `location` and `comment`. Keep the rotation working: ffmpeg applies the display-matrix rotation while decoding (autorotate is on by default), so the pixels are upright before the metadata is dropped. Confirm with a portrait phone clip.

**Verify:** harness M2 passes. Also run `ffprobe -show_entries stream_tags -of default=nw=1` on the output: no `location`, `creation_time` is acceptable, no `com.apple.*` keys.

### A3 (M3). Always re-encode audio
Delete the "already MP3 → copy" shortcut:
- `processAudio($inputPath, $outputBase)` always runs `-vn -t <max> -map_metadata -1 -c:a libmp3lame -q:a 2`.
- `classifyMedia` no longer returns `ext => 'mp3'`.
- Re-encoding 10 s of audio takes well under a second.

**Verify:** harness M3 passes (no tags, no cover stream, ≤ 10.1 s).

### A4 (M9). Sanitize log lines globally
In `logging.php` → `writeLog()`, before building the entry, replace control characters in `$message` and in each context value:
```php
$clean = fn($v) => preg_replace('/[\x00-\x1F\x7F]+/', ' ', (string)$v);
```
This protects every log line, not just uploads.

**Verify:** upload a file named `"a\nFAKE LINE.png"` on the bench (`curl -F 'file=@x.png;filename="a\nFAKE LINE.png"'`). `media.log` shows it on one line.

### A5 (M4). Accept only the formats phones, browsers and normal apps produce
**Chosen 2026-10-07** (Dave left the choice to the reviewer). Keep every format a family member could plausibly upload, and drop old desktop/broadcast formats. Those have the most complex demuxers and decoders and the longest CVE history, and nobody records in them any more.

| | Keep | Drop |
|---|---|---|
| Containers (`PROBE_ALLOWED_FORMATS`) | `mov,mp4,m4a,3gp,3g2,mj2` (iPhone, Android, most apps), `matroska,webm` (browser recordings, Android), `ogg` (Opus/Vorbis voice notes), `wav`, `mp3`, `flac`, `aac`, `aiff`, `caf` (iOS), `amr` (Android voice recorders) | `avi`, `flv`, `asf` (WMV/WMA), `mpegts`, `mpeg`, `mpegvideo` |
| Images | JPEG, PNG, GIF, WebP (GD); HEIC/HEIF, AVIF, BMP (ffmpeg) | TIFF, ICO, anything else |

1. Remove the dropped names from `PROBE_ALLOWED_FORMATS`, and remove their signature checks from `sniffMedia` (ASF, FLV, MPEG-PS, MPEG-TS, the `AVI ` RIFF branch). Rejecting at the sniff means ffprobe never parses them either. In the image branch, only send HEIC/HEIF/AVIF brands and BMP (`BM` header) to ffmpeg; TIFF and ICO return null.
2. **Codec allow-list:** a trusted container can still hold any codec, so check the probe's streams and reject unknown ones:
   - video: `h264`, `hevc`, `vp8`, `vp9`, `av1`, `mpeg4`, `h263`;
   - audio: `aac`, `mp3`, `opus`, `vorbis`, `flac`, `alac`, `amr_nb`, `amr_wb`, `pcm_*`;
   - images through ffmpeg: `hevc`, `av1`, `bmp`;
   - attached cover pictures (`mjpeg`, `png`) are allowed only as `attached_pic` streams; A2/A3 drop them anyway.

   Also pass the matching list to ffmpeg as `-codec_whitelist` on the input in `runFfmpeg` (A1), so the decoder itself refuses anything else. Check that the M3 `tagged.mp3` fixture still converts, since cover streams must not break it.
3. **Message** for anything rejected: "That file type isn't supported. Try a photo (JPEG, PNG, HEIC, WebP, GIF), a video (MP4, MOV, WebM) or audio (MP3, M4A, WAV, Ogg)."
4. **Update the [[ssapi]] note's "accepts any format" section** to record the narrowed list and why.

**Verify:** the harness gains fixtures made with ffmpeg: `x.avi`, `x.flv`, `x.ts`, `x.wmv` and `x.tiff` are all rejected; the M1–M3 fixtures, a 3GP with AMR audio, an Opus `.ogg` and a WebM all still convert.

**Commits:**
- `media: refuse oversized images/videos before decoding (pixel limits + ffmpeg -max_pixels)`
- `media: accept only common phone/browser formats; codec allow-list`
- `media: strip metadata (GPS, comments) from converted video/audio`
- `media: always re-encode audio`
- `logging: strip control characters from log lines`

---

## 4. Phase B: resource limits and hygiene (ssapi, about a day)
**Done 2026-10-07** (local bench only, not deployed): B4 fcdde8a, B1 cc6b125, B2 4eba368, B5 1b732f4, B3 4cc8921 (ssapi); clients ssreact ef01eeb. Notes in [[ssapi]].


### B1 (M5). Bound CPU, memory and concurrency
1. **Command prefix for ffmpeg and ffprobe** (build it once in a helper, `resourcePrefix($seconds)`):
   ```
   nice -n 10 prlimit --as=<bytes> -- timeout <seconds> ffmpeg ...
   ```
   - Use each tool only if it's executable, as `timeoutPrefix` does now.
   - `--as` (virtual memory) of about 2 GB is a safe ceiling for x264 at 1920 px. **Measure** with the harness `clip.mp4` and set the limit about 50% above the measured peak. Put it in `$CONFIG['media_ffmpeg_max_mem']`.
   - Add `-threads 2` to the video encode. TESTED: 7.6 s instead of 5.9 s for the 10 s 1080p60 clip, but it leaves cores for the website.
2. **Concurrency cap:** at most `media_max_concurrent` (default 2) conversions at once, image or audio/video, using lock files:
   ```php
   // Returns a lock handle, or null if all slots stayed busy for $waitSeconds.
   function acquireMediaSlot(int $waitSeconds = 20) {
       global $CONFIG;
       $dir = dirname($CONFIG['db_path']) . '/locks';
       if (!is_dir($dir)) mkdir($dir, 0700, true);
       $deadline = time() + $waitSeconds;
       do {
           for ($i = 0; $i < ($CONFIG['media_max_concurrent'] ?? 2); $i++) {
               $fh = fopen("$dir/media-$i.lock", 'c');
               if ($fh && flock($fh, LOCK_EX | LOCK_NB)) return $fh;   // released on fclose or when the process ends
               if ($fh) fclose($fh);
           }
           usleep(250000);
       } while (time() < $deadline);
       return null;
   }
   ```
   - Take a slot after classification, before processing. If you get `null`, return `503`: "The server is busy processing other uploads. Please try again in a moment."
   - **[ssreact]:** web and mobile treat 503 from `uploadMedia` as retryable: one automatic retry after 3 s, then show the message.

**Verify:**
- Start three `clip.mp4` uploads on the bench at once (`&` ×3) with `media_max_concurrent=2`. Two succeed. The third waits, then succeeds or gets 503, depending on timing. Record which.
- `ps` during the run shows ffmpeg at nice 10.

### B2 (M6). Write outputs to staging, then move into place
- All outputs (the final `.webp`, `.mp4`, `.mp3` and `thumb_*.webp`) are written in `private/tmp` first, then `rename()`d into `media/<uid>/<type>/` only after everything succeeded and **after** the DB insert.
- If `rename` fails across filesystems, fall back to `copy()` + `unlink()`.
- Register a `register_shutdown_function` at the start of `handle_uploadMedia` that deletes any staged files still listed. It runs on `bad()` (exit) and on fatal errors.

**Verify:**
- Force a failure after encoding (a temporary `throw` before the move). Nothing appears in `media/`, and `private/tmp` is empty afterwards.
- A normal upload still works.

### B3 (M7). Upload throttle, orphan sweep, 1 GB quota
1. **Throttle:** 60 uploads per user per hour, reusing the `auth_attempts`-based counter from ssapi plan S4 (`throttleSend`-style) with key `upload:<uid>`, window 3600 s. Over the limit → 429: "Too many uploads. Try again later."
2. **Orphan sweep:** media rows with `post_id IS NULL` and `created_at` older than 24 h are deleted (file, thumbnail, row).
   - Run it for the uploading user at the start of each upload; it's cheap and indexed by `user_id`.
   - Also run a **global** sweep of at most 50 rows on about 1% of uploads, so dormant accounts get cleaned too.
   - Reuse `mediaFilePath()`.
   - **Check first** that no client attaches media more than 24 h after uploading it. Drafts restore `mediaUrl` from local storage: web `ss_post_draft`, and the same key on mobile. So a draft older than 24 h would point at deleted media.
   - **Fix the clients [ssreact]:** when restoring a draft with media older than 24 h, drop the media part with a toast ("Your attached file expired; please add it again"). Alternatively, the server returns `400 'That media is invalid…'` on post and the client clears it. Do the client fix; it's a nicer experience.
3. **Quota: 1 GB of media per user (Dave, 2026-10-07).** `$CONFIG['media_max_user_bytes'] = 1_073_741_824`. Needs the `bytes` column (B4).
   - **What counts:** every stored file the user owns: originals, 960 px variants, posters and thumbnails. Use `SUM(bytes) FROM media WHERE user_id = ?`. Unattached uploads count until the orphan sweep removes them; run the user's sweep (step 2) **before** checking the quota so expired drafts don't hold space.
   - **When:**
     - Before conversion: if the user is already at or over the limit, reject straight away, without spending CPU.
     - After conversion, before the move into `media/` (B2): if `used + bytes of the new files` is over the limit, reject and delete the staged files.
   - **Response:** `413` with `{valid:false, code:'media_quota', message:'You have reached your media storage limit of 1 GB of media.'}`. Add an optional `$extra` array to `bad()` in `media.php` to carry `code`. Build the "1 GB" text from the config value, so changing the limit changes the message.
   - **API (additive):** `getMediaLimits` adds `storageUsedBytes` and `storageLimitBytes`.
   - **[ssreact] web and mobile:** on the create-post page, an upload rejected with `code:'media_quota'` shows that message as an error on the page, next to the media picker. Don't use a toast that disappears, and don't retry. When the user is already full (from `getMediaLimits`), show the same message up front and disable the media button; text-only posts still work. Settings shows "Media storage: 230 MB of 1 GB used".
   - **Existing files:** rows uploaded before B4 have `bytes = NULL`. Ship a small CLI script with B4, `ssapi/bin/media-backfill.php --bytes`, that fills in `filesize()` for each row (and its poster/thumbnail). Dave runs it on dev, then prod, right after deploying B4, and before the quota is enforced. C3 later extends the same script. The quota check treats any remaining NULL as 0. Log any user who is already over 1 GB; they can't upload more but keep what they have.
   - **Verify:** on the bench, set the limit to 5 MB. Uploads succeed until the total passes 5 MB, then the exact 413 message appears. Delete a post, and uploading works again. `getMediaLimits` reports the right numbers.

### B4. Media table columns (migration)
Through the schema migration mechanism (`migrationN` in `schema.php`), add to `media`: `width INTEGER`, `height INTEGER`, `duration REAL`, `bytes INTEGER`, `variant_path TEXT` (F2), `poster_path TEXT` (F3), `loop INTEGER DEFAULT 0` (D6), plus an index on `media(path)` (F2's join).

Fill the columns on upload. `bytes` = `filesize()` of the final file, summing variants.

### B5 (M8). Protect attached media
`handle_deleteMedia`: if `post_id` is not NULL → `409 'That file is attached to a post. Delete the post instead.'`.

The clients only call `deleteMedia` for drafts, so nothing breaks; confirm with `grep deleteMedia` in web/ and mobile/.

**Commits:** one per task: `media: …`.

---

## 5. Phase C: speed (ssapi + [ssreact], about 1–2 days)

### C1 (F1). One ffprobe call
`probeMedia` returns everything at once:
```
-show_entries format=format_name,duration:stream=codec_type,codec_name,width,height,r_frame_rate,color_transfer,color_primaries:stream_disposition=attached_pic
```
- Pass the probe array through `classifyMedia` → the handler → `processVideo`/duration check.
- Delete `ffprobeValue`, `mediaDuration` and `videoFrameRate`.

### C2 (F3). A better poster, exposed to clients
- `createVideoThumbnail`: seek to `min(0.5, duration/2)` before the input (`-ss 0.5 -i …`), so the poster is a real frame rather than the often-black first one. Store `poster_path` = `/media/<uid>/video/thumb_<base>.webp`.

### C3 (F2). A feed-size image variant + dimensions
1. `processImage` also writes a **960 px** WebP (`<base>_960.webp`, quality 80), only when the original is wider or taller than 960.
   - Store `variant_path`, `width` and `height` (of the 1920 version).
   - Reuse the decoded GD image: resample from `$src` twice, rather than decoding again.
2. **API (additive):** post objects gain `media: { type, url, width, height, variantUrl, posterUrl, duration }`, built by `LEFT JOIN media ON media.path = posts.media_url` in the post-list handlers (`fetchFollowedPosts`, `getUserPosts`, `getPostById`) and `postRowToApi`.
   - **Keep `mediaUrl`** for old clients (CLI/TUI, older app builds).
   - `variantUrl` and `posterUrl` are null when absent.
3. **Backfill script (CLI only):** `ssapi/bin/media-backfill.php`, guarded by `php_sapi_name() === 'cli'`.
   - For existing images: read dimensions and make the 960 variant.
   - For existing videos: probe dimensions/duration, set `poster_path` if `thumb_*.webp` exists, and generate one if not.
   - Supports `--dry-run`.
   - **Dave runs it** on dev, then on prod.
4. **[ssreact] web:** `PostMedia` uses `variantUrl ?? url` in the feed and the full `url` in the lightbox. It reserves space with `aspect-ratio: width / height` when known, which fixes the layout jump. `VideoPlayer` uses `poster={posterUrl}` when present and keeps the decode trick only as a fallback.
5. **[ssreact] mobile:** `expo-image` with `variantUrl`, the same aspect-ratio box, and `posterUrl` for `expo-video`'s poster.

**Verify:**
- An upload returns and stores dimensions, variant and poster.
- `fetchFollowedPosts` includes the new `media` object, and `mediaUrl` is unchanged.
- Bytes transferred for a 25-post feed of photos, before vs. after, measured in the browser's Network panel: expect a large drop.
- No layout jump while scrolling.

**Commits:**
- `media: single ffprobe pass`
- `media: poster from 0.5s, stored`
- `media: 960px variant + dimensions; post API media object`
- `media: backfill script`
- `web: use media variants, dimensions and posters`
- `mobile: same`

---

## 6. Phase D: reliability and quality (ssapi + [ssreact], about 1–2 days)

### D1 (R1). Keep transparency for every image type
In `processImage`, when resizing, always prepare `$dst` for alpha (`imagealphablending($dst, false)`, `imagesavealpha($dst, true)`, transparent fill), not only for `png`. JPEG has no alpha, so this costs nothing there.

**Verify:** harness R1 passes; a normal JPEG is unchanged.

### D2 (R2). iPhone HEIC photos
1. Dev and prod already run ffmpeg 7.1 (recorded in the [[ssapi]] note), so the multi-tile fix is in. Re-check with `ffmpeg -version` only if the host changes.
2. Test with a **real** iPhone HEIC. Dave uploads one to dev from his phone, or provides a non-personal sample to the agent. Check that the result is the whole photo, upright, at the right aspect ratio.
3. If ffmpeg is older than 7.1 or the result is wrong, either:
   - **(a)** upgrade ffmpeg on the server (distro backport or a static build); **DECISION** for Dave; or
   - **(b)** the phone converts photos before upload. **Already done**, by the mobile work: `mobile/src/lib/media-upload.ts` `renderImage` resizes to the server's limit and saves JPEG 0.92 with `expo-image-manipulator`. The web equivalent is `web/src/lib/media-image.ts` `renderImageFile` (JPEG, or PNG for formats that can be transparent).
4. **Switch the client-side conversion to WebP (Dave, 2026-10-07).** The server re-encodes to WebP anyway (that's how it strips metadata and enforces sizes, and it can't trust client files), so the upload format only affects upload size and transparency. WebP wins on both: about 25–35% smaller than JPEG at the same quality (faster on mobile data), and it keeps transparency, so the web no longer needs a PNG fallback.
   - **mobile:** `SaveFormat.WEBP`, `compress: 0.9`. **First** read the SDK 57 `expo-image-manipulator` docs (per `mobile/AGENTS.md`; don't trust memory). Older SDKs could only write WebP on Android. If iOS still can't, use WebP on Android and keep JPEG on iOS (`Platform.OS`). Check on a real iPhone and Android phone with a development build: the photo uploads, and the server output looks right.
   - **web:** `canvas.toBlob(cb, 'image/webp', 0.9)`. Safari can't encode WebP and silently returns PNG instead (much bigger), so check `blob.type`. If it isn't `image/webp`, encode again as JPEG, or as PNG when the image has transparency.
   - **Animated GIFs must skip this step** on both web and mobile. Drawing them to a canvas or manipulator keeps only the first frame, which would defeat D6. Send GIFs to the server as they are.

### D3 (R3). ffmpeg health check and honest errors
- Add `mediaCapabilities()`: `exec` is available, and `ffmpeg -version` / `ffprobe -version` succeed under `timeout 5`. Cache the result for 1 hour in `private/tmp/media-capabilities.json`.
- `getMediaLimits` adds `videoSupported` and `audioSupported` (additive).
- Uploads of video/audio when unsupported → `503 'Video and audio uploads are temporarily unavailable.'`, logged as ERROR.
- **[ssreact]:** web and mobile hide or disable the video/audio capture and pick options when the flags are false.

**Verify:** on the bench, set `FFMPEG` to a non-existent path (temporarily). `getMediaLimits` reports false, a video upload gives the clear 503, and images still work.

### D4 (R5). Align the web picker
`MEDIA_ACCEPT` in `web/src/pages/CreatePostPage.tsx` becomes `image/*,video/*,audio/*`. That's wider than A5's list, but the server is the gatekeeper and its message names the supported types. A precise MIME list would wrongly hide files whose browser-reported type is missing or odd.

Keep the client-side duration check. It already falls back gracefully when the browser can't read a format's duration (it returns 0 and lets the server decide).

### D5 (R4, optional). HDR → standard colour
- If the probe shows `color_transfer` of `arib-std-b67` (HLG) or `smpte2084` (PQ), and `ffmpeg -filters` lists `zscale`, prepend a tone-map to the video filters:
  ```
  zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709,tonemap=tonemap=hable:desat=0,zscale=t=bt709:m=bt709:r=tv,format=yuv420p
  ```
- Without `zscale`, convert as today and log a WARN.
- **Verify:** harness R4 picks the HDR path. Then check a real iPhone HDR clip on dev visually (Dave).

### D6. Animated GIFs become looping video (Dave, 2026-10-07: yes)
Today an animated GIF is flattened to its first frame, both by the clients (D2 step 4) and by GD on the server.
- **Detect:** ffprobe the GIF (C1's single probe). It is animated if `nb_frames > 1`, or if `-count_packets` reports more than one packet when `nb_frames` is missing. A single-frame GIF stays a WebP image.
- **Convert:** run the video path with GIF-specific args: `-an -t <media_max_seconds> -vf "scale=trunc(min(iw\,960)/2)*2:-2,format=yuv420p" -c:v libx264 -preset veryfast -crf 26 -movflags +faststart -map_metadata -1`. Even dimensions are required by H.264, and 960 px is plenty for GIFs. The same pixel limits (A1), resource caps (B1) and quota (B3.3) apply. A GIF with thousands of tiny frames is stopped by `-t` and `timeout`.
- **Store:** `type = 'video'`, plus a new `loop INTEGER DEFAULT 0` column on `media` (add it to B4's migration) set to 1. Make a poster the normal way (C2).
- **API (additive):** C3's post `media` object gains `loop: true`. Older clients just see a short silent video with controls, which still works.
- **[ssreact] web:** when `loop` is true, render `<video autoplay loop muted playsinline poster=…>` with no controls, styled like an image.
- **[ssreact] mobile:** `expo-video` with `loop`, `muted`, autoplay when on screen, no controls.
- **Harness:** `anim.gif` (Phase 0) → an MP4 with no audio stream, ≥ 2 s long, `loop = 1`. Add a single-frame GIF fixture → WebP.

**Commits:** one per task.

---

## 7. Ops checklist (Dave on el1; record results in the [[ssapi]] note)
1. `ffmpeg -version` on dev and prod (D2). Check `ffmpeg -filters | grep zscale` (D5).
2. PHP for the vhost:
   - `upload_max_filesize` and `post_max_size` ≥ 110M (video limit 100 MB, plus form overhead);
   - `max_execution_time`, or the per-request `set_time_limit` already in code, ≥ 120.
3. mod_fcgid (app/dev):
   - `FcgidMaxRequestLen` ≥ 115343360 (110 MB). The default is only 128 KB; uploads work today, so it has been raised, but confirm it covers 100 MB.
   - `FcgidIOTimeout` ≥ 120 s, so a conversion under load doesn't cut the response.
4. **Backups include `public_html/media/`**, not just `private/userdata.db`, and a restore has been tried once.
5. Disk: free space on the media volume, with an alert threshold.

---

## 8. Order and estimate

| Phase | Tasks | Effort |
|---|---|---|
| 0 | Harness + fixtures (sstests) | about 1 hour |
| A | A1 pixel limits, A2 metadata strip, A3 audio re-encode, A4 log sanitising, A5 narrower formats + codec allow-list | about 1 day. **Do first; deploy soon after (Dave).** |
| B | B1 resource caps + concurrency, B2 atomic outputs, B3 throttle + orphan sweep + 1 GB quota (with client messages), B4 columns + bytes backfill, B5 delete guard | 1–1.5 days |
| C | C1 single probe, C2 poster, C3 variants + API + backfill + clients | 1–2 days |
| D | D1 alpha, D2 HEIC test + WebP client conversion, D3 health check, D4 picker, D5 HDR (optional), D6 looping GIFs | about 2 days |

About **5–7 days of agent work** in total, roughly 1.5–2 weeks of calendar time on the $20 plan. Phase A alone closes the two high-severity problems.

**Decisions (Dave, 2026-10-07):**
1. HEIC: ffmpeg is already 7.1; the phone already converts before upload. Clients switch from JPEG to **WebP** (D2 step 4), with a fallback where the platform can't encode WebP.
2. **1 GB of media per user**, with a clear error on the create-post page (B3.3).
3. **Animated GIFs become looping video** (D6).
4. Formats: the reviewer's choice, recorded in A5. Keep phone/browser/common-app formats; drop AVI, FLV, WMV/ASF, MPEG-TS/PS, TIFF and ICO.

Still open: none for this plan. Ops checks (§7) are Dave's on el1.
