---
title: Test Database Guarantees Against a Real Database
impact: MEDIUM-HIGH
impactDescription: Proves constraints, indexes and migrations exist - something mocked repositories cannot
tags: testing, database, postgres, migrations, constraints
---

## Test Database Guarantees Against a Real Database

Some guarantees exist only in the database: CHECK constraints, unique and partial unique indexes, foreign keys, and where migrations actually create tables. E2E tests mock the repositories, so they cannot prove any of these; integration tests cannot reach tables that have no HTTP route yet. The database tier (`test/db/`, see `test-organize-by-tier`) covers them: it talks to a **migrated** database through the migration datasource and needs no API.

What belongs here:

- Constraints and indexes declared on entities (`@Check`, `@Index({ unique: true, where: ... })`, foreign keys), every one of them: a constraint without a spec is one a later migration can drop unnoticed.
- Migration behavior, such as tables landing in the configured schema.
- Write paths that only mean something against real SQL (bulk updates and deletes, backfills, dry runs that must write nothing).

**The tier runs on a database of its own** (`shop_test` in the examples), migrated like any other, never the database a running API uses. The specs that commit rows (the concurrent ones below) would hand their events to that API: its outbox dispatcher sends them and writes rows of its own, which then refuse the spec's cleanup through their foreign keys, and the integration tier's rows pile up in the same tables. The runner's config gives the tier its database name (see the runner's file in `references/`), and the README says how that database is created and migrated (see `docs-keep-readmes-current`).

How each case is written:

- Connect through the migration DataSource (`migrations/type-orm/config.ts` with TypeORM). It loads `.env` itself with `process.loadEnvFile()`, as it does for the migration CLI, and under Vitest that is enough. Under Jest it is not: Jest gives each spec file a copy of `process.env` that `process.loadEnvFile()` never reaches, so the DataSource would connect with its defaults. The Jest config's setup file parses `.env` into that copy before any spec loads (see `references/test-runner-jest.md`). Its globs name compiled files, so under the test runner it loads no entities: cases write and read with raw SQL, which is also what tests the database rather than the ORM.
- A service's bulk write (a cleanup's delete) runs through the service, since its query is what needs the real database. With TypeORM the spec names the entities before connecting (`dataSource.setOptions({ entities: [User, AuthSession] })`) and builds the service in a testing module with repositories from the case's query runner (`runner.manager.getRepository(AuthSession)`), so its statements join the transaction that is rolled back. The returned counts include rows that other runs left in the shared database: assert on the rows the case inserted.
- Run every case inside a transaction that is **rolled back** in `afterEach`, so the suite leaves no rows and test order does not matter.
- **A backfill migration** runs its own `up()` with the case's query runner (`await new BackfillProductPublishedAt1795088282595().up(runner)`), on rows the case inserted in the shape it fixes and in shapes it must leave alone, and the case checks both. When a later migration forbids the old shape (a `NOT NULL` added after the backfill), the case drops that constraint first, inside its transaction, so the rollback restores it.
- **A guarantee between concurrent transactions** (a row lock that makes writers wait, a limit that holds when 8 uploads arrive at once, a payment and a cancellation that share a gift card and a product and both finish, without a deadlock) needs connections that see each other's rows, so that spec is the exception: it commits its setup, runs the service's real transactions at the same moment (`Promise.allSettled`), asserts what each one ended with and the rows left, and deletes its rows in `afterAll`. It says so in a comment above its `describe`. Without it, nothing shows the lock is taken: a spec that runs the service inside the case's transaction passes with or without it.
- Wrap every **expected rejection in a savepoint**. Postgres aborts the whole transaction after a failed statement, so without one, everything after the first expected failure errors with "current transaction is aborted".
- Run one file at a time (`--no-file-parallelism` in Vitest, `--runInBand` in Jest). The tier needs a migrated database, so it is not part of the `test` script; run it before a PR that touches entities, migrations or services, and treat new specs here with the same review attention as production code.

**Incorrect (constraints asserted against mocks, or leaky real-database tests):**

```typescript
// A mocked repository cannot prove the unique index exists
it('rejects a duplicate email', async () => {
  userRepository.save.mockRejectedValue(new QueryFailedError('', [], new Error('duplicate key')));
  await expect(userService.create(dto)).rejects.toThrow();
});

// Real database, but rows are committed and the second assertion never runs
it('rejects a duplicate email, then accepts it after soft delete', async () => {
  await dataSource.query(`INSERT INTO users (email, name, password_hash) VALUES ('jane@example.com', 'Jane', 'hash')`);
  await expect(dataSource.query(`INSERT INTO users ...`)).rejects.toThrow(/UQ_users_email_active/);
  // Inside a transaction, this would now fail with "current transaction is aborted"
  await dataSource.query(`UPDATE users SET deleted_at = now() ...`);
});
```

**Correct (rolled-back transaction per case, savepoint per expected failure):**

```typescript
// test/db/user/user-constraints.spec.ts
import { QueryRunner } from 'typeorm';
import dataSource from '../../../migrations/type-orm/config';

describe('users constraints', () => {
  let runner: QueryRunner;

  const insertUser = async (email: string): Promise<string> => {
    const rows = (await runner.query(
      `INSERT INTO users (email, name, password_hash) VALUES ($1, 'Jane Doe', 'hash') RETURNING id`,
      [email],
    )) as { id: string }[];
    return rows[0].id;
  };

  // PostgreSQL aborts the whole transaction at a failed statement; rolling back to the savepoint keeps it usable
  const rejects = async (pattern: RegExp, run: () => Promise<unknown>): Promise<void> => {
    await runner.query('SAVEPOINT probe');
    await expect(run()).rejects.toThrow(pattern);
    await runner.query('ROLLBACK TO SAVEPOINT probe');
  };

  beforeAll(async () => {
    if (!dataSource.isInitialized) await dataSource.initialize();
  });

  afterAll(async () => {
    if (dataSource.isInitialized) await dataSource.destroy();
  });

  beforeEach(async () => {
    runner = dataSource.createQueryRunner();
    await runner.connect();
    await runner.startTransaction();
  });

  afterEach(async () => {
    await runner.rollbackTransaction();
    await runner.release();
  });

  describe('UQ_users_email_active', () => {
    it('rejects a duplicate active email, and frees it after a soft delete', async () => {
      await insertUser('jane@example.com');
      await rejects(/UQ_users_email_active/, () => insertUser('jane@example.com'));

      await runner.query(`UPDATE users SET deleted_at = now() WHERE email = 'jane@example.com'`);
      await expect(insertUser('jane@example.com')).resolves.toBeDefined();
    });
  });
});

// test/db/order/order-constraints.spec.ts
describe('orders constraints', () => {
  const insertOrder = (userId: string, totalCents: number) =>
    runner.query(`INSERT INTO orders (user_id, total_cents, currency, shipping_address) VALUES ($1, $2, 'EUR', '{}')`, [userId, totalCents]);

  describe('CHK_orders_total_cents_non_negative', () => {
    it('rejects a negative total and accepts zero', async () => {
      const userId = await insertUser('jane@example.com');
      await rejects(/CHK_orders_total_cents_non_negative/, () => insertOrder(userId, -1));
      await expect(insertOrder(userId, 0)).resolves.toBeDefined();
    });
  });
});
```

The orders spec opens and rolls back its runner as the users spec does, with the same `rejects()`, and inserts with its own `insertUser()` the user each order belongs to: an order constraint is the order module's, so its spec sits in `test/db/order/`.

Running this tier in CI needs a throwaway database (for example a Postgres service container) migrated before the run; without that it only runs where developers run it.

### With Prisma

- With Prisma no DataSource loads `.env` for the tier, so the `db` project's setup file does (`process.loadEnvFile('.env')` when the file exists under Vitest; under Jest see `references/test-runner-jest.md`), before the `env` that names the tier's own database is read.
- Constraint and migration specs use a plain `pg` client built from the app's mapping, `dbConfig()` (see `devops-use-config-module`), not `PrismaService`: these specs are about the database, and `BEGIN`, `ROLLBACK` and savepoints need a client the spec controls. `pg` is a direct dependency (see `references/project-setup.md`).
- Columns with a Prisma-side default (`@default(uuid())`, `@updatedAt`) have no database default, so raw inserts set them: `gen_random_uuid()`, `now()`.
- Every CHECK constraint has a spec here, since a migration adds it by hand and nothing else proves it exists (see `db-prisma-schema-conventions`). So does every partial unique index: its `WHERE` is behavior.
- A backfill migration's spec runs its `migration.sql` on the case's client, inside the case's transaction, after inserting the rows it must change and those it must leave alone. A batched backfill, a `DO` block that commits after each batch (see `db-prisma-use-migrations`), fails inside a transaction with `2D000`: its spec runs the file outside one, on rows it committed, and deletes them in `afterAll`.
- The tier's database is migrated with `prisma migrate deploy` before the run, never with `db push`: the point is to prove the migration history, hand-written SQL included.
- **A service's query against the real database** runs through `PrismaService` inside the case's transaction, held open and rolled back after the case (`openRolledBackTransaction()`, below). The service's own `$transaction` becomes a savepoint, and its work gets a client of its own, so a query on `this.prisma` inside the service's transaction throws instead of passing. The helper has limits: it refuses an `isolationLevel` (a savepoint has none) and drops `maxWait` and `timeout`, the service's locks last until the case ends, and a lock wait or an error at `COMMIT` cannot happen on the one connection. Row locks, conflicts and their retries, isolation levels and timeouts are tested like the concurrent specs above instead: a real `PrismaService` with its pool, committed setup, the service's transactions at the same moment, rows deleted in `afterAll`.
- **Each soft-delete filter has a case**, unless an e2e assertion names it (see `test-use-testing-module`): a deleted row the service must leave out. Prisma filters nothing by itself, so nothing else notices a dropped `deletedAt: null`. So does each omitted field: one table-driven spec over its own list of sensitive fields (not `PRISMA_OMIT`, which it guards) reads a row of each through `PrismaService` and checks the field is absent.
- **One spec forces real conflicts** through `PrismaTransactionService`: a deadlock through `retried()` (two transactions locking two rows in opposite orders, each waiting at a barrier after its first lock) and a serialization failure at `COMMIT` through `serializable()`. Both must finish, one of them run twice. A mocked error proves only the shape the test wrote; this proves the shapes Prisma throws, which an upgrade can change.
- **The time-zone spec runs on a database whose default time zone is not UTC** (`ALTER DATABASE shop_test SET timezone TO 'America/New_York'`, run once where the tier's database is created, the zone being an example), and checks that premise first: on a UTC database it passes with or without `TimeZone=UTC` in `PrismaService`. The premise is the zone's offset, never its name: the official `postgres` image's default zone is `Etc/UTC`, which a check that the name is not `UTC` lets through (checked on `postgres:18.4`: offset 0 there, -14400 seconds on a database set to America/New_York).

```typescript
// test/db/user/user-constraints.spec.ts
import { Client } from 'pg';
import { dbConfig } from '@/config/db.config';

describe('users constraints', () => {
  const db = dbConfig();
  const client = new Client({
    host: db.host,
    port: db.port,
    user: db.username,
    password: db.password,
    database: db.name,
    ssl: db.ssl ? { ca: db.sslCa } : false,
    options: `-c search_path=${db.schema},public`,
  });

  const insertUser = async (email: string): Promise<string> => {
    const { rows } = await client.query<{ id: string }>(
      `INSERT INTO users (id, email, name, password_hash, updated_at) VALUES (gen_random_uuid(), $1, 'Jane Doe', 'hash', now()) RETURNING id`,
      [email],
    );
    return rows[0].id;
  };

  // PostgreSQL aborts the whole transaction at a failed statement; rolling back to the savepoint keeps it usable
  const rejects = async (constraint: string, run: () => Promise<unknown>): Promise<void> => {
    await client.query('SAVEPOINT probe');
    await expect(run()).rejects.toMatchObject({ constraint });
    await client.query('ROLLBACK TO SAVEPOINT probe');
  };

  beforeAll(() => client.connect());
  afterAll(() => client.end());
  beforeEach(() => client.query('BEGIN'));
  afterEach(() => client.query('ROLLBACK'));

  describe('UQ_users_email_active', () => {
    it('rejects a duplicate active email, and frees it after a soft delete', async () => {
      await insertUser('jane@example.com');
      await rejects('UQ_users_email_active', () => insertUser('jane@example.com'));

      await client.query(`UPDATE users SET deleted_at = now() WHERE email = 'jane@example.com'`);
      await expect(insertUser('jane@example.com')).resolves.toBeDefined();
    });
  });
});
```

`pg` reports the violated constraint's name in the error's `constraint` field, which the savepoint helper matches.

The helper below, `test/helpers/database.ts`, lets the service under test join the case's transaction, which `rollBack()` undoes. It returns `tx` for the case's own statements and `prisma` for the services under test. Its `maxWait` and `timeout` fit the length of a case, not the app's limits (the values are examples).

```typescript
// test/helpers/database.ts
const ROLLED_BACK = Symbol('rolled back');

export type RolledBackTransaction = {
  tx: Prisma.TransactionClient;
  prisma: PrismaService;
  rollBack: () => Promise<void>;
};

export async function openRolledBackTransaction(prisma: PrismaService): Promise<RolledBackTransaction> {
  let rollBack!: () => void;
  let opened!: (tx: Prisma.TransactionClient) => void;
  const ready = new Promise<Prisma.TransactionClient>((resolve) => (opened = resolve));
  const ended = prisma
    .$transaction(
      (tx) => {
        opened(tx);
        return new Promise<never>((_, reject) => (rollBack = () => reject(ROLLED_BACK)));
      },
      { maxWait: 10_000, timeout: 60_000 },
    )
    .catch((error: unknown) => {
      if (error !== ROLLED_BACK) throw error;
    });
  const tx = await Promise.race([ready, ended.then(() => Promise.reject(new Error('The transaction ended before it opened')))]);
  return { tx, prisma: prismaInTransaction(prisma, tx), rollBack: async () => { rollBack(); await ended; } };
}

// A query on the outer client while the service's savepoint is open would run outside the service's transaction
function prismaInTransaction(prisma: PrismaService, tx: Prisma.TransactionClient): PrismaService {
  let savepoints = 0;
  let inSavepoint = false;
  const client = (isWork: boolean): PrismaService =>
    new Proxy(tx, {
      get: (target, property) => {
        if (property === '$transaction') return runInSavepoint;
        if (inSavepoint && !isWork) throw new Error(`this.prisma.${String(property)} inside the service's transaction`);
        return Reflect.get(target, property);
      },
    }) as unknown as PrismaService;
  const outer = client(false);
  const work = client(true);

  async function runInSavepoint(run: unknown, options?: { isolationLevel?: unknown }): Promise<unknown> {
    if (Array.isArray(run)) return Promise.all(run);
    if (options?.isolationLevel) throw new Error('A savepoint has no isolation level: test this with a pool');
    const name = `service_${(savepoints += 1)}`;
    await tx.$executeRawUnsafe(`SAVEPOINT ${name}`);
    inSavepoint = true;
    try {
      const result = await (run as (client: PrismaService) => Promise<unknown>)(work);
      await tx.$executeRawUnsafe(`RELEASE SAVEPOINT ${name}`);
      return result;
    } catch (error) {
      await tx.$executeRawUnsafe(`ROLLBACK TO SAVEPOINT ${name}`);
      throw error;
    } finally {
      inSavepoint = false;
    }
  }
  return outer;
}
```

In the service spec, `UserService` runs in the case's transaction, and `PrismaService` connects with the `DB_*` variables as the app reads them (`dbConfig()`). The time-zone spec's `client` is a plain `pg` client, so it has the database's own time zone.

```typescript
// test/db/user/user-service.spec.ts
const SESSION_ID = '00000000-0000-4000-8000-000000000001';

describe('UserService', () => {
  const prisma = new PrismaService(dbConfig());
  let transaction: RolledBackTransaction;
  let userService: UserService;

  beforeEach(async () => {
    transaction = await openRolledBackTransaction(prisma);
    const moduleRef = await Test.createTestingModule({
      providers: [UserService, { provide: PrismaService, useValue: transaction.prisma }],
    }).compile();
    userService = moduleRef.get(UserService);
  });
  afterEach(() => transaction.rollBack());
  afterAll(() => prisma.$disconnect());

  it('leaves a deleted user out', async () => {
    const [user] = await transaction.tx.$queryRaw<{ id: string }[]>`
      INSERT INTO users (id, email, name, password_hash, updated_at, deleted_at)
      VALUES (gen_random_uuid(), 'jane@example.com', 'Jane Doe', 'hash', now(), now()) RETURNING id`;
    const authUser: JwtUser = { id: user.id, email: 'jane@example.com', roles: [Role.User], sessionId: SESSION_ID };

    await expect(userService.findById(authUser, user.id)).rejects.toThrow(NotFoundException);
  });
});

// test/db/prisma/prisma-time-zone.spec.ts
it('stores the instant it is given on a database outside UTC', async () => {
  // By offset, not by name: a zone named Etc/UTC or GMT is UTC too
  const { rows: [{ offset }] } = await client.query<{ offset: string }>(
    `SELECT extract(timezone FROM '2026-06-01T12:00:00Z'::timestamptz) AS offset`,
  );
  expect(Number(offset)).not.toBe(0);

  const user = await transaction.prisma.user.create({
    data: { email: 'jane@example.com', name: 'Jane Doe', passwordHash: 'hash', createdAt: new Date('2026-06-01T12:00:00Z') },
  });
  const [stored] = await transaction.tx.$queryRaw<{ utc: string }[]>`
    SELECT to_char(created_at AT TIME ZONE 'UTC', 'YYYY-MM-DD HH24:MI') AS utc FROM users WHERE id = ${user.id}::uuid`;
  expect(stored.utc).toBe('2026-06-01 12:00');
});
```

The conflict spec runs real transactions on the pool, so its rows are committed and deleted in `afterAll`, and the pool's `lock_timeout` stays above `deadlock_timeout` (see `db-prisma-use-transactions`).

```typescript
// test/db/prisma/prisma-transaction.service.spec.ts
const barrier = (parties: number): (() => Promise<void>) => {
  let arrived = 0;
  let release!: () => void;
  const all = new Promise<void>((resolve) => (release = resolve));
  return () => {
    arrived += 1;
    if (arrived === parties) release();
    return all;
  };
};

it('retries the transaction that loses a deadlock', async () => {
  const bothLocked = barrier(2);
  let attempts = 0;
  const lockBoth = (first: string, second: string) =>
    new PrismaTransactionService(prisma).retried(async (tx) => {
      attempts += 1;
      await tx.$queryRaw`SELECT sku FROM inventory_items WHERE sku = ${first} FOR UPDATE`;
      // Already open on the retry
      await bothLocked();
      await tx.$queryRaw`SELECT sku FROM inventory_items WHERE sku = ${second} FOR UPDATE`;
    });

  const results = await Promise.allSettled([lockBoth('SKU-001', 'SKU-002'), lockBoth('SKU-002', 'SKU-001')]);

  expect(results.map((result) => result.status)).toEqual(['fulfilled', 'fulfilled']);
  expect(attempts).toBe(3);
});
```

Reference: [PostgreSQL SAVEPOINT](https://www.postgresql.org/docs/current/sql-savepoint.html) · [TypeORM QueryRunner](https://typeorm.io/docs/query-runner)
