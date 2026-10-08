---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi (backend), ssreact workspace (web/ = web app, mobile/ = phone app, packages/core/ = shared @ss/core)
replaces: Inbox/store-readiness-plan.md (its moderation spec is now Step 2 below, unchanged in substance)
---

# Access and public launch plan

**Dave's direction (2026-10-02):**
- **Step 1, now: family only.**
  - Android gets an **APK file**.
  - iPhone gets an **Unlisted App Store** listing, with TestFlight in the meantime.
  - Registration becomes **invite-only with free codes** that Dave hands out.
- **Step 1B, right after:** **report, block, admin moderation, terms and a privacy policy.** Apple's full App Store review applies to Unlisted apps too, and its guideline 1.2 requires these for apps with user posts. It's also good practice for family use.
- **Step 2, the end goal:** a **normal public app on both the App Store and Google Play**, as an **invite-only, paid** social network.

Steps 1 and 1B come first, for the family release. **Step 2 is the planned final stage, not optional.** It starts once the iPhone Unlisted listing is live and stable. Only its money, legal and business choices (§2.7) need Dave's decision before the agent builds the related parts.

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

# STEP 1 (now): invite-only registration, one invite per member

> **Redesigned 2026-10-08 by Dave**, replacing the earlier admin-made code batches. His words: "The codes should be auto generated in people's profile page. It should be an complicated code so people can't guess it. Symbols letters (upper and lower) numbers and it should be 8 digits long. There should be a note that says you can only invite one person so choose wisely! As owner i should have infinite codes. There should be a data field in a users info in the database that shows who invited them to the platform. It shouldnt be visible tho. For the current users put their invite as from me as if I invited them. On the register page there should be just a code field and doesnt even offer them an email field until they enter a valid code. Everybody's code on their profile page should generate when they click a generate code button and it should only be valid for 1 week."

## 1.1 Behaviour
- **Every member can invite exactly one person.** The **owner** (Dave) can invite as many as he likes.
- **Making a code:** on **your own profile page**, an "Invite someone" card has a **Generate code** button, and the note **"You can only invite one person, so choose wisely!"**
  - The code is valid for **7 days** and works **once**.
  - The invite is only **spent when someone joins with it**. An unused code that expires can be replaced by generating a new one.
  - **Members** have at most one active code at a time and can **cancel** it, for example if they sent it to the wrong person. Then they can generate a new one.
  - **The owner** can have any number of active codes.
  - Once someone joins with a member's code, the card says "You've used your invite." It never says who.
  - **If the person you invited later deletes their account, your invite stays spent** ("choose wisely").
- **The code:** 8 characters, with at least one uppercase letter, one lowercase letter, one number and one symbol, e.g. `kT7$mQ2x`.
  - Characters that look alike are left out: `0 O o`, `1 l I`.
  - Symbols come only from `! $ & @ ?`. They're on the first symbol page of iPhone and Android keyboards, and chat apps don't turn them into formatting: WhatsApp turns `*`, `_` and `~` into bold, italic and strikethrough, and phones curl quotes.
  - A code never **starts or ends** with a symbol, so a trailing `!` or `?` can't be mistaken for punctuation.
  - Codes are **case-sensitive**. Code fields turn off auto-capitalisation and autocorrect.
  - With 61 possible characters, there are about 10¹⁴ possible codes. Combined with the guessing limit below and the few codes alive at any time, guessing is hopeless.
- **Nothing to guess after a week:** expired codes and used codes are **deleted**. If nobody has generated a code in the last week, no valid code exists at all.
- **Registering:** the Register page shows **only an "Invite code" field** at first.
  - The email field appears only after the server accepts the code. The rest of registration (email code, password) is unchanged.
  - The server checks the code again when the account is created, and only then uses it up. If two people try the same code, exactly one gets in.
  - A code made by an account that has since been frozen or deleted no longer works.
- **Guessing protection:**
  - Every wrong code counts against the person's IP through the existing attempt limiter (20 per 15 minutes, then blocked for a while).
  - The message is always the same: "That invite code isn't valid." It doesn't say whether the code was wrong, expired or used, so it tells a guesser nothing.
- **Who invited whom:** `users.invited_by` stores the inviter's user id.
  - **No API ever returns it,** and it doesn't show in either app.
  - Dave can read it with `sqlite3` (§1.6).
  - **Existing users are set as invited by Dave** (§1.6).
- **The owner:** this step adds the `users.role` column, but only uses the value `owner`. Dave sets it on himself by hand. The [[staff-roles-plan]] adds moderator and admin to the same column later.
- **Mode switch:** `REGISTRATION_MODE` in `private/.env`, `invite` or `open`.
  - The **default in code is `invite`**, so a missing setting fails closed.
  - `open` is for local testing and is never set on prod without Dave.
  - Later, this becomes an owner-only app setting (staff roles plan §7).
- **Existing accounts and logins are unaffected.**

## 1.2 Backend (ssapi)
**Migration** (the next free `migrationN`; check `SCHEMA_VERSION` first, and never edit a shipped migration):
```sql
CREATE TABLE IF NOT EXISTS invite_codes (
  code TEXT PRIMARY KEY,          -- exactly as generated; compared case-sensitively (SQLite's default BINARY collation)
  created_by INTEGER NOT NULL,
  created_at TEXT NOT NULL,
  expires_at TEXT NOT NULL);      -- created_at + 7 days
CREATE INDEX IF NOT EXISTS idx_invite_codes_created_by ON invite_codes(created_by);
```
Then add these columns, using the column-exists guard pattern:
- **`users.role`:** `TEXT NOT NULL DEFAULT 'user'`.
- **`users.invited_by`:** `INTEGER`. NULL until the backfill in §1.6.
- **`users.invite_used_at`:** `TEXT`. Set when someone joins with this user's code.
- **`pending_users.invite_code`:** `TEXT`.

**New module `src/Invites/handlers.php`** (add it to `composer.json`'s `autoload.files`, then run `composer dump-autoload`):
```php
const INVITE_UPPER = 'ABCDEFGHJKLMNPQRSTUVWXYZ';   // no I, O
const INVITE_LOWER = 'abcdefghijkmnpqrstuvwxyz';   // no l, o
const INVITE_DIGITS = '23456789';                  // no 0, 1
const INVITE_SYMBOLS = '!$&@?';
const INVITE_LENGTH = 8;
const INVITE_DAYS = 7;

// 8 characters with at least one of each class; symbols never first or last.
function newInviteCode(): string {
    $all = INVITE_UPPER . INVITE_LOWER . INVITE_DIGITS . INVITE_SYMBOLS;
    $pick = fn($set) => $set[random_int(0, strlen($set) - 1)];
    while (true) {
        $code = '';
        for ($i = 0; $i < INVITE_LENGTH; $i++) {
            $code .= $pick(($i === 0 || $i === INVITE_LENGTH - 1) ? INVITE_UPPER . INVITE_LOWER . INVITE_DIGITS : $all);
        }
        if (preg_match('/[A-Z]/', $code) && preg_match('/[a-z]/', $code) && preg_match('/[0-9]/', $code)
            && strpbrk($code, INVITE_SYMBOLS) !== false) return $code;
    }
}

function registrationNeedsInvite(): bool {
    return strtolower(getenv('REGISTRATION_MODE') ?: 'invite') !== 'open';
}

function deleteExpiredInvites($pdo): void {
    $pdo->prepare('DELETE FROM invite_codes WHERE expires_at <= ?')->execute([date('Y-m-d H:i:s')]);
}

// The live code's row (with created_by), or exits with the one generic message.
// Rate-limited per IP. Codes from frozen or deleted accounts don't count.
function requireValidInvite($pdo, string $raw): array {
    $keys = ['ip' => attemptKeys('invite', '-')['ip']];
    checkAttemptLimit($pdo, $keys);
    deleteExpiredInvites($pdo);
    $s = $pdo->prepare('SELECT i.code, i.created_by, i.expires_at FROM invite_codes i
        JOIN users u ON u.id = i.created_by AND u.frozen_at IS NULL WHERE i.code = ?');
    $s->execute([trim($raw)]);   // trim only: codes are case-sensitive
    $row = $s->fetch(PDO::FETCH_ASSOC);
    if (!$row) { recordFailedAttempt($pdo, $keys); bad("That invite code isn't valid.", 400); }
    return $row;
}
```
`recordFailedAttempt` takes a keys map; passing only `ip` works, because its limits map has an `ip` entry. Confirm that still holds in the current code.

**Endpoints:**
- **`checkInviteCode`** (public, the Register page's first step): `requireValidInvite`, then `{valid: true, expiresAt}`.
- **`getMyInvites`** (logged in):
  - `{unlimited, canGenerate, used, codes: [{code, expiresAt}]}`.
  - `unlimited` is true for the owner. `used` means `invite_used_at` is set. `codes` are the caller's live codes.
- **`generateInviteCode`** (logged in):
  - **Owner:** always makes a new code.
  - **Member who has already used their invite:** 403 `You've already used your invite.`
  - **Member with a live code:** 400 `You already have an active code. Cancel it to make a new one.`
  - **Otherwise:** insert `newInviteCode()` with a 7-day expiry. Retry on the rare primary-key collision. Returns `{code, expiresAt}`.
  - Frozen accounts can't log in, so they can't generate.
- **`cancelInviteCode`** (logged in): params `code`. Deletes it only if it's the caller's own code; the same 400 message otherwise.

**Registration changes:**
- **`handle_sendRegisterOTP`:** when `registrationNeedsInvite()`, call `requireValidInvite($pdo, $_POST['inviteCode'] ?? '')` **first**, before the send throttles and before any email. Save the code on the `pending_users` row.
- **`handle_verifyRegisterOTP`:** unchanged.
- **`handle_finishRegister`:** inside the existing `BEGIN IMMEDIATE` transaction, before the `INSERT`:
  ```php
  if (registrationNeedsInvite()) {
      $s = $pdo->prepare('SELECT i.created_by FROM invite_codes i JOIN users u ON u.id = i.created_by AND u.frozen_at IS NULL
          WHERE i.code = ? AND i.expires_at > ?');
      $s->execute([$pendingRow['invite_code'] ?? '', date('Y-m-d H:i:s')]);
      $inviterId = $s->fetchColumn();
      if ($inviterId === false) { $pdo->exec('ROLLBACK'); bad('That invite code has expired or was already used. Ask for a new one.', 400); }
      $pdo->prepare('DELETE FROM invite_codes WHERE code = ?')->execute([$pendingRow['invite_code']]);
      $pdo->prepare('UPDATE users SET invite_used_at = ? WHERE id = ? AND role != ?')->execute([date('Y-m-d H:i:s'), $inviterId, 'owner']);
  }
  ```
  - Then insert the user with `invited_by = $inviterId` (NULL in `open` mode).
  - `BEGIN IMMEDIATE` makes the select-then-delete atomic, so two simultaneous registrations can't both use one code.
  - Add `invite_code` to the `SELECT` that reads the pending row.
- **Deleting an account** (`deleteAccount`, and the staff plan's `deleteUserAndData` later) also deletes that user's `invite_codes` rows.
- **`getMyInfo`:**
  - Add `registrationMode` (`invite`/`open`). It's additive.
  - **Never** add `invited_by` to it, or to any other response.
- **Messages:** every new message gets its Spanish in `src/I18n/es.php`, and `check-messages.php` must pass ([[language-plan]]).
- **API docs** (`ssreact/web/src/content/api-docs.html`, English): the new endpoints, the `inviteCode` parameter, and the code rules.
- **`.env.example`:** `REGISTRATION_MODE=invite`.

**Verify on the bench (`sstests/backend/invites/run.sh`):**
1. **Code format:** 1,000 calls to `newInviteCode()` all match the rules: 8 characters; every class present; only the allowed alphabet; no symbol first or last; no `0 O o 1 l I`.
2. **A member's one invite:**
   - Alice (a member) generates a code. A second generate → 400 (active code). Cancel, then generate again → OK.
   - Bob registers with Alice's code. Alice's `getMyInvites` now says `used`, and generating → 403.
   - In the database: Bob's `invited_by` = Alice's id, and the code row is gone.
3. **Owner:** with `role='owner'` set by hand, the owner generates 3 codes in a row → 3 live codes. Using one doesn't stop the owner generating more.
4. **The register checks:**
   - `checkInviteCode` with a wrong code, the right code in the wrong case, an expired code (set `expires_at` in the past) and a used code → all the same 400 message.
   - `sendRegisterOTP` without a valid code sends **no** email.
   - 21 wrong codes from one IP → 429.
5. **Race:** two pending registrations holding the same code, finished at the same moment → exactly one account.
6. **Frozen inviter:** freeze Alice while her code is live → the code stops working.
7. **Mode switch:** `REGISTRATION_MODE=open` → registration works without a code, and `invited_by` is NULL. Remove the setting → invite mode.
8. **Not leaked:** no response anywhere contains `invited_by`. Grep the responses of `getMyInfo`, `getUserInfo`, `getUsers` and `getStaff` (if present).
9. **Regression:** existing users still log in, and the i18n, moderation and media suites still pass.

**Commits:** `invites: schema (invite_codes, users.role/invited_by/invite_used_at)`, `invites: one code per member, unlimited for the owner`, `invites: code-first registration`, `api docs: invites`.

## 1.3 Web (ssreact)
- **Register page** (`OtpAuthFlow`, register mode):
  - **Step 0 is only an "Invite code" field** and Continue, with the hint "Simple Social is invite-only. Enter the code someone gave you." It calls `checkInviteCode`.
  - On success the existing email step appears, and the code is sent with `sendRegisterOTP`.
  - The field turns off `autoCapitalize`, `autoCorrect` and `spellCheck`, uses a monospace font, and has a show/hide toggle.
  - `/register?invite=<code>` (URL-encoded) fills the field in and checks it at once.
  - When the server says `registrationMode: open`, skip step 0. Before login that isn't known, so always show step 0 unless a check says otherwise.
- **Your own profile page: an "Invite someone" card,** shown only on your own profile (driven by `getMyInvites`):
  - **Member, no code yet:** the note **"You can only invite one person, so choose wisely!"** and a **Generate code** button. Confirm first: "Generate your one invite code? It works for 7 days."
  - **Member with a live code:** the code in large monospace, "Valid until <date>", **Copy**, **Share**, and **Cancel code**.
    - Share uses the Web Share API where it exists, otherwise copies the message. The message puts the code on its own line, with the website and the download page: "Join me on Simple Social!", then the code, then "Valid for 7 days. Codes are case-sensitive.", then the links.
  - **Member who has used their invite:** "You've used your invite." Nothing else, and never who.
  - **Owner:** "As the owner you can invite as many people as you like." It shows Generate code and a list of live codes, each with its expiry, Copy, Share and Cancel.
- **Client code:** the API methods go in `packages/core/src/api-client.ts` (`checkInviteCode`, `getMyInvites`, `generateInviteCode`, `cancelInviteCode`). The `sendRegisterOTP` client method gains `inviteCode` (the phone app already passes it).
- **Translate everything** with `tr()` and Spanish in `es.ts`. The note in Spanish: "Solo puedes invitar a una persona, ¡así que elige bien!" (the reviewer may adjust it).
- **Verify** with `pnpm dev` against the bench:
  - Register end to end: code first, then email, reading the OTP from `mail.log`.
  - The profile card in each state: member with no code, live code, used, and owner.
  - `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build`.

## 1.4 Phone (`ssreact/mobile/`)
The phone app's Register screen already has an invite field next to the email (`OtpAuthFlow`, `askInviteCode`). Rework it to match the web:
- **Code-first:** a code-only step, then the email step.
- **The code field:** `autoCapitalize="none"`, `autoCorrect={false}`, and a monospace font.
- **The profile card:** the same "Invite someone" card on your own profile in `ProfileView`, with the system share sheet.
- **Verify:** `npx expo lint` and `npx tsc --noEmit` pass. This is JavaScript only, so it can ship in the family build.

## 1.5 Other clients
- **Terminal clients** ([[simple-social-cli]], [[simple-social-cli-interactive]], [[simple-social-tui]]):
  - Login and everything else keeps working.
  - **Registration from them fails** until each one adds a code step before the email. That's acceptable for family use: people register on the web or phone, then log in anywhere.
  - Adding that step is a small, separate job in each repo, not part of this plan.

## 1.6 Step 1 rollout (Dave)
1. **Deploy ssapi** (dev first, then prod):
   1. Back up the database.
   2. Deploy, then run `composer dump-autoload` for the new `src/Invites` module.

   The migration runs on the first request. Leave `REGISTRATION_MODE` unset, which means invite mode.
2. **Make yourself the owner and record existing users as invited by you**, once on dev and once on prod:
   ```
   sqlite3 private/userdata.db "UPDATE users SET role = 'owner' WHERE email = '<your login email>';"
   sqlite3 private/userdata.db "UPDATE users SET invited_by = (SELECT id FROM users WHERE role = 'owner') WHERE invited_by IS NULL AND role != 'owner';"
   ```
3. **Deploy the web app**, and the phone app with its family build (phone plan Phase 8).
4. **To see who invited whom** (it's never shown in the app):
   ```
   sqlite3 private/userdata.db "SELECT u.email, i.email AS invited_by FROM users u LEFT JOIN users i ON i.id = u.invited_by ORDER BY u.id;"
   ```
5. **Invite family:** generate a code on your profile page and send it with the APK link (Android) or the TestFlight link (iPhone).

---

# STEP 1B (right after the family release): moderation, terms, privacy
**Phases A-C done 2026-10-07 (local only, not deployed).** A: ssapi d768449, 78cb8b3, acc5e37, e0c0fe2. B: ssreact 8c9312e, f18cf0c, 8dc47f8, b83a0bd, 31cc1ce, d331e78 (plus the React fix 5b3eeaa). C: ssreact 80e523a, 9bbe3ef. Phase D is Dave's. Details and deviations in [[ssapi]] and [[ssreact]].

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

**Not in Step 1B:** payments (Step 2). Member-generated invites are already in Step 1: one per member, unlimited for the owner.

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

- **B1. API client:** add the new methods, plus the `getMyInfo` fields in `types.ts`. After the phone plan's Phase 1, this goes in `packages/core/src/api-client.ts` (`@ss/core`), so web and phone share it directly.
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

## 1B.4 Phase C: phone (`ssreact/mobile/`), its Phase 9.0
The same features in native form:
- Long-press or a "…" action sheet on posts and comments → Report.
- Profile header → Report / Block.
- Settings → Blocked users.
- Terms gate screen after login and on register.
- Settings links to `/terms` and `/privacy` on the website.

See the phone plan's Phase 9. Because Phase B1 puts the API methods in `@ss/core` (`packages/core`), the phone app gets them directly through the shared workspace package.

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

# STEP 2 (the end goal): public launch on both stores as an invite-only, paid social network

**This is the planned final stage.** It starts after the iPhone Unlisted listing (phone plan, Phase 9) is live and stable. Items that depend on Dave's §2.7 decisions (price, LLC/organization accounts, payments stack, usernames) wait for those decisions; everything else can proceed. It's a substantial project. Rough agent effort is **4–7 weeks**, plus Dave's time on accounts, legal, money and moderation.

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
Step 1B's report/block/admin/terms work carries over. In Step 2, the **per-member allowance changes** from Step 1's one invite for life to the §2.1 model (e.g. a monthly allowance), using the same `invite_codes` table. Raise the moderation capacity to match (§2.6).

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
- **Step 1 (invites, redesigned 2026-10-08):** backend about 1 day, web about 1 day, phone about half a day.
- **Step 1B (moderation, terms, privacy):** backend 2–3 days, web 2–3 days, phone UI in phone plan 9.0. The legal text depends on Dave's review.
- **Step 2:** payments and entitlements 1–2 weeks; usernames (if chosen) about 1 week; the public store release (phone Phase 10) 2–3 weeks, including Google's 14-day test.
  - **Total: about 4–6 weeks** after Dave's go, not counting LLC and account setup time.
