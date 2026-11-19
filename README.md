# Nomi

An English-language Discord community bot with **57 slash commands**, built with Python 3.12, discord.py 2.7.1 and SQLite. Nomi combines moderation and self-roles with a configurable server economy and template responders. Original code; no connection to Carl-bot or Mimu accounts is needed.

**Монгол заавар: [START_HERE_MN.md](START_HERE_MN.md).** Full command reference: [COMMANDS.md](COMMANDS.md).

## Included

| Area | Features |
| --- | --- |
| Moderation | Kick, ban/unban, timeout, warnings, purge, slowmode, channel lock/restore |
| Automod | Literal blocked phrases, Discord invite filter, message rate limiting |
| Roles | Persistent toggle buttons, emoji reaction roles, new-member autorole |
| Economy | Per-server wallets, configurable currency, daily/work rewards, transfers, leaderboard |
| Shop | Inventory items, cosmetic role purchases, persisted orders and automatic delivery retries/refunds |
| Community | Welcome/goodbye, template autoresponders, slash-invoked tags, embed builder |
| Support | Private ticket channels, owner/staff access, read-only closure, text transcripts |
| Starboard | Unique human star counting, self-star exclusion, public source channels only |
| Utilities | Help, setup diagnostics, latency, server/member information, avatars, invite link |

The feature selection was inspired by [Carl-bot's listed community tools](https://carl.gg/) and [Mimu's server economy and slash-command approach](https://docs.mimu.bot/). This is an independent implementation, not a feature-for-feature clone. It does not include their dashboards, premium features, Mimu's scripting language/pets/global currency, leveling, giveaways, music, feeds, or scheduled messages.

## Discord setup

Create a bot application in the [Developer Portal](https://discord.com/developers/applications). Set its display name to Nomi and put its token in a local `.env` copied from `.env.example`. Enable **Server Members Intent** and **Message Content Intent** on its Bot page. These are required for Nomi's join/leave and message features. See [discord.py's intents guide](https://discordpy.readthedocs.io/en/stable/intents.html).

Install it to your server with the `bot` and `applications.commands` scopes. Use Guild Install, not User Install. This bot connects through the Gateway; it does not need an Interactions Endpoint URL. Token setup is described in [Discord's bot guide](https://docs.discord.com/developers/quick-start/getting-started).

Recommended permission integer: `1374658358358`. This covers viewing/sending messages, embeds, attachments, history, reactions/external emoji, sending in threads, managing messages/roles/channels, kicking, banning and moderating members. Nomi does not request Administrator. Channel overrides can still deny access. Position its role above any roles or members it should manage.

Use `DEV_GUILD_ID` for commands in one test server. Leave it blank to sync global commands. On startup Nomi syncs its command tree; do not restart repeatedly to force propagation. Switching from test-server commands to global commands may leave the old guild commands visible in the test server. To remove those, use discord.py's `tree.clear_commands(guild=...)` followed by `tree.sync(guild=...)` in a deliberate maintenance script; Nomi does not silently delete another scope's commands.

## Run on Windows

Extract the project, then double-click `start.bat`. On first launch it creates `.env` and asks you to edit it. On the next launch it creates a virtual environment, installs the pinned dependency if needed, and runs Nomi. Keep the console open.

Or run manually from the project folder:

```powershell
python -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
# Edit .env locally.
.venv\Scripts\python.exe main.py
```

On Linux/macOS, use `.venv/bin/python` and copy `.env.example` to `.env` with your normal file manager or shell. Stop gracefully with Ctrl+C.

## Docker deployment

Set `.env`, then use:

```sh
docker compose up -d --build
docker compose logs -f nomi
```

The image runs as a non-root user. The named `nomi-data` volume stores SQLite data; normal container restarts preserve it. Do not remove the volume when updating. Run **one replica**; this package is intended for a single bot process, not a distributed/sharded deployment. Docker build and live Discord operation have not been executed in this workspace.

## Behavior and boundaries

- **All commands use `/`.** Autoresponders additionally react to ordinary messages. Saved responses are invoked with `/tag name`; arbitrary new top-level slash command names are not generated.
- Manage Server is required for configuration and shop administration. Moderation actions enforce runtime permissions, the caller's role hierarchy, and the bot's hierarchy. `/mod` is visible by default to members with Moderate Members; use Discord's Integration command permissions if your staff roles split capabilities differently. Runtime action permissions still apply.
- Self-roles, shop roles and autoroles must have no server permissions, no dangerous channel permission grants, and sit below Nomi. Autoroles wait for membership screening; pending members cannot claim self-roles. Granting or removing these cosmetic roles is not a membership verification system.
- Wallets and cooldown timestamps are per server. Daily is a rolling 24-hour cooldown; work is one hour. Wallet cap: 1 trillion. Currency has no payment or real-money integration.
- Purchases commit to SQLite before role delivery. Permanent role delivery failures are refunded once; temporary Discord errors leave the order pending. Pending orders retry every minute and after restart. A pending member must complete screening before delivery. Inventory entries for role purchases are receipts, not an ongoing entitlement that re-grants a manually removed role. Ordinary shop items are collectible inventory entries.
- Autoresponders support case-insensitive `exact`, `contains` and `startswith` matching, with one response per user per five seconds. Supported placeholders: `{user}`, `{username}`, `{server}`, `{count}`. No code evaluation, regex, arbitrary role actions, or Mimu/Carl scripting syntax. Mentions do not ping.
- Automod deletes matching messages; it does not automatically ban or timeout members. Staff with Manage Messages are exempt. Blocked phrases are literal substrings. Invite filtering and anti-spam are basic protection, not a comprehensive anti-raid service. Spam and responder rate limits reset on restart.
- `/mod lock` saves and changes only the @everyone Send Messages overwrite. Explicit member/role allow overwrites and threads can still allow messages. `/mod unlock` restores the saved value.
- Each ticket is visible to its owner, the configured support role, Nomi, and Discord administrators. Closing preserves the channel and history, prevents the owner from sending, and allows a new ticket. Administrators bypass channel restrictions. Staff remain able to reply. Delete closed channels manually when no longer needed. The transcript exports at most 500 messages and links attachments; it does not download them.
- Starboard excludes source channels with any explicit View Channel deny, including the usual private ticket channels. NSFW sources cannot be mirrored to SFW destinations. Existing posts refresh on reaction events; channel permission changes alone do not retroactively erase copied text. Keep source channels public or remove old highlights when changing their audience. Stars added or removed while the bot is offline are reconciled on the next star event for that message.
- Edit/delete logs contain message text for cached messages only. Messages outside the cache, events during downtime, and bulk-deletion history are not fully audited. Choose a private staff log channel. There is no automatic data retention policy.
- Role buttons restore after restart. Reaction role changes that occur while the bot is offline are not backfilled; users should remove/re-add their reaction. Panel removal stops future assignments but keeps current roles.

## Data and maintenance

SQLite lives at `data/nomi.sqlite3` by default. Its WAL sidecar files can contain recent data. Stop Nomi before copying the entire `data` directory for backup, or use SQLite's backup API. Keep `.env` private. User IDs, wallets, transaction records, warnings, ticket ownership and server settings persist until the operator removes them. Message text is not stored in SQLite; configured Discord log channels and exported transcripts can contain text.

For a small community the synchronous, short database transactions keep deployment simple. A large public bot would need additional scaling work, metrics, migration tooling and lifecycle/data-retention controls.

## Verification

```sh
python main.py --check
python -m unittest discover -s tests -v
```

These checks do not load `.env`, require a token, or connect to Discord. The check loads all extensions and validates the command schema. The **37 automated tests** cover transaction rollback, concurrent reward claims, cooldown persistence, per-server isolation, purchase refunds/retries, permissions and hierarchy, membership screening, ticket access, automod and persistent views. Results are recorded in [VERIFICATION.md](VERIFICATION.md).

Before using moderation on a real community, test in a private server: check a regular member cannot use staff actions; buy a cheap cosmetic role; click a role button after restarting; verify an unrelated member cannot see a ticket; and try warning/timeout on a test account below Nomi's role.
