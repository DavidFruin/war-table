---
status: review (findings and recommendations; nothing implemented)
written: 2026-11-16
repo: ssapi @ 4af5192 (5,690 lines of PHP outside vendor/)
---

# ssapi review, November 2026: faster, safer, simpler, and what I'd do differently

Dave asked four questions about [[ssapi]], plus how **Fruin Auth** could take over the hand-rolled authentication. This is a read-through of every file at `4af5192`. Nothing was run against dev or prod, and nothing was changed.

**The Fruin Auth part is not done.** The `fruin-auth` repo could not be opened from the reviewing session (see section 5). That section lists what ssapi would hand over and what it would keep, so it's ready once the repo can be read.

Severity: **High** = fix soon, **Medium** = worth scheduling, **Low** = when convenient.

---

## 1. More secure

| # | Finding | Where | Severity |
|---|---|---|---|
| S1 | **Password reset and registration can be hijacked in the 10 minutes after someone verifies their email code.** `verifyOTP` marks the *email* as verified. `resetPassword` then accepts any caller who sends that email and a new password; it never checks that the caller is the one who entered the code, and it has no attempt limit. Someone who knows a member's email can call `resetPassword` in a loop, and the moment the member verifies a real reset code, the attacker's password wins. `finishRegister` has the same gap for a new account. | `src/Auth/handlers.php` `handle_verifyOTP`, `handle_resetPassword`, `handle_verifyRegisterOTP`, `handle_finishRegister` | **High** |
| S2 | **Anyone can lock a member out of logging in.** Five wrong passwords for an email lock that email for 15 minutes, and the lock is checked before the password, so the real owner is refused too. It can be repeated forever. | `checkAttemptLimit`, `handle_login` | Medium |
| S3 | **Refresh tokens never change.** A session's refresh token is the same for its whole life, and the session slides forward 30 days on each use. A stolen one works indefinitely, and the owner can't tell. The web app keeps it in `localStorage`, where any script on the page can read it. | `auth.php` `sessionRefresh` | Medium |
| S4 | **The Content-Security-Policy is still "report only"**, so it blocks nothing. With tokens in `localStorage`, an enforced policy is the main defence against a script-injection bug. | `deploy/root.htaccess` | Medium |
| S5 | **Public endpoints reveal which emails have accounts:** `sendOTP` answers "No account found with this email", and login is measurably faster for an unknown email (no password hash is checked). Members can already list every email (`getUsers`, by design), but these work for outsiders. | `handle_sendOTP`, `handle_login` | Low |
| S6 | **One-time codes are stored as plain text** (`users.reset_otp`, `pending_users.otp`). Anyone who can read the database or a backup during the 10-minute window can use them. Store a hash. | `handle_sendOTP`, `handle_sendRegisterOTP` | Low |
| S7 | **No foreign keys.** Deleting an account or post relies on about 100 lines of hand-written cleanup (`deleteUserAndData`, `deletePostById`). Every new table has to be remembered there, or rows are orphaned. | `api.php`, `schema.php` | Low (integrity) |
| S8 | **Timestamps use the server's local time zone** (`date('Y-m-d H:i:s')` everywhere), and expiry checks compare those strings. A time-zone or daylight-saving change on the server shifts every expiry by an hour. Use UTC. | throughout | Low |

**The fix for S1** (small): `verifyOTP` and `verifyRegisterOTP` return a random one-time token, stored hashed. `resetPassword` and `finishRegister` require it. Alternatively, drop the separate verify step and send the code together with the new password. Add an attempt limit to both final steps either way.

**For S2:** keep the per-IP limit, and replace the hard per-email lock with a growing delay, or exempt a correct password from the lock.

**For S3:** issue a new refresh token on every refresh and reject the old one. A reused old token then reveals the theft, and that session can be revoked.

---

## 2. Faster

| # | Finding | Where | Gain |
|---|---|---|---|
| F1 | **SQLite isn't in WAL mode.** In the default mode, any write blocks every reader. Likes, comments, session touches and notifications all write. One line fixes it, once per database: `PRAGMA journal_mode=WAL`, with `synchronous=NORMAL`. (The first improvement plan proposed this; it was never done.) | `schema.php` `db()` | **Biggest single win** |
| F2 | **Follows are still a JSON list inside each user row.** Showing someone's followers reads *every* user and decodes each list. The feed, follow, unfollow, block and account deletion each decode and rewrite it. Two people following at once can overwrite each other. A `follows(follower_id, followed_id, created_at)` table makes all of these one indexed query. | `src/Follows/handlers.php`, feed in `src/Posts/handlers.php` | Large as users grow |
| F3 | **Every feed post carries its full list of likes** (who and when). Send `likeCount` and `likedByMe`, and fetch the list only when someone opens it (`getPostLikes` already exists). | `getLikesForPostIds`, `postRowToApi` | Smaller payloads |
| F4 | **Each page of posts runs a `COUNT(*)` as well as the page query.** Fetch one extra row to know whether there's more. Paging by offset also gets slower the deeper someone scrolls; paging "before this post's time" doesn't. | `fetchPostsPageForUser`, `handle_fetchFollowedPosts` | Fewer queries |
| F5 | **`getUsers` returns every member in one response**, for search and @mentions. Fine for a family; it needs a search-as-you-type endpoint before the app is public. | `src/Users/handlers.php` | Later |
| F6 | **Threaded comments load every comment on the post**, then pick the page in PHP. Fine until a post has thousands. | `handle_getPostComments` | Later |
| F7 | **Push notifications run after the response but still hold the PHP worker** on dev and prod (mod_fcgid), for up to 4 seconds per device. A burst of likes can tie up workers. A small queue table drained by a cron job would free them. | `respond()`, `pushNotification` | Under load |
| F8 | **Housekeeping runs on user requests:** expired invites are deleted on every invite read, and sessions and attempt rows are swept on every login and refresh. Each is a write. Move them to a daily cron job, or run them on 1 request in 100. | `deleteExpiredInvites`, `sessionSweep` | Small |

---

## 3. Simpler

- **The JWT does no work.** Every request verifies the token *and* looks the session up in the database. An opaque random session token, looked up the same way, gives identical behaviour and removes the hand-written JWT code and the signing secret. (This matters less if Fruin Auth takes over tokens.)
- **Two entry points repeat the same plumbing.** `api.php` and `media.php` each define their own `respond`, `bad`, `good`, `logMsg` and `requireAuth`. One bootstrap file would do.
- **About a third of the code is commentary on its own history** ("this used to…", plan item numbers like P3, S14). Git already holds that. Keep the comments that explain *why the code is the way it is now*.
- **Dead columns:** `users.posts`, `users.followers`, `users.jwt`, `users.is_admin`, `pending_users.password`. Each new reader has to work out that they're unused.
- **Every migration repeats the same "add column if missing" loop.** One `addColumns($pdo, $table, [...])` helper would shrink `schema.php` by about a third.
- **Comments have two listing paths** (flat and threaded) that apply the visibility rules separately. Once the terminal apps use the threaded one, delete the flat one.
- **"Unliked your post" and "unfollowed you" notifications** add code and noise; most apps don't send them. (This was open decision S9 in the first plan.)
- **Globals and `$_POST` everywhere:** `$CONFIG`, `$action`, `$DEFERRED`, and `$_POST` rewritten from the raw body. Passing a small request object to handlers makes them testable without a web server.

---

## 4. If it were mine from the beginning

1. **Usernames or display names, not email addresses, as public identity.** This is the largest design debt. Emails are shown to every member, sit in @mentions and notifications, and are why share links and store reviews need special care. Email would be private and used only for login.
2. **Don't write authentication by hand.** Use an identity service (this is what Fruin Auth is for) or, at the least, plain server sessions with `HttpOnly` cookies on the web, so no script can read the login.
3. **A real relational schema from day one:** integer ids for posts (not `userId.time` strings), tables for follows and likes, foreign keys with cascading deletes, UTC timestamps. The 100-line account-deletion function would be one `DELETE`.
4. **One front controller with a small router and ordinary REST conventions** (paths, HTTP verbs, status codes), not one `action` field on POST. It works with standard tools, caches and docs.
5. **Counts and cursors in the API from the start** (`likeCount`, `likedByMe`, "next page" cursors), so clients never depend on whole lists.
6. **Background work in a queue:** push notifications, email and video conversion, run by a worker, never inside a web request.
7. **Send email through a mail service** (with SPF, DKIM and bounce handling), not PHP's `mail()`. The Hotmail/Outlook block is this problem.
8. **Tests and CI next to the code.** Dave's rule is that tests never deploy to the server; a `tests/` folder excluded from the deploy keeps that rule and still runs on every push.
9. **Keep what's right:** SQLite (with WAL) is the correct size for this app; the module layout, the migration versioning, the media pipeline's safety limits and the rank rules for staff are all sound.

---

## 5. Fruin Auth taking over authentication

**Blocked:** the `fruin-auth` repo isn't visible to the reviewing session. It was tried under `fruininc`, `Fruin-Inc` and `DavidFruin`. The only mention in this vault is [[andrews-claude]], which says Fruin Auth is Drew's project and nothing about how it works. To finish this section, the session needs the repo's exact `owner/name` and Claude's GitHub connection approved for that organization.

**What ssapi would hand over** (about 750 lines):
- **Code:** `auth.php` (JWT and sessions), and in `src/Auth/handlers.php`: login, logout, refresh, password reset, registration codes and attempt limits.
- **Data:** the `users.password`, `reset_otp` and `reset_expires` columns, and `pending_users`.

Findings S1, S2, S3, S5 and S6 disappear with it.

**What ssapi keeps, whatever Fruin Auth is:**
- **Its own `users` row per person** (role, language, theme, frozen, invited_by, terms accepted), linked to the Fruin Auth identity by a stable id, never by email alone.
- **Authorization:** roles, blocks, frozen accounts and the rank rules.
- **Invite-only joining.** It moves from "before the email code" to "the first time a Fruin Auth identity arrives with no ssapi account".
- **A local record of each signed-in device,** if the device list, "sign out other devices" and push subscriptions tied to a session are to keep working.

**Questions the repo has to answer before a plan can be written:**
1. **What is it?** A sign-in service with redirects (OpenID Connect or OAuth), a token-checking API, or a shared library and database.
2. **How does ssapi verify a login on each request?** Is it a signed token checked locally with a public key, or a call to Fruin Auth? Can a login be revoked at once (ssapi's freeze depends on that)?
3. **Which clients can it serve?** The web app, the phone app (which needs a native-app flow) and the three terminal apps (which can't open a browser easily).
4. **Can the existing accounts move without everyone resetting their password?** ssapi's hashes are standard bcrypt (`password_hash`).
5. **Who sends the emails,** and from which domain?
6. **Does it run on the same server as ssapi today,** or only on the planned cluster? [[andrews-claude]] says that network "does not exist today".
