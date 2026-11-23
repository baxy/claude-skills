---
title: Generate Migrations from the Schema
impact: HIGH
impactDescription: Keeps the database schema and the ORM model in sync, without drift
tags: database, migrations, schema
---

## Generate Migrations from the Schema

The ORM never changes the schema by itself: no automatic schema sync, in any environment including local development. Every schema change goes through a migration, and every **structural** migration (tables, columns, constraints, indexes, foreign keys) is **generated** by the ORM's CLI from the change to the model. Do not hand-write structural migrations: hand-written DDL drifts from the names and types the ORM generates, and once it is applied, the next generated migration tries to undo the difference. Only migrations that do not touch table structure, data backfills and one-off scripts, are written by hand. This rule says what every ORM must do; the commands, options and examples are in `db-typeorm-use-migrations` for TypeORM and `db-prisma-use-migrations` for Prisma.

Workflow:

1. Change the model.
2. Generate the migration, then review the generated file. Look for `DROP` and `RENAME` statements you did not intend (see "Destructive changes" below).
3. Apply it.
4. Generate once more. It must produce no new migration; if it does, the previous migration drifted from the model. Fix that before moving on.

**When you cannot run the CLI** (no database or shell available), stop after step 1: change the model, write no structural migration file, and hand over the exact commands for steps 2 to 4 with the migration name (`pnpm migration:generate migrations/create-users-table`, not a placeholder). The change is not finished until someone runs them: say so first in your report, since nothing else in the repository shows it, and the app fails on its first query against the missing table. A hand-written stand-in labeled "regenerate this" still gets committed sometimes, and it is the drift this rule prevents. Data-only migrations can still be written by hand.

- **Never edit a migration that has been applied anywhere but your machine.** The databases it already ran on never get the change while new ones do, and the environments drift apart unnoticed. Fix forward with a new migration. With Prisma, fix forward on your machine too: Prisma Migrate notices the edit and asks to reset the database (see `db-prisma-use-migrations`).
- **Schema changes are additive where possible** (a nullable column, a column with a default, a new table), so the version still running during a rollout keeps working.
- **A new enum value is used only by a later migration.** PostgreSQL lets a transaction use an enum value only once the transaction that added it has committed, so a migration that adds a value (`ALTER TYPE ... ADD VALUE`) does not also name it, in a partial index's condition, a default or a backfill. The use goes in the next migration. TypeORM runs each migration in a transaction of its own (see `db-typeorm-use-migrations`); Prisma Migrate commits each plain statement of a file on its own, and runs a file that contains a `DO` block as one transaction (see `db-prisma-use-migrations`).
- **A constraint that existing rows must meet comes after they meet it.** A CHECK or a foreign key added to a table that already has rows checks every one of them. The migration fails on the first row that breaks it (a status an operator set by hand, a column the previous release leaves empty), and while it checks, it holds a lock that blocks every write to the table. Old rows that may break it are filled in a data migration first. On a table that already holds many rows, add the constraint `NOT VALID` in one migration, which checks only new writes, and run `ALTER TABLE ... VALIDATE CONSTRAINT` in the next, which checks the old rows without blocking writes. With TypeORM, append `NOT VALID` to the generated statement: the constraint keeps its generated name, so the next generate finds no drift. With Prisma, the CHECK is SQL added by hand to the migration (see `db-prisma-use-migrations`), where `NOT VALID` goes too.

**Destructive changes.** Dropping or renaming a column or a table, or narrowing a column's type, breaks the version that still runs during the rollout (see below). A rename is also easy to lose data with, since a generated migration can express it as a drop plus an add. Split a destructive change into expand and contract, across separate releases, each deployed once the one before runs everywhere:

1. Expand: add the new columns (nullable or with a default) in a generated migration, and fill them in a hand-written data migration after it (the backfill, below). The code writes the old and the new columns, since the previous release still runs and knows only the old one. It trusts the new columns of a row only while they agree with the old one: a row the previous release created or changed during the rollout has new columns that are empty or stale (a name changed after its first and last name were copied), and for that row the old column wins.
2. Read only the new columns. The migration first copies again the rows the previous release changed during the rollout, every row whose new columns are empty or disagree with the old one, not only the empty ones. The code still writes the old column, which the release before still reads.
3. Stop writing the old column: make it nullable, and unmap it from the model. An ORM model that still maps a column uses it, since the ORM names the column in every query of the model, so dropping it while such a release runs fails all of that release's queries. Unmapped, it stays in the schema, so the migration generator does not drop it, and no query names it (see `db-typeorm-use-migrations` or `db-prisma-use-migrations`).
4. Contract: drop the column.

Each structural step still comes from the model.

**A backfill that runs while the previous release still writes** works in batches (a range of ids per statement, each committed on its own), so no row stays locked while that release serves its users. The form that commits each batch is in `db-typeorm-use-migrations` for TypeORM and `db-prisma-use-migrations` for Prisma. Each batch computes the new values from the row it updates, in the `UPDATE`'s `SET` (`SET first_name = split_part(name, ' ', 1)`), never from a second read of the table joined in (`UPDATE users SET ... FROM (SELECT ... FROM users) AS p`). When the update waits for a row the previous release is changing, PostgreSQL re-reads the row it updates once the lock is released, but not the joined one, so it writes values computed from data that changed meanwhile. The backfill writes only the new columns: the old one belongs to the release still running, and a backfill that tidies it too (collapsing spaces in a name) overwrites a change made meanwhile and changes data nobody asked to change.

**Applying migrations when deploying.** Apply them in a deploy step of their own (a one-off task or init job) that finishes before the new version takes traffic, never from the application at startup, where every replica would run them at the same time. Until the rollout completes, the previous version still runs against the new schema, which is why destructive changes are split into expand and contract.

Reference: [Parallel Change (expand and contract)](https://martinfowler.com/bliki/ParallelChange.html)
