---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi (backend), ssreact (web), ssreact-native (phone)
replaces: Inbox/store-readiness-plan.md (its moderation spec is now Step 2 below, unchanged in substance)
---

# Access and public launch plan

**Dave's direction (2026-10-02):**
- **Step 1, now: family only.**
  - Android gets an **APK file**.
  - iPhone gets an **Unlisted App Store** listing, with TestFlight in the meantime.
  - Registration becomes **invite-only with free codes** that Dave hands out.
- **Step 1B, right after:** **report, block, admin moderation, terms and a privacy policy.** Apple's full App Store review applies to Unlisted apps too, and its guideline 1.2 requires these for apps with user posts. It's also good practice for family use.
- **Step 2, later:** open Simple Social to the public as an **invite-only, paid** social network on the App Store and Google Play.

Steps 1 and 1B are needed for the family release. Step 2 is a separate, larger project. **Don't start any part of Step 2 without Dave's explicit go.**

How these fit with the other plans:
- `Inbox/ssapi-improvement-plan.md` comes first.
- `Inbox/ssreact-native-port-plan.md` builds the phone app. Its Phase 8 is the family release (APK + TestFlight), Phase 9 is the iPhone Unlisted App Store listing (needs Step 1B), and Phase 10 is the public launch (Step 2).
- This plan provides the invite system (Step 1), moderation/terms/privacy (Step 1B) and everything a public paid launch needs (Step 2).

---

## 0. Rules for the implementing agent
Everything in section 0 of the ssapi improvement plan applies here too:
- one task per commit;
- stop after each phase;
- verify on the local ssapi bench (its Phase 0);
- never deploy and never touch prod;
- no tests folder in ssapi;
- the vault is public.

Additional rules:
1. **Prerequisites:** the ssapi plan's P3 (migrations), S4 (OTP throttle), S8 (input helpers), S14 (atomic registration) and P9 must have landed. If they haven't, stop and tell Dave.
2. **Additive and backwards compatible:** existing accounts are untouched, and log in and work exactly as before. Only **new registrations** change.
3. **Be honest about access.** If invites are ever paused, say why truthfully: "Invites are paused while we grow at a pace our server can handle" is fine **when it's true**. Step 2 makes it true by design, because invite numbers are a real, chosen cap. Don't put a false claim in the app, the site or a store listing; that is also an App Store rejection risk.

---

# STEP 1 (now): invite-only registration with free codes

## 1.1 Behaviour
- **New accounts need an invite code.** Existing users aren't affected.
- Codes are created by **admins only** in Step 1. Dave is the admin, using the existing, unused `users.is_admin` column, which he sets by hand with `sqlite3`. There is no API to grant admin.
- **Each code has:**
  - a maximum number of uses (default 1);
  - an optional expiry (default 30 days);
  - an optional private note, such as "for Mum";
  - a way to revoke it.
- The code is checked at the **first** registration step, before any email is sent, and again atomically when the account is created.
  - That also stops strangers from using the registration form to send emails (it complements ssapi plan S4).
- **Format:** 10 characters from an unambiguous alphabet (`ABCDEFGHJKMNPQRSTUVWXYZ23456789`), shown as `XXXXX-XXXXX`. Input is case-insensitive, and spaces and dashes are ignored.
- **Mode switch:** `REGISTRATION_MODE` in `private/.env`, either `invite` or `open`.
  - The **default in code is `invite`**, so a missing setting fails closed.
  - `open` exists for local testing and is never set on prod without Dave.
- **Guessing protection:** failed code checks count per IP through the existing attempt limiter (scope `invite`, 20 per 15 minutes). Error messages are generic: "That invite code isn't valid." The one exception is "That invite code has already been used", which helps family members who reuse a code.

## 1.2 Backend (ssapi)
**Migration** (the next `migrationN`):
```sql
CREATE TABLE IF NOT EXISTS invite_codes (
  code TEXT PRIMARY KEY,               -- normalized: uppercase, no dashes
  created_by INTEGER NOT NULL,
  created_at TEXT NOT NULL,
  max_uses INTEGER NOT NULL DEFAULT 1,
  uses INTEGER NOT NULL DEFAULT 0,
  expires_at TEXT,
  revoked_at TEXT,
  note TEXT);
```
Add `invited_by_code TEXT` to `users` and `invite_code TEXT` to `pending_users`, using the column-exists guard pattern.

**Helpers (`src/Auth/handlers.php`):**
```php
const INVITE_ALPHABET = 'ABCDEFGHJKMNPQRSTUVWXYZ23456789';

function normalizeInviteCode($raw) {
    return strtoupper(preg_replace('/[^A-Za-z0-9]/', '', (string)$raw));
}

function newInviteCode() {
    $out = '';
    for ($i = 0; $i < 10; $i++) $out .= INVITE_ALPHABET[random_int(0, strlen(INVITE_ALPHABET) - 1)];
    return $out;
}

function registrationNeedsInvite() {
    return strtolower(getenv('REGISTRATION_MODE') ?: 'invite') !== 'open';
}

// Read-only check (used before sending the OTP email). Returns the normalized code, or exits via bad().
function checkInviteCode($pdo, $raw) {
    $keys = attemptKeys('invite', '-');                  // per-IP only
    checkAttemptLimit($pdo, ['ip' => $keys['ip']]);
    $code = normalizeInviteCode($raw);
    $s = $pdo->prepare('SELECT max_uses, uses, expires_at, revoked_at FROM invite_codes WHERE code = ?');
    $s->execute([$code]);
    $row = $s->fetch(PDO::FETCH_ASSOC);
    $valid = $row && $row['revoked_at'] === null
        && ($row['expires_at'] === null || $row['expires_at'] > date('Y-m-d H:i:s'));
    if (!$valid) { recordFailedAttempt($pdo, ['ip' => $keys['ip']]); bad("That invite code isn't valid.", 400); }
    if ((int)$row['uses'] >= (int)$row['max_uses']) bad('That invite code has already been used.', 400);
    return $code;
}
```
`recordFailedAttempt` iterates the keys array with an `email`/`ip` limit map. Passing only `ip` works, because `$limits['ip']` exists. Check that this holds after any ssapi-plan changes to that function.

**Flow changes:**
- **`handle_sendRegisterOTP`:** if `registrationNeedsInvite()`, then `$code = checkInviteCode($pdo, $_POST['inviteCode'] ?? '')` **before** the S4 throttle and before any email. Store `invite_code` on the `pending_users` row.
- **`handle_verifyRegisterOTP`:** unchanged.
- **`handle_finishRegister`:** inside the `BEGIN IMMEDIATE` transaction from ssapi plan S14, before the `INSERT`:
  ```php
  if (registrationNeedsInvite()) {
      $code = $pendingRow['invite_code'] ?? null;  // SELECT invite_code along with otp/dateCreated
      $claim = $pdo->prepare('UPDATE invite_codes SET uses = uses + 1
          WHERE code = ? AND revoked_at IS NULL AND uses < max_uses
          AND (expires_at IS NULL OR expires_at > ?)');
      $claim->execute([$code, date('Y-m-d H:i:s')]);
      if ($claim->rowCount() !== 1) { $pdo->exec('ROLLBACK'); bad('That invite code is no longer valid. Ask for a new one.', 400); }
  }
  ```
  Then insert the user with `invited_by_code = $code`. The conditional `UPDATE` is what makes a single-use code impossible to redeem twice, even with two simultaneous registrations.
- **Admin endpoints**, all calling `requireAdmin()` (users.is_admin = 1, otherwise 403):
  - `adminCreateInvites(count=1..50, maxUses=1..100, expiresDays=0..365 (0 = never), note?)` → `{codes: ['ABCDE-FGHJK', …]}`. Retry on the rare primary-key collision.
  - `adminListInvites(offset)` → codes with their uses, max, expiry, revoked state, note, and who used them (`SELECT email FROM users WHERE invited_by_code = ?`).
  - `adminRevokeInvite(code)`.
- **`getMyInfo`:** add `isAdmin` (bool) and `registrationMode` (`invite`/`open`). Both are additive.
- **`.env.example`:** add `REGISTRATION_MODE=invite`.

**Verify on the bench:**
1. Make alice an admin and run `adminCreateInvites(count=2)` → two codes.
2. `sendRegisterOTP` with no code → 400. With a bad code → 400, and `mail.log` stays unchanged. With a valid code in lowercase with spaces → 200, and the email is captured.
3. Finish registration → the user is created with `invited_by_code` set, and the code's `uses` is 1.
4. Reuse the same code → "already been used" before any email.
5. Two concurrent `finishRegister` calls holding the same single-use code (two pending emails, both with that code) → exactly one account.
6. A revoked code and an expired code → rejected.
7. 21 bad codes from one IP → 429.
8. `REGISTRATION_MODE=open` → registration works without a code. Remove the setting → back to invite mode.
9. A non-admin calling the admin endpoints → 403.
10. Existing users still log in normally.

**Commits:**
- `invites: schema + validation at registration`
- `invites: admin create/list/revoke`
- `invites: getMyInfo isAdmin/registrationMode`

## 1.3 Web (ssreact)
- **Registration** (`OtpAuthFlow`, register mode): when `registrationMode` is `invite`, add an "Invite code" field to step 1 next to the email field. Before `getMyInfo` is available (logged out), always show the field, and treat it as optional only if the server says `open`.
  - Simplest option: always show the field, with the hint "Simple Social is invite-only. Ask the person who invited you for a code."
  - Send `inviteCode` with `sendRegisterOTP`.
  - **Clients:** the API method is in `api.ts`, or in `src/core/api-client.ts` once the phone plan's Phase 1 has landed.
- **Landing / Register copy:** "Simple Social is invite-only." That is a true statement.
- **Admin, `/admin/invites`** (route guarded on `user.isAdmin`; link from Settings for admins):
  - A form: count, max uses, expiry days, note → shows the new codes with a copy button.
  - A table of existing codes, with who used each one and a Revoke button.
- **Verify** with `pnpm dev` against the bench: register with a code end to end (read the OTP from `mail.log`). The admin page creates, lists and revokes codes, and non-admins can't see the page. Run `pnpm lint && pnpm build`.

## 1.4 Phone (ssreact-native)
The same invite field on Register (phone plan, Phase 3.1). There is no admin UI on the phone; Dave uses the web `/admin/invites` page.

## 1.5 Other clients
- **Terminal clients** ([[simple-social-cli]], [[simple-social-cli-interactive]], [[simple-social-tui]]) and **the vanilla web app** ([[simple-social]]):
  - Login and everything else keeps working.
  - **Registration from them will fail** with "That invite code isn't valid" until each one adds an invite-code prompt to its register flow.
  - For family-only use that's acceptable: people register on the web or phone app, then log in anywhere.
  - Adding the prompt to each terminal client is a small, separate job in each repo, **not part of this plan**.
- Note this in the hand-off.

## 1.6 Step 1 rollout (Dave)
This rides on the same deploy that puts ssapi on prod (ssapi plan D6 / the phone plan's prerequisite):
1. Set `REGISTRATION_MODE=invite` in prod's `private/.env`, or leave it unset, which means invite mode.
2. `UPDATE users SET is_admin=1 WHERE email='<Dave>'`.
3. Create codes on `/admin/invites` and send one to each family member, along with the APK link (Android) or the TestFlight invite / unlisted App Store link (iPhone) (phone plan, Phase 8).

---

# STEP 1B (right after the family release): moderation, terms, privacy
**Why:**
- The iPhone app goes on the App Store as **Unlisted** (phone plan, Phase 9), and Unlisted apps go through **full App Store review**.
- Apple's guideline 1.2 requires apps with user posts to have:
  - a way to filter objectionable material;
  - **reporting** with timely responses;
  - **blocking** abusive users;
  - published **contact information**;
  - terms users agree to.
- It's also useful for the family app itself and is required again for Step 2.

**Scope for Step 1B:** everything below. The parts about store listings in "Phase D" apply to the Unlisted submission (phone plan, Phase 9).

**Not in Step 1B:** payments and member-generated invites (Step 2).

## 1B.1 Behaviour spec (agree this with Dave before coding)

**Report:**
- Any logged-in user can report a **post**, a **comment** or a **user**, choosing a reason (`spam`, `harassment`, `hate`, `sexual`, `violence`, `self_harm`, `illegal`, `other`) and adding optional details of up to 500 characters.
- One report per reporter per target; repeats are accepted silently, with no duplicate row.
- Reporting **automatically hides that content from the reporter**. Users expect that, and it is one of the ways Apple's filtering requirement is met.
- Each new report emails Dave (the admin address) straight away, so the "timely response" expectation, roughly 24 hours, can be met.

**Block (two-way invisibility):**
- When A blocks B, neither sees the other's posts, comments, likes or profile, and neither appears in the other's search.
- Existing follows in both directions are removed. Neither can follow the other again or comment on the other's posts.
- B's likes, comments and mentions no longer notify A, and vice versa.
- Unblock restores visibility only. Follows are not restored.
- B is not told about the block. B's requests about A behave as if A's content doesn't exist (404 / empty lists).

**Freeze (admin action):**
- A frozen account can't log in, and all of its sessions are revoked.
- Its posts and comments are hidden from everyone, which uses the same filter as blocking.
- Admins can unfreeze.

**Admin:**
- `users.is_admin` already exists and is unused (see [[simple-social]] "dead columns"). Reuse it.
- Dave sets it by hand with `sqlite3` on the server. There is **no API to grant admin**.
- Admins get a `/admin` page in ssreact. The phone app has no admin UI.

**Terms acceptance:**
- There is a current `TERMS_VERSION` (an integer).
- The app and the website block use of the app until the logged-in user has accepted that version: one screen, with a checkbox and an "Agree" button.
- Registration shows the same agreement.
- The server records `terms_version_accepted` and when it was accepted. It **doesn't** reject requests from users who haven't accepted, because that would break the CLI and TUI clients. Enforcement is in the store clients (phone and web).

**Optional, DECISION:** a word filter on posts and comments, set by Dave, either rejecting or masking listed words. Report + block + hide-on-report usually satisfies Apple's "filter" requirement, so this starts off.

---

## 1B.2 Phase A: backend (ssapi)

### A1. Migration (next `migrationN` after the ssapi plan's)
```sql
CREATE TABLE IF NOT EXISTS blocks (
  blocker_id INTEGER NOT NULL,
  blocked_id INTEGER NOT NULL,
  created_at TEXT NOT NULL,
  PRIMARY KEY (blocker_id, blocked_id));
CREATE INDEX IF NOT EXISTS idx_blocks_blocked ON blocks(blocked_id);

CREATE TABLE IF NOT EXISTS reports (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  reporter_id INTEGER NOT NULL,
  target_type TEXT NOT NULL CHECK (target_type IN ('post','comment','user')),
  target_id TEXT NOT NULL,
  target_user_id INTEGER,           -- author of the reported content (or the reported user)
  reason TEXT NOT NULL,
  details TEXT,
  snapshot TEXT,                    -- copy of the reported text at report time (content may be deleted later)
  created_at TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open','dismissed','actioned')),
  resolved_at TEXT, resolved_by INTEGER, resolution TEXT,
  UNIQUE (reporter_id, target_type, target_id));
CREATE INDEX IF NOT EXISTS idx_reports_status ON reports(status, created_at);
```
Add columns to `users`. Use the same column-exists guard pattern the code already uses:
- `frozen_at TEXT`
- `terms_version_accepted INTEGER NOT NULL DEFAULT 0`
- `terms_accepted_at TEXT`

### A2. The visibility helper, the core of block and freeze
In `api.php`, next to the other shared helpers:
```php
// Users whose content the viewer must not see, and who must not interact with the viewer:
// anyone the viewer blocked, anyone who blocked the viewer, and every frozen account.
// Computed once per request.
function hiddenUserIds($pdo, $viewerId) {
    static $cache = [];
    if (isset($cache[$viewerId])) return $cache[$viewerId];
    $stmt = $pdo->prepare('SELECT blocked_id FROM blocks WHERE blocker_id = ?
        UNION SELECT blocker_id FROM blocks WHERE blocked_id = ?
        UNION SELECT id FROM users WHERE frozen_at IS NOT NULL');
    $stmt->execute([$viewerId, $viewerId]);
    return $cache[$viewerId] = array_map('intval', $stmt->fetchAll(PDO::FETCH_COLUMN));
}

// Returns [" AND <col> NOT IN (?,?,..)", [ids]] or ['', []] -- append to a WHERE clause.
function hiddenFilter($pdo, $viewerId, $column) {
    $ids = hiddenUserIds($pdo, $viewerId);
    if (!$ids) return ['', []];
    return [" AND $column NOT IN (" . implode(',', array_fill(0, count($ids), '?')) . ')', $ids];
}

function isHiddenFrom($pdo, $viewerId, $userId) {
    return in_array((int)$userId, hiddenUserIds($pdo, $viewerId), true);
}
```
`$column` is always a hard-coded identifier chosen by the code, never user input.

**Also hide content the viewer has reported:**
```php
function reportedByViewer($pdo, $viewerId, $type) {
    $s = $pdo->prepare('SELECT target_id FROM reports WHERE reporter_id = ? AND target_type = ?');
    $s->execute([$viewerId, $type]);
    return $s->fetchAll(PDO::FETCH_COLUMN);
}
```
Filter posts by `posts.id NOT IN (...)` and comments by `c.id NOT IN (...)` in the same places as the block filter.

### A3. Apply the filter everywhere content or users are listed
Each row is a small edit. Merge the filter's parameters into the existing `execute([...])` arrays in the right order.

| Handler | Change |
|---|---|
| `fetchFollowedPosts` | `posts.user_id` filter + reported-post filter, in **both** the COUNT and the page query |
| `getUserPosts`, `getUserInfo`, `getMyFollowers`/`getMyFollows` with `userId` | if `isHiddenFrom(target)` → `bad('User not found', 404)` |
| `getPostById`, `getPostLikes` | 404 if the owner is hidden or the post was reported by the viewer; filter likers by `user_id` |
| `getPostComments`, comment counts (P4 helper) | filter `c.user_id` + reported comments |
| `getUsers`, `getUserEmails` | filter `id` (search, and mention suggestions, come from `getUsers`) |
| `getMyFollowers` / `getMyFollows` lists | filter hidden ids out of the result |
| `getNotifications`, `getUnseenNotificationCount` | filter `n.actor_id` |
| `getPostPreviews` | drop previews of posts by hidden users |
| `createNotification` | `if (isHiddenFrom($pdo, $recipientId, $actorId)) return;` at the top, so nothing is created or pushed |
| `followUser` | `bad('User not found', 404)` if hidden |
| `createComment`, `likePost` | `bad('Post not found', 404)` if the post's owner is hidden |
| `hydrateMentions` / batch | leave as is (a mention of a hidden user still shows the email) **or** return `email: null` so it renders as "@deleted user". **DECISION.** Recommend `null`, which needs no client change. |

### A4. New endpoints (all authenticated; add them to `$HANDLERS`)
- **`reportContent(targetType, targetId, reason, details?)`**
  - Validate the type and the reason against the allowlist.
  - Check that the target exists and get its author.
  - Refuse to let users report themselves.
  - Throttle to 20 reports per user per day. Reuse `throttleSend` from ssapi plan S4 with key `report:<uid>` and a 24h window, or add a window parameter.
  - Save `snapshot` (the post or comment text, or the user's email).
  - `INSERT OR IGNORE`.
  - Then `defer()` (ssapi plan P2) an email to `ADMIN_REPORT_EMAIL` from `private/.env`. Use the subject "Simple Social report #<id>: <type> (<reason>)" and include the snapshot and a link to `/admin`. **No reporter email address in the message.**
  - Response: `{valid:true}`.
- **`blockUser(userId)`**
  - Refuse to let users block themselves; the target must exist.
  - `INSERT OR IGNORE` into `blocks`.
  - Remove the follow in both directions from the `follows` data. That's the JSON today, or the table if ssapi plan D2 has landed. **Write both while both exist.**
  - Delete existing notifications between the two users in both directions.
- **`unblockUser(userId)`**: delete the row.
- **`getBlockedUsers()`**: `[{id, email, created_at}]` of the people *I* blocked. Never return who blocked me.
- **`acceptTerms(version)`**: `version` must equal `TERMS_VERSION`. Store it with a timestamp.
- **`getMyInfo` (additive fields):**
  - `termsVersionAccepted`, `termsVersionCurrent`, `isAdmin` (bool).
  - Put `TERMS_VERSION` in config with a matching `TERMS_URL`.
- **Admin endpoints**, each calling `requireAdmin($pdo, $user)` (checks `users.is_admin = 1`, else 403):
  - `adminListReports(status='open', offset)`: reports joined with reporter and target-author emails, newest first, `pageParams()`.
  - `adminResolveReport(reportId, action, note?)`:
    - `action` ∈ `dismiss | delete_content | freeze_user | delete_and_freeze`.
    - `delete_content` reuses the same deletion code as `deletePost`/`deleteComment`. Factor out `deletePostById($pdo, $postId)` and `deleteCommentById($pdo, $id)` so the owner checks stay in the user handlers.
    - Set the status, `resolved_*` and `resolution`.
    - Resolving one report resolves **all** open reports on the same target with the same action.
  - `adminFreezeUser(userId)` / `adminUnfreezeUser(userId)`: set or clear `frozen_at`. Freezing also calls `sessionRevokeAllForUser()`.
- **`handle_login`** (and `sessionRefresh`): if `frozen_at` is set → `bad('This account has been suspended. Contact <contact address> if you think this is a mistake.', 403)`. The message uses `CONTACT_EMAIL` from config.
- **`deleteAccount`**: also delete the user's `blocks` rows in both directions and their own `reports` as reporter. **Keep** reports about them, with the snapshot, as the moderation record. Set `target_user_id` to NULL.

### A5. Verify Phase A on the bench
Seed users alice, bob and carol, and make alice an admin (`UPDATE users SET is_admin=1 WHERE email='alice@test.local'`). Then check each of these:
1. bob posts and carol follows bob. carol blocks bob → carol's feed has no bob posts, `getUserPosts(bob)` → 404, carol's search lacks bob, bob's search lacks carol, and the follow is gone. bob comments on carol's post → 404. bob likes carol's post → no notification for carol.
2. carol reports bob's post → that post disappears from carol's feed only. `mail.log` contains the report email without carol's address. A repeat report → still `valid:true`, and still one row.
3. alice calls `adminListReports` → sees it. `adminResolveReport(delete_and_freeze)` → the post is gone, bob's login → 403 suspended, bob's existing JWT → 401 on the next call, bob's other posts are hidden from everyone.
4. A non-admin calling any `admin*` endpoint → 403.
5. `acceptTerms(TERMS_VERSION)` → `getMyInfo` shows the accepted version. A wrong version → 400.
6. A plain old-client request flow (getMyInfo, feed, post, like) works without `acceptTerms`, which is backwards compatible.
7. `EXPLAIN QUERY PLAN` on the filtered feed query still uses the posts index.

**Commits:** one per endpoint group: `moderation: blocks`, `moderation: reports`, `moderation: admin actions + freeze`, `terms: acceptance tracking`.

---

## 1B.3 Phase B: web (ssreact)

- **B1. API client:** add the new methods, plus the `getMyInfo` fields in `types.ts`. If ssreact-native's Phase 1 core extraction has landed, this goes in `src/core/api-client.ts`, so the phone app gets it through sync-core.
- **B2. Report UI:**
  - A "…" menu on `PostCard`, `CommentItem` and the profile header with "Report".
  - It opens a dialog with the reason radio list, an optional details field and a confirmation toast ("Thanks, we'll review this").
  - After reporting, remove the item from the current list straight away.
- **B3. Block UI:**
  - Profile "…" → "Block <email>", confirmed with an `AlertDialog` explaining what blocking does. After blocking, navigate back to the feed and clear `feed-cache`.
  - Settings gets a "Blocked users" card with Unblock buttons.
- **B4. Terms gate:**
  - After login, and on app load with a stored user, if `termsVersionAccepted < termsVersionCurrent`, show a full-screen non-dismissible acceptance page. It links Terms and the Privacy Policy, and has a checkbox + Agree. It works like the existing session-expired modal pattern.
  - On register, the final step shows the same checkbox, and `acceptTerms` is called right after the first login.
- **B5. Legal pages**, as new public routes `/terms` and `/privacy` like `/conduct`, linked from the footer or Header, the landing page and Settings:
  - **Terms of Use (DRAFT for Dave):**
    - zero tolerance for objectionable content and abusive users, building on the existing Code of Conduct page's rules;
    - reporting and blocking;
    - account suspension;
    - account deletion;
    - contact address.
  - **Privacy Policy (DRAFT for Dave):**
    - what's collected: email, password hash, posts, comments, media, likes, follows, push tokens, device/browser name, IP addresses in logs, and log retention (pending the ssapi plan's open decision);
    - that **emails are visible to all users** (the recorded design);
    - no ads, no tracking, no selling of data;
    - where data lives (the SQLite DB on the server);
    - how to delete your account;
    - contact address.
  - Contact address: **DECISION**, Dave provides one.
- **B6. Admin page** `/admin` (route guarded on `user.isAdmin`):
  - Open reports list (target type, reason, snapshot, author, reporter count), each with Dismiss / Delete content / Freeze user / Delete + freeze. Confirm destructive actions.
  - A "Resolved" tab.
  - Link it from Settings, admins only.
- **Verify on `pnpm dev` against the bench:** the full A5 scenarios again through the UI, plus `pnpm lint && pnpm build`.

---

## 1B.4 Phase C: phone (ssreact-native), its Phase 9.0
The same features in native form:
- Long-press or a "…" action sheet on posts and comments → Report.
- Profile header → Report / Block.
- Settings → Blocked users.
- Terms gate screen after login and on register.
- Settings links to `/terms` and `/privacy` on the website.

See the phone plan's Phase 9. Because Phase B1 puts the API methods in `src/core`, the phone app gets them through `sync-core`.

---

## 1B.5 Phase D: before the Unlisted App Store submission (Dave + agent)
1. **Prod must run ssapi, with this plan and the Expo push additions included.** Store builds point at `app.davidfruin.com`, which today runs [[simple-social]]'s **separate copy** of the backend, which has none of this (ssapi plan D6). **This is the critical-path decision for the whole store release.** Dave decides how ssapi replaces that copy on app and dev. After deploying, his checks: log in, view the feed, create and delete a post, upload media, and the vanilla web app and the TUI still work.
2. Set `ADMIN_REPORT_EMAIL`, `CONTACT_EMAIL`, `TERMS_VERSION=1` and `TERMS_URL` in each host's `private/.env`.
3. `UPDATE users SET is_admin=1 WHERE email='<Dave's account>'` on prod (Dave runs it).
4. Dave reviews and approves the Terms and Privacy texts. Deploy ssreact on react.davidfruin.com, and wherever the prod web frontend lives once migrated. **The store listings need public URLs for both pages.** If ssreact isn't on app.davidfruin.com yet, host them on a public URL Dave chooses: either the vanilla app's static pages or react.davidfruin.com.
5. **Reviewer demo account:** a normal account on prod for Apple and Google reviewers, following some test content. Its credentials go **only** into App Store Connect / Play Console review notes. Never into a repo or this vault.
6. Do an end-to-end moderation drill on prod with two test accounts: report → email arrives → resolve from `/admin`. Then delete the test data.

---



---

# STEP 2 (later, GATED): public launch as an invite-only, paid social network

**Nothing here starts without Dave's explicit go.** It's a substantial project. Rough agent effort is **4–7 weeks**, plus Dave's time on accounts, legal, money and moderation.

## 2.1 What "public, invite-only, paid" means (proposed model, to agree with Dave)
- **Invites stay free, and payment is a subscription.** Members get a small, real number of invite codes, for example 3 per month, issued by the server. New members pay a monthly or yearly subscription after registering.
- **Family stays free:** existing accounts get a permanent `comp` (complimentary) entitlement.
- **Growth pacing is real and honest:** the number of invites issued **is** the scaling control. Dave can raise or lower the invites per member, or pause new issuance, and the site can truthfully say "Invites are limited while we grow."
- **Don't sell invite codes as access keys.**
  - Apple's guideline 3.1.1 forbids unlocking app functionality with codes or license keys bought outside in-app purchase.
  - The rules on selling digital access outside the stores differ by country and have changed recently (for example, US court-ordered changes in 2025 around linking to external purchases).
  - **Check the current App Store and Play rules at the time.** The safe design is a subscription sold through **in-app purchase** on iOS and Android, and **Stripe** on the web, with all of them unlocking the same server-side entitlement.

## 2.2 Payments and entitlements
- **Server (ssapi):**
  - new columns on `users`: `subscription_status` (`none | active | grace | expired | comp`), `subscription_source` (`apple | google | web | comp`), `subscription_expires_at`;
  - a `subscription_events` audit table.
  - **Access rule:** a new account without an active or comp entitlement can log in, but gets a "Subscribe to continue" screen. Its API calls return 402 except for `getMyInfo`, the subscription endpoints, logout and `deleteAccount`.
  - Existing family accounts are migrated to `comp`.
- **iOS and Android:** in-app subscriptions.
  - **Recommended:** **RevenueCat** (`react-native-purchases`). It handles receipt validation, renewals and refunds, and sends a webhook to ssapi, which updates the entitlement. It's free below a revenue threshold; check current pricing.
  - Doing it directly instead means App Store Server Notifications V2 + Google Real-Time Developer Notifications with server-side verification, which is more work for a solo developer.
- **Web:** Stripe Checkout + Customer Portal, plus a webhook endpoint (signature-verified) that updates the same columns.
  - Use Stripe Tax, or a merchant-of-record service, for sales tax and VAT on web sales. The app stores handle tax for in-app purchases.
- **Store requirements for subscriptions:**
  - price, period and auto-renew terms shown before purchase;
  - links to the Terms and Privacy;
  - a **Restore Purchases** button;
  - account deletion must explain that the store subscription is cancelled separately.
- **Fees:** Apple's and Google's small-business rates are about 15% under $1M a year. Stripe charges about 3% per transaction.

## 2.3 Business and legal (Dave, with professional advice: this plan isn't legal or tax advice)
- **Individual or organization developer account:**
  - With in-app sales, an **individual** account shows Dave's legal name as the seller everywhere.
  - Google Play also **publicly shows a monetizing developer's contact address**.
  - Many solo developers form an LLC (with a D-U-N-S number) and switch to organization accounts before charging money.
  - **DECISION before Step 2.**
- Agreements, banking and tax forms in App Store Connect (Paid Apps Agreement) and Play Console (payments profile).
- Terms of Use with subscription and refund terms, and a Privacy Policy covering payments (processors receive limited data), GDPR/CCPA rights if EU or California users join, and a **minimum age (13+, 16+ in parts of the EU)** enforced at registration with an age confirmation.
- Content obligations for a public service hosting photos and video. For example, US providers must report apparent child sexual abuse material to NCMEC when they become aware of it. Research the obligations, and decide on a response process and tooling (for example, a hash-matching service) before opening to the public.

## 2.4 Moderation (already built in Step 1B)
Step 1B's report/block/admin/terms work carries over. In Step 2, **members also gain the ability to generate their own invites** (§2.1), using the same `invite_codes` table with `created_by` = the member and a per-member monthly allowance. Raise the moderation capacity to match (§2.6).

## 2.5 Public store release
This is the phone plan's **Phase 10** (built on the same App Store listing from Phase 9). It covers: store identity, listing content, privacy questionnaires, age rating, review notes, TestFlight external/production, Google's closed test (at least 12 testers for 14 days for personal accounts) and the Apple submission checklist. It also includes the subscription disclosures from §2.2. Android moves from the APK to Google Play here.

## 2.6 Running a public service
- **Capacity:** one PHP server with one SQLite file is fine for family and for a paced invite-only launch: hundreds to low thousands of users, especially with WAL (ssapi plan P3b) and the query fixes.
  - Watch disk (media), DB size, and slow requests.
  - Before pushing past that: move media to object storage and consider Postgres or MySQL. That is a separate, later plan.
- **Backups:** nightly `sqlite3 userdata.db ".backup …"` + a media sync off the server, with a tested restore.
- **Monitoring:** uptime checks, error-log alerts and disk alerts.
- **Moderation duty:** reports email the admin. Commit to a response time (about 24h, which Apple expects) or add more moderators (`is_admin`).
- **Prod/dev separation:** keep test accounts and test data off prod.

## 2.7 Step 2 open decisions for Dave
1. The **business model**: price, monthly/yearly, invites per member, family `comp`.
2. **LLC + organization accounts** before charging money (to avoid showing his legal name and address).
3. **Payments stack:** RevenueCat + Stripe (recommended) or direct integrations.
4. Contact email, Terms and Privacy approval, minimum age.
5. Moderation response commitment, and who moderates.
6. **Usernames instead of public emails:** strongly recommended before going public. Showing every member's email to all members is a privacy risk at public scale; it's an acceptable design for a family-only app.
7. Whether to also list on the web: Stripe, with the regional rules on in-app links to web purchases checked at the time.

## Estimates
- **Step 1 (invites):** backend about 1 day, web about 1 day; the phone field is included in the phone plan.
- **Step 1B (moderation, terms, privacy):** backend 2–3 days, web 2–3 days, phone UI in phone plan 9.0. The legal text depends on Dave's review.
- **Step 2:** payments and entitlements 1–2 weeks; usernames (if chosen) about 1 week; the public store release (phone Phase 10) 2–3 weeks, including Google's 14-day test.
  - **Total: about 4–6 weeks** after Dave's go, not counting LLC and account setup time.
