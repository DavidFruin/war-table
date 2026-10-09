---
status: built and tested on a local bench 2026-10-09 (all three phases); not deployed. Revised 2026-10-08: owner role, public badges, moderation-page-only actions, owner-only settings
written: 2026-10-08
for: Sonnet 5 (medium effort), implementing agent
repos: ssapi @ 8db90d3, ssreact @ 62b07c8 (packages/core, web/, mobile/), sstests
---

# Staff roles plan: owner, admins and moderators

**What Dave asked for (2026-10-08), in order:**
1. "I want there to be two levels of administrator. Admin and moderator. Admins can appoint other admins and moderators and can change the settings of the app when we later make an app settings dashboard."
2. "I want my role to be a third role called owner and it should be public who is a moderator admin and owner of the app. When someone is looking at your profile it should have a badge that says either "user" "moderator" "admin" or "owner". Moderators have control to freeze comments posts and accounts. Administrator can appoint and unappoint moderators and can delete accounts or posts. Owner can appoint admins and [unappoint] admins. But the owner is an account that cannot be deleted even by itself and moderators and admins cannot freeze or delete comments posts or the owners account."
3. "Keep app settings to owner only. Change freezing and deleting can only be done at the moderation page. If they want to do that they just report it and then go to the moderation page."

**Today:**
- There is one level. `users.is_admin` is set by hand with `sqlite3` on the server (`src/Moderation/handlers.php`, `requireAdmin()`), and there is no API to grant it.
- Admins resolve reports on the web `/admin` page: dismiss, **delete** the reported post or comment, or freeze the account. They can unfreeze accounts from the Resolved tab.
- The phone app has no staff tools.
- Roles are read from the database on every request, which this plan keeps, so a role change takes effect on the person's next request with no new login.

**After this plan:**
- **Four roles, public:** every profile shows a badge reading User, Moderator, Admin or Owner.
- **Moderators freeze** posts, comments and accounts. Freezing hides them and is reversible.
- **Admins also delete** them, which is permanent, and appoint or remove moderators.
- **The owner also appoints or removes admins,** and is the only one who will change app settings.
- **Freezing and deleting happen only on the moderation page** (the website's `/admin`). Staff who spot something report it like anyone else, then handle the report there. There are no staff buttons on posts, comments or profiles.
- **Nobody acts on anyone at their own level or above.** Nobody can freeze or delete the owner, its posts or its comments, and the owner account can't be deleted, not even by the owner.
- **Everything stays in the same SQLite database.** The owner is set there by hand, and the app never makes or removes an owner.
- **App settings are not built here.** Section 7 sets the rules the later, owner-only dashboard must follow.

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

Rank: **user 0 < moderator 1 < admin 2 < owner 3.** Every freeze, unfreeze and delete below happens on the moderation page (R10).

| | Moderator | Admin | Owner |
|---|---|---|---|
| Freeze and unfreeze posts and comments | users' | users' and moderators' | users', moderators' and admins' |
| Freeze and unfreeze accounts | users | users and moderators | users, moderators and admins |
| Delete other people's posts and comments | — | users' and moderators' | users', moderators' and admins' |
| Delete accounts (must be frozen first) | — | users and moderators | users, moderators and admins |
| Appoint and remove moderators | — | ✓ | ✓ |
| Appoint and remove admins | — | — | ✓ |
| Handle reports | about users | about users and moderators | all (about itself: dismiss only) |
| See who made a report | — | ✓ | ✓ |
| Activity log | — | ✓ | ✓ |
| App settings (later, section 7) | — | — | ✓ |

Everyone can still report, block, and delete their own posts and comments, as today.

**The rules:**
- **R1. Rank:** staff can only act on people **below** their own role: their posts, their comments and their account. Reports about someone go only to staff who outrank them. This is Dave's owner rule applied at every level: a misbehaving moderator is any admin's to deal with, and a misbehaving admin is the owner's. **DECISION**, default yes.
- **R2. The owner:**
  - It is stored in SQLite as `users.role = 'owner'` and set by hand on the server (section 6). No endpoint ever sets or removes `owner`.
  - Nobody can freeze, delete or change the owner, its posts or its comments.
  - The owner account can't be deleted, not even by the owner: "The owner account can't be deleted. Ownership can only be changed on the server."
  - Handing over ownership is the same one-line `sqlite3` command.
  - **Reports about the owner's posts, comments or account** go only to the owner, whose only option there is **Dismiss**. The owner can still delete its own post the normal way. Content stays reportable, which Apple's guideline 1.2 expects, and those reports don't disappear with nobody able to see them.
  - Apple also requires in-app account deletion. The owner account is the operator's own and App Review deletes the demo account, so the exception is low risk; the message explains it.
- **R3. Nobody changes their own role,** or freezes or deletes themselves through the moderation page. Your own posts you delete the normal way.
- **R4. Freezing is reversible; deleting is permanent.**
  - A frozen post or comment is hidden from everyone except its author, who sees it labelled "Hidden by a moderator", with no likes or comments while frozen. **DECISION**, default: the author sees it.
  - Frozen accounts work as today: no login, and all their content hidden.
  - While something is frozen its photos and videos stay on the server, so a link someone already copied still works. Deleting removes the files.
- **R5. Deleting an account** needs the account frozen first, plus the admin typing the account's email to confirm. It removes all of the account's posts, comments, photos and videos, and can't be undone. **DECISION**, default yes.
- **R6. Moderators don't see who made a report.** **DECISION**, default hidden.
- **R7. Roles are public.** Every profile shows exactly one badge: User, Moderator, Admin or Owner. Dave asked for "User" too; **DECISION**, the alternative is badges only for staff.
- **R8. A frozen account can't be given a role.** Unfreeze it first.
- **R9. The server enforces everything.** The apps only hide buttons people can't use.
- **R10. The moderation page only** (Dave):
  - There are only two ways to freeze or delete anything:
    - **resolving a report** on the Reports tab;
    - **the Frozen tab**, which unfreezes things or deletes what's already frozen.
  - Staff who want to act on something report it first, like anyone else. There are no staff options in the "…" menus on posts, comments or profiles, in either app.
  - **The server enforces it too:**
    - Freezing happens only through `adminResolveReport`.
    - `adminDeleteContent` and `adminDeleteAccount` work only on items that are already frozen.
    - The direct `adminFreezeUser` endpoint is removed; no app calls it.

---

## 2. Phase 1: server (ssapi, about 1.5 days)

### 1.1 Migration (`migration6`, `SCHEMA_VERSION` 6)
Follow `migration5`'s pattern (`PRAGMA table_info` before each `ALTER`):
- `users.role`: **the invites migration (access plan Step 1) adds this column first**, using only `owner`.
  - Here, add it only if it's still missing: `TEXT NOT NULL DEFAULT 'user'`.
  - Then **always** run `UPDATE users SET role = 'admin' WHERE is_admin = 1 AND role = 'user'`, so Dave's owner role isn't overwritten.
  - Leave `is_admin` in the table but stop reading it, with a comment in `schema.php`.
- `posts.frozen_at TEXT`, `posts.frozen_by INTEGER`, `comments.frozen_at TEXT`, `comments.frozen_by INTEGER`.
- The `staff_actions` table:
  ```sql
  CREATE TABLE IF NOT EXISTS staff_actions (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      actor_id INTEGER, actor_email TEXT NOT NULL,
      action TEXT NOT NULL,      -- set_role | resolve_report | unfreeze_user | unfreeze_content
                                 -- | delete_content | delete_account | update_setting (later)
      target_user_id INTEGER, target_email TEXT,
      details TEXT,              -- JSON: {"from":"user","to":"moderator"}, {"reportId":12,"resolution":"freeze_content","text":"first 200 chars"}, …
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
        bad(['moderator' => 'Moderators only', 'admin' => 'Admins only', 'owner' => 'Only the owner can do that'][$min], 403);
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
**The Reports tab: existing endpoints, same names, so older app builds still work:**
- **`adminListReports`** (moderator+):
  - Only reports whose target account is **below** the caller's rank, plus reports with no account left.
  - **For the owner,** also reports about the owner itself (R2).
  - `reporterEmail` is `null` for moderators (R6).
  - Each row gains `targetUserRole`.
- **`adminResolveReport`** (moderator+) is **the only way to freeze anything** (R10).
  - **Resolutions:**
    - `dismiss`;
    - `freeze_content` (new; sets `frozen_at`/`frozen_by` on the reported post or comment);
    - `freeze_user`;
    - `delete_content`;
    - `delete_and_freeze`.
  - Moderators may use only `dismiss`, `freeze_content` and `freeze_user`. The two delete resolutions are admin+.
  - **Checks:**
    - Every resolution, dismiss included, first calls `requireOutranks` on the reported account. The one exception is the owner dismissing a report about itself (R2).
    - `freeze_content` on a user report → 400.
  - As now, resolving one report resolves every open report on the same target. Log `resolve_report`.

**The Frozen tab, new:**
- **`adminListFrozen`** (moderator+):
  - Frozen accounts, posts and comments **the caller outranks**, newest first, paged with `pageParams()`.
  - Each as `{type, id, userId, email, text (first 200 chars), mediaUrl, frozenAt, frozenByEmail}`.
- **`adminUnfreezeUser`** (existing, moderator+): add `requireOutranks` and logging.
- **`adminUnfreezeContent`** (new, moderator+): params `type` (`post|comment`) and `id`. Call `requireOutranks` on the author, clear `frozen_at`/`frozen_by`, then log.
- **`adminDeleteContent`** (new, admin+):
  - Params `type` and `id`. **Frozen items only**: otherwise 400 `Only frozen posts and comments can be deleted here. Report it first.`
  - Call `requireOutranks` on the author, then `deletePostById` / `deleteCommentById` (media files removed as today).
  - Log the action with a text snapshot.
- **`adminDeleteAccount`** (new, admin+): params `userId` and `confirmEmail`.
  1. `requireOutranks`.
  2. The account must be frozen (R5): 400 `Freeze this account before deleting it`.
  3. `confirmEmail` must match, case-insensitively: 400 `The email doesn't match`.
  4. Log **before** deleting (the snapshot keeps the email), then call `deleteUserAndData`.
- **Remove `adminFreezeUser`** from `api.php`'s handler map, and from the core API client. No app calls it (checked web and mobile at `62b07c8`; check ssterminal too if it's in the session). Freezing now happens only through a report (R10).

**The Team tab, and roles, new:**
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
- **`adminListActivity`** (admin+): newest first, paged, `{id, actorEmail, action, targetEmail, details, createdAt}`.

**Changed:**
- **`getUserInfo`** (a profile) adds `role` (R7).
- **`getMyInfo`** adds `role`, and **keeps `isAdmin`** (true for admin and owner) for older app builds.
- **Delete `requireAdmin()`** once nothing calls it.

Report emails stay as they are (`ADMIN_REPORT_EMAIL`). The **API docs** (`ssreact/web/src/content/api-docs.html`, English) get the roles, the new and removed endpoints, and which role each needs.

### 1.6 Verify on the bench (`sstests/backend/staff/run.sh`)
Bench users: **O** (owner), **A1** and **A2** (admins), **M1** and **M2** (moderators), **U1** and **U2** (users). "Reported" means U2 reports it first; every staff action below goes through the endpoints the moderation page uses.
- **Migration:** a version-5 database with `is_admin = 1` becomes version 6 with `role = 'admin'`.
- **Public roles:** U1's `getUserInfo` on O, A1, M1 and U2 returns owner, admin, moderator and user. `getStaff` lists O, A1, A2, M1 and M2.
- **Freezing through a report:**
  - M1 resolves the report on U1's post with `freeze_content`. U2 no longer sees the post in the feed, on U1's profile or by id. U1 still sees it, with `frozen: true`. Comment counts drop. Liking it → 404.
  - The post shows in M1's `adminListFrozen`. `adminUnfreezeContent` brings it back.
- **No other way to freeze:** `adminFreezeUser` → `Unknown action`. `adminDeleteContent` on a post that isn't frozen → 400.
- **Who sees which reports (R1, R2):**
  - M1's list has reports about users only, with no reporter email.
  - A1's list also has reports about moderators, with reporter emails.
  - O's list has all of them, including reports about admins and about O.
- **Acting on reports outside your rank:**
  - M1 resolving a report about M2 or A1 (by id) → 403.
  - A1 resolving a report about A2 → 403.
  - O freezes A1 through a report, then unfreezes A1 → OK.
- **Reports about O:**
  - Anyone but O resolving one → 403.
  - O dismissing one → OK.
  - O using `freeze_content` or a delete on one → 403.
  - O deleting its own account → 400.
- **Deleting:**
  - M1 using `delete_content` → 403.
  - A1 resolves the report on U2's post with `delete_content` → OK, and its media files are gone.
  - A1 freezes U2 through a report. `adminDeleteAccount` with the wrong email → 400; with the right email → OK, and U2's posts, comments, media and sessions are gone.
  - `adminDeleteAccount` on an unfrozen account → 400.
  - A1 calling `adminDeleteAccount` on A2 → 403.
- **Roles:**
  - A1 makes U1 a moderator, and U1's **next** `adminListReports` call works without a new login.
  - A1 making U1 an admin → 403 ("Only the owner…"), and A1 demoting A2 → 403.
  - O makes U1 an admin, then back to a user → OK.
  - Setting `owner` → 403. Changing your own role → 400. Frozen account to moderator → 400.
- **Activity:** `adminListActivity` lists all of the above with emails; M1 → 403.
- **Spanish:** one 403 with `X-SS-Lang: es` comes back in Spanish.
- **Regression:** the moderation, media and i18n suites still pass, and so does `check-messages.php`.

**Done 2026-10-09 (local bench only, not deployed):** ssapi `8b83556`, `e5724a9`, `735bd1b`, `4e7d898`, `af4a490`, `d703ca8` (the migration is `migration7`, schema version 7: invites took 6), ssreact `9496859` (API docs), sstests `staff/run.sh` with 122 checks, all pass. Notes and choices in [[ssapi]].

**Commits:**
- `staff: roles, frozen content columns, staff_actions (migration6)`
- `staff: hide frozen posts and comments`
- `staff: shared account deletion; owner can't be deleted`
- `staff: rank rules and freeze_content on the report endpoints; remove adminFreezeUser`
- `staff: frozen list, unfreeze/delete content, delete account`
- `staff: roles, staff list, activity`
- `api docs: roles`

---

## 3. Phase 2: shared code + web (about 1–1.5 days)

### 2.1 `@ss/core`
- **Types:** `Role = 'user' | 'moderator' | 'admin' | 'owner'`, plus `ROLE_RANK`.
  - `User` gains `role`. `userFrom` uses `info.role ?? (info.isAdmin ? 'admin' : 'user')`, so an older server still works.
  - The profile type gains `role`, posts and comments gain `frozen?`, and `ModerationReport` gains `targetUserRole`, with `reporterEmail` allowed to be `null`.
  - New types `FrozenItem`, `StaffMember` and `StaffAction`.
- **Permission helpers that mirror the server table** (section 1):
  - `isStaff(me)`, `outranks(me, targetRole)`;
  - `resolutionsFor(me, report)`, the buttons a report gets;
  - `canDeleteFrozen(me, item)`;
  - `rolesICanGive(me)`, `canChangeRole(me, member)`.
- **API client:**
  - add `adminListFrozen`, `adminUnfreezeContent`, `adminDeleteContent`, `adminDeleteAccount`, `adminSetRole`, `getStaff` and `adminListActivity`;
  - `adminResolveReport` accepts `freeze_content`;
  - **remove `adminFreezeUser`**.
- **Tests:** every cell of the section-1 table through the helpers, plus the `userFrom` fallback.

### 2.2 Web
- **Badge:** a small `RoleBadge` on **every** profile, next to the email (R7). Keys `role.user`, `role.moderator`, `role.admin` and `role.owner`.
  - Spanish: Usuario/a, Moderador/a, Admin and Propietario/a. These follow the language plan's no-gendered-words rule; the Spanish reviewer may change them.
- **No staff buttons on posts, comments or profiles** (R10). Staff use the normal Report option like everyone else.
- **The author's own frozen post or comment:** muted, with a "Hidden by a moderator" label and no like or comment buttons.
- **The moderation page `/admin`:**
  - **Access:** staff only. The title is Moderation, Admin or Owner to match the role.
  - **Reports** (Open / Resolved):
    - Each report shows only the resolutions its viewer may use (`resolutionsFor`): Dismiss; Freeze post/comment; Freeze account; plus, for admin+, Delete post/comment and Delete and freeze.
    - Confirmations say whether the action can be undone, e.g. "Freeze this post? You can unfreeze it from the Frozen tab." or "Delete this post? This can't be undone."
    - On reports about itself, the owner sees only Dismiss.
  - **Frozen:**
    - Frozen accounts, posts and comments, each with **Unfreeze**.
    - For admin+: **Delete**, and **Delete account**, which opens a confirmation where the admin types the account's email.
    - This replaces the Resolved tab's Unfreeze button.
  - **Team:** the staff list (`getStaff`) with badges and a Frozen marker.
    - Admins can add someone as a moderator by email, or remove a moderator.
    - The owner can do the same for admins.
    - Ask for confirmation before every change. The owner's row and your own row have no controls.
  - **Activity:** admin+ only. One line per entry, built with `tr()` from `action` + `details`, e.g. "dave@… made ana@… a moderator", "ana@… froze a post by leo@… (report #12)" or "dave@… deleted the account leo@…".
  - **No App settings tab yet** (section 7).
  - If an action comes back 403 because the role was just removed, refresh the user and go to `/feed` with a toast: "Your role has changed."
- **Settings:** the Admin card shows for all staff, with wording to match the role.
- **Translate the moderation page.** Remove `src/pages/AdminPage.tsx` from the lint ignore list in `web/eslint.config.js`, and drop "Admin" from the language plan's English-only list.

### 2.3 Verify
- **Checks:** `pnpm typecheck`, `pnpm lint` and `pnpm test` pass.
- **On the local bench, with one login per role:**
  - No staff options appear in any "…" menu.
  - **A moderator** reports a user's post, then freezes it from the Reports tab. The author sees the label and someone else doesn't. The moderator then unfreezes it from the Frozen tab.
  - **A moderator** never sees delete options.
  - **An admin** deletes a frozen account with the typed email.
  - **The owner** appoints and removes an admin, and sees only Dismiss on a report about its own post.
  - Badges show on every profile.
  - Removing a moderator's role while their page is open sends them to the feed on their next action.
- **Spanish:** check everything in Spanish at 375 px wide.

**Done 2026-10-09 (local bench only, not deployed):** ssreact `676f3e8`, `10e6fd5`, `5b4ed70`, `433846f` (page and its translation in one commit), `dc704e0` (report dialog wording), sstests `staff/web-staff.sh` (54 browser checks, all pass). Notes in [[ssreact]].

**Commits:** `core: roles and permission helpers`, `web: role badges and frozen label`, `web: moderation page (reports, frozen, team, activity)`, `web: translate the moderation page`.

---

## 4. Phase 3: phone app (`ssreact/mobile/`, about two hours)
The moderation page stays on the website, and the phone app gets no staff options in its menus (R10).
- **Badge:** on every profile in `ProfileView`.
- **The author's own frozen post or comment:** the "Hidden by a moderator" label, as on the web.
- **Settings:** for staff, a card that opens the website's moderation page (`${apiBase}/admin?lang=${lang}`).
- **Verify:** `npx expo lint` and `npx tsc --noEmit` pass. Then, on a development build:
  - the badge shows;
  - a frozen post shows its label to its author;
  - the staff card shows for a moderator but not for a user.
- This is JavaScript only, so it can ship in any phone update after the language plan's phone build.

**Done 2026-10-09 (local only; not tried on a phone; JavaScript only):** ssreact `807d1c8`, `be956b9`. Notes in [[ssreact]].

**Commits:** `mobile: role badges and frozen label`, `mobile: moderation link in Settings`.

---

## 5. What older app builds see
- The server keeps `isAdmin` and the `admin*` report endpoints, and the old resolutions still work for admins.
- `adminFreezeUser` is gone, but no app called it.
- An old web build won't show badges, the Frozen tab or the Team tab, and treats the owner as an admin (`isAdmin`).
- A moderator on an old build sees nothing new until they update.

---

## 6. Deploy (Dave)
1. **Server, dev then prod:**
   1. Back up the database.
   2. Deploy ssapi.
   3. Run `composer dump-autoload` for the new `src/Staff` module.

   `migration6` runs on the first request, and your current admin flag becomes `role = 'admin'`.
2. **Make yourself the owner**, right after deploying, on dev and then on prod. **Skip this if you already did it with the invites deploy** (access plan §1.6):
   ```
   sqlite3 private/userdata.db "UPDATE users SET role = 'owner' WHERE email = '<your login email>';"
   ```
   Until you do, nobody can appoint admins. Handing over ownership later is the same command for the new owner, plus `role = 'admin'` (or `'user'`) for yourself.
3. **The web app, then the phone app** with its next update.
4. **Appoint people** from the moderation page's Team tab. Setting a role with `sqlite3` still works, but the Team tab checks the rules and records the change in Activity, which a hand edit doesn't.

---

## 7. Later: the app settings dashboard (design rules only; don't build now)
- **The owner only** (Dave, 2026-10-08). Every settings endpoint uses `requireRole($pdo, $user, 'owner')`, and the settings screen only appears for the owner.
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
| 1 | Server: roles, frozen content, rank rules, report resolutions, frozen list, delete account, roles/staff/activity endpoints, tests | about 1.5 days |
| 2 | Core + web: badges, frozen label, moderation page with Reports, Frozen, Team and Activity, translated | 1–1.5 days |
| 3 | Phone: badges, frozen label, link to the moderation page | about two hours |

About **3 days of agent work**.

**Dave's answers so far (2026-10-08):**
- App settings are the owner's only.
- Freezing and deleting happen only on the moderation page, through reports.

He reviewed the rest of the revised plan without changing it, so the defaults below stand unless he says otherwise.

**Remaining defaults:**
1. The rank rule at every level (R1) [yes].
2. The author still sees their own frozen post or comment, labelled "Hidden by a moderator" (R4) [yes].
3. An account must be frozen before an admin can delete it, and the admin types its email to confirm (R5) [yes].
4. Moderators don't see who made a report (R6) [hidden].
5. A "User" badge on everyone, as asked (R7) [yes; the alternative is badges only on staff].
6. Report emails are unchanged, still only going to `ADMIN_REPORT_EMAIL` [unchanged].
