---
status: proposal
written: 2026-10-02
for: an agent with el1 SSH access (this session didn't have one)
repos: ssapi @ 73e05fd, ssreact @ 865ff6f
---

# el1 verification checklist — ssapi improvement plan

[[ssapi-improvement-plan]] was implemented and verified on a local PHP
bench this session (22 commits, all pushed to `master` on both
[[ssapi]] and [[ssreact]] — see that plan and the two projects' own
notes for the full writeup). Everything below is what that session
could **not** check, because it had no access to `el1` (the host
behind `app.davidfruin.com`/`dev.davidfruin.com`/`react.davidfruin.com`).

**Nothing here asks you to deploy anything.** Every item is read-only
(a lookup, a curl check, a `ls`) except #5, which is Dave's own call,
not an agent's. If an item's answer changes what the code should do,
report back rather than editing and pushing yourself — these are
checks, not a continuation of the implementation pass.

---

## 1. Real `users`/`pending_users` schema

The improvement plan's Phase 0 needed these two tables' real definitions
— they predate the repo and are created in no source file. The session
used a from-source reconstruction instead (confirmed against `api.php`'s
own `ALTER TABLE`/`INSERT` statements, not a guess), but it was never
checked against the real database.

```bash
sqlite3 <dev's private dir>/userdata.db '.schema users' '.schema pending_users'
```

Paste the output back. It's table definitions only, no data, so this
is safe to paste into this public vault.

## 2. `ls *.php` on app and dev docroots

Needed before `ssapi/.htaccess` can get the strict PHP-entry-point
allowlist that `ssreact/public/.htaccess` already has (full writeup:
plan's S2). Right now `ssapi/.htaccess` only has the safe, additive
deny rules — converting it to "deny every `.php` except
`api.php`/`media.php`/`index.php`" risks breaking the vanilla frontend
(`simple-social`'s own copy, not this repo) if it relies on some other
PHP file reachable through that docroot.

```bash
ls *.php    # run in both app.davidfruin.com's and dev.davidfruin.com's docroots
```

If the only files are `api.php`, `media.php`, and (maybe)
`index.maintenance.php`, the allowlist is safe to add as-is. If
anything else shows up, it needs its own line in the allowlist before
the conversion happens.

## 3. Duplicate emails on real data

S14 (atomic registration) added a case-insensitive unique index on
`users.email`. The migration already skips creating that index safely
if duplicates exist (logs it rather than failing), so nothing breaks
either way — but it's worth knowing before relying on the index being
there.

```bash
sqlite3 <dev's private dir>/userdata.db \
  "SELECT LOWER(email), COUNT(*) FROM users GROUP BY 1 HAVING COUNT(*) > 1"
```

Repeat against prod's DB if reachable. Empty output = index is live
there. Any rows = report them (emails only, that's fine for this vault)
so the duplicates can be resolved before the index matters.

## 4. Post-deploy checks (only after Dave deploys S2/S12/P7 to react.davidfruin.com)

These need the actual deployed site, so they're not useful until
`ssreact/public/.htaccess` and `dist/` are live there. Once they are:

**4a. The plan's own curl loop** (same one used to confirm the original
problem):
```bash
for p in logs/api.log logs/media.log config.php composer.json src/Auth/handlers.php vendor/autoload.php media/; do
  printf '%-28s %s\n' "$p" "$(curl -s -o /dev/null -w '%{http_code}' https://react.davidfruin.com/$p)"; done
```
Expect all `403`/`404`. Anything else means the deployed `.htaccess`
isn't what's in the repo, or Apache config on the host overrides it.

**4b. Still-working checks** (the allowlist/cache rules shouldn't have
broken anything real):
```bash
curl -s -o /dev/null -w '%{http_code}\n' https://react.davidfruin.com/feed       # expect 200
curl -s -o /dev/null -w '%{http_code}\n' -X POST https://react.davidfruin.com/api.php -d action=login   # expect a real JSON response, not a 403/404
curl -sI https://react.davidfruin.com/assets/<any .js file>   # expect Cache-Control: public, max-age=31536000, immutable
curl -sI -H 'Accept-Encoding: gzip' https://react.davidfruin.com/   # expect Content-Encoding: gzip
```

**4c. The one thing the local bench genuinely could not test:** whether
push notifications (P2, deferred to run after the response is sent)
actually release the client early under this host's real PHP-FPM.
`php -S` (the bench) has no FastCGI worker pool, so `fastcgi_finish_request()`
was never exercised for real — the deferred mechanism was proven correct
(ECDH+curl genuinely runs after `respond()` returns, and still cleans up
a dead subscription correctly), just not proven *fast* on real infra.

```bash
# Trigger a like/comment/follow against a recipient with at least one
# real push subscription, timed from outside:
time curl -s -X POST https://react.davidfruin.com/api.php \
  -H "Authorization: Bearer <a real token>" -d action=likePost -d postId=<a real post>
```
Expect well under 1 second. If it's taking multiple seconds (roughly
however long a slow push attempt takes), `fastcgi_finish_request()`
isn't actually releasing the client on this host, and P2's real-world
benefit doesn't exist yet even though the code is correct.

## 5. Not an agent's call — flag for Dave, don't act

- Click through every ssreact page with DevTools open once the CSP
  (S12, currently Report-Only) is live, watching the console for
  violations, **before** renaming the header to the enforcing
  `Content-Security-Policy`. This is Dave's decision to make the switch,
  not something to do automatically even if zero violations show up.
- S9 (drop vs. dedupe unlike/unfollow notifications) and the D1–D8
  structural items (real follows table, HttpOnly-cookie refresh tokens,
  dropping dead columns, likeCount-not-likes-array, and D6 — how/when
  `ssapi` replaces `simple-social`'s backend copy on app/dev, since prod
  gets none of this session's fixes until that happens) are all still
  open. Don't implement any of them without Dave explicitly picking an
  option first — that's what GATED/DECISION meant in the original plan.

---

## Reporting back

Update [[ssapi]]'s war-table note (its "Status (2026-10-02)" section
has the full checklist this file is based on) with whatever you find —
don't just leave results in this Inbox file. If #2 or #3 turn up
something that changes what the code should do, that's a new task for
an implementing agent, not something to fix inline during a
verification pass.
