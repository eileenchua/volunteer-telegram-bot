# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev:local          # Start bot with local PGlite database (reads .env.local)
npm run setup:fresh        # Reset + migrate + seed local DB from scratch
npm run setup:local        # Migrate + seed (no reset)

# Testing
npm test                   # Run all tests (uses in-memory PGlite)
npm run test:local         # Run tests with .env.local loaded
npm run test:watch         # Watch mode
npm run test:coverage      # Coverage report

# Type checking & linting
npm run type-check         # tsc --noEmit only
npm run lint               # tsc --noEmit + eslint --fix

# Database
npm run db:generate        # Generate Drizzle migration from schema changes
npm run db:migrate:local   # Apply migrations to local PGlite DB
npm run db:studio:local    # Open Drizzle Studio GUI
```

Run a single test file:

```bash
npx vitest run tests/database.test.ts
```

## Architecture

This is a Telegram bot for volunteer management, built with:

- **grammY** — Telegram Bot API framework (`src/bot.ts` is the entrypoint)
- **Drizzle ORM** — type-safe database queries (`src/drizzle.ts`, `src/schema.ts`)
- **PGlite** — in-memory/file-backed Postgres for local dev and tests; real Postgres for staging/production

### Database environment switching

`src/drizzle.ts` selects the database at startup based on `NODE_ENV`:

- `development` → PGlite (file at `local-db/volunteer-bot.db`, or in-memory when `PGLITE_STORAGE=memory`)
- `staging` / `production` → `postgres-js` connecting to `STAGING_DATABASE_URL` / `PRODUCTION_DATABASE_URL`

Tests always set `NODE_ENV=development` and `PGLITE_STORAGE=memory` (see `vitest.config.ts`).

### Schema

Five tables in `src/schema.ts`:

- `volunteers` — core entity; tracks `commitments` (current period) and `cumulative_commitments` (all-time); status enum: `probation | active | lead | inactive`
- `events` — community events with format/status enums
- `tasks` — belong to events
- `task_assignments` — many-to-many between tasks and volunteers
- `admins` — Telegram handles with admin role

### Command handlers (`src/commands/`)

| File            | Responsibility                                                                        |
| --------------- | ------------------------------------------------------------------------------------- |
| `volunteers.ts` | `/onboard`, `/commit`, `/uncommit`, `/mystatus`, `/mytasks`, volunteer status reports |
| `admins.ts`     | Admin login, volunteer CRUD, `setStatus`, `setCommitCount`, `resetQuarter`            |
| `events.ts`     | Event creation wizard, edit/remove, list                                              |
| `broadcast.ts`  | Broadcast messages to volunteers or event participants                                |

`src/bot.ts` wires all commands to the grammY bot instance and sets up session middleware.

### Tests

Tests live in `tests/`. `tests/setup.ts` creates all tables using raw SQL before each suite and truncates them before each test — no migration files are run during tests. When adding new columns to schema, update the `CREATE TABLE` statements in `tests/setup.ts` as well.

### Migrations

Drizzle migrations live in `drizzle/`. Generate a new migration after schema changes with `npm run db:generate`, then apply with `npm run db:migrate:local`.