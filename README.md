# Personal Telegram Bot — Cloudflare Worker + D1 + KV + GramIO

A lightweight, privacy-oriented personal Telegram bot. Runs entirely on
Cloudflare (Worker + D1 + KV), built with [GramIO](https://gramio.dev), a
type-safe TypeScript Telegram Bot API framework.

This is a **personal** bot, not an enterprise platform: no permanent
user/message-history tables, minimal D1 persistence, temporary KV state only
where a feature genuinely needs it.

## Stack

- **Cloudflare Workers** — the whole bot, one reusable URL
- **Cloudflare D1** — minimal persistent SQL (content, buttons, published
  message ids, optional scheduled tasks)
- **Cloudflare KV** — temporary conversation state only, always with a TTL
- **GramIO** — Telegram Bot API framework
- **Cloudflare Cron Triggers** — scheduled publishing, checked every minute

Nothing else: no VPS, no Docker, no Postgres/Redis, no always-on process.

## Project layout

```
telegram-personal-bot/
├── src/
│   ├── index.ts          # Worker entry: fetch (webhook) + scheduled (cron)
│   ├── webhook.ts         # secret-token validation, parses the Update
│   ├── router.ts          # registers commands + update-type routing
│   ├── telegram.ts        # Bot factory, isOwner() check
│   ├── env.ts              # Env bindings/secrets type
│   ├── handlers/
│   │   ├── message.ts      # conversation continuation + normal-user path
│   │   ├── callback.ts     # join-request approve/decline buttons
│   │   ├── owner.ts        # every owner-only admin command
│   │   └── join-request.ts # notifies the owner of new join requests
│   ├── admin/
│   │   ├── content.ts      # create/edit content
│   │   ├── publish.ts      # publish / edit / delete published messages
│   │   └── buttons.ts      # add buttons, propagate URL changes
│   └── storage/
│       ├── d1.ts            # all D1 queries
│       └── kv.ts            # temporary conversation state (TTL'd)
├── migrations/001_init.sql
├── scripts/
│   ├── register-webhook.ts   # run locally — never exposes the token publicly
│   └── unregister-webhook.ts
├── package.json
├── tsconfig.json
├── wrangler.jsonc
├── .dev.vars.example
└── .gitignore
```

## Setup

1. **Create the bot** with [@BotFather](https://t.me/BotFather); copy the token.
2. **Get your Telegram user id** from [@userinfobot](https://t.me/userinfobot) — this is `OWNER_ID`.
3. `npm install`
4. Create the D1 database and KV namespace:
   ```
   npx wrangler d1 create personal-bot-db
   npx wrangler kv namespace create STATE
   ```
   Copy the returned `database_id` / `id` values into `wrangler.jsonc`.
5. Run the migration:
   ```
   npm run db:migrate:local    # for local dev
   npm run db:migrate:remote   # once, against production
   ```
6. Copy `.dev.vars.example` to `.dev.vars` and fill in `TELEGRAM_BOT_TOKEN`,
   `TELEGRAM_WEBHOOK_SECRET` (any long random string), and `OWNER_ID`.

## Local development

```
npm run dev
```

Wrangler's local dev server reads `.dev.vars` automatically. To actually
receive Telegram updates locally you'll need a tunnel (e.g. `cloudflared
tunnel` or `ngrok`) pointed at the dev server, then register that tunnel URL
with the `register` script below.

## Deploying

```
npm run deploy
```

Then set the real secrets on the deployed Worker (these are **not** read
from `.dev.vars` in production):

```
npx wrangler secret put TELEGRAM_BOT_TOKEN
npx wrangler secret put TELEGRAM_WEBHOOK_SECRET
npx wrangler secret put OWNER_ID
```

Deploy gives you a permanent URL like `https://telegram-personal-bot.<your-subdomain>.workers.dev`.

## Registering the webhook

Registration runs **locally**, not through an in-Worker endpoint — this
keeps the bot token from ever being sent to a public URL as part of a
registration flow:

```
npm run register -- https://telegram-personal-bot.<your-subdomain>.workers.dev
```

This calls Telegram's `setWebhook` with your Worker's `/telegram/webhook`
path and the secret token, and restricts delivery to the update types this
bot actually handles (`message`, `callback_query`, `chat_join_request`).

To remove it later: `npm run unregister`.

## Owner commands

| Command | Purpose |
|---|---|
| `/new <title> \| <body>` | Create a piece of content |
| `/list` | List recent content with ids |
| `/publish <contentId> <chatId1,chatId2,...>` | Publish to one or more chats |
| `/edit <contentId> \| <new body>` | Edit content and update every published copy |
| `/unpublish <contentId>` | Delete every published copy |
| `/addbutton <contentId> <text> \| <url>` | Add a button to content |
| `/buttons <contentId>` | List a content item's buttons with ids |
| `/setbuttonurl <buttonId>` | Start a short flow to change a button's URL everywhere it's posted |
| `/schedule <contentId> <chatIds> <2026-09-20T10:00>` | Schedule a future publish |
| `/reply <ticketId> <message>` | Reply to an anonymous support request |
| `/cancel` | Cancel an in-progress flow |

All of these silently refuse for anyone whose Telegram id doesn't match
`OWNER_ID` — there's no admins table and no username-based check (spec
section 5).

## Normal users

Any plain text message from a non-owner is relayed to the owner as an
anonymous, numbered request (`Anonymous request #ab12cd`) without exposing
the sender's Telegram id or username. The owner replies with `/reply
<ticketId> <message>`. The ticket→user mapping lives in KV with a 24-hour
TTL — no permanent ticket history unless you extend it.

## Privacy

- No `users`, `message_history`, or `analytics` tables — D1 only stores
  `bot_config`, `content`, `buttons`, `published_messages`, and
  `scheduled_tasks`.
- KV holds only short-lived conversation/session state, always with a TTL.
- The bot token is a Worker secret, never stored in D1 or committed to
  source control.

## A note on how this was built

This project uses GramIO's documented APIs (`Bot`, `.command()`, `.on()`,
`.callbackQuery()`, `context.send()`, `InlineKeyboard`, `CallbackData`,
`bot.api.<method>`, `bot.handleUpdate()`) and Cloudflare's standard D1/KV
bindings. A few context property names (`context.chat`, `context.from`) are
GramIO conventions I'm confident about but couldn't execute-verify in this
sandbox (no network/npm access here), so the handlers read sender/chat IDs
via `context.payload` — the raw Telegram `Message`/`CallbackQuery`/
`ChatJoinRequest` object — instead, since that shape is guaranteed by the
Bot API regardless of GramIO's exact shortcut naming.

What I *did* verify: the whole TypeScript project type-checks cleanly
against stubs built from GramIO's documented API surface, and the full D1
schema plus every query in `src/storage/d1.ts` (upserts, cascading
deletes, the due-tasks filter) was executed against a real SQLite engine
with realistic data — all passed, including `ON DELETE CASCADE` behavior.
One open question from that: the cascade test only passed with `PRAGMA
foreign_keys = ON` explicitly set, and I don't have certainty whether
Cloudflare D1 enables foreign-key enforcement by default. Worth checking
after your first `/unpublish` or content-delete — if buttons/published rows
don't get cleaned up automatically, add `PRAGMA foreign_keys = ON;` at the
top of `migrations/001_init.sql` and re-run it, or D1 may already handle
this per-statement (Cloudflare's docs are the source of truth here, not my
training data).

Also worth running `npm run dev` early to confirm the GramIO wiring behaves
as expected before relying on it — and note that `wrangler d1` migration
commands (`wrangler d1 migrations apply`) exist as an alternative to the
plain `wrangler d1 execute` used in the `db:migrate:*` scripts here, if you
want migration tracking as the schema grows.
