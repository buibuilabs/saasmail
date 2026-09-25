[saasmail](../README.md) › [Docs](README.md) › **Keeping this clone updated**

# Keeping this fork updated

Operator runbook for **this** checkout (`~/code/saasmail` on Ubuntu WSL), the
[buibuilabs/saasmail](https://github.com/buibuilabs/saasmail) fork. Upstream’s
generic notes live in [Updating](updating.md).

Agents: start here when the user asks how to update, pull upstream, or
redeploy. Commands run in **WSL Ubuntu**, not Windows. Node **22** via nvm
(`nvm use 22`; default alias is already `22`). Production deploys use
`yarn deploy`.

## Deploy this instance

```bash
cd ~/code/saasmail
yarn deploy
```

That builds the fork as committed (Discord webhook formatting, GooglyAI Mail
title, favicons, and `wrangler.jsonc`) and publishes `mail.googlyai.app`.

Header text is still Settings → App branding (`brand_name` in D1). It is
already `GooglyAI Mail`. The tab title follows that value after `/api/config`
loads, and `index.html` uses the same title for the first paint.

## This deployment

| Item                     | Value                                                                                  |
| ------------------------ | -------------------------------------------------------------------------------------- |
| Live UI                  | https://mail.googlyai.app                                                              |
| Worker name              | `saasmail`                                                                             |
| Cloudflare account ID    | `24d0fd4dee694044dbe6f9afdac31891`                                                     |
| Inbound / send-from zone | `googlyai.app`                                                                         |
| Git `origin`             | `git@github.com:buibuilabs/saasmail.git`                                               |
| Git `upstream`           | `https://github.com/choyiny/saasmail.git`                                              |
| Fork?                    | Yes. This checkout is the buibuilabs fork. Custom code lives in the repo.              |
| Config                   | `wrangler.jsonc` is committed for this fork's CI deploy. `.dev.vars` stays gitignored. |
| Deploy                   | `yarn deploy` from WSL.                                                                |

Local Durable Object migration uses `new_sqlite_classes` (this account rejects
KV-backed `new_classes`). That setting is in the committed `wrangler.jsonc`.

## Update from upstream

[`.github/workflows/sync-upstream.yml`](../.github/workflows/sync-upstream.yml)
merges `choyiny/saasmail` `main` into this fork’s `main` every Monday, and
whenever someone runs the **Sync upstream** workflow by hand. It pushes the
merge commit. If the merge conflicts, the job fails and nothing is pushed —
resolve it locally, then push.

After a sync lands, from WSL:

```bash
cd ~/code/saasmail
git pull origin main
yarn install --frozen-lockfile
yarn db:migrate:prod
yarn deploy
```

`.dev.vars` is ignored, so pulls do not overwrite local secrets. After a pull,
skim `wrangler.jsonc.example` for new bindings or vars and add anything this
fork needs to the committed `wrangler.jsonc`.

`yarn db:migrate:prod` applies **pending** D1 SQL only. See
[Data recovery](data-recovery.md#migrations-and-data).

## Day-to-day commands

```bash
cd ~/code/saasmail
yarn dev
yarn deploy
yarn db:migrate:prod
npx wrangler tail saasmail
```

## Finish mail plumbing (if inbound/outbound still broken)

These are Cloudflare dashboard settings, not git:

1. **Inbound** — zone `googlyai.app` → **Email → Email Routing** → enable →
   catch-all → **Send to a Worker** → `saasmail`.
2. **Outbound** — [Email Service](https://dash.cloudflare.com/?to=/:account/email-service)
   → add `googlyai.app` → wait until **Verified**.

Compose **From** addresses come from **Inboxes**
(https://mail.googlyai.app/inboxes), not from Wrangler. Sending still requires
Email Service verification.

## What not to do

- Do not `wrangler d1 create` / recreate `saasmail-db` or the R2 bucket.
- Do not set `DISABLE_PASSKEY_GATE` or `DEMO_MODE` as production secrets.
- Do not rewrite `origin` / `upstream` remotes.
- Do not force-push to `choyiny/saasmail` or to this fork’s `main`.
- Do not commit `.dev.vars` or `.brand/`.

---

**See also:** [Data recovery](data-recovery.md) · [Updating](updating.md) (upstream) ·
[Setup](setup.md) · [Configuration](configuration.md) · [CHANGELOG](../CHANGELOG.md)
