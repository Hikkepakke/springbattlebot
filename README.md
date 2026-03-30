# Spring Battle Bot

A Telegram bot for the annual **SIK vs KIK Spring Battle** — a fitness competition between two student guilds. Participants log their exercise (running/walking, biking, steps) by sending photos to the bot, and the bot tracks distances and leaderboards for each guild.

## Table of Contents

- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Quick Start (Development)](#quick-start-development)
- [Environment Variables](#environment-variables)
- [Common Commands](#common-commands)
- [Bot Commands (Telegram)](#bot-commands-telegram)
- [Database](#database)
- [Production Deployment](#production-deployment)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)

## How It Works

1. A participant opens a private chat with the bot and sends `/start`.
2. They choose their guild: **SIK** or **KIK**.
3. To log exercise, they send a **photo** (e.g. a Strava screenshot or step counter).
4. The bot asks which sport and how much distance. Alternatively the photo caption can contain this info (e.g. `running, 5.5`).
5. Steps are automatically converted to kilometers (× 0.0007).
6. Anyone can check guild standings with `/status` and personal stats with `/personal`.
7. Admins can pull daily reports and full stats.

## Prerequisites

- **Docker** and **Docker Compose** — the entire dev environment (bot + database) runs in containers.
  - [Install Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/Mac) or Docker Engine (Linux).
- **A Telegram Bot Token** — create one via [@BotFather](https://t.me/BotFather) on Telegram.
- **Node.js 20+** and **npm** — only needed if you want to run commands outside Docker (e.g. `npm install` on the host).

## Quick Start (Development)

### 1. Clone the repository

```bash
git clone <repo-url>
cd springbattlebot
```

### 2. Create the `.env` file

Copy the example and fill in your values:

```bash
cp .env.example .env
```

Edit `.env` so it looks like this:

```env
POSTGRES_USER=username
POSTGRES_PASSWORD=password
POSTGRES_DB=database
POSTGRES_URL=postgresql://username:password@springbattlebot-db:5432/database

BOT_TOKEN=<your-telegram-bot-token>
ADMINS={"list":[]}
ACCEPTING_SUBMISSIONS=true
```

Replace `<your-telegram-bot-token>` with the token from @BotFather.

To grant yourself admin access (for `/daily` and `/all` commands), put your numeric Telegram user ID in the `ADMINS` list. You can find your ID by messaging [@userinfobot](https://t.me/userinfobot) on Telegram.

```env
ADMINS={"list":[123456789]}
```

### 3. Start everything

```bash
docker compose up --build
```

This starts two containers:
- **springbattlebot-db** — PostgreSQL 14 database
- **springbattlebot-bot** — the Node.js bot (with hot-reload via nodemon)

The bot waits for the database to be healthy before starting.

### 4. Run database migrations

Open a **second terminal** (while the containers are running) and run:

```bash
npm run db:migrate
```

This creates the required tables (`users`, `logs`, `log_events`) and enums in the database. **You must do this before the bot can handle any messages.**

### 5. (Optional) Seed test data

```bash
npm run db:seed
```

This inserts a handful of fake users and log entries so you can test stats commands without real participants.

### 6. Test the bot

Open Telegram, find your bot, and send `/start`. You should see the welcome message and guild selection buttons.

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `POSTGRES_USER` | Yes | PostgreSQL username (used by both the DB container and the app) |
| `POSTGRES_PASSWORD` | Yes | PostgreSQL password |
| `POSTGRES_DB` | Yes | PostgreSQL database name |
| `POSTGRES_URL` | Yes | Full connection string. Use host `springbattlebot-db` when running in Docker Compose, or `127.0.0.1` when running the bot on the host directly. |
| `BOT_TOKEN` | Yes | Telegram bot token from @BotFather |
| `ADMINS` | Yes | JSON object with a `list` array of numeric Telegram user IDs, e.g. `{"list":[123456789]}` |
| `ACCEPTING_SUBMISSIONS` | No | Set to `true` to allow photo submissions. If unset or anything else, the bot rejects new logs. |
| `CRON_GROUP_ID` | No | Telegram chat ID for automated daily summary messages. If unset, no cron job runs. |
| `NODE_ENV` | No | Set to `production` for webhook mode; otherwise uses long polling. |
| `DOMAIN` | No | Public domain for the Telegram webhook (production only). |

## Common Commands

All commands are run from the **repository root**.

| Command | What it does |
|---|---|
| `npm run dev` | Starts bot + database via `docker compose up` |
| `npm run db:migrate` | Runs database migrations (via `docker exec` into the bot container) |
| `npm run db:seed` | Seeds the database with test data (via `docker exec` into the bot container) |
| `npm run db:generate` | Generates new Drizzle migration files after schema changes (runs locally in `bot/`) |
| `npm run install` | Installs npm dependencies in the `bot/` folder |

### Rebuilding after dependency changes

If you add or remove packages in `bot/package.json`, rebuild the bot image:

```bash
docker compose build bot
docker compose up
```

### Full reset (wipe database)

```bash
docker compose down -v
docker compose up --build
npm run db:migrate
```

The `-v` flag removes the database volume, giving you a clean slate.

## Bot Commands (Telegram)

All commands work in **private chat** only (direct message to the bot).

### Everyone

| Command | Description |
|---|---|
| `/start` | Register and choose your guild (SIK or KIK) |
| `/status` | See current battle standings — totals per sport, who's leading |
| `/personal` | See your own total km per sport |
| `/mydaily` | See your own km logged today |
| `/reset_guild` | Switch your guild (also updates all your past logs) |
| `/update_name` | Sync your display name from your Telegram profile |
| `/cancel` | Cancel an in-progress logging session |

### Logging exercise

1. Send a **photo** (screenshot of steps, Strava, etc.) to the bot in private chat.
2. **Option A — caption shortcut:** Add a caption like `running, 5.5` or `s, 8000` (for steps). Accepted sport keywords: `running`, `walking`, `r`, `w`, `biking`, `cycling`, `b`, `c`, `steps`, `activity`, `s`, `a`.
3. **Option B — guided flow:** If no caption or it can't be parsed, the bot shows sport buttons, then asks for the distance/steps as a text message.

### Admin only

These require your Telegram user ID in the `ADMINS` list.

| Command | Description |
|---|---|
| `/daily` | Pick today or yesterday, get a detailed daily report with top 5 per guild and all-time top 3 |
| `/all` | Full stats: participant counts, totals per sport per guild, top 5 leaderboards |

## Database

The bot uses **PostgreSQL 14** with **Drizzle ORM**. The schema has three tables:

- **`users`** — Telegram user ID, display name, guild (SIK/KIK)
- **`logs`** — Each exercise entry: user, guild (denormalized), sport, distance in km, timestamp
- **`log_events`** — Tracks in-progress logging sessions (one per user, cleared when done)

### Connecting to the database directly

While the containers are running:

```bash
docker exec -it springbattlebot-db psql -U username -d database
```

This opens an interactive PostgreSQL shell where you can run SQL queries.

### Schema changes

1. Edit `bot/src/db/schema.ts`
2. Generate a migration: `npm run db:generate`
3. Apply it: `npm run db:migrate`

Migration SQL files live in `bot/drizzle/`.

## Production Deployment

Production uses a separate Compose file and a multi-stage Dockerfile:

```bash
docker compose -f docker-compose.prod.yml up --build -d
```

Key differences from development:
- Uses `bot/Dockerfile.prod` (multi-stage build, compiled JS, pruned dependencies)
- Runs migrations automatically on startup via `prod-entrypoint.sh`
- Uses Telegram **webhook** mode (requires `NODE_ENV=production` and `DOMAIN`)
- No bind-mounted source code (the image is self-contained)
- Production `.env` goes in `bot/.env` (see `docker-compose.prod.yml`)
- Daily cron summary runs at **01:00 Helsinki time** (instead of every minute)

## Troubleshooting

### `relation "users" does not exist`

Migrations haven't been applied yet. Run:

```bash
npm run db:migrate
```

### `password authentication failed for user "username"`

The credentials in `.env` don't match what the database was initialized with. If you changed `POSTGRES_USER` or `POSTGRES_PASSWORD` after the database volume was first created, you need to wipe the volume:

```bash
docker compose down -v
docker compose up --build
npm run db:migrate
```

### `the database system is starting up`

The bot tried to query before Postgres was ready. This should be fixed by the healthcheck in `docker-compose.yml`. If it still happens, restart:

```bash
docker compose restart bot
```

### `sh: nodemon: not found`

The container's `node_modules` is missing or incomplete. Rebuild:

```bash
docker compose down -v
docker compose up --build
```

### `missing some environment variables...`

The bot requires both `BOT_TOKEN` and `ADMINS` to be set in `.env`. Make sure they exist and are not empty.

### Bot doesn't respond to messages

- Check that `ACCEPTING_SUBMISSIONS=true` is set (for photo submissions).
- Make sure you're messaging the bot in a **private chat**, not a group.
- Check container logs: `docker compose logs bot`

## Project Structure

```
springbattlebot/
├── .env.example              # Template for environment variables
├── .env                      # Your local environment variables (not committed)
├── docker-compose.yml        # Development: bot + database
├── docker-compose.prod.yml   # Production: bot + database
├── package.json              # Root scripts (docker compose wrappers)
└── bot/
    ├── Dockerfile            # Dev image (with nodemon + tsx)
    ├── Dockerfile.prod       # Production multi-stage image
    ├── package.json          # Bot dependencies and scripts
    ├── index.ts              # Main bot logic (commands, handlers, cron)
    ├── tsconfig.json
    ├── drizzle.config.ts     # Drizzle ORM configuration
    ├── prod-entrypoint.sh    # Production entrypoint (runs migrations)
    ├── drizzle/              # Generated SQL migration files
    └── src/
        └── db/
            ├── db.ts         # Database connection
            ├── schema.ts     # Drizzle table/enum definitions
            ├── migrate.ts    # Migration runner
            └── seed.ts       # Test data seeder
```
