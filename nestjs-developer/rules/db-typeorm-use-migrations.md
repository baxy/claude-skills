---
title: Generate Migrations from Entities with TypeORM
impact: HIGH
impactDescription: Keeps the database schema and the TypeORM entities in sync, without drift
tags: database, migrations, typeorm, schema
---

## Generate Migrations from Entities with TypeORM

The TypeORM form of `db-use-migrations`: read that rule first for the workflow, destructive changes and deploys.

`synchronize` is hardcoded to `false` in the TypeORM options, in every environment including local development, and is never read from config or an environment variable. Structural migrations come from `migration:generate`; do not scaffold an empty file with `migration:create` and fill in DDL yourself. Hand-written SQL drifts from TypeORM's naming for constraints, indexes and foreign keys, and the next `migration:generate` produces spurious statements that drop and recreate those objects just to rename them. `migration:create` is only for data backfills, one-off scripts and the migration that validates a constraint added `NOT VALID` (`ALTER TABLE ... VALIDATE CONSTRAINT`, see `db-use-migrations`).

- The workflow's steps are `migration:generate`, `migration:run`, then `migration:generate` again as the drift check.
- Filenames are lowercase and dash-separated; the class name stays PascalCase: `1777494397675-create-order-items-table.ts` containing `CreateOrderItemsTable1777494397675`.
- An applied migration is never edited because TypeORM records applied migrations by name, not by content: `migration:run` skips an edited one wherever it already ran, without a warning.
- `migration:generate` emits a `RENAME COLUMN` only when exactly one column of the table changed its name and nothing else; any other rename comes out as a drop plus an add.
- A column unmapped before it is dropped (see the destructive changes of `db-use-migrations`) keeps its property, as `@Column({ nullable: true, select: false, insert: false, update: false })`: no query of the entity names it, and `migration:generate` sees no change. Removing the property would generate the `DROP COLUMN` a release too early.
- Leave table and index names unqualified in migration SQL. TypeORM's `schema` option qualifies the migrations table and entity queries, but not the raw SQL inside migration files, which follows the connection's `search_path`. When the app uses a non-`public` schema, set the same `search_path` on the migration connection and on the app's (schema first, then `public`): the app's own raw SQL, an extension's operators (`pg_trgm`'s `%`, `citext` comparisons) and the types an extension adds resolve through it too, and TypeORM's `schema` option sets no `search_path`. An extension a migration creates lands in the first schema of the migration connection's path, so an app connection without that path neither finds it nor fails at startup: its searches answer 500 and its `citext` columns compare as text. Strip the schema prefix that `migration:generate` emits: with the `public` schema it qualifies the `DROP INDEX` statements of `down()` (`DROP INDEX "public"."..."`) and every enum type, in `CREATE TYPE`, the column type and `DROP TYPE` (`"public"."product_status"`), since TypeORM 1.1 names enum types with their schema whatever it is; with any other schema it qualifies every statement in both directions (`CREATE TABLE "shop"."users"`).
- The deploy step runs `migration:run`. Do not set `migrationsRun: true`: every replica would run the migrations at startup, at the same time. Roll back with `migration:revert`, which undoes the last applied migration per call.
- The migration DataSource sets `migrationsTransactionMode: 'each'`. TypeORM's default, `'all'`, runs every pending migration in one transaction, so a fresh database applies an enum value and its first use (see `db-use-migrations`) in the same transaction, which PostgreSQL refuses.
- A batched backfill (see `db-use-migrations`) sets `transaction = false` on its class, so TypeORM runs it outside a transaction and each batch, one `UPDATE` through the migration's query runner, commits on its own (below).
- Run `migration:revert` once on the throwaway database for every new migration. A generated `down()` that removes enum values rebuilds the type under the column, and fails when the column has a default or a partial index's condition names the type; drop those first and restore them after, by hand, with a comment saying so. Revert it also with rows the migration made possible (an order with no shipping address once pickups exist): a `down()` that cannot put them back in the old shape (a `SET NOT NULL` over rows that are now `NULL`) first refuses, with a message saying which rows block it, instead of failing halfway through a revert of several migrations.

**Incorrect (synchronize, ad-hoc DDL, or hand-written structural migrations):**

```typescript
// synchronize turned on, or read from config
TypeOrmModule.forRoot({
  type: 'postgres',
  synchronize: true, // Can drop columns, tables, or data
});
synchronize: dbSettings.synchronize, // One wrong environment variable away from the line above

// DDL at runtime, outside any migration
await this.dataSource.query('ALTER TABLE users ADD COLUMN loyalty_points INT');

// Hand-written structural migration: index and constraint names will not
// match TypeORM's, so the next migration:generate shows drift
// migrations/1705312800000-AddUserLoyaltyPoints.ts   (also: PascalCase filename)
export class AddUserLoyaltyPoints1705312800000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "public"."users" ADD "loyalty_points" integer DEFAULT 0`);
    await queryRunner.query(`CREATE INDEX "users_loyalty_points_idx" ON "users" ("loyalty_points")`);
  }
  // ...
}
```

**Correct (generate from entities, create only for data changes):**

First change the entity:

```typescript
@Entity('users')
@Index('IDX_users_loyalty_points', ['loyaltyPoints'])
export class User {
  // ...
  @Column({ type: 'integer', default: 0 })
  loyaltyPoints: number;
}
```

Then generate the migration, review it and run it, and confirm there is no drift:

```bash
pnpm run migration:generate migrations/add-user-loyalty-points
pnpm run migration:run
pnpm run migration:generate migrations/drift-check   # must report no changes
```

The generated migration:

```typescript
// migrations/1705312800000-add-user-loyalty-points.ts
export class AddUserLoyaltyPoints1705312800000 implements MigrationInterface {
  name = 'AddUserLoyaltyPoints1705312800000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "users" ADD "loyalty_points" integer NOT NULL DEFAULT '0'`);
    await queryRunner.query(`CREATE INDEX "IDX_users_loyalty_points" ON "users" ("loyalty_points") `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX "IDX_users_loyalty_points"`);
    await queryRunner.query(`ALTER TABLE "users" DROP COLUMN "loyalty_points"`);
  }
}
```

A data-only change is written with `migration:create`. A backfill that runs while the previous release still writes is batched: it walks the table in id order and updates a range of ids per statement (1000 rows here, an example). It updates only the rows whose new column is still empty, so it can run again: when a batch fails, the batches before it stay committed, TypeORM does not record the migration, and the next `migration:run` fills the rest.

```typescript
// migrations/1705313000000-backfill-user-first-names.ts
export class BackfillUserFirstNames1705313000000 implements MigrationInterface {
  // Outside a transaction, so each batch commits on its own and no row stays locked while the previous release writes
  transaction = false;

  public async up(queryRunner: QueryRunner): Promise<void> {
    let lastId = '00000000-0000-0000-0000-000000000000';
    for (;;) {
      const [batchEnd]: { id: string }[] = await queryRunner.query(
        'SELECT id FROM (SELECT id FROM users WHERE id > $1 ORDER BY id LIMIT 1000) AS batch ORDER BY id DESC LIMIT 1',
        [lastId],
      );
      if (!batchEnd) {
        return;
      }
      await queryRunner.query(
        `UPDATE users SET first_name = split_part(name, ' ', 1) WHERE id > $1 AND id <= $2 AND first_name IS NULL`,
        [lastId, batchEnd.id],
      );
      lastId = batchEnd.id;
    }
  }

  public async down(): Promise<void> {
    // Backfill is not reversible; nothing to undo structurally
  }
}
```

The migration DataSource for a non-public schema sets `search_path` as a connection option, schema first, then `public`, and the app's DataSource sets the same option. Its `entities` glob names the compiled copy of the entities (see `references/project-setup.md`), and it sets the app's naming strategy, uuid options and TLS (see `db-typeorm-schema-conventions`):

```typescript
// migrations/type-orm/config.ts
const ca = process.env.DB_SSL_CA_FILE ? readFileSync(process.env.DB_SSL_CA_FILE, 'utf8') : undefined;

export default new DataSource({
  type: 'postgres',
  schema: process.env.DB_SCHEMA,
  ssl: process.env.DB_SSL === 'true' ? { ca } : false,
  entities: [join(__dirname, '../../src/**/*.entity.js')],
  migrations: [join(__dirname, '../*.js')],
  // PostgreSQL refuses a new enum value until the transaction that added it commits
  migrationsTransactionMode: 'each',
  namingStrategy: new SnakeNamingStrategy(),
  uuidExtension: 'pgcrypto',
  installExtensions: false,
  // A startup option, so every pooled connection starts with it
  extra: { options: `-c search_path=${process.env.DB_SCHEMA},public` },
});
```

Reference: [TypeORM Migrations](https://typeorm.io/docs/migrations/why)
