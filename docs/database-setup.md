# Database Setup — PostgreSQL

## Default database

The `clean` architecture template is configured for **PostgreSQL**. The TypeORM connection is defined in `src/config/database.config.ts` and reads all credentials from environment variables.

## Required schemas

The template uses two schemas inside a single PostgreSQL database. Both must exist before running migrations or starting the application.

```sql
CREATE SCHEMA IF NOT EXISTS trn;
CREATE SCHEMA IF NOT EXISTS cat;
```

### `trn` — Transactional

Holds tables that change frequently as part of business operations: user accounts, sessions, assignments, audit logs, and any entity whose rows are created, updated, or deleted during normal application use.

**Examples in this template:** `trn.users`, `trn.role_user`

### `cat` — Catalogues

Holds reference data that changes rarely and is shared across the application: roles, permissions, statuses, types, and lookup tables. These rows are typically seeded once and modified only by administrators.

**Examples in this template:** `cat.roles`, `cat.permissions`, `cat.permission_role`

### Why two schemas?

Separating transactional from catalogue data gives you:

- **Clear ownership** — a migration that touches `cat` is implicitly a reference-data change; one that touches `trn` is a business-logic change.
- **Access control** — you can grant a reporting role read-only access to `cat` without exposing `trn`.
- **Backup granularity** — catalogue data is stable and small; transactional data is large and changes constantly. Separate schemas make targeted backups and restores straightforward.

## Step-by-step local setup

```sql
-- 1. Create the database
CREATE DATABASE your_db_name;

-- 2. Connect to it
\c your_db_name

-- 3. Create schemas
CREATE SCHEMA IF NOT EXISTS trn;
CREATE SCHEMA IF NOT EXISTS cat;
```

## Environment variables

Copy `.env.example` to `.env` and fill in your values:

```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=your_password
DB_NAME=your_db_name
```

## Running migrations

```bash
# Generate a migration after entity changes
pnpm migration:generate src/database/migrations/MigrationName

# Run pending migrations
pnpm migration:run

# Revert the last migration
pnpm migration:revert
```
