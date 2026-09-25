[saasmail](../README.md) › [Docs](README.md) › **Data recovery**

# Data recovery

Where this instance’s data lives, what migrations do to it, and how to restore
it. Pair with [Keeping this clone updated](keep-updating.md).

Agents: do **not** delete D1, R2, or the Worker as a “fix.” Time Travel or an
SQL export is the recovery path. Commands run in **WSL Ubuntu** from
`~/code/saasmail`.

## What persists where

| Store          | Resource                                                        | What it holds                                                                                      | Survives `yarn deploy`?      |
| -------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------- |
| D1             | `saasmail-db` (`d5646acc-89aa-463f-9ee8-5a79f18f2661`)          | Users, passkeys, inboxes, people, mail metadata/bodies, templates, sequences, suppressions, drafts | Yes                          |
| R2             | `saasmail-attachments`                                          | Attachment bytes                                                                                   | Yes                          |
| Queue          | `saasmail-sequence-emails`                                      | In-flight sequence jobs (ephemeral)                                                                | n/a                          |
| Durable Object | `NotificationsHub`                                              | Live notification fan-out, not the mail archive                                                    | Redeploy keeps the namespace |
| Worker secrets | `BETTER_AUTH_SECRET`, `UNSUBSCRIBE_SECRET`, `VAPID_PRIVATE_KEY` | Session signing, unsubscribe HMAC, Web Push                                                        | Yes (not in git)             |
| Dashboard      | Email Routing + Email Service on `googlyai.app`                 | MX / catch-all → worker, send-from verification                                                    | Independent of deploys       |

Redeploying the Worker does **not** reset D1 or R2. Recreating `saasmail-db`
**would** wipe mail. The gitignored `wrangler.jsonc` is how this checkout finds
the existing database; losing that file is annoying, not fatal — IDs are in
this page.

## Migrations and data

`yarn db:migrate:prod` is **not** a restore and **not** a factory reset.

- Wrangler applies only SQL files in `migrations/` that this D1 has not run
  yet, in journal order (`migrations/README.md`).
- Typical files are `CREATE TABLE`, `ALTER TABLE … ADD COLUMN`, indexes, and
  occasional `UPDATE` backfills. Existing rows stay.
- Re-running when you are current is a no-op.
- It does not recreate D1, R2, queues, or secrets.

Before applying a pull, glance at new `migrations/NNNN_*.sql`. Stop and ask if
you see `DROP TABLE`, destructive `DELETE`, or a rename implemented as
drop + create.

Local D1 (`yarn db:migrate:dev`, `yarn test:e2e`) is a **different** database
under `.wrangler/`. E2E **wipes local D1** only. Production is untouched.

## Built-in recovery: D1 Time Travel

Always on for this production D1. No extra setup. Restore to any minute in the
**last 30 days** (Workers Paid). Bookmarks older than 30 days cannot be used.

```bash
cd ~/code/saasmail

# Current bookmark
npx wrangler d1 time-travel info saasmail-db

# Bookmark at a past timestamp (RFC3339 or Unix seconds)
npx wrangler d1 time-travel info saasmail-db --timestamp="2026-09-17T20:00:00+00:00"

# Restore in place — destructive. Last resort (bad migration, runaway DELETE).
npx wrangler d1 time-travel restore saasmail-db --timestamp="2026-09-17T20:00:00+00:00"
```

Restore overwrites the live database. In-flight queries fail. The CLI prints a
bookmark you can use to **undo** the restore. Time Travel does **not** roll
back R2 attachments, Worker code, or Cloudflare Email dashboard settings.

Docs: [D1 Time Travel](https://developers.cloudflare.com/d1/reference/time-travel/).

## Longer than 30 days: SQL export

Time Travel is not archival. If mail is worth keeping past 30 days, dump D1
occasionally to somewhere durable (disk, another R2 bucket, off-account):

```bash
cd ~/code/saasmail
npx wrangler d1 export saasmail-db --remote --output "$HOME/saasmail-d1-$(date +%Y%m%d).sql"
```

Schema-only:

```bash
npx wrangler d1 export saasmail-db --remote --output "$HOME/saasmail-d1-schema.sql" --no-data
```

Restoring from a SQL file is a separate D1 (or a careful execute against the
existing one). Prefer Time Travel for “undo the last hour.” Use the export for
“Cloudflare account is gone” / “I need a copy from months ago.”

## R2 attachments

Time Travel does not cover `saasmail-attachments`. Objects remain unless
deleted. For a full disaster copy, sync the bucket elsewhere (another R2
bucket or `wrangler r2 object` / rclone). Mail rows in D1 without the R2
object show as missing attachments.

## Secrets and dashboard (not in D1)

A D1 restore does not replace:

| Item                    | How to recover                                                                                                               |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `BETTER_AUTH_SECRET`    | Must match the original or every session/passkey path breaks. Do not rotate casually. Stored as a Worker secret, not in git. |
| `UNSUBSCRIBE_SECRET`    | Rotating invalidates outstanding unsubscribe links.                                                                          |
| `VAPID_PRIVATE_KEY`     | Must match `vars.VAPID_PUBLIC_KEY` in `wrangler.jsonc` or push dies.                                                         |
| Email Routing catch-all | Re-create: zone `googlyai.app` → Email Routing → catch-all → Worker `saasmail`.                                              |
| Email Service domain    | Re-verify `googlyai.app` at Email Service.                                                                                   |

Do not paste secret **values** into this file or into git.

## Practical default

No custom backup pipeline is required on day one. Use Time Travel if a
migration or query goes wrong. Start periodic `d1 export` once there is mail
you would hate to lose. Keep `wrangler.jsonc` and `.dev.vars` on the WSL disk
(already gitignored).

## Inventory for restore-from-scratch (worst case)

Only if the Worker and bindings are gone and Time Travel cannot help:

1. Recreate D1 / R2 / queue **with new IDs only if the old ones are deleted**.
   Prefer binding the existing IDs in `wrangler.jsonc`.
2. Copy `wrangler.jsonc.example` → `wrangler.jsonc`; fill account id, D1 id,
   `routes` → `mail.googlyai.app`, `send_email`, `BASE_URL`, VAPID public key.
   Keep Durable Object migration as `new_sqlite_classes`.
3. Re-put secrets (`BETTER_AUTH_SECRET` must be the old value if you still
   have user rows).
4. Restore D1 from Time Travel or import a SQL export.
5. `yarn deploy`.
6. Re-attach Email Routing + Email Service.

---

**See also:** [Keeping this clone updated](keep-updating.md) ·
[Configuration](configuration.md) · [Setup](setup.md) ·
[migrations/README.md](../migrations/README.md)
