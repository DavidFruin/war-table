---
status: proposal
written: 2026-10-08
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi @ 6fa06d2, ssreact @ 0595e5c (packages/core, web/, mobile/), sstests
---

# Staff roles plan: admins and moderators

Dave (2026-10-08): "I want there to be two levels of administrator. Admin and moderator. Admins can appoint other admins and moderators and can change the settings of the app when we later make an app settings dashboard."

**Today:**
- There is one level. `users.is_admin` is set by hand with `sqlite3` on the server, and there is no API to grant it (`src/Moderation/handlers.php`, `requireAdmin()`).
- Admins can list and resolve reports, delete reported posts and comments, and freeze or unfreeze accounts: the four `admin*` endpoints and the web `/admin` page.
- The phone app has no admin screens.
- The role is read from the database on every request, which this plan keeps. A role change therefore takes effect on the person's very next request, with no new login.

**After this plan:**
- Three roles: **user**, **moderator** and **admin**.
- Admins appoint and remove admins and moderators from a new **Team** tab, and see an **Activity** log of everything staff did.
- Moderators handle reports, except reports about other staff.
- Dave's own account is protected as the **owner**, so nobody can lock him out from inside the app.
- **App settings are not built here.** Section 6 sets the rules the later settings dashboard must follow, so it slots straight in.

---

## 0. Rules for the implementing agent
- The usual rules apply: check and claim `Areas/active-work.md` first, one task per commit, push after each commit, **never deploy**, never touch prod. Dave deploys.
- Tests go in [[sstests]] (`sstests/backend/staff/`), never in ssapi.
- The database change is the next free `migrationN`. `SCHEMA_VERSION` is 5 as of `6fa06d2`, so it's most likely `migration6`. Check first, and never edit a migration that has shipped.
- **Every new server message goes into `src/I18n/es.php`** with its Spanish (`sstests/backend/i18n/check-messages.php` must still pass). **Every new web text goes through `tr()`** with Spanish in `packages/core/src/i18n/es.ts` (the web lint rule enforces it). See [[language-plan]].
- **Mobile:** follow `mobile/AGENTS.md`. If the language plan's Phase 4 (phone) is in progress, do this plan's phone task after it.
- Skip anything marked **DECISION** until Dave answers. Use the stated default if he says "go" without answering.

---

## 1. Who can do what

| | User | Moderator | Admin | Owner (Dave) |
|---|---|---|---|---|
| Report and block (unchanged) | ✓ | ✓ | ✓ | ✓ |
| See reports; dismiss, delete reported content, freeze and unfreeze **regular users** | | ✓ | ✓ | ✓ |
| The same when the reported person is **staff** (moderator or admin) | | | ✓ | ✓ |
| See who reported something | | | ✓ | ✓ |
| See the team list | | ✓ (view only) | ✓ | ✓ |
| Appoint, change or remove admins and moderators | | | ✓ | ✓ |
| Freeze and unfreeze staff | | | ✓ | ✓ |
| See the Activity log | | | ✓ | ✓ |
| Change app settings (later, section 6) | | | ✓ | ✓ |
| Can lose their role or be frozen | — | by an admin | by another admin | **never from the app** |

**The rules behind the table:**
- **R1. The owner.**
  - `OWNER_EMAIL` in the server's `private/.env` names the owner. The owner is always treated as an admin, whatever the database says.
  - Nobody can change the owner's role or freeze the owner from the app. Changing the owner is a server edit only.
  - This is the lock-out protection: with several admins, any of them could otherwise remove all the others, Dave included.
  - **DECISION**, default yes.
- **R2. No changing your own role.** Another admin does it. This stops accidental self-demotion.
- **R3. Never zero admins.** The last unfrozen admin can't delete their account: "You're the only admin. Make someone else an admin before deleting your account."
  - R2 already means an admin can only lose the role, or be frozen, at another admin's hands, so those paths can't empty the admin list.
  - With an owner set this can't happen at all; the check guards a server with `OWNER_EMAIL` unset.
- **R4. Moderators never see reports about staff.**
  - Reports whose target belongs to a moderator or admin (a post, a comment or the account itself) are left out of moderators' report lists, and every action on them, including dismiss, is admin-only.
  - This prevents moderators burying complaints about themselves or each other, and from seeing who complained.
  - **DECISION**, default yes.
- **R5. Moderators don't see who reported.** `reporterEmail` is `null` for moderators and kept for admins. In a family app, knowing who reported you invites friction. **DECISION**, default hidden.
- **R6. Frozen accounts can't hold a new role.** Giving a role to a frozen account is refused ("Unfreeze this account first"). Freezing a staff member keeps their role, but they can't log in, so it does nothing until they're unfrozen.
- **R7. The server enforces everything.** The apps only hide buttons people can't use; every rule above is checked on the server.

---

## 2. Phase 1: server (ssapi, about a day)

### 1.1 Migration (`migration6`, `SCHEMA_VERSION` 6)
Follow `migration5`'s pattern (`PRAGMA table_info` check before `ALTER`):
```php
// users.role replaces users.is_admin (left in place, no longer read).
if (!in_array('role', $have, true)) {
    $pdo->exec("ALTER TABLE users ADD COLUMN role TEXT NOT NULL DEFAULT 'user'");
    $pdo->exec("UPDATE users SET role = 'admin' WHERE is_admin = 1");
}
// Who did what. Emails are copied in, so the log stays readable after an
// account is deleted.
$pdo->exec('CREATE TABLE IF NOT EXISTS staff_actions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    actor_id INTEGER, actor_email TEXT NOT NULL,
    action TEXT NOT NULL,              -- set_role | freeze | unfreeze | resolve_report | update_setting (later)
    target_user_id INTEGER, target_email TEXT,
    details TEXT,                      -- JSON, e.g. {"from":"user","to":"moderator"} or {"reportId":12,"resolution":"dismiss"}
    created_at TEXT NOT NULL)');
$pdo->exec('CREATE INDEX IF NOT EXISTS idx_staff_actions_created ON staff_actions(created_at)');
$pdo->exec('CREATE INDEX IF NOT EXISTS idx_users_role ON users(role)');
```
- Leave `is_admin` in the table but stop reading it. Add a comment in `schema.php`, and note that the ssapi improvement plan's D7 ("drop dead columns") now covers it too.
- Dave's existing admin flag on dev and prod becomes `role = 'admin'` automatically.

### 1.2 New module `src/Staff/handlers.php`
Add it to `composer.json`'s `autoload.files` and run `composer dump-autoload`. It holds the role helpers and the new endpoints:
```php
const ROLES = ['user', 'moderator', 'admin'];
const ROLE_RANK = ['user' => 0, 'moderator' => 1, 'admin' => 2];

// The owner (OWNER_EMAIL in private/.env) is always an admin and can't be
// changed or frozen from the app (staff-roles plan, R1).
function isOwnerEmail(?string $email): bool {
    global $CONFIG;
    $owner = strtolower(trim($CONFIG['owner_email'] ?? ''));
    return $owner !== '' && $email !== null && strtolower($email) === $owner;
}

// ['id', 'email', 'role' (effective), 'isOwner', 'frozen'], or null for an unknown id.
function staffInfo($pdo, int $userId): ?array {
    $s = $pdo->prepare('SELECT id, email, role, frozen_at FROM users WHERE id = ?');
    $s->execute([$userId]);
    $row = $s->fetch(PDO::FETCH_ASSOC);
    if (!$row) return null;
    $owner = isOwnerEmail($row['email']);
    $role = $owner ? 'admin' : (in_array($row['role'], ROLES, true) ? $row['role'] : 'user');
    return ['id' => (int)$row['id'], 'email' => $row['email'], 'role' => $role, 'isOwner' => $owner, 'frozen' => $row['frozen_at'] !== null];
}

// Stops the request unless the caller has at least $min. Returns the caller's staffInfo.
function requireRole($pdo, $user, string $min): array {
    $me = staffInfo($pdo, (int)$user['sub']);
    if (!$me || ROLE_RANK[$me['role']] < ROLE_RANK[$min]) {
        bad($min === 'admin' ? 'Admins only' : 'Moderators and admins only', 403);
    }
    return $me;
}

// Rules R1 and R4 for any action aimed at another account.
function requireCanActOn(array $me, ?array $target): void {
    if ($target === null) return;   // deleted account: nothing to protect
    if ($target['isOwner']) bad("That account can't be changed from the app.", 403);
    if ($target['role'] !== 'user' && $me['role'] !== 'admin') {
        bad('Only an admin can act on an admin or moderator.', 403);
    }
}

function logStaffAction($pdo, array $me, string $action, ?array $target, array $details = []): void { /* INSERT INTO staff_actions … */ }

// Unfrozen admins, counting the owner (rule R3, used by deleteAccount).
function activeAdminCount($pdo): int { /* role = 'admin' AND frozen_at IS NULL, plus the owner if their row says otherwise */ }
```
`config.php`: `$CONFIG['owner_email'] = getenv('OWNER_EMAIL') ?: '';` Add `OWNER_EMAIL=` to `.env.example`, with a comment.

### 1.3 Change the existing admin endpoints
Keep their names (`admin*`), so no client breaks; moderators now use them too.
- **Replace `requireAdmin()`** (`src/Moderation/handlers.php`) with `requireRole($pdo, $user, 'moderator')` in all four, keeping `$me` for the checks below. Delete `requireAdmin()` once nothing calls it.
- **`adminListReports`:**
  - For moderators (`$me['role'] === 'moderator'`), leave out reports whose `target_user_id` is staff (R4). Do it in SQL, e.g. `AND (r.target_user_id IS NULL OR r.target_user_id NOT IN (<staff ids>))`. Build the id list from `role IN ('moderator','admin')` plus the owner's id.
  - Set `reporterEmail` to `null` (R5).
  - Add `targetUserIsStaff` to every row, for the admin UI.
- **`adminResolveReport`:** load `staffInfo` for `target_user_id` and call `requireCanActOn($me, $target)` before **any** action, including dismiss. Then call `logStaffAction(… 'resolve_report', …, ['reportId' => …, 'resolution' => $act])`.
- **`adminFreezeUser` / `adminUnfreezeUser`:** call `requireCanActOn` before acting, then `logStaffAction('freeze' | 'unfreeze')`.
- Keep the existing `writeLog(...)` lines as well.

### 1.4 New endpoints (register them in `api.php`'s handler map)
- **`adminListStaff`** (moderator+): every account whose effective role isn't `user`, as `{ userId, email, role, isOwner, frozen }`. Admins first, then by email.
- **`adminSetRole`** (admin):
  - **Params:** `email` (for appointing someone new; matched case-insensitively) or `userId`, plus `role` (`user|moderator|admin`).
  - **Order of checks:**
    1. Invalid role → 400 `Invalid role`.
    2. Unknown account → 404 (`No account found with this email` is already in es.php).
    3. Owner → 403 (R1).
    4. Self → 400 `You can't change your own role` (R2).
    5. Frozen and the new role isn't `user` → 400 `Unfreeze this account before giving it a role` (R6).
  - **Then:** `UPDATE users SET role = ?`, then `logStaffAction('set_role', …, ['from' => …, 'to' => …])`, then respond with the member's new `{ userId, email, role, isOwner, frozen }`.
  - The same role again is a no-op success and isn't logged.
- **`adminListActivity`** (admin): newest first, paged with `pageParams()`, as `{ id, actorEmail, action, targetEmail, details (decoded), createdAt }` plus `hasMore`.

### 1.5 Other server changes
- **`getMyInfo`:**
  - Add `role` (effective) and `isOwner`.
  - **Keep `isAdmin`** (`role === 'admin'`), so older app builds keep working.
  - Read `role` instead of `is_admin`.
- **`deleteAccount`:** refuse the last unfrozen admin (R3), 400 with the message above.
- **API docs** (`ssreact/web/src/content/api-docs.html`, English): the roles, the new endpoints, `role`/`isOwner` in `getMyInfo`, and which endpoints moderators can call.
- **Report emails stay as they are** (`ADMIN_REPORT_EMAIL`). **DECISION**, optional: also email every admin and moderator. Default no; moderators check the Moderation page.

### 1.6 Verify on the bench (`sstests/backend/staff/run.sh`)
Bench users: **O** (owner, via `OWNER_EMAIL`), **A1** and **A2** (admins), **M** (moderator), **U1** and **U2** (users). Make the checks PASS/FAIL like the other suites:
- **Migration:** a database at version 5 with `is_admin = 1` on one user becomes version 6 with that user's `role = 'admin'`. `getMyInfo` returns the right `role` / `isOwner` / `isAdmin` for each user.
- **Report lists:** U1 calling `adminListReports` → 403.
  - M sees reports about U2 but not the report about A1, and `reporterEmail` is null.
  - A1 sees both, with reporter emails.
- **Report actions:** M dismisses, deletes and freezes on a U2 report → OK.
  - M acting on the A1 report (even dismissing it) → 403, and M freezing A1 → 403.
  - A1 resolving the A1-targeted report → refused (it's their own account); A2 resolving it → OK.
- **Roles:** M calling `adminSetRole` → 403.
  - A1 makes U1 a moderator by email, and U1's **next** `adminListReports` call succeeds with no new login.
  - A1 removes it again, and the next call → 403.
  - A1 changing O → 403, freezing O → 403, changing their own role → 400.
- **Last admin:** with `OWNER_EMAIL` unset, A2 demoted and A1 the only admin, A1 deleting their account → 400.
- **Frozen accounts:** freeze U2, then try to make U2 a moderator → 400.
- **Activity:** `adminListActivity` lists every action above with emails, newest first; M → 403.
- **Spanish:** one 403 with `X-SS-Lang: es` comes back in Spanish.
- **Regression:** the moderation and i18n suites still pass, and so does `check-messages.php`.

**Commits:** `staff: users.role + staff_actions (migration6)`, `staff: roles on the moderation endpoints`, `staff: team and activity endpoints`, `staff: getMyInfo role, last-admin guard, API docs`.

---

## 3. Phase 2: shared code + web (about a day)

### 2.1 `@ss/core`
- **`types.ts`:**
  - `User` gains `role: 'user' | 'moderator' | 'admin'` and `isOwner: boolean`.
  - `userFrom` maps them, with `role: info.role ?? (info.isAdmin ? 'admin' : 'user')` so an older server still works.
  - Add helpers `isStaff(user)` and `isAdminRole(user)`.
- **`moderation.ts`:**
  - Types `StaffMember` and `StaffAction`.
  - `ModerationReport` gains `targetUserIsStaff`, and `reporterEmail` can be `null`.
- **`api-client.ts`:** `adminListStaff()`, `adminSetRole(target: { email: string } | { userId: number }, role)`, `adminListActivity(offset)`.
- **Tests:** the `userFrom` fallback, and `isStaff`.

### 2.2 `/admin` becomes the staff page (`web/src/pages/AdminPage.tsx`)
- **Access:** staff (moderator or admin); anyone else is sent to `/feed`, as now.
  - The page title is **"Admin"** for admins and **"Moderation"** for moderators.
  - If an admin endpoint answers 403 (the role was just removed), refresh the user (`getMyInfo`) and go to `/feed` with a toast: "Your role has changed."
- **Tabs:**
  - **Reports** (Open / Resolved, the existing tabs) for all staff. Show "Reported by" only when the email is present.
  - **Team** for all staff.
  - **Activity** for admins only.
  - **No App settings tab yet**; the dashboard plan adds it (section 6).
- **Team tab:**
  - One row per staff member: email, role badge, an **Owner** badge on the owner, and a **Frozen** badge when frozen.
  - **Admins see:**
    - A role control on each row except the owner's and their own. Those two get a short note instead: "Set on the server" and "Another admin can change your role".
    - An **Add someone** form: email + role (Moderator or Admin) + Add.
    - A confirmation before every change, e.g. "Make ana@… an admin? Admins can appoint and remove other admins and moderators, and will be able to change the app's settings." or "Remove ana@…'s moderator role?"
  - Server errors show inline under the form or row.
  - **Moderators see the list read-only.**
- **Activity tab:** one line per entry, newest first, with a Load more button. Build the text client-side from `action` + `details` with `tr()` keys, e.g.:
  - "dave@… made ana@… a moderator"
  - "ana@… froze leo@…"
  - "ana@… resolved report #12 (dismissed)"
- **Translate the whole page.** It was English-only because only Dave saw it; moderators may be Spanish speakers.
  - Remove `src/pages/AdminPage.tsx` from the lint rule's ignore list in `web/eslint.config.js`.
  - Change the language plan's "English only" list to drop Admin.
  - Spanish role names: "Admin" and "Moderador/a" (neutral, per the language plan's no-gendered-words rule; the Spanish reviewer may change them).
- **Settings** (`SettingsPage.tsx`): the existing Admin card shows for **all staff** (`isStaff(user)`). Moderators get new `settings.moderation.*` keys ("Moderation", "Review reports from the community", "Open").

### 2.3 Verify
- **Checks:** `pnpm typecheck`, `pnpm lint` and `pnpm test` pass.
- **On the local bench, with three logins:**
  - **Owner:** can manage everyone.
  - **Admin:** can't change the owner or themselves; adding, changing and removing a moderator works and shows up in Activity.
  - **Moderator:** sees "Moderation", the Reports and Team tabs only, no reporter emails, and no reports about staff. Removing their role while their page is open sends them to the feed on their next action.
  - **Regular user:** visiting `/admin` lands on `/feed`.
- **Spanish:** check the page in Spanish at 375 px wide.

**Commits:** `core: staff roles`, `web: staff page (roles, team, activity)`, `web: translate the staff page`.

---

## 4. Phase 3: phone app (`ssreact/mobile/`, about an hour)
Moderation stays on the website, so the phone app needs no staff screens.
- In Settings, for staff only (`isStaff(user)`), add a card after Blocked users. It's titled "Moderation" ("Admin" for admins) and says "Reports are handled on the website." Its button opens `${apiBase}/admin?lang=${lang}` in the browser, the same way the info links do. Use `tr()`.
- Verify with `npx expo lint` and `npx tsc --noEmit`. On a development build: a moderator account shows the card, and a regular account doesn't.
- This is JavaScript only, so it can ship in any phone update.

**Commit:** `mobile: staff link in Settings`.

---

## 5. Deploy (Dave)
1. **Server, dev then prod:**
   1. Back up the database.
   2. Add `OWNER_EMAIL=<your login email>` to `private/.env`.
   3. Deploy ssapi.
   4. Run `composer dump-autoload` for the new `src/Staff` module.

   `migration6` runs on the first request. The deploy is backward compatible: `isAdmin` stays, and the `admin*` endpoint names don't change.
2. **The web app.**
3. **The phone app** with its next update.
4. **Appoint people** from Admin → Team. Roles live in the same SQLite database as everything else (`users.role`). Setting one by hand with the `sqlite3` tool still works, e.g. `UPDATE users SET role = 'moderator' WHERE email = '…'`, but the Team tab doesn't need a server login, checks the rules in section 1, and records the change in Activity, which a hand edit doesn't.

---

## 6. Later: the app settings dashboard (design rules only; don't build now)
When the dashboard is planned, it must follow these rules, so roles and settings fit together:
- **Admins only:** `requireRole($pdo, $user, 'admin')` on every settings endpoint.
- **Logged:** every change writes a `staff_actions` row (`update_setting`, `{"key": …, "from": …, "to": …}`), so Activity shows who changed what.
- **Stored** in an `app_settings` table (`key`, `value` as JSON, `updated_at`, `updated_by`). Each setting is declared once in code with its type, default, minimum/maximum and description. Today's `config.php` values become the defaults, and code reads settings through one cached helper (`appSetting('media_max_user_bytes')`).
- **Good candidates:**
  - the per-user media quota, upload limits and uploads per hour;
  - whether registration is open or invite-only;
  - the terms version, the contact email and who gets report emails.
- **Never settings:** secrets (`.env` stays the only place for keys and passwords), the owner, or anything that could lock admins out.

---

## 7. Estimate and decisions

| Phase | Work | Effort |
|---|---|---|
| 1 | Server: roles, owner, guards, team/activity endpoints, tests | about a day |
| 2 | Core + web: staff page with Team and Activity tabs, translated | about a day |
| 3 | Phone: Settings link for staff | about an hour |

About **2 days of agent work**.

**Decisions for Dave (defaults in brackets):**
1. Protect your account as the owner, set on the server so nobody can demote or freeze it from the app (R1) [yes].
2. Moderators can't see or act on reports about other staff; only admins handle those (R4) [yes].
3. Moderators don't see who made a report (R5) [hidden].
4. Report emails: only `ADMIN_REPORT_EMAIL` as now, or also every admin and moderator [only as now].
5. Admins can freeze other admins (never the owner) [yes].
