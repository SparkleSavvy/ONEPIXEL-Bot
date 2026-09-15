# ONEPIXEL-Bot

A Discord bot for managing a **local** Minecraft Java server with a full RBAC system, an interactive control panel, and an RCON console bridge.

- **Language / stack:** Python 3.12+, discord.py 2.7, aio-mc-rcon, aiosqlite, psutil, pydantic-settings
- **Persistence:** SQLite (auto-created on first run)
- **Platform:** Windows / Linux / macOS (anywhere Python + a running Minecraft server live)

---

## Table of contents

- [Overview](#overview)
- [Features](#features)
- [Requirements](#requirements)
- [Quick start](#quick-start)
  1. [Create a Discord application](#1-create-a-discord-application)
  2. [Enable RCON on the Minecraft server](#2-enable-rcon-on-the-minecraft-server)
  3. [Install dependencies](#3-install-dependencies)
  4. [Configure the bot](#4-configure-the-bot)
  5. [Run the bot](#5-run-the-bot)
- [Configuration reference](#configuration-reference)
- [Command reference](#command-reference)
  - [Server control](#server-control)
  - [Console](#console)
  - [RBAC](#rbac)
  - [Panel](#panel)
- [The control panel](#the-control-panel)
- [The RBAC system](#the-rbac-system)
  - [How it works](#how-it-works)
  - [Permissions](#permissions-table)
  - [Setting up your first roles](#setting-up-your-first-roles)
  - [Linking a bot role to a Discord role](#linking-a-bot-role-to-a-discord-role)
  - [Guardrails](#guardrails)
- [Audit log](#audit-log)
- [Database](#database)
- [Logging](#logging)
- [How it works under the hood](#how-it-works-under-the-hood)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## Overview

ONEPIXEL-Bot lets your Discord community start, stop, restart and monitor the Minecraft server directly from Discord — without giving anyone shell access. It adds:

- slash commands (`/start`, `/stop`, …),
- a **persistent but panelless-style interactive embed** with buttons (auto-refreshing),
- an **RCON console** for admin commands and chat,
- a **per-guild RBAC system** so you can decide exactly who may do what.

The bot, the database, and the Minecraft server all run on the same machine (`127.0.0.1` by default). Every guild the bot joins gets its own isolated role/permission state, its own panel message, and its own server-process record.

---

## Features

| Area | Capability |
|------|-----------|
| Server lifecycle | `/start`, `/stop`, `/restart` with sane cooldowns |
| Status monitoring | players (count + names + max), uptime, RAM, CPU, version via psutil + RCON |
| Control panel | persistent embed with **Start / Stop / Restart / Console / Refresh** buttons, auto-refreshes every 30 s |
| RCON console | `/cmd` — run any server command; `/say` — speak in the in-game chat; Console modal on the panel |
| RBAC | custom bot roles (not Discord roles) with 8 granular permissions, optional Discord-role mapping, per-guild |
| Auditing | every action (start, stop, `/cmd`, RBAC changes, …) is recorded in `audit_log` |
| Robustness | server PID tracked in the DB; orphaned (crashed) processes detected and cleaned on startup |

---

## Requirements

- **Python 3.12+**
- A **Minecraft Java server** (Vanilla, Paper, Spigot, Fabric/Quilt with a RCON-mod, …) running on the same machine as the bot
- A **Discord bot token** and the **application commands** scope enabled

---

## Quick start

### 1. Create a Discord application

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications) and create a new application.
2. Open **Bot** → **Reset Token** → copy the token (put it in `.env` as `DISCORD_BOT_TOKEN`).
3. Invite the bot using **OAuth2 → URL Generator** with scopes **`bot`** and **`applications.commands`**. Grant the permissions you want the bot itself to have (none are strictly required — the bot manages roles in its own database, not Discord).
4. (Optional but recommended during setup) Invite it to a single test guild and set `TEST_GUILD_ID` in `.env` so commands appear **instantly** in that guild instead of after Discord's global sync delay (up to 1 hour).

### 2. Enable RCON on the Minecraft server

RCON is required for: player listing, `/cmd`, `/say`, the Console modal, and graceful `/stop`. Edit `server.properties` in the server directory:

```properties
enable-rcon=true
rcon.port=25575
rcon.password=<a strong, unique password>
```

Restart the Minecraft server afterwards so the settings take effect.

> If RCON is disabled, `/start`, `/stop` (via RCON), `/restart` and the status player/version fields will not work fully — see [Troubleshooting](#troubleshooting).

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

Installs `discord.py`, `aio-mc-rcon`, `pydantic-settings`, `aiosqlite`, `psutil` and `python-dotenv`.

### 4. Configure the bot

```bash
cp .env.example .env
```

Edit `.env` — at minimum set **`DISCORD_BOT_TOKEN`**, **`RCON_PASSWORD`**, and fill in `MC_START_SCRIPT` / `MC_SERVER_DIR` if your setup differs from the defaults. See the [Configuration reference](#configuration-reference).

### 5. Run the bot

```bash
python -m bot.main
```

On startup the bot:

1. creates the folder and SQLite database (`data/bot.db` by default),
2. re-registers the persistent panel view (buttons keep working after restarts),
3. for every guild it is already in, creates the `Owner` role and assigns it to the guild owner,
4. scans the DB and **marks any dead/crashed server process as stopped**.

Then it logs in to Discord and syncs the slash commands.

---

## Configuration reference

All settings are read from **environment variables** or a `.env` file in the project root (`bot/config.py`).

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `DISCORD_BOT_TOKEN` | — | ✅ | Bot token from the Developer Portal. |
| `TEST_GUILD_ID` | *(empty)* | ❌ | Dev guild ID. When set, commands are synced **to that guild only and instantly** (avoiding the global 1-hour sync delay). Leave empty for global/balanced rollout. |
| `MC_SERVER_HOST` | `127.0.0.1` | ❌ | Server host, shown in the panel description. |
| `MC_SERVER_PORT` | `25565` | ❌ | Game port, shown in the panel description. |
| `RCON_HOST` | `127.0.0.1` | ❌ | Host of the RCON endpoint. |
| `RCON_PORT` | `25575` | ❌ | Port of the RCON endpoint. |
| `RCON_PASSWORD` | *(empty)* | ✅ **for RCON** | Must match `rcon.password` in `server.properties`. |
| `MC_START_SCRIPT` | `java -Xmx2G -Xms1G -jar server.jar nogui` | ❌ | Shell command used to launch the server. Must be runnable from `MC_SERVER_DIR`. |
| `MC_SERVER_DIR` | `./server` | ❌ | Working directory for the server process (where `server.jar`, `world`, `logs/` live). |
| `DATABASE_PATH` | `data/bot.db` | ❌ | SQLite database file. Parent directory is created automatically. |
| `PANEL_REFRESH_INTERVAL` | `30` | ❌ | Seconds between automatic panel-embed refreshes. |

Notes:

- pydantic parses ints from `.env`, so values like `25565` and `30` are already numbers.
- Leave `TEST_GUILD_ID` empty in production if you want commands in every guild; otherwise you can set a real guild so updates apply instantly for your community.
- `MC_START_SCRIPT` runs through the system shell with stdout/stderr discarded; the bot tracks the PID, not the console output. Use RCON (or the server's own `logs/latest.log`) to see server output.

---

## Command reference

All commands are **guild-only** (usable only in text channels of a server, not in DMs). Response messages are ephemeral (visible only to the invoker) unless noted.

| Category | Command | Permission required |
|----------|---------|---------------------|
| Server | `/start` | `SERVER_START` |
| Server | `/stop` | `SERVER_STOP` |
| Server | `/restart` | `SERVER_RESTART` |
| Server | `/status` | `VIEW_STATUS` |
| Console | `/cmd <command>` | `CONSOLE_EXECUTE` |
| Console | `/say <message>` | `CONSOLE_SAY` |
| Panel | `/panel` | `VIEW_PANEL` |
| RBAC | `/rbac …` (all subgroups) | `MANAGE_ROLES` |

### Server control

- **`/start`** — launches the server via `MC_START_SCRIPT` in `MC_SERVER_DIR`. If the server is already running, returns `Server is already running.`
- **`/stop`** — sends RCON `stop`, then waits (up to 30 s) for the process to exit. If the server was not online, returns `Server is not running.`
- **`/restart`** — graceful stop, then a 3-second pause, then start. If it wasn't running, it just starts.
- **`/status`** — current status:

  ```
  🟢 Server: Online
  Players: Steve, Alex
  Max players: 20
  Uptime: 2h 14m 5s
  RAM: 1483 MB
  CPU: 42.3%
  Version: 1.21.*
  ```

  Status indicator: 🟢 online · 🔴 offline · 🟡 starting/stopping.

`/start`, `/stop` and `/restart` share a **per-guild cooldown of 1 call per 10 seconds** to protect against button mashing (each independent action per server, not per user).

### Console

- **`/cmd <command>`** — executes an arbitrary RCON command and returns its textual response inside a code block (truncated to 1900 chars). Examples: `/cmd list`, `/cmd op Steve`, `/cmd time set day`, `/cmd give Steve diamond 4`.
- **`/say <message>`** — sends a message to the in-game chat (executes `say <message>` via RCON). Useful for announcements.

> Because Discord also tries to parse your message as a slash command, prefix anything that starts with `/` inside the `command` argument — not needed: the argument is passed verbatim. However, Discord will offer unknown-command corrections; simply ignore them. Free text is fine.

> RCON packets are capped (≈4 KB); very long commands may be rejected by the protocol. The Console modal caps input at 1446 characters; `/cmd` accepts more but the server may still refuse oversized packets.

### RBAC

See the dedicated [RBAC section](#the-rbac-system) for full details. Command summary:

```
/rbac role  create <name>
/rbac role  delete <name>
/rbac role  list
/rbac role  link-discord-role <name> <discord_role>

/rbac permission grant  <role> <permission>
/rbac permission revoke <role> <permission>

/rbac user grant  <member> <role>
/rbac user revoke <member> <role>
/rbac user roles <member>
```

Role-name parameters have autocomplete; the `permission` parameter is a pick-list of the 8 permissions.

### Panel

`/panel` posts the interactive control panel into the channel where it was invoked, or **updates** the existing panel message if one already exists for the guild.

---

## The control panel

The panel is a persistent embed (`Minecraft Server — Online/Offline/…`) that live-updates every `PANEL_REFRESH_INTERVAL` seconds.

```
🟢 Minecraft Server — Online
`127.0.0.1:25565`

Players (2/20): Steve, Alex
Uptime      2h 14m 5s
RAM         1483 MB
CPU         42.3%
Version     1.21.*
```

Buttons (each re-checks permission at click time — a permission change takes effect immediately):

| Button | Permission | Behavior |
|--------|-----------|----------|
| 🟢 **Start** | `SERVER_START` | Starts the server, refreshes the panel, replies `Server start requested.` |
| 🔴 **Stop** | `SERVER_STOP` | Sends RCON `stop`, waits for exit, refreshes, replies `Server stop requested.` |
| ⚪ **Restart** | `SERVER_RESTART` | Stops + 3 s + starts, refreshes, replies `Server restart requested.` |
| ℹ️ **Console** | `CONSOLE_EXECUTE` | Opens a modal where you type an RCON command and see its response |
| ↻ **Refresh** | *(none)* | Manually re-reads status and updates the embed |

Behavior notes:

- Button interactions are **ephemeral** — the action reply is only visible to the person who clicked.
- The panel is **persistent**: buttons keep working after bot restarts because the same `custom_id`s (`panel:start`, …) are re-registered at startup and the message is stored in the DB.
- `Refresh` requires no permission — viewing status was deliberately left open so anyone can check who is online.
- If `/panel` is invoked again, the existing panel message is updated in place (whichever channel it currently lives in); if it was deleted, a new one is created in the current channel.
- If the panel message is deleted, the background refresh task notices and cleans up the stored reference.

---

## The RBAC system

### How it works

Access control is **completely independent of Discord roles**. The bot manages its own roles in the SQLite database, per guild. This means you can grant "can stop the server" to someone who has no Discord moderation role, and vice-versa — without polluting Discord.

Model:

```
role                (a named set of permissions, e.g. "Admin", "Moderator", "Player")
  ├── role_permission   (which of the 8 permissions the role grants)
  ├── user_role         (which Discord users were assigned this bot-role)
  └── discord_role_link (OPTIONAL: a Discord role that automatically grants this bot-role)
```

A user has a permission if **any** of:

1. they have a direct user→bot-role assignment with that permission, **or**
2. they hold a Discord role that is linked to a bot-role with that permission.

Permission checks are performed live (SQLite + cached guild members), so changes apply instantly — no reload needed.

> **Data model note:** a `discord_role_link` **replaces** the previous link for that bot-role in that guild (`INSERT OR REPLACE`). A bot-role can be linked to at most one Discord role per guild.

### Permissions table

| Permission | Grants access to |
|-----------|------------------|
| `SERVER_START` | `/start`, panel **Start** |
| `SERVER_STOP` | `/stop`, panel **Stop** |
| `SERVER_RESTART` | `/restart`, panel **Restart** |
| `CONSOLE_EXECUTE` | `/cmd`, panel **Console** modal |
| `CONSOLE_SAY` | `/say` |
| `VIEW_STATUS` | `/status` |
| `VIEW_PANEL` | `/panel` |
| `MANAGE_ROLES` | the entire `/rbac` command group |

### Setting up your first roles

Worked example — a Discord server with the owner, a moderator, and everybody else.

**Step 1 — create roles**

```
/rbac role create Moderator
/rbac role create Player
```

**Step 2 — give them permissions**

```
/rbac permission grant Moderator SERVER_RESTART
/rbac permission grant Moderator CONSOLE_EXECUTE
/rbac permission grant Moderator CONSOLE_SAY
/rbac permission grant Moderator VIEW_STATUS
/rbac permission grant Player VIEW_STATUS
/rbac permission grant Player VIEW_PANEL
```

**Step 3 — assign users (either way works)**

```
/rbac user grant @Steve Moderator
/rbac user grant @Alex Player
```

**Step 4 — verify**

```
/rbac role list          → shows each role + its count of permissions
/rbac user roles @Steve  → Moderator
```

Result: the owner can do everything (automatic `Owner` role); moderators can restart and use the console; players can check status and open the panel but **not** start/stop.

### Linking a bot role to a Discord role

Instead of (or in addition to) listing users one by one, you can make permissions follow an existing Discord role:

```
/rbac role link-discord-role Player @Members
```

Now **every Discord member who has the `@Members` Discord role** automatically gets the bot-role `Player` (and its permissions). Re-linking the same bot-role to a different Discord role replaces the mapping.

> **Limitation:** the lookup uses the guild member cache in memory. New members/roles are picked up once Discord events populate the cache (nothing extra needs to be configured).

### Guardrails

The `Owner` bot-role is created automatically when the bot joins a guild and is assigned to the guild owner. It is protected:

- ✅ cannot be deleted (`/rbac role delete Owner` → error)
- ✅ its permissions cannot be revoked
- ✅ it cannot be removed from the guild owner (but can be removed from *other* users, e.g. if ownership changes)

These protections ensure at least one account always retains full control and you never lock yourself out.

---

## Audit log

Every meaningful action is written per-guild to the `audit_log` table. There is currently no slash-command to read it — query the database directly:

```bash
sqlite3 data/bot.db "SELECT datetime(timestamp) AS when, guild_id, user_id, action, detail FROM audit_log ORDER BY id DESC LIMIT 20;"
```

| `action` | Meaning | `detail` |
|----------|---------|----------|
| `start` / `stop` / `restart` | server lifecycle via slash | *(empty)* |
| `panel_start` / `panel_stop` / `panel_restart` | server lifecycle via panel buttons | *(empty)* |
| `console` | `/cmd` or panel Console | the executed command |
| `say` | `/say` | the message text |
| `rbac_role_create` / `rbac_role_delete` | role lifecycle | role name |
| `rbac_role_link` | Discord role mapping | `botRole -> discordRole` |
| `rbac_permission_grant` / `rbac_permission_revoke` | permission change | `role/permission` |
| `rbac_user_grant` / `rbac_user_revoke` | user assignment | `discordUserId/role` |

---

## Database

SQLite database at `DATABASE_PATH` (default `data/bot.db`), WAL mode, foreign keys enabled. Schema versioned via `schema_version` (currently v1) for future migrations.

| Table | Purpose |
|-------|---------|
| `schema_version` | migration version counter |
| `role` | bot roles, unique per `(name, guild_id)` |
| `role_permission` | permissions per role (cascade-delete with role) |
| `user_role` | direct user→role assignments (unique per user+guild+role) |
| `discord_role_link` | bot-role → Discod role mapping (unique per bot-role+guild) |
| `server_process` | one row per guild: pid, status (`stopped/starting/online/stopping`), started_at |
| `panel_message` | one row per guild: channel_id + message_id of the panel |
| `audit_log` | all audited actions |

Statuses are adjusted automatically: a dead PID → `stopped` (both in `get_status` and on startup).

---

## Logging

- Console + file `logs/bot.log`.
- `RotatingFileHandler`: max 5 MB per file, 3 backups.
- `INFO` level (set in `bot/main.py::_setup_logging`).

The log contains connection info, server lifecycle events, PID changes, orphaned-process cleanup, RCON failures and command errors — useful for diagnosing issues described below.

---

## How it works under the hood

- **`bot/main.py`** — `MinecraftBot(commands.Bot)`; `setup_hook` initializes the DB, registers the persistent view, guarantees `Owner` roles for all guilds, cleans orphaned PIDs, loads the 4 cogs and syncs commands.
- **`bot/core/server_manager.py`** — process lifecycle. `start_server` spawns `MC_START_SCRIPT` as a subprocess in `MC_SERVER_DIR`, stores its PID, marks `online`. `stop_server` calls RCON `stop`, then polls the PID up to 30 s. Status combines psutil stats (RAM, CPU, uptime) with RCON (`list` → players, `version` → version).
- **`bot/core/rcon_client.py`** — thin async wrapper over `aio-mc-rcon` (`Client.connect()` → `send_cmd()` → `close()` per call).
- **`bot/core/permissions.py`** — RBAC engine: `has_permission()` (direct assignments + Discord-role links), `require_permission()` (app-command check that replies *"Insufficient permissions."*), `setup_owner()`.
- **`bot/core/audit.py`** — single insert helper for `audit_log`.
- **`bot/cogs/`** — four cogs: `ServerControl`, `Console`, `AdminRBAC` (nested `Group`s), `Panel` (slash command + `tasks.loop` auto-refresh).
- **`bot/ui/`** — `PanelView` (persistent `discord.ui.View`, `timeout=None`), `ConsoleModal` (RCON input modal), `panel_embed` (embed builder).
- **Persistent views:** on startup `bot.add_view(PanelView(bot))` re-attaches handlers to the stored button `custom_id`s so old panel messages keep functioning after a restart.
- **Nested commands:** `/rbac` is an `app_commands.Group`; `role`, `permission`, `user` are subgroups (`Group` with `parent=rbac`) — matches Discord's "one extra nesting level" limit.
- **Cooldowns:** `@app_commands.checks.cooldown(1, 10, key=…guild_id)` — per-guild bucket on lifecycle commands; a friendly *"wait Ns"* reply is sent via the global tree error handler.

---

## Troubleshooting

**Commands don't show up in Discord**
- Make sure the bot was invited with the `applications.commands` scope.
- Global commands can take up to **1 hour** to appear. Set `TEST_GUILD_ID` during development for instant (guild-only) sync.
- Check `logs/bot.log` — the startup sync step raises if Discord rejects anything.

**`/status` shows 🔴 Offline while the server is clearly running**
- Use psutil checking the stored PID. If the server was started manually (not via the bot), it is simply not tracked. Start it with `/start` (or restart the bot after ensuring it is running via the bot) and it will be tracked.

**Player list / version show "unknown", and `/cmd` fails with `RCON error`**
- RCON is disabled or misconfigured. Verify `server.properties` (`enable-rcon=true`, port, password), restart the server, and double-check `RCON_PORT` / `RCON_PASSWORD` in `.env` match.

**`/stop` waits and then still says "Server stopped" but the process is alive**
- RCON was unreachable, so `stop` never reached the server; the bot waits 30 s and then marks the PID stopped anyway. Fix RCON, or stop the process manually once. (Known limitation — the bot never force-kills.)

**`/start` says "Failed to start server process."**
- Usually a bad `MC_START_SCRIPT` or a wrong `MC_SERVER_DIR`. The exception is logged. `ls`/`cd` into the directory and run the command by hand to confirm it works before wiring it into the bot.

**Permissions not granted / still "Insufficient permissions."**
- Double-check the role exists (`/rbac role list`), has the permission (`/rbac permission grant`), and the user is assigned (`/rbac user roles @user`). Remember `Owner` role deletion/revocation is blocked — create separate roles.
- Discord-role links require the user to actually have that Discord role **and** the user to be in the guild member cache. If you just changed roles, give Discord a moment.

**The panel stops updating / button says "This interaction failed"**
- The panel message was deleted → the bot cleans it up and the next `/panel` call posts a fresh one.
- Panel updates and edits can race with Discord rate limits under heavy use; the refresh loop logs failures and simply tries again next cycle.

---

## FAQ

**Does the bot manage Discord roles?**
No. It manages its own roles stored in SQLite. `discord_role_link` only *reads* a user's Discord roles to grant bot-role permissions automatically — it never creates/modifies/removes Discord roles.

**Can multiple guilds use the bot independently?**
Yes. Roles, permissions, user assignments, the server-process record, the panel message and the audit log are all keyed per guild. Each guild effectively controls the same local server (the bot is meant for one server on localhost).

**Can a user start the server via both buttons and `/start`?**
Yes — identical permission (`SERVER_START`), identical function.

**Do I need RCON for everything?**
No. `/start` and the status indicator work without RCON. RCON is needed for the player list/version fields, `/cmd`, `/say`, the Console modal, and graceful `/stop`.

**How do I update commands after a code change?**
Just restart the bot — `setup_hook` re-syncs the tree each run.
