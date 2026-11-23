---
title: Generate Migrations from the Schema with Prisma
impact: HIGH
impactDescription: Keeps the database and the Prisma schema in sync, without drift
tags: database, migrations, prisma, schema
---

## Generate Migrations from the Schema with Prisma

The Prisma form of `db-use-migrations`: read that rule first for the workflow, destructive changes and deploys. Structural migrations come from Prisma Migrate, and nothing changes the schema at runtime. `prisma db push` changes the schema without a migration and is never used, not even locally. Hand-written SQL inside a generated migration is for data (a backfill, the copy step of an expand and contract), for CHECK constraints, which the schema cannot express (see `db-prisma-schema-conventions`), for `NOT VALID` on a constraint over existing rows and the later migration that validates it (see `db-use-migrations`), and for the extension an index needs: `CREATE EXTENSION IF NOT EXISTS pg_trgm;` at the top of the migration that first creates a `gin_trgm_ops` index (see `perf-optimize-database`), since the generated SQL creates the index and not the extension. The drift check does not report it. `pg_trgm` is a trusted extension (PostgreSQL 13 and later), so the migration's role needs only the `CREATE` privilege on the database, not a superuser.

- The workflow's steps are `prisma migrate dev --create-only --name <name>` (writes the migration without applying it), a review of its SQL, `prisma migrate dev` (applies it locally), then `prisma generate`, which Prisma 7 no longer runs after a migration.
- The drift check is `prisma migrate diff --from-config-datasource --to-schema prisma --exit-code`. It compares the migrated local database with the schema: exit code 0 means no drift, 2 means the schema has changes no migration covers.
- Names are lowercase snake_case and say what the migration does (`--name create_order_items`); Prisma prefixes the timestamp.
- A data-only migration is `prisma migrate dev --create-only --name backfill_display_names` with the schema unchanged: Prisma writes an empty `migration.sql` for you to fill. How the file runs decides how it is written (below).
- An applied migration is never edited, not even one that only your local database has run, which `db-use-migrations` allows for other tools: `prisma migrate deploy` applies only the migrations it has not recorded, so the databases where it already ran never get the change, and `migrate dev` sees the edit by its checksum and asks to reset the local database, which an agent may not do without the user (below). Fix forward with a new migration, a local mistake included.
- A renamed field or model is generated as a drop plus an add, never as a rename. Renaming only the Prisma field while keeping `@map("old_column")` needs no migration at all.
- A field unmapped before its column is dropped (see the destructive changes of `db-use-migrations`) is made optional and marked `@ignore`: Prisma Client leaves it out of every query, and the column stays in the schema. Removing the field would generate the drop a release too early.
- Prisma Migrate has no down migrations: roll back with a new migration that reverses the change. A migration that failed halfway in an environment is marked with `prisma migrate resolve --rolled-back <name>` once its effects are undone, a release step rather than part of a feature change.
- `prisma migrate reset` drops every table, so it only runs against a throwaway local database. When it detects an AI agent, Prisma 7 refuses `migrate reset`, `db push --force-reset` and `db push --accept-data-loss` until the user consents through `PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION`, whose value is the user's consenting message. Ask the user; never set it yourself.
- The deploy step runs `prisma migrate deploy`, and on a database another tool built, the baseline checks first (below).
- **Prisma Migrate gets the database password on its command line.** Every `migrate` command starts Prisma's schema engine with the database URL, password included, as an argument (`--datasource`), which other users of the machine, the host of a container and process collectors can read; nothing in the project's code avoids it on 7.10.1. Run Prisma Migrate against a shared database only from its own job or container, never from a shared machine, with a database role separate from the app's.
- `prisma.config.ts` checks the server's certificate (`sslaccept=strict`, see `db-prisma-schema-conventions`): this is the connection that carries the schema owner's credentials.

**How Prisma Migrate runs a migration file.** Measured with `migrate deploy` on 7.10.1 and PostgreSQL 18:

- A file of plain statements runs statement by statement, each committed on its own, with no transaction around the file. When one fails, the statements before it stay applied, the migration is recorded as failed (`P3018`), and every later deploy stops (`P3009`) until `prisma migrate resolve` marks it.
- A file that contains a `DO` block runs as one transaction: a failure in the block, or in a plain statement after it, undid the whole file. A function body in `$$` quotes does not do this.
- So a `COMMIT` inside a `DO` block works only when the block is alone in its file. With any other statement in the file, it fails with `2D000` ("invalid transaction termination"), and the file is undone.

**A batched backfill** (see `db-use-migrations`) is a migration file whose only statement is a `DO` block that walks the table in id order and commits after each batch (1000 rows here, an example). It updates only the rows whose new column is still empty, so it can run again: when a batch fails, the batches before it stay committed and the migration is marked failed. Fix the cause, mark it with `prisma migrate resolve --rolled-back <name>`, and the next deploy runs the file again and fills the rest. Its database-tier spec runs it outside a transaction (see `test-database-tier`).

```sql
-- prisma/migrations/20261121120000_backfill_user_first_names/migration.sql
-- Alone in its file: with any other statement here, the COMMIT fails with 2D000
DO $$
DECLARE
  last_id uuid := '00000000-0000-0000-0000-000000000000';
  batch_end uuid;
BEGIN
  LOOP
    SELECT id INTO batch_end FROM (SELECT id FROM users WHERE id > last_id ORDER BY id LIMIT 1000) AS batch ORDER BY id DESC LIMIT 1;
    EXIT WHEN batch_end IS NULL;
    UPDATE users SET first_name = split_part(name, ' ', 1) WHERE id > last_id AND id <= batch_end AND first_name IS NULL;
    last_id := batch_end;
    COMMIT;
  END LOOP;
END
$$;
```

**Moving an existing database onto Prisma Migrate** (a project leaving TypeORM or another tool):

1. Write the schema so it matches the database, names and types included. `prisma db pull` can draft it.
2. The baseline migration is generated from the schema: `prisma migrate diff --from-empty --to-schema prisma --script > prisma/migrations/20261121000000_baseline/migration.sql`. Then add by hand what the schema cannot express: CHECK and exclusion constraints, extensions (`CREATE EXTENSION IF NOT EXISTS`), and the rows the old migrations inserted (a default category). A new database gets everything from it.
3. On a database the old tool built, the deploy step checks the live schema before it records the baseline, and stops on any difference:
   - `prisma migrate diff --from-config-datasource --to-schema prisma --script`, whose only statement may be the `DROP TABLE` of the old tool's history table (TypeORM's `migrations`). An index or a column added by hand shows here, and so does a missing one;
   - the CHECK and exclusion constraints by name, `SELECT conname FROM pg_constraint WHERE connamespace = current_schema()::regnamespace AND contype IN ('c', 'x')`, against the baseline's list: the diff cannot see them, and a database missing one passes it;
   - the old tool's history, on every deploy while its table exists: exactly the migrations the baseline replaces, none missing and none unknown, so a migration the old release ran during a rollback is noticed.
4. Then it records the baseline as applied, once, with `prisma migrate resolve --applied 20261121000000_baseline`, and runs `prisma migrate deploy`. A bare `migrate deploy` on such a database stops with `P3005` and writes nothing.
5. The old history table stays while a rollback to the old release is possible: without it, that release would run all its migrations again. Meanwhile the drift check exits 2 on it, and `prisma migrate dev` reports drift and asks to reset the database, so developers drop the table on their local databases, and a drift check on a shared database allows that one statement. A later release drops it with a hand-written `DROP TABLE IF EXISTS "migrations";` migration: the generator never writes it, since the shadow database it builds from the migrations never has the table.
6. A committed snapshot of the old schema (`pg_dump --schema-only` of a database the old tool's last release built) proves the baseline: a check builds a database from empty with `migrate deploy` and compares its dump with the snapshot, apart from each tool's history table.

**Incorrect (runtime DDL, `db push`, hand-written structural migrations):**

```typescript
// DDL at runtime, outside any migration
await this.prisma.$executeRaw`ALTER TABLE users ADD COLUMN loyalty_points INT`;
```

```bash
# Schema pushed straight to the database: no migration, nothing to deploy anywhere else
pnpm prisma db push
```

```sql
-- prisma/migrations/20261118120000_AddUserLoyaltyPoints/migration.sql written by hand (and PascalCase):
-- the index name and the column type are guesses, so the drift check reports them
ALTER TABLE "public"."users" ADD "loyalty_points" integer DEFAULT 0;
CREATE INDEX "users_loyalty_points_index" ON "users" ("loyalty_points");
```

**Correct (generated from the schema; by hand only what the schema cannot express):**

First change the schema:

```prisma
// prisma/user.prisma
model User {
  // ...
  loyaltyPoints Int @default(0) @map("loyalty_points")

  @@index([loyaltyPoints], map: "IDX_users_loyalty_points")
  @@map("users")
}
```

Then create the migration, review it, apply it and regenerate the client, and confirm there is no drift:

```bash
pnpm prisma migrate dev --create-only --name add_user_loyalty_points
pnpm prisma migrate dev
pnpm prisma generate
pnpm prisma migrate diff --from-config-datasource --to-schema prisma --exit-code   # must exit 0
```

The generated migration, with the CHECK constraint added by hand in review:

```sql
-- prisma/migrations/20261118120000_add_user_loyalty_points/migration.sql
-- AlterTable
ALTER TABLE "users" ADD COLUMN     "loyalty_points" INTEGER NOT NULL DEFAULT 0;

-- CreateIndex
CREATE INDEX "IDX_users_loyalty_points" ON "users"("loyalty_points");

-- By hand: the Prisma schema cannot express a CHECK constraint
ALTER TABLE "users" ADD CONSTRAINT "CHK_users_loyalty_points_non_negative" CHECK ("loyalty_points" >= 0);
```

The `users` table is small in this example, so the CHECK checks its rows at once. On a table that holds many rows, add it `NOT VALID` here and validate it in the next migration (see `db-use-migrations`).

Reference: [Prisma Migrate: development and production](https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production) · [Customizing migrations](https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/customizing-migrations)
