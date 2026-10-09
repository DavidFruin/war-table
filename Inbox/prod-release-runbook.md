---
status: ready to follow once Dave has checked dev (written 2026-10-08)
for: Dave (and an agent he gives el1 access to)
---

# Prod release runbook: moderation, media, English/Spanish, reports, invite codes

**What this is.** Prod (`app.davidfruin.com`) still runs the backend and web app from 2026-10-06. Everything built since is on **dev** and tested on a local bench. This is the order to put all of it on prod. Nothing here has been done on prod. **Prod only on Dave's go.**

**What prod gets, in plain words**
- **Moderation:** report, block, an admin page, Terms and Privacy pages, freezing accounts.
- **Media:** safer, faster uploads (size limits, a 1 GB quota per person, better video, looping GIFs, smaller feed images).
- **English/Spanish:** a language picker, a "choose your language" popup at everyone's next login, Spanish server messages, emails and push notifications.
- **Reports:** a post or comment you reported stays visible to you with a red "Reported" label, and can't be reported twice.
- **Invite codes:** registration becomes invite-only. Each member invites one person, the owner as many as they like. A welcome message appears for new members.

**The database changes on its own** at the first request after the new backend is in place (versions 3, 4, 5, 6 and 7, one after another). That is why the backup in step 1 matters.

---

## A. Before touching prod (on dev, with Dave)

Dev already has all of it (backend `c9f2f2b`, web `ddc3042`). Check these on **dev.davidfruin.com**:

1. **Log in** with a normal member account: the language popup shows, then the terms dialog (if not accepted), then (for a brand-new member only) the welcome.
2. **Settings** has the Language card; switching works and survives a reload.
3. **Report** a post from a second account: it stays, with a red "Reported", and the report menu is gone from it. The report email arrives (needs `ADMIN_REPORT_EMAIL`, see C2).
4. **Admin page** (`/admin`) opens for the admin account and lists the report.
5. **Invite someone** on your profile: generate a code, copy it. In a private window open `/register`, enter the code: it should say "Invited by …". Finish registering with a real test email and check the mail code arrives. Then log in as that new member and look at the welcome. (Hotmail/Outlook addresses will not get the mail code; see D1.)
6. **Upload** a photo, a video and a voice recording; the feed shows them.
7. Open the **Spanish** pages (`?lang=es`) and have a Spanish speaker read them. The wording is a draft until they have.
8. **Terms and Privacy** texts: read and approve them (they say "Draft" until `LEGAL_DRAFT` is turned off in `web/src/content/legal.ts`). The store listings need these as public pages.
9. **Phone** (Expo Go or a development build pointed at dev): the register step, the invite card and its share sheet, the welcome, the language popup, report/block. These screens have only been type-checked, never run on a device.

If anything is wrong, stop here and tell the agent; nothing on prod has changed yet.

## B. Server checks (el1; record the results in [[ssapi]])

From the media plan's ops list (section 7), once on prod:
1. `ffmpeg -version` is recent, and `ffmpeg -filters | grep zscale` (HDR video).
2. PHP: `upload_max_filesize` and `post_max_size` at least 110M; `max_execution_time` at least 120 (or the per-request value in the code).
3. `FcgidMaxRequestLen` at least 115343360 and `FcgidIOTimeout` at least 120 s.
4. **Backups include `public_html/media/`**, not just the database, and a restore has been tried once.
5. Free disk space on the media volume, with an alert.
6. PHP **GD** extension in the **command-line** PHP (the backfill in D4 needs it).

## C. The release (prod, in this order)

Replace `$D` with `/home/davidfruin/domains/app.davidfruin.com`.

1. **Back up, twice.**
   - Database: `sqlite3 $D/private/userdata.db ".backup '$D/private/userdata.db.pre-release-YYYYMMDD'"`
   - Code: `cp -a $D/ssapi $D/ssapi.bak-YYYYMMDD`, and `cp -a $D/public_html/app $D/public_html/app.bak-YYYYMMDD`
2. **Settings in `$D/private/.env`** (add these lines; never commit this file):
   - `ADMIN_REPORT_EMAIL=` where reports are emailed
   - `CONTACT_EMAIL=` shown to suspended people and in Settings
   - `TERMS_VERSION=1` and `TERMS_URL=/terms`
   - Leave `REGISTRATION_MODE` **unset** (that means invite-only). Do **not** set it to `open` on prod.
3. **Deploy the backend** (from the ssapi checkout on Dave's machine; it needs a locally built `vendor/`):
   ```
   cd ~/dev/ssapi && git pull && composer dump-autoload --no-dev --optimize
   rsync -rltz --no-owner --no-group --delete --exclude='.git' --exclude='deploy/' ./ el1:$D/ssapi/
   rsync -rltz --no-owner --no-group deploy/public/ el1:$D/public_html/
   rsync -tz deploy/root.htaccess el1:$D/public_html/.htaccess
   ssh el1 "chown -R davidfruin:davidfruin $D/ssapi $D/public_html/.htaccess $D/public_html/api.php $D/public_html/media.php"
   ```
   Then make **one harmless request** (`curl -s -d action=getMediaLimits https://app.davidfruin.com/api.php`) so the database upgrades, and check it reads version 7: `sqlite3 $D/private/userdata.db 'pragma user_version'`. Then `chown davidfruin:davidfruin $D/private/userdata.db*`.
4. **Make yourself the owner and record everyone as invited by you.** Pick the account that is your admin login on prod (on dev it is `me@davidfruin.com`):
   ```
   sqlite3 $D/private/userdata.db "UPDATE users SET role = 'owner' WHERE email = '<your admin login email>';"
   sqlite3 $D/private/userdata.db "UPDATE users SET invited_by = (SELECT id FROM users WHERE role = 'owner') WHERE invited_by IS NULL AND role != 'owner';"
   sqlite3 $D/private/userdata.db "UPDATE users SET is_admin = 1 WHERE email = '<your admin login email>';"
   ```
   (the third line is only needed if that account isn't already an admin).
5. **Media backfill** (fills in sizes and the smaller feed images for existing uploads). Dry run first, then for real, as the site's user:
   ```
   cd $D/ssapi && php bin/media-backfill.php --bytes --dry-run && php bin/media-backfill.php --details --dry-run
   php bin/media-backfill.php --bytes --details
   chown -R davidfruin:davidfruin $D/public_html/media $D/private
   ```
6. **Deploy the web app** (from the ssreact checkout): `cd ~/dev/ssreact && git pull && scripts/deploy-web.sh app` (it asks you to type `prod`). Then `ssh el1 "chown -R davidfruin:davidfruin $D/public_html/app"`.
7. **Check prod** (no test data on prod beyond what's listed):
   - `https://app.davidfruin.com/` loads and the bundle name matches the build (`curl -s https://app.davidfruin.com/ | grep -o 'assets/index-[^"]*js'`).
   - Log in as yourself; a wrong password gives the normal error.
   - `/terms`, `/privacy`, `/register` load; `config.php` and `vendor/autoload.php` return 403.
   - `sendRegisterOTP` without a code is refused: `curl -s -d action=sendRegisterOTP -d email=x@example.invalid https://app.davidfruin.com/api.php` should say "That invite code isn't valid."
   - Your own profile shows the Invite someone card; generate a code and register one real family member with it.
   - Feed shows old photos and videos.
8. **Phone.** Build the family APK and TestFlight build (phone plan Phase 8). This build has **native changes** (Spanish permission prompts, the per-app language setting), so it needs a real build; an over-the-air update alone isn't enough. Point it at prod only for a deliberate release test.
9. **Tell people.** The next time anyone logs in they see the language popup, then (if new) the welcome. Existing members keep working exactly as before.

## D. Known issues and things to decide

1. **Hotmail/Outlook email.** Microsoft is currently blocking the server's IP, so people with Hotmail, Outlook or Live addresses never get their registration or password-reset code. With invite-only registration this affects each new Hotmail invitee. Fixes: ask Microsoft to delist the IP (sender.office.com), or send mail through a mail service. Until then, invitees need a Gmail (or similar) address, or you create the account by hand.
2. **Terms and Privacy are drafts** until you approve them and turn `LEGAL_DRAFT` off. The store listings need them as public pages.
3. **Terminal clients** (sscli, sswiz, sstui) can't register while it's invite-only; logging in still works.
4. **Two web bugs left as they are, on purpose:** the page doesn't always start at the top after navigating, and admins can't add a note to a report. "View post" on a report can still fail when the author is frozen or blocked.
5. **The phone screens for invites and the language popup are unverified on a device.**
6. **Staff roles** (moderators, admins) are a separate, later plan ([[staff-roles-plan]]); this release only adds the owner.
7. **Step 1C (store-review fixes) was in progress when this was written** (2026-10-08; access plan Step 1C):
   - **What it changes:** reported posts **fold away** for the reporter instead of the red-label behaviour above; a word filter reads `private/blocked-words.txt`; the minimum age is 13.
   - **If it's finished before the prod release, ship it in the same release:** create `$D/private/blocked-words.txt` before step C3.
   - **Keep `TERMS_VERSION=1` on prod.** Nobody on prod has accepted any terms yet, so everyone sees the new terms, age line included, at their first login anyway. Only dev, where people already accepted version 1, needs `TERMS_VERSION=2`.
   - Check dev again for the fold-away and the filter before the prod release.

## E. If something goes wrong

- **Backend broken:** put the old code back (`mv $D/ssapi $D/ssapi.failed && mv $D/ssapi.bak-YYYYMMDD $D/ssapi`). The database changes are additive (new tables and columns), so the old code still runs against the upgraded database. Only restore the database backup if the data itself is damaged, and note that anything people posted after the backup would be lost.
- **Web broken:** `rsync -rltz --delete $D/public_html/app.bak-YYYYMMDD/ $D/public_html/app/` (on the server).
- **Locked out of registering by mistake:** that is the intended behaviour; make a code on your profile.
- **Need to pause new sign-ups completely:** just don't hand out codes; nothing else needs changing.

## F. After the release

- Mark this runbook done with the date and the commit hashes deployed (`git -C ~/dev/ssapi log -1 --format=%h`, same for ssreact).
- Record the server-check results (B) in [[ssapi]].
- Delete the backups a few days later.

Related: [[access-and-public-launch-plan]] (invite codes, §1.6), [[media-pipeline-plan]] (§7), [[language-plan]] (§6), [[ssapi]], [[ssreact]].

## Added 2026-10-09: staff roles and the store-review fixes (Step 1C)
Also part of this release now (all built, tested on a local bench):
- **Staff roles** (`Inbox/staff-roles-plan.md`): schema version 7; `composer dump-autoload` for the new `src/Staff` module. The old `is_admin` users become admins at the first request. **Make yourself the owner** with the `UPDATE users SET role = 'owner' ...` command (once, if you haven't already), then appoint people from the website's moderation page (`/admin`, Team tab).
- **Word filter**: create `private/blocked-words.txt` (one word or phrase per line, readable only by you and the web server). No file means no filtering.
- **Minimum age 13**: set `TERMS_VERSION=2` in `private/.env`.
- The old `adminFreezeUser` endpoint is gone; web builds older than this release don't know the new moderation page.
