---
title: Keep the Schema Persistence-Only and Consistent with Prisma
impact: MEDIUM-HIGH
impactDescription: One schema split by owning module, with snake_case tables and columns and named indexes and constraints
tags: database, prisma, schema, models, conventions, postgres
---

## Keep the Schema Persistence-Only and Consistent with Prisma

The Prisma form of `db-schema-conventions`: read that rule first for the conventions and why they hold. There are no entity classes: the schema lives in `prisma/`, and services use the model types and enums of the generated client.

Layout:

- `prisma/schema.prisma` holds only the `generator` and `datasource` blocks. The models live in one file per owning module: `Order` and `OrderItem` in `prisma/order.prisma`, `User` in `prisma/user.prisma`. Only that module's services write its models (see `arch-data-access-in-services`).
- `prisma.config.ts` at the project root points the CLI at the `prisma/` folder and holds the database URL. A `url` in the `datasource` block is an error in Prisma 7.
- The client is generated into `src/prisma/generated/`, which is gitignored and rebuilt by `prisma generate`. Feature modules import model types and enums from `@/prisma/generated/client` and query through `PrismaService` (below).
- Two preview features are enabled: `partialIndexes`, so partial indexes are declared in the schema like any other index, and `strictUndefinedChecks` (below).

Conventions:

- The model is **singular PascalCase** and the table **plural snake_case**: `model Order { ... @@map("orders") }`. Always write the `@@map`.
- Prisma has no naming strategy: every multi-word field carries `@map("snake_case")` (`userId String @map("user_id")`). Without it the column is literally `"userId"`.
- Ids are `String @id @default(uuid()) @db.Uuid`. A foreign key is `@db.Uuid` too, named `{relation}Id` and mapped, with the relation field beside it.
- Timestamps: `createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz(3)` and `updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz(3)`. A `DateTime` without `@db.Timestamptz` is a `timestamp` without time zone. Prisma Client fills both from the app server's clock: it sends the `@default(now())` value with every create, and `@updatedAt` has no database default, so raw SQL writes set it.
- A column the database compares with its own `now()` takes the database's clock and keeps `now()`'s precision: `nextAttemptAt DateTime @default(dbgenerated("transaction_timestamp()")) @map("next_attempt_at") @db.Timestamptz(6)` for a row a query claims with `next_attempt_at <= now()`. With `@default(now())`, an instance whose clock runs ahead of the database's writes a time still in the database's future, and the claim that runs right after the commit finds nothing due. With `Timestamptz(3)`, PostgreSQL rounds `now()`'s microseconds to the millisecond, upward about half the time (`.0006` is stored as `.001`), so the stored time is later than the `now()` that stamped it: a claim in the same transaction, or right after the commit, misses the row. On PostgreSQL 18, 510 of 1000 rows stamped and claimed in one transaction were not due with `Timestamptz(3)`, and none with `Timestamptz(6)`; a database-tier spec that runs the claim in the case's transaction fails as often. `transaction_timestamp()` is `now()` under another name: Prisma reads a `now()` or `CURRENT_TIMESTAMP` default back as its own `now()`, so `dbgenerated("now()")` shows as drift on every check.
- Enums (see `db-schema-conventions` for when a set gets one): the values are named as the database stores them, without `@map`: `enum OrderStatus { pending paid shipped canceled @@map("order_status") }`. Prisma Client's values are the names in the schema, so `Pending @map("pending")` makes the client read and write `'Pending'`: the API answers `"Pending"`, `@IsEnum()` accepts only that, and a raw `${status}::order_status` cast fails with `22P02`. Code that wants PascalCase names gets a const object whose values are the database's, typed by the generated union (below). A set that grows with features stays a `String` (`String[]` for several), typed in code with a TypeScript enum.
- Soft deletes: `deletedAt DateTime? @map("deleted_at") @db.Timestamptz(3)`. Prisma has no soft delete: deleting is `update({ where: { id }, data: { deletedAt: new Date() } })`, and **every query on the model filters `deletedAt: null`** itself. Nothing filters for you, so a dropped filter returns deleted rows with every test passing unless a test names it: each filter has one (see `test-use-testing-module` and `test-database-tier`). A value unique among live rows is a partial unique index, `@@unique([email], map: "UQ_users_email_active", where: { deletedAt: null })`. Such an index makes `email` a `findUnique` key without its condition: `findUnique({ where: { email } })` compiles and can return a soft-deleted row, so always add `deletedAt: null`.
- Structured data: `Json` (`jsonb`), with `?` or a `@default` when the value is optional.
- Sensitive fields (password hashes, token hashes) are added to `PRISMA_OMIT` in the same change (below), and to the database-tier spec that checks each one stays out of results (see `test-database-tier`).
- Indexes and unique constraints are named with `map:`: `@@index([userId, status], map: "IDX_orders_user_id_status")`, `@unique(map: "UQ_users_email")`, `@@unique([...], map: "UQ_...")`. Primary and foreign keys keep Prisma's names (`orders_pkey`, `orders_user_id_fkey`).
- A partial index's object form takes equality, `null` and `not` only (`where: { deletedAt: null }`, `where: { kind: "nightly" }`); `in` is refused. A condition it cannot express is `where: raw("...")`, written exactly as PostgreSQL prints it back (`pg_indexes.indexdef` once the index exists): `raw("(status = ANY (ARRAY['pending'::order_status, 'paid'::order_status]))")`. PostgreSQL stores any other spelling (`status IN ('pending', 'paid')`) in its own form, and the drift check then reports the index as dropped and added on every run.
- A CHECK constraint is the one structural object the schema cannot express. Add it by hand to the SQL of the migration that introduces the column, named `CHK_{table}_{purpose}` (see `db-prisma-use-migrations`), and cover it with a database-tier spec (see `test-database-tier`): nothing else checks it is still there.

**An `undefined` value in a query means no filter.** Prisma drops a `where` key whose value is `undefined`, so `updateMany({ where: { id: job.orderId } })` with a missing id updates every row of the table, `deleteMany` deletes them, and `findFirst({ where: { email, deletedAt: null } })` returns some other user. With `strictUndefinedChecks`, an explicit `undefined` anywhere in a query throws a `PrismaClientValidationError` instead (checked on 7.10.1). Then:

- An optional filter is `status: query.status ?? Prisma.skip`, or a conditional spread for a value that is never falsy.
- A DTO instance holds `undefined` too: TypeScript compiles each declared field to a class field (with the ES2022 or later target `nest new` generates), so every optional field the request left out is an own property set to `undefined`, and `data: dto` throws. A write takes the DTO's defined fields (`definedFields(dto)`, see `db-prisma-use-transactions`).

**`contains`, `startsWith` and `endsWith` do not escape LIKE's wildcards.** Prisma puts the value into a `LIKE` pattern (`ILIKE` with `mode: 'insensitive'`) as it is, so a `%` or `_` in a client's search term matches anything, and `?search=%` lists every row. Checked on 7.10.1: `startsWith: '%'` and `contains: '_'` returned every row, and the escaped values only the rows that hold the character. Escape `\`, `%` and `_` with a backslash, PostgreSQL's default escape character, before the filter, so a client's text inside the pattern matches itself; which index serves such a filter is in `perf-optimize-database`.

```typescript
// shared/utils/escape-like.ts
export function escapeLike(value: string): string {
  return value.replace(/[\\%_]/g, (character) => `\\${character}`);
}

// user/services/user.service.ts
where: { name: { startsWith: escapeLike(query.search), mode: 'insensitive' }, deletedAt: null },
```

### `PrismaService`, the one client

The `prisma` module provides `PrismaService`: a `PrismaClient` on the `@prisma/adapter-pg` driver adapter, which Prisma 7 requires and which owns the connection pool. It reads the same `db` namespace as any other database setting (see `devops-use-config-module`), leaves the sensitive fields out of every query, logs queries at `debug` when `db.logging` is on (so they show with `APP_LOG_LEVEL=debug`, see `devops-use-logging`), and closes the pool once the HTTP server has closed (see `devops-graceful-shutdown`). It also sets:

- **The session's time zone, UTC, and the search path**, in the pool's startup `options`. The adapter sends a `DateTime` as a UTC time without its offset, and PostgreSQL reads it in the session's time zone: on a database whose default zone was Europe/Berlin, 12:00Z was stored as 10:00Z. Prisma read it back as 12:00Z, so only raw SQL, `now()` comparisons and reports see the shift. A pooler in transaction mode (PgBouncer) drops startup options; behind one, they are the role's settings (`ALTER ROLE shop_app SET timezone = 'UTC'`, and its `search_path` and `lock_timeout`).
- **The lock wait limit, `lock_timeout`, in the same startup options**, so every statement that waits on a lock fails with `55P03` once the limit passes, in a transaction or not (see `db-prisma-use-transactions`).
- **A warn line for a connection the database ends**: `onPoolError` for an idle pooled connection, `onConnectionError` for one a transaction holds. The adapter logs both at `debug` only, so a failover or a restart would leave no trace.
- **`transactionOptions` from settings**: `maxWait`, how long a transaction waits for a free connection, and `timeout`, how long it may run (Prisma's defaults are 2 and 5 seconds). With the lock wait limit (see `db-prisma-use-transactions`), they are settings of the `db` namespace, since the right values depend on the load: the defaults below are examples. Each must be a whole number, and the lock wait limit must sit between PostgreSQL's `deadlock_timeout` (1 second by default) and the transaction's `timeout`, so `validate-env.ts` refuses the service otherwise (see `devops-use-config-module`): `parseInt('2s')` is 2, a lock wait limit of 2 milliseconds.

It opens no connection at startup: with the pg adapter, `$connect()` resolves at once and the first query fails with `P1001` while the database is down. The app therefore starts without a database, and readiness keeps it out of traffic until the database answers (see `micro-use-health-checks`). The module is not global: every module that queries imports `PrismaModule` (see `arch-module-sharing`).

```typescript
// config/db.config.ts
transactionMaxWaitMs: parseInt(process.env.DB_TRANSACTION_MAX_WAIT_MS || '2000', 10),
transactionTimeoutMs: parseInt(process.env.DB_TRANSACTION_TIMEOUT_MS || '5000', 10),
lockTimeoutMs: parseInt(process.env.DB_LOCK_TIMEOUT_MS || '2000', 10),

// config/validate-env.ts
const lockTimeoutMs = Number(env.DB_LOCK_TIMEOUT_MS || 2000);
if (lockTimeoutMs <= 1000 || lockTimeoutMs >= Number(env.DB_TRANSACTION_TIMEOUT_MS || 5000)) {
  throw new Error('Refusing to start: DB_LOCK_TIMEOUT_MS must be over 1000 (deadlock_timeout's default) and under DB_TRANSACTION_TIMEOUT_MS');
}

// prisma/constants/prisma-omit.constant.ts
export const PRISMA_OMIT = {
  user: { passwordHash: true },
} as const;

// prisma/services/prisma.service.ts
// The options are type arguments too: without them, the omitted fields stay in the result types, $on('query') does
// not compile and transactionOptions is refused
type PrismaServiceOptions = {
  adapter: PrismaPg;
  omit: typeof PRISMA_OMIT;
  log: [{ emit: 'event'; level: 'query' }];
  transactionOptions: { maxWait: number; timeout: number };
};

@Injectable()
export class PrismaService extends PrismaClient<PrismaServiceOptions> implements OnApplicationShutdown {
  private readonly logger = new Logger(PrismaService.name);

  constructor(@Inject(dbConfig.KEY) dbSettings: ConfigType<typeof dbConfig>) {
    super({
      adapter: new PrismaPg(
        {
          host: dbSettings.host,
          port: dbSettings.port,
          user: dbSettings.username,
          password: dbSettings.password,
          database: dbSettings.name,
          // Without ca, Node's own list of CAs
          ssl: dbSettings.ssl ? { ca: dbSettings.sslCa } : false,
          max: dbSettings.poolSize,
          // TimeZone: the adapter sends a DateTime without its offset, which PostgreSQL reads in the session's zone
          options: `-c search_path=${dbSettings.schema},public -c TimeZone=UTC -c lock_timeout=${dbSettings.lockTimeoutMs}`,
        },
        {
          schema: dbSettings.schema,
          onPoolError: (error) => this.logger.warn({ err: error }, 'The database ended an idle pooled connection'),
          onConnectionError: (error) => this.logger.warn({ err: error }, 'The database ended a connection in a transaction'),
        },
      ),
      omit: PRISMA_OMIT,
      log: [{ emit: 'event', level: 'query' }],
      transactionOptions: { maxWait: dbSettings.transactionMaxWaitMs, timeout: dbSettings.transactionTimeoutMs },
    });
    if (dbSettings.logging) {
      // Never event.params: they hold the values
      this.$on('query', (event) => this.logger.debug({ query: event.query, durationMs: event.duration }));
    }
  }

  async onApplicationShutdown(): Promise<void> {
    await this.$disconnect();
  }
}

// prisma/prisma.module.ts
@Module({
  providers: [PrismaService, PrismaTransactionService],
  exports: [PrismaService, PrismaTransactionService],
})
export class PrismaModule {}
```

`db.config.ts` keeps the three limits next to the other `db` settings (see `devops-use-config-module`), the lock wait limit below the transaction timeout with room for the work. `validate-env.ts` adds the three to `WHOLE_NUMBERS` (see `devops-use-config-module`), then checks the limits against each other. A field in `PRISMA_OMIT` is never selected unless a query opts back in with `omit: { passwordHash: false }` (see `api-use-dto-serialization`). In the pool's startup options, the search path is there because raw SQL names its tables unqualified, and `lock_timeout` keeps any lock wait from outlasting the limit. The query log leaves out `event.params`, which hold the values (see `devops-use-logging`). `$disconnect()` also ends the adapter's pool.

`prisma.config.ts` is read by the Prisma CLI only, outside Nest. Like the TypeORM migration DataSource, it reads `process.env` directly (see `devops-use-config-module`), with the same `DB_*` variables and the same fallbacks as `db.config.ts`: `||`, which also replaces an empty value, where a destructuring default would keep `DB_SCHEMA=` as an empty schema name. The Prisma 7 CLI does not load `.env` itself.

`DB_SSL` is `true` everywhere but `APP_ENV` `dev`, which `validate-env.ts` checks (see `devops-use-config-module`). With it on, the CLI needs `sslaccept=strict`. `sslmode=require` alone encrypts without checking the certificate in Prisma's schema engine: the CLI connected to a server whose certificate an unknown CA had signed for another host, and would have sent that server the password. With `sslaccept=strict` it refused it (`P1011`); with `sslcert` set to the CA's file it connected, and it still refused a host the certificate does not name. A private CA (a managed database's own) is a file both sides read, named by `DB_SSL_CA_FILE`: `sslcert=<absolute path>` here, and `ssl: { ca }` in the app's pool options, from `db.sslCa`, which `db.config.ts` reads from that file (see `devops-use-config-module`); `ssl: true` checks against Node's own list of CAs only.

```typescript
// prisma.config.ts
import 'dotenv/config';
import { defineConfig } from 'prisma/config';

const { env } = process;
const caFile = env.DB_SSL_CA_FILE ? `&sslcert=${encodeURIComponent(env.DB_SSL_CA_FILE)}` : '';
const tls = env.DB_SSL === 'true' ? `&sslmode=require&sslaccept=strict${caFile}` : '';

export default defineConfig({
  // The folder: every .prisma file in it is part of one schema
  schema: 'prisma',
  migrations: { path: 'prisma/migrations' },
  datasource: {
    url:
      `postgresql://${encodeURIComponent(env.DB_USERNAME || '')}:${encodeURIComponent(env.DB_PASSWORD || '')}` +
      `@${env.DB_HOST || 'localhost'}:${env.DB_PORT || '5432'}/${encodeURIComponent(env.DB_NAME || '')}` +
      `?schema=${encodeURIComponent(env.DB_SCHEMA || 'public')}${tls}`,
  },
});
```

**Incorrect (no mapping, wrong types, unnamed index, model in another module's file):**

```prisma
// prisma/user.prisma - the order model appended to the user module's file
model Order {
  id         String    @id @default(uuid())   // text, not uuid
  userId     String                           // Column "userId": quoted in every raw query
  user       User      @relation(fields: [userId], references: [id])
  status     String                           // Free text instead of an enum
  createdAt  DateTime  @default(now())        // timestamp without time zone
  deletedAt  DateTime?                        // Column "deletedAt"

  @@index([userId])                           // Prisma's name, not IDX_
}                                             // No @@map: table "Order"

enum OrderStatus {
  Pending @map("pending")                     // Prisma Client reads and writes 'Pending'
}
```

**Correct (owned files, mapped names, explicit types, named indexes):**

```prisma
// prisma/schema.prisma
generator client {
  provider        = "prisma-client"
  output          = "../src/prisma/generated"
  previewFeatures = ["partialIndexes", "strictUndefinedChecks"]
}

datasource db {
  provider = "postgresql"
}
```

```prisma
// prisma/order.prisma
model Order {
  id              String      @id @default(uuid()) @db.Uuid
  userId          String      @map("user_id") @db.Uuid
  user            User        @relation(fields: [userId], references: [id])
  status          OrderStatus @default(pending)
  totalCents      Int         @map("total_cents")
  currency        String      @db.Char(3)
  shippingAddress Json        @map("shipping_address")
  items           OrderItem[]
  createdAt       DateTime    @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt       DateTime    @updatedAt @map("updated_at") @db.Timestamptz(3)
  deletedAt       DateTime?   @map("deleted_at") @db.Timestamptz(3)

  @@index([userId, status], map: "IDX_orders_user_id_status")
  @@index([userId], map: "IDX_orders_user_id_active", where: { deletedAt: null })
  @@index([createdAt], map: "IDX_orders_created_at_open", where: raw("(status = ANY (ARRAY['pending'::order_status, 'paid'::order_status]))"))
  @@map("orders")
}

model OrderItem {
  id             String  @id @default(uuid()) @db.Uuid
  orderId        String  @map("order_id") @db.Uuid
  order          Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId      String  @map("product_id") @db.Uuid
  product        Product @relation(fields: [productId], references: [id])
  sku            String
  quantity       Int
  unitPriceCents Int     @map("unit_price_cents")

  @@index([orderId], map: "IDX_order_items_order_id")
  @@map("order_items")
}

enum OrderStatus {
  pending
  paid
  shipped
  canceled

  @@map("order_status")
}
```

```typescript
// order/types/order-status.enum.ts
import type { OrderStatus as PrismaOrderStatus } from '@/prisma/generated/client';

export const OrderStatus = {
  Pending: 'pending',
  Paid: 'paid',
  Shipped: 'shipped',
  Canceled: 'canceled',
} as const satisfies Record<string, PrismaOrderStatus>;

export type OrderStatus = PrismaOrderStatus;
```

```prisma
// prisma/user.prisma
model User {
  id           String    @id @default(uuid()) @db.Uuid
  email        String
  name         String
  passwordHash String    @map("password_hash")
  orders       Order[]
  createdAt    DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt    DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)
  deletedAt    DateTime? @map("deleted_at") @db.Timestamptz(3)

  @@unique([email], map: "UQ_users_email_active", where: { deletedAt: null })
  @@map("users")
}
```

The CHECK constraint is appended by hand after the generated DDL:

```sql
-- prisma/migrations/20261118120000_create_orders/migration.sql
ALTER TABLE "orders" ADD CONSTRAINT "CHK_orders_total_cents_non_negative" CHECK ("total_cents" >= 0);
```

Reference: [Prisma: database mapping](https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/database-mapping) · [Prisma: indexes](https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes) · [prisma.config.ts](https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference)