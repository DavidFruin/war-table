---
status: proposal
written: 2026-10-02
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi (backend), ssreact (web: admin screen, legal pages, report/block UI), ssreact-native (report/block UI, terms gate)
---

# Store readiness plan: moderation, terms, privacy

**Why this exists:** Dave decided on 2026-10-02 that the phone app ([[ssreact-native]]) must ship on **both the Apple App Store and Google Play**, using **individual** developer accounts.

Both stores have rules for apps where users post content. Apple's App Review Guideline 1.2, "User-Generated Content", requires:
- a way to **filter objectionable material**;
- a way to **report** offensive content, with **timely responses**;
- a way to **block** abusive users;
- published **contact information**.

Apple also expects users to agree to terms that forbid objectionable content. Google Play's UGC policy asks for much the same.

[[simple-social]] has none of these yet; they are only listed under "Future ideas" as an admin panel and "report a person". This plan adds them. It runs **in parallel with** the phone port (`Inbox/ssreact-native-port-plan.md`), and the phone app's store submission depends on it.

---

## 0. Rules for the implementing agent

Everything in section 0 of `Inbox/ssapi-improvement-plan.md` applies here too:
- one task per commit;
- stop after each phase;
- verify on the local ssapi bench (its Phase 0);
- never deploy and never touch prod;
- no tests folder in ssapi;
- the vault is public.

Additional rules:
1. **Sequencing:** start after the ssapi plan's Phase 3 has landed. This plan relies on its migration mechanism (`ensureSchema`/`migrationN`, P3), on `pageParams`/`jsonIdList` (S8), and on carrying `$user['email']` through the session lookup (P9). If P3 hasn't landed, stop and tell Dave instead of inventing a second migration system.
2. **Everything is additive.** The vanilla web app, the CLI, the wizard and the TUI all keep working unchanged. They get server-side block and freeze filtering automatically, but no report or block UI. That UI for them is a separate, later job.
3. **The legal text is Dave's.** The agent drafts the Terms of Use and the Privacy Policy, marked **DRAFT, needs Dave's review**. They are not legal advice. Dave approves or edits them before any store submission.
4. **Moderation data is sensitive.** Report details and block lists are never shown to anyone except the reporter, the blocker and admins.

---

## 1. Behaviour spec (agree this with Dave before coding)

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

## 2. Phase A: backend (ssapi)

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

## 3. Phase B: web (ssreact)

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

## 4. Phase C: phone (ssreact-native), folded into its port plan
The same features in native form:
- Long-press or a "…" action sheet on posts and comments → Report.
- Profile header → Report / Block.
- Settings → Blocked users.
- Terms gate screen after login and on register.
- Settings links to `/terms` and `/privacy` on the website.

See the phone plan's Phase 7. Because Phase B1 puts the API methods in `src/core`, the phone app gets them through `sync-core`.

---

## 5. Phase D: before the first store submission (Dave + agent)
1. **Prod must run ssapi, with this plan and the Expo push additions included.** Store builds point at `app.davidfruin.com`, which today runs [[simple-social]]'s **separate copy** of the backend, which has none of this (ssapi plan D6). **This is the critical-path decision for the whole store release.** Dave decides how ssapi replaces that copy on app and dev. After deploying, his checks: log in, view the feed, create and delete a post, upload media, and the vanilla web app and the TUI still work.
2. Set `ADMIN_REPORT_EMAIL`, `CONTACT_EMAIL`, `TERMS_VERSION=1` and `TERMS_URL` in each host's `private/.env`.
3. `UPDATE users SET is_admin=1 WHERE email='<Dave's account>'` on prod (Dave runs it).
4. Dave reviews and approves the Terms and Privacy texts. Deploy ssreact on react.davidfruin.com, and wherever the prod web frontend lives once migrated. **The store listings need public URLs for both pages.** If ssreact isn't on app.davidfruin.com yet, host them on a public URL Dave chooses: either the vanilla app's static pages or react.davidfruin.com.
5. **Reviewer demo account:** a normal account on prod for Apple and Google reviewers, following some test content. Its credentials go **only** into App Store Connect / Play Console review notes. Never into a repo or this vault.
6. Do an end-to-end moderation drill on prod with two test accounts: report → email arrives → resolve from `/admin`. Then delete the test data.

---

## 6. Open decisions for Dave
1. **The contact email address** shown in the Terms, the Privacy Policy, the suspension message and the store listings. It must be monitored, because Apple expects timely responses to reports.
2. Whether a mentioned blocked user renders as "@deleted user" (recommended) or still shows the email.
3. Whether to add the optional word filter.
4. **Approving** the Terms and Privacy texts.
5. **How ssapi gets to prod** (D6). The store release can't happen without it.
6. Not required, but reviewers may ask about it: emails are visible to everyone, by the recorded design. A separate username would be more private. It's a product decision, listed so it's a conscious choice.

## 7. Estimate
- Phase A: 2–3 days of agent work.
- Phase B: 2–3 days.
- Phase C: inside the phone port.
- Phase D: depends on Dave (D6 and the legal text).

Phases A and B can run while the phone port is in its Phases 1–5.
