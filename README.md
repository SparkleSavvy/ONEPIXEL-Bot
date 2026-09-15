# ONEPIXEL-Bot

A **Telegram bot** for managing a **local** Minecraft Java server: start / stop / restart, a live status panel, an RCON console bridge, and an invite-only access system with one-time codes, bans, and revocable tokens.

- **Language / stack:** Python 3.12+, python-telegram-bot 21.x, aio-mc-rcon, aiosqlite, psutil, pydantic-settings
- **Persistence:** SQLite (auto-created on first run)
- **Platform:** Windows / Linux / macOS (anywhere Python + a running Minecraft server live)

---

## Table of contents

- [Overview](#overview)
- [Features](#features)
- [Requirements](#requirements)
- [Quick start](#quick-start)
  1. [Create a Telegram bot](#1-create-a-telegram-bot)
  2. [Enable RCON on the Minecraft server](#2-enable-rcon-on-the-minecraft-server)
  3. [Install dependencies](#3-install-dependencies)
  4. [Configure the bot](#4-configure-the-bot)
  5. [Run the bot](#5-run-the-bot)
- [Configuration reference](#configuration-reference)
- [Command reference](#command-reference)
- [The control panel](#the-control-panel)
- [Access, codes and bans](#access-codes-and-bans)
- [Audit log](#audit-log)
- [Database](#database)
- [Logging](#logging)
- [How it works under the hood](#how-it-works-under-the-hood)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## Overview

ONEPIXEL-Bot lets you control the Minecraft server from Telegram without giving anyone shell access. Access is **opt-in**: nobody can use the bot by default. The owner issues one-time access codes, users redeem them once, and every Telegram account is authenticated against the SQLite database. Five wrong codes auto-ban the account; only the owner can unban.

The bot, the database, and the Minecraft server all run on the same machine (`127.0.0.1` by default).

---

## Features

| Area | Capability |
|------|-----------|
| Server lifecycle | `▶ Пуск`, `⏹ Стоп`, `🔄 Перезапуск` via panel buttons — each button requires its own right |
| Status monitoring | players (count + names + max), uptime, RAM, CPU, version via psutil + RCON |
| Control panel | single text card per chat, **edited in place** (no message spam), with buttons that re-check access and rights at click time |
| RCON console | `/cmd` — run commands (right + optional command whitelist); `/say` — speak in the in-game chat |
| Per-user rights | `/perm` — grant/revoke individual rights (`panel`, `start`, `stop`, `restart`, `cmd`, `say`, `cmd:<pattern>`); nobody has rights by default |
| Token management | `/tokens` with one-button revoke — no need to copy codes into `/revoketoken` |
| Owner notifications | online/offline transitions are reported to the owner's chat |
| Auto-stop | background check: stop the server after N minutes with no players (`/autostop`) |
| Access system | one-time access codes (`/grant`), redeem (`/redeem`), revocable tokens with notes |
| Anti-abuse | 5 wrong codes → automatic ban of the Telegram account; owner can never be auto-banned |
| Auditing | every action (start, stop, `/cmd`, grant, ban, …) is recorded in `audit_log` |
| Robustness | server PID tracked in the DB; orphaned (crashed) processes detected and cleaned; PID lockfile prevents two bot instances |

---

## Requirements

- **Python 3.12+**
- A **Minecraft Java server** (Vanilla, Paper, Spigot, Fabric/Quilt with a RCON mod, …) running on the same machine as the bot
- A **Telegram bot token** (from @BotFather) and your **own Telegram user id** (owner)

---

## Quick start

### 1. Create a Telegram bot

1. Open Telegram, message **@BotFather**, and send `/newbot`.
2. Choose a name and username; BotFather replies with an HTTP API token like `1234567890:AA...`.
3. Get your **owner Telegram id**: message **@userinfobot** (or use `@myidbot`) — it replies with your numeric id.

### 2. Enable RCON on the Minecraft server

RCON is required for: player listing, `/cmd`, `/say`, and graceful stop. Edit `server.properties` in the server directory:

```properties
enable-rcon=true
rcon.port=25575
rcon.password=<a strong, unique password>
```

Restart the Minecraft server afterwards so the settings take effect.

> If RCON is disabled, start/stop/restart still work (process-based), but the player list, version and `/cmd`/`/say` will not.

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

Installs `python-telegram-bot` (with job queue), `aio-mc-rcon`, `pydantic-settings`, `aiosqlite`, `psutil` and `python-dotenv`.

### 4. Configure the bot

```bash
cp .env.example .env
```

Edit `.env` — set **`TELEGRAM_BOT_TOKEN`** and **`ADMIN_TELEGRAM_ID`** (your own id). Set **`RCON_PASSWORD`**, and adjust `MC_START_SCRIPT` / `MC_SERVER_DIR` if your setup differs from the defaults. See the [Configuration reference](#configuration-reference).

### 5. Run the bot

```bash
python -m tgpanel.main
```

On startup the bot:

1. acquires a PID lockfile (`data/tgpanel.pid`) — a second instance refuses to start,
2. creates the folder and SQLite database (`data/bot.db` by default),
3. scans the DB and **marks any dead/crashed server process as stopped**,
4. starts the Telegram polling loop and the background monitor job.

---

## Configuration reference

All settings are read from **environment variables** or a `.env` file in the project root (`bot/config.py`).

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | — | ✅ | Bot token from @BotFather. |
| `ADMIN_TELEGRAM_ID` | — | ✅ | Your numeric Telegram id — the owner. |
| `SCOPE_ID` | `1` | ❌ | Fixed scope id used in place of a Discord/guild id in `server_process`, `guild_settings` and `audit_log`. |
| `RCON_HOST` | `127.0.0.1` | ❌ | Host of the RCON endpoint. |
| `RCON_PORT` | `25575` | ❌ | Port of the RCON endpoint. |
| `RCON_PASSWORD` | *(empty)* | ✅ **for RCON** | Must match `rcon.password` in `server.properties`. |
| `MC_START_SCRIPT` | `java -Xmx2G -Xms1G -jar server.jar nogui` | ❌ | Shell command used to launch the server. Must be runnable from `MC_SERVER_DIR`. |
| `MC_SERVER_DIR` | `./server` | ❌ | Working directory for the server process (where `server.jar`, `world`, `logs/` live). |
| `DATABASE_PATH` | `data/bot.db` | ❌ | SQLite database file. Parent directory is created automatically. |
| `PANEL_REFRESH_INTERVAL` | `30` | ❌ | Seconds between monitor ticks (status change notifications, auto-stop check). |
| `MAX_LOGIN_ATTEMPTS` | `5` | ❌ | Wrong `/redeem` attempts before the account is auto-banned. |

Notes:

- pydantic parses ints from `.env`, so values like `25565` and `30` are already numbers.
- `MC_START_SCRIPT` runs through the system shell with stdout/stderr discarded; the bot tracks the PID, not the console output. Use RCON (or the server's own `logs/latest.log`) to see server output.
- The owner id is never auto-banned and cannot be banned manually (`/ban`).

---

## Command reference

Commands with the 🔒 marker are **owner-only**. All other commands and the panel buttons require the specific **right** for the action (granted via `/perm`); redeeming a code by itself grants nothing.

| Category | Command | Access |
|----------|---------|--------|
| General | `/start` | anyone |
| Access | `/redeem <code>` | anyone |
| Panel | `/panel` | right `panel` |
| Console | `/cmd <command>` | right `cmd` or matching `cmd:<pattern>` |
| Console | `/say <message>` | right `say` |
| Rights | `/perm list|grant|revoke` | 🔒 owner |
| Auto-stop | `/autostop [minutes]` | 🔒 owner |
| Access | `/grant [note]` | 🔒 owner |
| Access | `/tokens` | 🔒 owner |
| Access | `/revoketoken <code>` | 🔒 owner |
| Bans | `/ban <telegram_id>` | 🔒 owner |
| Bans | `/unban <telegram_id>` | 🔒 owner |
| Bans | `/banned` | 🔒 owner |
| Users | `/users` | 🔒 owner |

### General

- **`/start`** — welcome message. Shows a different reply depending on whether you are the owner, have access, are banned, or are unknown.
- **`/panel`** — posts the control panel into the current chat. If a panel already exists in that chat, it is **updated in place** (never duplicated).

### Access

- **`/redeem <code>`** — redeems a one-time access code issued by the owner. A wrong or already-used code counts as one failed attempt; after `MAX_LOGIN_ATTEMPTS` failed attempts the account is **auto-banned**. Redeem alone does **not** grant any rights — the owner must issue them via `/perm`.
- **`/grant [note]`** — creates a one-time **16-character** code, e.g. `/grant myserver-token`. The note appears in `/tokens`.
- **`/tokens`** — lists all active codes (code, note, state) with an **«Отозвать»** button under each. Clicking it revokes the token in one tap — same effect as `/revoketoken`.
- **`/revoketoken <code>`** — deletes the code; if it had already been redeemed, the redeemer's access is revoked too.

### Rights (`/perm`)

Rights live in the `user_permission` table and are granted **per Telegram account**. The owner implicitly has all rights and can never be denied. By default a redeeming user has **zero rights** — every command asks for its own one.

| Right | Grants |
|-------|--------|
| `panel` | `/panel` and the `🔄 Обновить` button |
| `start` | `▶ Пуск` button |
| `stop` | `⏹ Стоп` button |
| `restart` | `🔄 Перезапуск` button |
| `cmd` | all RCON commands via `/cmd` |
| `say` | `/say` |
| `cmd:<pattern>` | only matching `/cmd` commands (glob, case-insensitive), e.g. `cmd:list`, `cmd:whitelist add`, `cmd:whitelist*` |

Usage (owner-only):

- `/perm list` — all users and their rights
- `/perm list <id>` — one user
- `/perm grant <id> <right>` — grant (repeatable; `cmd:<pattern>` is just another right value)
- `/perm revoke <id> <right>` — revoke

If a user has the `cmd` right, every command is allowed; otherwise only the `cmd:<pattern>` entries that match (via `fnmatch` on the lower-cased command). `/perm revoke` on a `cmd:` pattern removes just that pattern.

### Bans

- **`/ban <telegram_id>`** — bans the account (owner can't be banned). The ban kicks in immediately and blocks access even if the account had redeemed a code.
- **`/unban <telegram_id>`** — lifts the ban and resets the failed-attempt counter. A previously granted access is restored (the `granted_at` timestamp is kept).
- **`/banned`** — lists banned accounts as `id (username)`, with the **current** username (it is refreshed on every contact with the bot), plus the ban time.

### Console

- **`/cmd <command>`** — executes an RCON command, e.g. `/cmd list`, `/cmd op Steve`, `/cmd time set day`. Requires the `cmd` right or a matching `cmd:<pattern>`. Response is returned in a code block (truncated to 4000 chars).
- **`/say <message>`** — broadcasts a message in the in-game chat (executes `say <message>` via RCON).

### Auto-stop

- **`/autostop`** — shows the current idle setting.
- **`/autostop 30`** — stops the server automatically after 30 minutes with zero players.
- **`/autostop 0`** — disables auto-stop.

---

## The control panel

The panel is a plain text message that is **edited in place** — the bot never spams new messages. The current panel message id is stored per chat in the `panel_state` table, so repeated `/panel` calls and button presses keep updating **one** message.

```
🟢 Сервер Minecraft

Статус: Онлайн
Игроки: 2/20
Сейчас: Steve, Alex
RAM: 1483 МБ
CPU: 42.3%
Аптайм: 2ч 14м 5с
Версия: 1.21.*
```

Status indicator: 🟢 online · 🔴 offline · 🟡 starting · 🟠 stopping.

Buttons (each re-checks access at click time):

| Button | Required right | Behavior |
|--------|----------------|----------|
| ▶ Пуск | `start` | Starts the server, panel updates |
| ⏹ Стоп | `stop` | RCON `stop`, waits for exit (up to 60 s, then force-kills), panel updates |
| 🔄 Перезапуск | `restart` | Stop + 3 s + start, panel updates |
| 🔄 Обновить | `panel` | Re-reads status and edits the panel |

Behavior notes:

- If the panel message was deleted, the next render sends a new one and stores the new id. While the status is `Остановка...`, only the `🔄 Обновить` button is shown (nothing to start yet).
- Clicking a button shows the action via the Telegram callback toast; the panel itself is updated afterwards.
- Unauthorized, banned, or right-less users get an alert and nothing changes.

---

## Access, codes and bans

Access control is **entirely DB-based** — no first-use registration, no roles. "Who has access" = whoever is in the `panel_user` table with a `granted_at` timestamp (and is not banned). "Who can do what" = the `user_permission` table (the owner has everything implicitly). By default nobody has access and nobody has rights.

Flow:

1. The owner runs `/grant` (optionally with a note like a username or purpose) and gets a one-time code.
2. The user messages the bot with `/redeem <code>`. The first person to redeem the code gets access; the code can never be reused.
3. The owner grants the needed rights: `/perm grant <id> panel` and, say, `/perm grant <id> start`. Optional per-command whitelist: `/perm grant <id> cmd:list`.
4. A wrong or already-used code increments the account's `failed_attempts`. At `MAX_LOGIN_ATTEMPTS` the account is written to `is_banned = 1` automatically.
5. The owner manually unban with `/unban` — there is no auto-unban.

Details:

- The `username` column is refreshed **every time a user interacts** with the bot, so ban reports always show the current handle.
- Owner (`ADMIN_TELEGRAM_ID`) is excluded from auto-ban, manual `/ban`, and from every right check.
- `revoke` (`/revoketoken`, or the button in `/tokens`) clears `granted_at`, while a ban keeps it — so an unban restores full access without a new code.

---

## Audit log

Every meaningful action is written to the `audit_log` table (with the fixed `SCOPE_ID`). There is no command to read it — query the database directly:

```bash
sqlite3 data/bot.db "SELECT datetime(timestamp) AS when, guild_id, user_id, action, detail FROM audit_log ORDER BY id DESC LIMIT 20;"
```

| `action` | Meaning | `detail` |
|----------|---------|----------|
| `panel_start` / `panel_stop` / `panel_restart` | server lifecycle via panel buttons | result text |
| `console` | `/cmd` | the executed command |
| `say` | `/say` | the message text |
| `grant` | code issued | the note |
| `redeem` | code redeemed | the code |
| `revoketoken` | code deleted | the code |
| `perm_grant` / `perm_revoke` | right granted/revoked | `target_id:right` |
| `ban` / `unban` | manual ban/unban (or auto-ban) | target id; `auto: max login attempts` on auto-ban |
| `autostop` | auto-stop setting change | minutes |

---

## Database

SQLite database at `DATABASE_PATH` (default `data/bot.db`), WAL mode, foreign keys enabled. Schema versioned via `schema_version` for future migrations.

| Table | Purpose |
|-------|---------|
| `schema_version` | migration version counter |
| `server_process` | row for `SCOPE_ID`: pid, status (`stopped/starting/online/stopping`), started_at |
| `guild_settings` | row for `SCOPE_ID`: notify channel (unused), `auto_stop_minutes` |
| `panel_user` | one row per Telegram user: username, `is_banned`, `failed_attempts`, `banned_at`, `granted_at` |
| `access_code` | one-time codes: code, note, `redeemed_by`, `redeemed_at` |
| `panel_state` | one row per chat: the panel message id |
| `user_permission` | one row per (user, right): `telegram_user_id`, `permission` (`panel`/`start`/`stop`/`restart`/`cmd`/`say`/`cmd:<pattern>`) |
| `audit_log` | all audited actions |

Statuses are adjusted automatically: a dead PID → `stopped` (both in `get_status` and on startup); if `MC_SERVER_DIR` still hosts a live process with a dead tracked PID, the process is adopted (its PID becomes the tracked one).

---

## Logging

- Console + file `logs/bot.log`.
- `RotatingFileHandler`: max 5 MB per file, 3 backups.
- `INFO` level (set in `tgpanel/main.py`).

The log contains startup info, server lifecycle events, PID changes, orphaned-process cleanup, RCON failures and command errors.

---

## How it works under the hood

- **`tgpanel/main.py`** — entry point. PID lockfile (a second instance exits), `post_init` (create DB, start monitor job), polling loop.
- **`tgpanel/handlers.py`** — all commands and the panel-button / token-revoke callback handlers. Every command refreshes the caller's username, re-checks the ban status and then the required right against `user_permission` (owner bypasses everything).
- **`tgpanel/panel.py`** — status card rendering, button layout, and the panel lifecycle: edit the stored message or send a new one (`panel_state`). `Message is not modified` is ignored; a deleted panel is recreated.
- **`tgpanel/dashboard.py`** — periodic monitor job: detects online/offline transitions (notifies the owner) and enforces auto-stop.
- **`bot/core/server_manager.py`** — process lifecycle. `start_server` spawns `MC_START_SCRIPT` as a subprocess in `MC_SERVER_DIR`, stores its PID. `stop_server` calls RCON `stop`, polls the PID up to 60 s, then force-kills if needed. Status combines psutil stats (RAM, CPU, uptime) with RCON (`list` → players, `version` → version). Orphaned processes are adopted or cleaned.
- **`bot/core/rcon_client.py`** — thin async wrapper over `aio-mc-rcon` (`connect()` → `send_cmd()` → `close()` per call).
- **`bot/core/audit.py`** — single insert helper for `audit_log`.
- **`bot/db/`** — `session.py` (connection + schema init), `access_code_repo.py`, `panel_user_repo.py`, `panel_state_repo.py`, `permission_repo.py` (rights + `cmd:<pattern>` matching), `settings_repo.py`.

---

## Troubleshooting

**The bot doesn't reply at all**
- Check `logs/bot.log`. Common causes: wrong `TELEGRAM_BOT_TOKEN`, or the bot was restarted while another instance is still alive (PID lockfile) — kill the old process and remove `data/tgpanel.pid`.

**`/redeem <code>` always says wrong, but the code is right**
- Codes are compared exactly; make sure there are no stray spaces/spoiler formatting when you paste the code into Telegram.

**The panel shows 🔴 Офлайн while the server is clearly running**
- The server was started manually (not via the bot) and hasn't been adopted yet. Use the panel's `▶ Пуск`/refresh (the bot then tracks the process), or restart the bot while the server is running.

**Player list / version show "unknown", and `/cmd` fails**
- RCON is disabled or misconfigured. Verify `server.properties` (`enable-rcon=true`, port, password), restart the server, double-check `RCON_PORT` / `RCON_PASSWORD` in `.env`.

**`⏹ Стоп` waits and then the process is still alive**
- RCON was unreachable, so `stop` never reached the server; the bot waits 60 s and then force-kills the PID. Fix RCON if you want graceful stops.

**`▶ Пуск` says it failed to start**
- Usually a bad `MC_START_SCRIPT` or a wrong `MC_SERVER_DIR`. The exception is logged; run the command by hand in that directory to confirm it works.

**A user got banned for nothing / the ban table is wrong**
- Bans are automatic only from wrong `/redeem` attempts. `/banned` shows the current username (refreshed on contact), `/unban <id>` clears the ban.

**A user has access (redeemed a code) but a button/command doesn't work**
- Rights are granted separately. The owner runs `/perm grant <id> <right>` (e.g. `start`, `panel`, `cmd:list`). `/perm list <id>` shows what the user has.

---

## FAQ

**Can someone use the bot before receiving a code?**
No. By default nobody is in `panel_user`, so every command except `/start` and `/redeem` replies "Нет доступа". After redeeming a code the user has access but still **zero rights** until the owner grants them with `/perm`.

**Does the owner have to redeem a code?**
No. The owner id has implicit full access, every right, and can never be banned.

**Can the same code be used by two people?**
No — a code is single-use. The next attempt treats it as "already used" (and counts as a failed attempt).

**Is the panel auto-refreshing?**
No — use the `🔄 Обновить` button. The background monitor job only checks status transitions and auto-stop (and notifies the owner), it does not touch the panel message. Auto-refresh can be added later.

**Do I need RCON for everything?**
No. Start/stop/restart and the status indicator work without RCON. RCON is needed for the player list/version fields, `/cmd`, `/say`, and graceful stops.
