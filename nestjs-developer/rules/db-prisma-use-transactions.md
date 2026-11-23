---
title: Use Transactions for Multi-Step Operations with Prisma
impact: HIGH
impactDescription: Every write of a multi-step operation commits or rolls back together, and checks on the current state cannot race
tags: database, transactions, prisma, consistency, concurrency
---

## Use Transactions for Multi-Step Operations with Prisma

The Prisma form of `db-use-transactions`: read that rule first for when a transaction or a conditional update is needed, and why.

- **Interactive**: `this.prisma.$transaction(async (tx) => { ... })`, when a step depends on an earlier result or on a check in between. Every query inside goes through `tx`; a query on `this.prisma` runs outside the transaction and commits on its own.
- **Sequential**: `this.prisma.$transaction([queryA, queryB])`, for independent writes that must commit together. It is no snapshot: under `READ COMMITTED` each of its statements reads what is committed when it starts, so a row committed between them shows in the second only. Like an interactive one, it holds a connection for its whole length and fails with `P2028` once `maxWait` passes while the pool is busy, where a plain query waits in the pool's queue. A page and its total are two plain queries (see `perf-optimize-database`), or this form with `{ isolationLevel: Prisma.TransactionIsolationLevel.RepeatableRead }` when the two must agree.
- **A nested write** (a `create` with a nested `create` of its children) is one transaction by itself.
- **A service method that joins the caller's transaction** takes `tx: Prisma.TransactionClient` as its first parameter and uses it for every query.
- **An interactive transaction times out** after `timeout`, and waits at most `maxWait` for a connection, both set in `PrismaService` from settings (see `db-prisma-schema-conventions`). Do not raise the timeout to make a slow transaction pass. A wait for a row lock counts toward the timeout, and the timer does not interrupt a statement that waits: with a 1-second timeout and the row locked elsewhere for 3 seconds, the request failed with `P2028` after 3 seconds, its connection held the whole time and its work thrown away. The pool's `lock_timeout` bounds the wait (below).
- **A conditional update** is `updateMany` with the expected state in its `where`. Act on the `count` it returns.
- **A conflict** (SQLSTATE `40001` or `40P01`) comes in three shapes, each carrying PostgreSQL's code in the driver adapter's error: `P2034` on a model query and `P2010` on a raw one (the `FOR UPDATE` row locks below), both with `meta.driverAdapterError.cause.originalCode`, and on the `COMMIT` the `DriverAdapterError` itself, rethrown unconverted, with `cause.originalCode`. A retry that checks `P2034` misses a deadlock on a row lock and every conflict at commit, so the conflict check reads PostgreSQL's code.

**Incorrect (writes without a transaction, `this.prisma` inside one, an external call inside one):**

```typescript
@Injectable()
export class OrderService {
  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    // If any later step fails, the order row stays written
    const order = await this.prisma.order.create({
      data: { userId: authUser.id, status: OrderStatus.Pending, totalCents: 0, currency: dto.currency, shippingAddress: dto.shippingAddress },
    });

    for (const item of dto.items) {
      await this.prisma.orderItem.create({ data: { ...item, orderId: order.id } });
      await this.inventoryService.reserve(item.sku, item.quantity); // Commits on its own
    }
    return order;
  }

  async cancel(authUser: JwtUser, id: string): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      await tx.order.update({ where: { id }, data: { status: OrderStatus.Canceled } });
      await this.prisma.orderItem.deleteMany({ where: { orderId: id } }); // this.prisma: outside the transaction
      await this.paymentService.refund(id);                              // A network call holding the transaction open
    });
  }
}
```

**Correct (one transaction for the database writes; the external call after commit):**

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly prismaTransactionService: PrismaTransactionService,
    private readonly productService: ProductService,
    private readonly inventoryService: InventoryService,
    private readonly paymentService: PaymentService,
  ) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    // Every transaction locks inventory items in sku order, so two orders cannot deadlock on them
    const items = [...dto.items].sort((a, b) => a.sku.localeCompare(b.sku));
    const lines = await this._priceLines(items);

    const order = await this.prismaTransactionService.retried(async (tx) => {
      const created = await tx.order.create({
        data: {
          userId: authUser.id,
          status: OrderStatus.Pending,
          totalCents: lines.reduce((sum, line) => sum + line.quantity * line.unitPriceCents, 0),
          currency: dto.currency,
          shippingAddress: dto.shippingAddress,
          items: { create: lines },
        },
      });

      for (const line of lines) {
        await this.inventoryService.reserve(tx, line.sku, line.quantity);
      }

      return created;
    });

    await this.paymentService.authorize(order);
    return order;
  }

  private async _priceLines(items: OrderItemDto[]): Promise<Prisma.OrderItemCreateWithoutOrderInput[]> {
    const products = await this.productService.findBySkus(items.map((item) => item.sku));
    const productsBySku = new Map(products.map((product) => [product.sku, product]));
    return items.map(({ sku, quantity }) => {
      const product = productsBySku.get(sku);
      if (!product) {
        throw new BadRequestException(`Unknown product ${sku}`, { errorCode: 'UNKNOWN_PRODUCT' });
      }
      return { productId: product.id, sku, quantity, unitPriceCents: product.priceCents };
    });
  }
}
```

Prices come from the product module, never from the request: the client sends only what it buys and how many (`OrderItemDto`), and an unknown sku is a 400. The nested create stores the order and its items in one write. Inventory items belong to the inventory module, so `InventoryService` writes them, inside the order's transaction. Other orders lock the same inventory items, so the transaction runs through the retry (below), and it takes them in `sku` order, the one order every transaction on them follows. The payment call comes after the commit: a failed charge leaves a pending order to retry or cancel, never half-written rows.

`InventoryService.reserve()` takes the caller's `tx`, so its write joins the caller's transaction. Its conditional update makes the stock check and the decrement one statement, so two buyers cannot both pass it, and its exception rolls back the caller's whole transaction:

```typescript
// inventory/services/inventory.service.ts
@Injectable()
export class InventoryService {
  async reserve(tx: Prisma.TransactionClient, sku: string, quantity: number): Promise<void> {
    const { count } = await tx.inventoryItem.updateMany({
      where: { sku, quantity: { gte: quantity } },
      data: { quantity: { decrement: quantity } },
    });
    if (count === 0) {
      throw new BadRequestException(`Not enough stock for ${sku}`, { errorCode: 'INSUFFICIENT_STOCK' });
    }
  }
}
```

A redemption checks the balance between its steps, so it takes the interactive form, and other transactions change the same balance, so it runs through the retry:

```typescript
// gift-card/services/gift-card.service.ts
@Injectable()
export class GiftCardService {
  constructor(private readonly prismaTransactionService: PrismaTransactionService) {}

  async redeem(giftCardId: string, orderId: string, amountCents: number): Promise<void> {
    await this.prismaTransactionService.retried(async (tx) => {
      const debited = await tx.giftCard.updateMany({
        where: { id: giftCardId, balanceCents: { gte: amountCents } },
        data: { balanceCents: { decrement: amountCents } },
      });
      if (debited.count === 0) {
        throw new BadRequestException('The gift card balance is too low', { errorCode: 'INSUFFICIENT_GIFT_CARD_BALANCE' });
      }

      await tx.giftCardRedemption.create({ data: { giftCardId, orderId, amountCents } });
    });
  }
}
```

**A state transition** puts the expected status in the `where` of one `updateMany` and acts on `count`:

```typescript
// order/services/order.service.ts
async markAsPaid(id: string): Promise<void> {
  const { count } = await this.prisma.order.updateMany({
    where: { id, status: OrderStatus.Pending, deletedAt: null },
    data: { status: OrderStatus.Paid },
  });
  if (count) {
    return;
  }

  const order = await this.prisma.order.findFirst({ where: { id, deletedAt: null } });
  if (!order) {
    throw new NotFoundException(`Order #${id} not found`);
  }
  throw new BadRequestException(`An order in status ${order.status} cannot be paid`, { errorCode: 'ORDER_NOT_PAYABLE' });
}
```

Two payment callbacks for one order cannot both mark it paid. When no row changed, the order was either never pending or paid by a concurrent call.

**A plain update** is one `update()` with the request's fields and `deletedAt: null` in its `where`. Prisma writes only the fields it is given, and returns the row. The DTO's fields the request left out are `undefined`, which `strictUndefinedChecks` refuses (see `db-prisma-schema-conventions`), so the update takes the defined ones. With no field left, Prisma sends no `UPDATE` at all: it reads the row and returns it, still `P2025` when it is missing, and `@updatedAt` does not move. When an empty PATCH should still count as a change, the update sets `updatedAt` itself:

```typescript
// shared/utils/defined-fields.ts
export function definedFields<T extends object>(dto: T): Partial<T> {
  return Object.fromEntries(Object.entries(dto).filter(([, value]) => value !== undefined)) as Partial<T>;
}

// user/services/user.service.ts
async update(authUser: JwtUser, id: string, dto: UpdateUserDto): Promise<UserRecord> {
  this._assertVisible(authUser, id);
  const fields = definedFields(dto);
  const data = Object.keys(fields).length > 0 ? fields : { updatedAt: new Date() };
  return this.prisma.user.update({ where: { id, deletedAt: null }, data });
}
```

A DTO instance holds every optional field it declares, `undefined` when not sent, hence `definedFields()`. In this example an empty PATCH still marks the user as updated. A missing or deleted user is `P2025`, which the exception filter answers with 404; an account the caller may not see gets the same 404 from `_assertVisible()` (see `error-throw-http-exceptions`).

**A row lock**, for a limit of this module checked by a count and then an insert. Prisma has no lock option on its queries, so the lock is a raw `SELECT ... FOR UPDATE` in the interactive transaction. Uploads to one product wait for each other on its row, and a wait past the pool's `lock_timeout` fails with `55P03` (below):

```typescript
// product/services/product-image.service.ts
private _saveWithinLimit(image: ProductImageDraft): Promise<ProductImage> {
  return this.prisma.$transaction(async (tx) => {
    const locked = await tx.$queryRaw<{ id: string }[]>`
      SELECT id FROM products WHERE id = ${image.productId}::uuid AND deleted_at IS NULL FOR UPDATE`;
    if (locked.length === 0) {
      throw new NotFoundException(`Product #${image.productId} not found`);
    }
    const count = await tx.productImage.count({ where: { productId: image.productId } });
    if (count >= PRODUCT_IMAGE_MAX_COUNT) {
      throw new BadRequestException(`A product can have at most ${PRODUCT_IMAGE_MAX_COUNT} images`, {
        errorCode: 'PRODUCT_IMAGE_LIMIT_REACHED',
      });
    }
    return tx.productImage.create({ data: { ...image, position: count } });
  });
}
```

**An advisory lock**, for a limit no row of this module stands for. The limit is one clients see, so it lives in `constants/`. One customer's placements wait for each other on a lock held until the transaction ends, whose key names its purpose, and the pool's `lock_timeout` bounds this wait too; the user row is the user module's, so it is not locked here:

```typescript
// order/constants/order-limits.constant.ts
export const ORDER_MAX_PENDING = 3;

// order/services/order.service.ts
create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
  return this.prisma.$transaction(async (tx) => {
    await tx.$executeRaw`SELECT pg_advisory_xact_lock(hashtextextended(${`order-pending:${authUser.id}`}, 0))`;
    const pendingOrders = await tx.order.count({ where: { userId: authUser.id, status: OrderStatus.Pending, deletedAt: null } });
    if (pendingOrders >= ORDER_MAX_PENDING) {
      throw new BadRequestException(`A customer can have at most ${ORDER_MAX_PENDING} unpaid orders`, {
        errorCode: 'UNPAID_ORDER_LIMIT_REACHED',
      });
    }
    return tx.order.create({
      data: { userId: authUser.id, status: OrderStatus.Pending, totalCents: 0, currency: dto.currency, shippingAddress: dto.shippingAddress, items: { create: dto.items } },
    });
  });
}
```

**Every lock wait has a limit.** Any statement may wait on a lock: a row lock, an advisory lock, a conditional update of a row another transaction is writing (the gift card's balance, the inventory item above), with or without a transaction around it. `PrismaService` sets `lock_timeout` in the pool's startup options (see `db-prisma-schema-conventions`), so every connection has it and no service has to remember a first statement: below the transaction's `timeout` with room for the work (for example 2 seconds against 5). Keep it above PostgreSQL's `deadlock_timeout` (1 second by default): below it, both sides of a deadlock failed with `55P03` before PostgreSQL could pick one to end, so neither was retried. A wait past it fails at the lock and frees the connection, and nothing is written: the client may send the request again. A transaction that needs another limit (a job that may wait longer) sets its own first, `` tx.$executeRaw`SELECT set_config('lock_timeout', ${`${ms}ms`}, true)` `` (`SET LOCAL` takes no bind parameter, `42601` with one), and it ends with the transaction. Checked on PostgreSQL 18 through `@prisma/adapter-pg` 7.10.1: the startup option set `1500ms` on every connection, a transaction's `set_config(..., true)` raised it to `4s` for that transaction only, and an `UPDATE` outside any transaction and an advisory lock behind a holder both failed after 1.5 seconds. The wait fails with SQLSTATE `55P03` in the driver adapter's error: `P2010` on a raw query, `P2039` on a model query (both checked on 7.10.1). The global filter answers it with 503 and `Retry-After`, as it answers a transaction that found no connection within `maxWait` (`P2028` without `meta.timeout`), and logs both at warn, without a Sentry event (see `error-use-exception-filters`). A transaction that outlived its `timeout` is different: `P2028` with `meta.timeout` and `meta.timeTaken` on its next query or its `COMMIT`. With lock waits bounded, that one means the work itself is too slow, a bug, so it stays a 500 reported to Sentry.

**A retry on a conflict**, for a transaction that locks rows other transactions also lock, and `SERIALIZABLE` for a check over rows that no one key names. The helpers recognize a conflict by PostgreSQL's code:

```typescript
// prisma/utils/postgres-error-code.ts
import { Prisma } from '../generated/client';

// A query's error carries the SQLSTATE under meta; a failed COMMIT rethrows the driver adapter's error itself
export function postgresErrorCode(error: unknown): string | undefined {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    const meta = error.meta as { driverAdapterError?: { cause?: { originalCode?: string } } } | undefined;
    return meta?.driverAdapterError?.cause?.originalCode;
  }
  if (error instanceof Error && error.name === 'DriverAdapterError') {
    return (error.cause as { originalCode?: string } | undefined)?.originalCode;
  }
  return undefined;
}

// prisma/utils/transaction-conflict.ts
import { ConflictException } from '@nestjs/common';
import { postgresErrorCode } from './postgres-error-code';

// Serialization failure, deadlock
const CONFLICT_CODES = new Set(['40001', '40P01']);

export function isTransactionConflict(error: unknown): boolean {
  return CONFLICT_CODES.has(postgresErrorCode(error) ?? '');
}

export function transactionConflictException(): ConflictException {
  return new ConflictException('Too many changes at the same moment; try again', { errorCode: 'TRANSACTION_CONFLICT' });
}

// prisma/services/prisma-transaction.service.ts
const TRANSACTION_ATTEMPTS = 3;

@Injectable()
export class PrismaTransactionService {
  constructor(private readonly prisma: PrismaService) {}

  // The work runs again on a conflict, so it only touches the database: a call to another system would be repeated
  retried<T>(work: (tx: Prisma.TransactionClient) => Promise<T>): Promise<T> {
    return this._retryOnConflict(() => this.prisma.$transaction(work));
  }

  serializable<T>(work: (tx: Prisma.TransactionClient) => Promise<T>): Promise<T> {
    return this._retryOnConflict(() => this.prisma.$transaction(work, { isolationLevel: Prisma.TransactionIsolationLevel.Serializable }));
  }

  private async _retryOnConflict<T>(run: () => Promise<T>): Promise<T> {
    for (let attempt = 1; ; attempt += 1) {
      try {
        return await run();
      } catch (error) {
        if (!isTransactionConflict(error)) {
          throw error;
        }
        if (attempt === TRANSACTION_ATTEMPTS) {
          throw transactionConflictException();
        }
      }
    }
  }
}
```

`postgresErrorCode()` returns PostgreSQL's SQLSTATE behind a failed Prisma call, whatever the query's error (`P2002`, `P2010` for raw SQL, `P2034`, `P2039`...). Everything that knows Prisma's transactions and errors is the prisma module's, so `shared/` never imports Prisma (see `arch-shared-folder`): a module that retries a transaction injects `PrismaTransactionService`, which `PrismaModule` exports next to `PrismaService`, and the global filter imports the conflict check from the module's `utils/`. `TRANSACTION_ATTEMPTS` is an implementation detail, so it stays in the file. A conflict that outlasts the attempts becomes a 409, not a 500. The whole transaction runs again on a conflict, so every check in it reads what the other one wrote.

**A chunked bulk write** falls back to one row at a time for any error a row's own data causes, recognized by PostgreSQL's code as well:

```typescript
// prisma/utils/row-level-error.ts
import { postgresErrorCode } from './postgres-error-code';

export function isRowLevelError(error: unknown): boolean {
  const code = postgresErrorCode(error) ?? '';
  // SQLSTATE classes 22 (data exception) and 23 (integrity constraint violation)
  return code.startsWith('22') || code.startsWith('23');
}

// product/services/product-import.service.ts
private async _importChunk(rows: ProductImportRow[]): Promise<void> {
  try {
    await this.prisma.$transaction((tx) => this._importRows(tx, rows));
  } catch (error) {
    if (!isRowLevelError(error)) {
      throw error;
    }
    for (const row of rows) {
      try {
        await this.prisma.$transaction((tx) => this._importRows(tx, [row]));
      } catch (rowError) {
        if (!isRowLevelError(rowError)) {
          throw rowError;
        }
        await this._markRowFailed(row, rowError);
      }
    }
  }
}
```

The import runs one transaction per chunk, and the chunk's rows one by one when one is refused. Any other error, such as a lost connection or a conflict, is thrown, and the job tries the chunk again. A refused row is marked failed with its reason, by SQLSTATE, and the next rows go on.

Reference: [Prisma transactions](https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions)
