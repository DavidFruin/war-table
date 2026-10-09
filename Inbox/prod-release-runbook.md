---
status: PARTLY DONE on prod 2026-10-09 (backend ssapi d703ca8 at schema 7, web ssreact be956b9, .env settings, blocked-words.txt, backups *-20261009); still to do: C4 owner commands and C5 media backfill (blocked for the agent), then C8/C9. Was: ready for prod on Dave's go (dev deployed and tested 2026-10-09). Before starting, approve Terms/Privacy (LEGAL_DRAFT) and do section B.
for: Dave (and an agent he gives el1 access to)
---

# Prod release runbook: moderation, staff roles, media, English/Spanish, invite codes, store-review fixes

**What this is.** Prod (`app.davidfruin.com`) still runs the backend and web app from 2026-10-06. Everything built since is on **dev** and tested on a local bench. This is the order to put all of it on prod. Nothing here has been done on prod. **Prod only on Dave's go.**

**What prod gets, in plain words**
- **Moderation:** report, block, Terms and Privacy pages, and a moderation page (`/admin`) with Reports, Frozen, Team and Activity tabs.
- **Staff roles:** owner, admin, moderator and user, shown as a badge on every profile. Moderators freeze, admins also delete and appoint moderators, and the owner also appoints admins. Freezing and deleting happen only on the moderation page.
- **Media:** safer, faster uploads (size limits, a 1 GB quota per person, better video, looping GIFs, smaller feed images).
- **English/Spanish:** a language picker, a "choose your language" popup at everyone's next login, Spanish server messages, emails and push notifications.
- **Reports:** a post or comment you reported **folds away** for you ("You reported this post", with Show), and can't be reported twice.
- **Word filter:** posts and comments containing a word from your private list (`private/blocked-words.txt`) are refused.
- **Minimum age 13:** in the Terms, the Privacy page and the sign-up checkbox.
- **Invite codes:** registration becomes invite-only. Each member invites one person, the owner as many as they like. A welcome message appears for new members.

**The database changes on its own** at the first request after the new backend is in place (versions 3, 4, 5, 6 and 7, one after another). That is why the backup in step 1 matters.

---

## A. Before touching prod (on dev, with Dave)

**Dev is current (Dave, 2026-10-09):** Step 1C and staff roles (backend `d703ca8`, schema version 7; ssreact up to `be956b9`) are deployed to dev, and Dave tested them there. No agent recorded the deploy itself in [[ssapi]]; add the details there if they matter. The steps below are kept for reference.

**First, put the latest on dev:**
1. Run the same steps as C1, C3 and C6 below, with dev's folder and `scripts/deploy-web.sh dev`.
2. Create dev's `private/blocked-words.txt` with a test word.
3. Set `TERMS_VERSION=2` in dev's `.env`. Dev members already accepted version 1, so this shows them the updated terms.

An agent can do this with Dave's go.

**Then check these on dev.davidfruin.com:**

1. **Log in** with a normal member account: the language popup shows, then the terms dialog (if not accepted), then (for a brand-new member only) the welcome.
2. **Settings** has the Language card; switching works and survives a reload.
3. **Report** a post from a second account: it folds away for that account, Show reveals it with a red "Reported", and it can't be reported again. The report email arrives (needs `ADMIN_REPORT_EMAIL`, see C2).
4. **Moderation page** (`/admin`), as the owner:
   - The report is listed. Freeze the post: it disappears for others and shows "Hidden by a moderator" to its author. Unfreeze it from the Frozen tab.
   - In Team, make a test account a moderator. Log in as it and check it sees Reports, Frozen and Team, but no delete options and nothing about staff.
   - Activity lists all of it.
5. **Badges:** every profile shows User, Moderator, Admin or Owner.
6. **Word filter:** a post containing the test word is refused with the "isn't allowed" message, and the text stays in the box.
7. **Sign-up** shows the "at least 13 years old" checkbox. The terms dialog appears once for existing members (version 2).
8. **Invite someone** on your profile: generate a code, copy it. In a private window open `/register`, enter the code: it should say "Invited by …". Finish registering with a real test email and check the mail code arrives. Then log in as that new member and look at the welcome. (Hotmail/Outlook addresses will not get the mail code; see D1.)
9. **Upload** a photo, a video and a voice recording; the feed shows them.
10. Open the **Spanish** pages (`?lang=es`) and have a Spanish speaker read them. The wording is a draft until they have.
11. **Terms and Privacy** texts: read and approve them (they say "Draft" until `LEGAL_DRAFT` is turned off in `web/src/content/legal.ts`). The store listings need these as public pages.
12. **Phone** (Expo Go or a development build pointed at dev): the register step, the invite card and its share sheet, the welcome, the language popup, report/block and fold-away, badges, and the staff link in Settings. These screens have only been type-checked, never run on a device.

If anything is wrong, stop here and tell the agent; nothing on prod has changed yet.

## B. Server checks (el1; record the results in [[ssapi]])

From the media plan's ops list (section 7), once on prod:
1. `ffmpeg -version` is recent, and `ffmpeg -filters | grep zscale` (HDR video).
2. PHP: `upload_max_filesize` and `post_max_size` at least 110M; `max_execution_time` at least 120 (or the per-request value in the code).
3. `FcgidMaxRequestLen` at least 115343360 and `FcgidIOTimeout` at least 120 s.
4. **Backups include `public_html/media/`**, not just the database, and a restore has been tried once.
5. Free disk space on the media volume, with an alert.
6. PHP **GD** extension in the **command-line** PHP (the backfill in C5 needs it).

## C. The release (prod, in this order)

Replace `$D` with `/home/davidfruin/domains/app.davidfruin.com`.

1. **Back up, twice.**
   - Database: `sqlite3 $D/private/userdata.db ".backup '$D/private/userdata.db.pre-release-YYYYMMDD'"`
   - Code: `cp -a $D/ssapi $D/ssapi.bak-YYYYMMDD`, and `cp -a $D/public_html/app $D/public_html/app.bak-YYYYMMDD`
2. **Settings in `$D/private/.env`** (add these lines; never commit this file):
   - `ADMIN_REPORT_EMAIL=` where reports are emailed
   - `CONTACT_EMAIL=` shown to suspended people and in Settings
   - `TERMS_VERSION=2` and `TERMS_URL=/terms`. Nobody on prod has accepted any version yet, so 1 or 2 makes no difference there; 2 keeps prod and dev the same.
   - Leave `REGISTRATION_MODE` **unset** (that means invite-only). Do **not** set it to `open` on prod.
   - **Also create `$D/private/blocked-words.txt`**: one word or phrase per line, readable only by you and the web server. No file means no filtering.
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
5. **The phone screens added since October 7 are unverified on a device:** invites, the language popup, badges, fold-away and the staff link.
6. **Staff roles:**
   - **First request:** the old `is_admin` accounts become admins.
   - **The owner commands in C4 are still needed.** After the release, appoint people from `/admin` → Team.
   - **The old `adminFreezeUser` endpoint is gone.** No app used it.
## E. If something goes wrong

- **Backend broken:** put the old code back (`mv $D/ssapi $D/ssapi.failed && mv $D/ssapi.bak-YYYYMMDD $D/ssapi`). The database changes are additive (new tables and columns), so the old code still runs against the upgraded database. Only restore the database backup if the data itself is damaged, and note that anything people posted after the backup would be lost.
- **Web broken:** `rsync -rltz --delete $D/public_html/app.bak-YYYYMMDD/ $D/public_html/app/` (on the server).
- **Locked out of registering by mistake:** that is the intended behaviour; make a code on your profile.
- **Need to pause new sign-ups completely:** just don't hand out codes; nothing else needs changing.

## F. After the release

- Mark this runbook done with the date and the commit hashes deployed (`git -C ~/dev/ssapi log -1 --format=%h`, same for ssreact).
- Record the server-check results (B) in [[ssapi]].
- Delete the backups a few days later.

Related: [[access-and-public-launch-plan]] (invite codes §1.6, store-review fixes §1C.4), [[staff-roles-plan]] (§6), [[media-pipeline-plan]] (§7), [[language-plan]] (§6), [[ssapi]], [[ssreact]].
