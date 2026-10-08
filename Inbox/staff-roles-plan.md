---
status: proposal (revised 2026-10-08 with Dave's owner, badge and freeze/delete rules)
written: 2026-10-08
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi @ 8db90d3, ssreact @ 62b07c8 (packages/core, web/, mobile/), sstests
---

# Staff roles plan: owner, admins and moderators

Dave (2026-10-08), first: "I want there to be two levels of administrator. Admin and moderator. Admins can appoint other admins and moderators and can change the settings of the app when we later make an app settings dashboard."

Then, revising it: "I want my role to be a third role called owner and it should be public who is a moderator admin and owner of the app. When someone is looking at your profile it should have a badge that says either "user" "moderator" "admin" or "owner". Moderators have control to freeze comments posts and accounts. Administrator can appoint and unappoint moderators and can delete accounts or posts. Owner can appoint admins and [unappoint] admins. But the owner is an account that cannot be deleted even by itself and moderators and admins cannot freeze or delete comments posts or the owners account."

**Today:**
- There is one level. `users.is_admin` is set by hand with `sqlite3` on the server (`src/Moderation/handlers.php`, `requireAdmin()`), and there is no API to grant it.
- Admins resolve reports (dismiss, **delete** the reported post or comment, freeze the account) and freeze or unfreeze accounts, through the four `admin*` endpoints and the web `/admin` page.
- The phone app has no staff tools.
- Roles are read from the database on every request, which this plan keeps, so a role change takes effect on the person's next request with no new login.

**After this plan:**
- **Four roles, public:** every profile shows a badge reading User, Moderator, Admin or Owner.
- **Moderators freeze** posts, comments and accounts. Freezing hides them and is reversible.
- **Admins also delete** them, which is permanent, and appoint or remove moderators.
- **The owner also appoints or removes admins.**
- **Nobody can act on anyone at their own level or above.** The owner, its posts and comments can't be touched by anyone, and the owner account can't be deleted, not even by the owner.
- **Staff act where they see things:** the "…" menu on posts, comments and profiles, in the web and phone apps. The website's staff page has the report queue, the frozen list, the team and the activity log.
- **Everything stays in the same SQLite database.** The owner is set there by hand, and the app never makes or removes an owner.
- **App settings are not built here.** Section 7 sets the rules the later dashboard must follow.

---

## 0. Rules for the implementing agent
- The usual rules apply: check and claim `Areas/active-work.md` first, one task per commit, push after each commit, **never deploy**, never touch prod. Dave deploys.
- Tests go in [[sstests]] (`sstests/backend/staff/`), never in ssapi.
- The database change is the next free `migrationN`. `SCHEMA_VERSION` is 5 as of `8db90d3`, so most likely `migration6`. Check first, and never edit a migration that has shipped.
- **Every new server message** goes into `src/I18n/es.php` with its Spanish, and `sstests/backend/i18n/check-messages.php` must pass. **Every new app text** goes through `tr()` with Spanish in `packages/core/src/i18n/es.ts`. See [[language-plan]].
- **Mobile:** follow `mobile/AGENTS.md`. The language plan's Phase 4 (phone) was in progress at `62b07c8`, so do this plan's phone phase after it.
- Skip anything marked **DECISION** until Dave answers. Use the stated default if he says "go" without answering.

---

## 1. Who can do what

Rank: **user 0 < moderator 1 < admin 2 < owner 3.**

| | Moderator | Admin | Owner |
|---|---|---|---|
| Freeze and unfreeze posts and comments | users' | users' and moderators' | users', moderators' and admins' |
| Freeze and unfreeze accounts | users | users and moderators | users, moderators and admins |
| Delete other people's posts and comments | — | users' and moderators' | users', moderators' and admins' |
| Delete accounts (must be frozen first) | — | users and moderators | users, moderators and admins |
| Appoint and remove moderators | — | ✓ | ✓ |
| Appoint and remove admins | — | — | ✓ |
| Handle reports | about users | about users and moderators | all |
| See who made a report | — | ✓ | ✓ |
| Activity log | — | ✓ | ✓ |
| App settings (later, section 7) | — | ✓ | ✓ |

Everyone can still report, block, and delete their own posts and comments, as today.

**The rules:**
- **R1. Rank:** staff can only act on people **below** their own role: their posts, their comments and their account. This is Dave's owner rule applied at every level, so an admin can't delete another admin and a moderator can't freeze another moderator. A misbehaving admin is the owner's to deal with; a misbehaving moderator is any admin's. **DECISION**, default yes.
- **R2. The owner:**
  - It is stored in SQLite as `users.role = 'owner'` and set by hand on the server (section 6). No endpoint ever sets or removes `owner`.
  - Nobody can freeze, delete or change the owner, its posts or its comments.
  - The owner account can't be deleted, not even by the owner: "The owner account can't be deleted. Ownership can only be changed on the server."
  - Handing over ownership is the same one-line `sqlite3` command.
  - Apple requires in-app account deletion for users. The owner account is the operator's own, and App Review deletes the demo account, so this is low risk; the message explains why.
- **R3. Nobody changes their own role.** Freezing, deleting or demoting yourself is refused too; your own posts you delete the normal way.
- **R4. Freezing is reversible; deleting is permanent.**
  - A frozen post or comment is hidden from everyone except its author, who sees it labelled "Hidden by a moderator", with no likes or comments while frozen. **DECISION**, default: the author sees it.
  - Frozen accounts work as today: no login, and all their content hidden.
  - While something is frozen its photos and videos stay on the server, so a link someone already copied still works. Deleting removes the files.
- **R5. Deleting an account** needs the account frozen first, plus the admin typing the account's email to confirm. It removes all of the account's posts, comments, photos and videos, and can't be undone. **DECISION**, default yes.
- **R6. Moderators don't see who made a report.** **DECISION**, default hidden.
- **R7. Roles are public.** Every profile shows exactly one badge: User, Moderator, Admin or Owner. Dave asked for "User" too; **DECISION**, the alternative is badges only for staff.
- **R8. A frozen account can't be given a role.** Unfreeze it first.
- **R9. The server enforces everything.** The apps only hide buttons people can't use.

---

## 2. Phase 1: server (ssapi, about 1.5 days)

### 1.1 Migration (`migration6`, `SCHEMA_VERSION` 6)
Follow `migration5`'s pattern (`PRAGMA table_info` before each `ALTER`):
- `users.role TEXT NOT NULL DEFAULT 'user'`, then `UPDATE users SET role = 'admin' WHERE is_admin = 1`. Dave makes himself owner by hand after deploying (section 6). Leave `is_admin` in the table but stop reading it, with a comment in `schema.php`.
- `posts.frozen_at TEXT`, `posts.frozen_by INTEGER`, `comments.frozen_at TEXT`, `comments.frozen_by INTEGER`.
- The `staff_actions` table:
  ```sql
  CREATE TABLE IF NOT EXISTS staff_actions (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      actor_id INTEGER, actor_email TEXT NOT NULL,
      action TEXT NOT NULL,      -- set_role | freeze_user | unfreeze_user | freeze_content | unfreeze_content
                                 -- | delete_content | delete_account | resolve_report | update_setting (later)
      target_user_id INTEGER, target_email TEXT,
      details TEXT,              -- JSON: {"from":"user","to":"moderator"}, {"type":"post","id":"…","text":"first 200 chars"}, …
      created_at TEXT NOT NULL)
  ```
  Emails and a short text snapshot are copied in, so the log stays readable after something is deleted.
- Indexes on `users(role)` and `staff_actions(created_at)`.

### 1.2 New module `src/Staff/handlers.php`
Add it to `composer.json`'s `autoload.files` and run `composer dump-autoload`.
```php
const ROLES = ['user', 'moderator', 'admin', 'owner'];
const ROLE_RANK = ['user' => 0, 'moderator' => 1, 'admin' => 2, 'owner' => 3];

// ['id', 'email', 'role', 'frozen'], or null for an unknown id.
function userRole($pdo, int $userId): ?array { /* SELECT id, email, role, frozen_at FROM users WHERE id = ? */ }

// Stops the request unless the caller has at least $min. Returns the caller.
function requireRole($pdo, $user, string $min): array {
    $me = userRole($pdo, (int)$user['sub']);
    if (!$me || ROLE_RANK[$me['role']] < ROLE_RANK[$min]) {
        bad(['moderator' => 'Moderators only', 'admin' => 'Admins only'][$min] ?? 'Not allowed', 403);
    }
    return $me;
}

// Rules R1-R3 for any action on another person, their posts or their comments.
function requireOutranks(array $me, ?array $target): void {
    if ($target === null) return;   // the account is already gone
    if ($target['role'] === 'owner') bad("The owner's account, posts and comments can't be changed.", 403);
    if (ROLE_RANK[$me['role']] <= ROLE_RANK[$target['role']]) bad('You can only act on people below your own role.', 403);
}

function logStaffAction($pdo, array $me, string $action, ?array $target, array $details = []): void { /* INSERT … */ }
```
`requireOutranks` also covers yourself, because equal rank is refused.

### 1.3 Hiding frozen posts and comments (R4)
- **The filter:** `frozenFilter(string $alias, $viewerId)` returns `[" AND ($alias.frozen_at IS NULL OR $alias.user_id = ?)", [$viewerId]]`. Append it the same way `hiddenFilter()` is used.
- **Apply it everywhere posts or comments are shown:**
  - the feed (`fetchFollowedPosts`), profile posts (`getUserPosts`, `getMyPosts`), `getPostById` and the comment lists;
  - **comment counts**, which leave out frozen comments for everyone (`getPostCommentCounts` and the `commentCount` field).

  `grep -rn "FROM posts\|FROM comments\|JOIN posts\|JOIN comments" src api.php` finds 21 queries as of `8db90d3`. Go through every one, and list in the commit message which got the filter and why the rest don't need it (deletes, internal lookups).
- **Flag:** objects the author can still see get `frozen: true`.
- **No interactions while frozen:** liking or commenting on a frozen post gives 404 `Post not found`, the author included.

### 1.4 Account deletion, shared
- Move everything in `handle_deleteAccount` (`api.php`) after the password check into `deleteUserAndData($pdo, int $uid)`, with no change in behaviour, so self-deletion and admin deletion use the same code.
- `handle_deleteAccount` refuses the owner (R2).

### 1.5 Endpoints
**Existing ones keep their names, so older app builds still work:**
- **`adminListReports`** (moderator+):
  - Only reports whose target account is **below** the caller's rank, plus reports with no account left.
  - `reporterEmail` is `null` for moderators (R6).
  - Each row gains `targetUserRole`.
- **`adminResolveReport`** (moderator+):
  - Resolutions: `dismiss`, `freeze_content` (new), `freeze_user`, `delete_content` and `delete_and_freeze`. Moderators may use only the first three; the delete ones are admin+.
  - Every resolution, dismiss included, first calls `requireOutranks` on the reported account.
  - `freeze_content` on a user report → 400.
  - Log `resolve_report`.
- **`adminFreezeUser` / `adminUnfreezeUser`** (moderator+): `requireOutranks`, then log.

**New:**
- **`staffFreezeContent` / `staffUnfreezeContent`** (moderator+):
  - Params `type` (`post|comment`) and `id`. Call `requireOutranks` on the author, then set or clear `frozen_at`/`frozen_by`.
  - Freezing also resolves any open reports on that item as actioned (`freeze_content`).
  - Log the action.
- **`adminDeleteContent`** (admin+):
  - Params `type` and `id`. Call `requireOutranks` on the author, then `deletePostById` / `deleteCommentById` (media files removed as today).
  - Resolve any open reports on it, then log the action with a text snapshot.
- **`adminDeleteAccount`** (admin+): params `userId` and `confirmEmail`.
  1. `requireOutranks`.
  2. The account must be frozen (R5): 400 `Freeze this account before deleting it`.
  3. `confirmEmail` must match, case-insensitively: 400 `The email doesn't match`.
  4. Log **before** deleting (the snapshot keeps the email), then call `deleteUserAndData`.
- **`adminSetRole`:** params `email` or `userId`, plus `role` (`user|moderator|admin`).
  - `role = 'owner'`, or a target who is the owner → 403 `Ownership can only be changed on the server.`
  - The caller (admin+) must **outrank the target's current role**, and the **new role must be below the caller's**:
    - admins can switch people between user and moderator;
    - the owner can set user, moderator or admin.
    - Otherwise → 403 `Only the owner can appoint or remove admins.` when an admin is involved, or the rank message.
  - Yourself → 400 `You can't change your own role`.
  - Frozen target with a non-`user` role → 400 `Unfreeze this account before giving it a role` (R8).
  - The same role again is a no-op success and isn't logged. Otherwise log `set_role` with `{from, to}`.
- **`getStaff`** (any logged-in user; roles are public, R7):
  - Every unfrozen account whose role isn't `user`, as `{userId, email, role}`, ordered owner, admins, moderators, then by email.
  - For staff callers, also include frozen staff with `frozen: true`, for the Team tab.
- **`adminListFrozen`** (moderator+):
  - Frozen accounts, posts and comments **the caller outranks**, newest first, paged with `pageParams()`.
  - Each as `{type, id, userId, email, text (first 200 chars), mediaUrl, frozenAt, frozenByEmail}`.
- **`adminListActivity`** (admin+): newest first, paged, `{id, actorEmail, action, targetEmail, details, createdAt}`.

**Changed:**
- **`getUserInfo`** (a profile) adds `role` (R7).
- **`getMyInfo`** adds `role`, and **keeps `isAdmin`** (true for admin and owner) for older app builds.
- **Delete `requireAdmin()`** once nothing calls it.

Report emails stay as they are (`ADMIN_REPORT_EMAIL`). The **API docs** (`ssreact/web/src/content/api-docs.html`, English) get the roles, the new endpoints and which role each needs.

### 1.6 Verify on the bench (`sstests/backend/staff/run.sh`)
Bench users: **O** (owner), **A1** and **A2** (admins), **M1** and **M2** (moderators), **U1** and **U2** (users).
- **Migration:** a version-5 database with `is_admin = 1` becomes version 6 with `role = 'admin'`.
- **Public roles:** U1's `getUserInfo` on O, A1, M1 and U2 returns owner, admin, moderator and user. `getStaff` lists O, A1, A2, M1 and M2.
- **Freezing:**
  - M1 freezes U1's post. U2 no longer sees it in the feed, on U1's profile or by id. U1 still sees it, with `frozen: true`. Comment counts drop. Liking it → 404.
  - M1 then unfreezes it, and everything is back.
- **Rank:**
  - M1 freezing M2's post → 403, and M1 freezing A1 → 403.
  - A1 freezes M1's comment → OK. A1 freezing A2 → 403.
  - O freezes and unfreezes A1 → OK.
  - **Anyone** freezing or deleting O's post, comment or account → 403. O deleting its own account → 400.
- **Deleting:**
  - M1 calling `adminDeleteContent` → 403.
  - A1 deletes U2's post → OK, and its media files are gone.
  - A1 calls `adminDeleteAccount` on U2 while U2 isn't frozen → 400. After freezing U2: wrong email → 400; right email → OK, and U2's posts, comments, media and sessions are gone.
  - A1 deleting A2 → 403.
- **Roles:**
  - A1 makes U1 a moderator, and U1's **next** `adminListReports` call works without a new login.
  - A1 making U1 an admin → 403 ("Only the owner…"), and A1 demoting A2 → 403.
  - O makes U1 an admin, then back to a user → OK.
  - Setting `owner` → 403. Changing your own role → 400. Frozen account to moderator → 400.
- **Reports:**
  - M1 sees reports about users only, with no reporter email.
  - A1 also sees reports about moderators, with reporter emails. O sees all.
  - M1 calling `delete_content` → 403; M1 calling `freeze_content` → OK.
- **Activity:** `adminListActivity` lists all of the above with emails; M1 → 403.
- **Frozen list:** `adminListFrozen` for M1 shows only users' frozen items; A1's also shows moderators' items.
- **Spanish:** one 403 with `X-SS-Lang: es` comes back in Spanish.
- **Regression:** the moderation, media and i18n suites still pass, and so does `check-messages.php`.

**Commits:**
- `staff: roles, frozen content columns, staff_actions (migration6)`
- `staff: hide frozen posts and comments`
- `staff: shared account deletion; owner can't be deleted`
- `staff: rank rules on the moderation endpoints`
- `staff: freeze/delete content and accounts, roles, staff list, frozen list, activity`
- `api docs: roles`

---

## 3. Phase 2: shared code + web (about 1.5 days)

### 2.1 `@ss/core`
- **Types:** `Role = 'user' | 'moderator' | 'admin' | 'owner'`, plus `ROLE_RANK`.
  - `User` gains `role`. `userFrom` uses `info.role ?? (info.isAdmin ? 'admin' : 'user')`, so an older server still works.
  - The profile type gains `role`, and posts and comments gain `frozen?`.
- **Permission helpers that mirror the server table** (section 1):
  - `isStaff(me)`, `outranks(me, targetRole)`;
  - `canFreeze(me, targetRole)`, `canDelete(me, targetRole)`;
  - `canSetRole(me, targetRole, newRole)`, `rolesICanGive(me)`.
- **API client:** `getStaff`, `staffFreezeContent`, `staffUnfreezeContent`, `adminDeleteContent`, `adminDeleteAccount`, `adminSetRole`, `adminListFrozen` and `adminListActivity`.
  - `adminResolveReport` accepts `freeze_content`.
- **A staff list cache:** userId → role, from `getStaff`. Load it at start, refresh it after any Team change, and treat anyone missing as `user`. The apps use it to decide which staff buttons to show on someone's post.
- **Tests:** every cell of the section-1 table through the helpers, plus the `userFrom` fallback.

### 2.2 Web
- **Badge:** a small `RoleBadge` on **every** profile, next to the email (R7). Keys `role.user`, `role.moderator`, `role.admin` and `role.owner`.
  - Spanish: Usuario/a, Moderador/a, Admin and Propietario/a. These follow the language plan's no-gendered-words rule; the Spanish reviewer may change them.
- **Staff actions in the existing "…" menus** (`PostCard`, `CommentItem`, `ProfilePage`), shown only when the core helpers allow them:
  - **Post or comment:** **Freeze** (moderator+), with a confirmation that says it can be undone from the staff page. **Delete** (admin+), with a confirmation that says it's permanent.
  - **Profile:** **Freeze account** (moderator+).
  - Deleting an account happens from the staff page's Frozen tab, because frozen profiles are hidden.
- **The author's own frozen post or comment:** muted, with a "Hidden by a moderator" label and no like or comment buttons.
- **The staff page `/admin`:**
  - **Access:** staff only. The title is Moderation, Admin or Owner to match the role.
  - **Reports** (Open / Resolved): each role sees only the actions it may take.
  - **Frozen:** accounts, posts and comments with Unfreeze (moderator+) and Delete (admin+). **Delete account** opens a confirmation where the admin types the account's email.
  - **Team:** the staff list with badges and a Frozen marker.
    - Admins can add someone as a moderator by email, or remove a moderator.
    - The owner can do the same for admins.
    - Ask for confirmation before every change. The owner's row and your own row have no controls.
  - **Activity:** admin+ only. One line per entry, built with `tr()` from `action` + `details`, e.g. "dave@… made ana@… a moderator", "ana@… froze a post by leo@…" or "dave@… deleted the account leo@…".
  - **No App settings tab yet** (section 7).
  - If an action comes back 403 because the role was just removed, refresh the user and go to `/feed` with a toast: "Your role has changed."
- **Settings:** the Admin card shows for all staff, with wording to match the role.
- **Translate the staff page.** Remove `src/pages/AdminPage.tsx` from the lint ignore list in `web/eslint.config.js`, and drop "Admin" from the language plan's English-only list.

### 2.3 Verify
- **Checks:** `pnpm typecheck`, `pnpm lint` and `pnpm test` pass.
- **On the local bench, with one login per role:**
  - Each role sees only its own buttons.
  - A moderator freezes a user's post: the author sees the label, someone else doesn't see it, and the moderator unfreezes it from the Frozen tab.
  - An admin deletes a frozen account with the typed email.
  - The owner appoints and removes an admin.
  - Nobody gets buttons on the owner's things.
  - Badges show on every profile.
  - Removing a moderator's role while their page is open sends them to the feed on their next action.
- **Spanish:** check everything in Spanish at 375 px wide.

**Commits:** `core: roles and permission helpers`, `web: role badges`, `web: staff actions in menus`, `web: staff page (reports, frozen, team, activity)`, `web: translate the staff page`.

---

## 4. Phase 3: phone app (`ssreact/mobile/`, half a day to a day)
- **Badge:** on every profile in `ProfileView`.
- **Staff actions:** in the `PostCard`, `CommentItem` and `ProfileView` menus, with the same core helpers and confirmations as the web. The author sees the "Hidden by a moderator" label.
- **Settings:** for staff, a card that opens the website's staff page (`${apiBase}/admin?lang=${lang}`) for the report queue, frozen list, team and activity. Those stay website-only.
- **Verify:** `npx expo lint` and `npx tsc --noEmit` pass. Then, on a development build, with a moderator login and an admin login:
  - Each sees only its own buttons.
  - Freezing a post works.
  - The badge shows.
- This is JavaScript only, so it can ship in any phone update after the language plan's phone build.

**Commits:** `mobile: role badges`, `mobile: staff actions in menus`, `mobile: staff link in Settings`.

---

## 5. What older app builds see
- The server keeps `isAdmin` and all `admin*` endpoint names, and the old `adminResolveReport` resolutions still work for admins.
- An old web build won't show badges or staff menus. It treats the owner as an admin (`isAdmin`).
- A moderator on an old build sees nothing new until they update.

---

## 6. Deploy (Dave)
1. **Server, dev then prod:**
   1. Back up the database.
   2. Deploy ssapi.
   3. Run `composer dump-autoload` for the new `src/Staff` module.

   `migration6` runs on the first request, and your current admin flag becomes `role = 'admin'`.
2. **Make yourself the owner**, right after deploying, on dev and then on prod:
   ```
   sqlite3 private/userdata.db "UPDATE users SET role = 'owner' WHERE email = '<your login email>';"
   ```
   Until you do, nobody can appoint admins. Handing over ownership later is the same command for the new owner, plus `role = 'admin'` (or `'user'`) for yourself.
3. **The web app, then the phone app** with its next update.
4. **Appoint people** from Admin → Team. Setting a role with `sqlite3` still works, but the Team tab checks the rules and records the change in Activity, which a hand edit doesn't.

---

## 7. Later: the app settings dashboard (design rules only; don't build now)
- **Admins and the owner** can change settings, per Dave's first message. Every settings endpoint uses `requireRole($pdo, $user, 'admin')`.
- **Every change is logged** as a `staff_actions` row (`update_setting`, `{"key": …, "from": …, "to": …}`).
- **Stored in SQLite:** an `app_settings` table (`key`, `value` as JSON, `updated_at`, `updated_by`).
  - Each setting is declared once in code with its type, default, minimum/maximum and description.
  - Today's `config.php` values become the defaults, read through one cached helper, e.g. `appSetting('media_max_user_bytes')`.
- **Good candidates:**
  - the per-user media quota, upload limits and uploads per hour;
  - whether registration is open or invite-only;
  - the terms version, the contact email and who gets report emails.
- **Never settings:** secrets (`.env` stays the only place for keys and passwords), anything about the owner, or anything that could lock staff out.

---

## 8. Estimate and decisions

| Phase | Work | Effort |
|---|---|---|
| 1 | Server: roles, frozen content, rank rules, delete account, staff/frozen/activity endpoints, tests | about 1.5 days |
| 2 | Core + web: badges, staff menus, staff page with Reports, Frozen, Team and Activity, translated | about 1.5 days |
| 3 | Phone: badges, staff menus, link to the staff page | half a day to a day |

About **3.5–4 days of agent work**.

**Decisions for Dave (defaults in brackets):**
1. The rank rule at every level, so nobody can act on someone at their own level or above (R1) [yes].
2. The author still sees their own frozen post or comment, labelled "Hidden by a moderator" (R4) [yes].
3. An account must be frozen before an admin can delete it, and the admin types its email to confirm (R5) [yes].
4. Moderators don't see who made a report (R6) [hidden].
5. A "User" badge on everyone, as asked (R7) [yes; the alternative is badges only on staff].
6. App settings can be changed by admins and the owner, as in the first message (section 7) [admins and owner].
7. Report emails are unchanged, still only going to `ADMIN_REPORT_EMAIL` [unchanged].
