---
title: Use Transactions for Multi-Step Operations with TypeORM
impact: HIGH
impactDescription: Every write of a multi-step operation commits or rolls back together, and checks on the current state cannot race
tags: database, transactions, typeorm, consistency, concurrency
---

## Use Transactions for Multi-Step Operations with TypeORM

The TypeORM form of `db-use-transactions`: read that rule first for when a transaction or a conditional update is needed, and why. Use `dataSource.transaction()`, through `DatabaseTransactionService.retried()` when the transaction locks rows other transactions also lock (below). A query runner (`dataSource.createQueryRunner()`) is for manual control, such as several commits on one connection in a batched backfill (see `db-typeorm-use-migrations`).

- **Every query inside goes through the transaction's manager** (`manager.save()`, `manager.update()`, `manager.getRepository(X)`). An injected repository or `this.dataSource` runs outside the transaction and commits on its own.
- **A service method that joins the caller's transaction** takes `manager: EntityManager` as its first parameter and uses it for every query.
- **A conditional update** is `update()` with the expected state in its criteria, or a query builder `update()` with it in `where()`. Act on the `affected` count it returns.
- **`update()` does not skip soft-deleted rows**, unlike the `find` methods: put `deletedAt: IsNull()` in its criteria.
- **Never `save()` a loaded entity to update it.** `save()` reloads the row, soft-deleted rows included, and writes every column whose value in memory differs from the reloaded one. A concurrent change to another column is overwritten with the stale value, and a soft delete that landed between the read and the save is undone. `save()` is for inserts.
- **A conflict** (SQLSTATE `40001` or `40P01`), on a query or on the `COMMIT`, is thrown as a `QueryFailedError` with the code in `driverError.code`.

**Incorrect (multiple saves without transaction):**

```typescript
// Multiple saves without transaction
@Injectable()
export class OrderService {
  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    // If any step fails, the rows before it stay written
    const order = await this.orderRepository.save({ userId: authUser.id, status: OrderStatus.Pending });

    for (const item of dto.items) {
      await this.orderItemRepository.save({ orderId: order.id, sku: item.sku, quantity: item.quantity });
      await this.inventoryService.reserve(item.sku, item.quantity); // Commits on its own
    }

    return order;
  }
}
```

**Correct (one transaction for the database writes; the external call after commit):**

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly databaseTransactionService: DatabaseTransactionService,
    private readonly productService: ProductService,
    private readonly inventoryService: InventoryService,
    private readonly paymentService: PaymentService,
  ) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    // Every transaction locks inventory items in sku order, so two orders cannot deadlock on them
    const items = [...dto.items].sort((a, b) => a.sku.localeCompare(b.sku));
    const lines = await this._priceLines(items);

    const order = await this.databaseTransactionService.retried(async (manager) => {
      const created = await manager.save(
        manager.create(Order, {
          userId: authUser.id,
          status: OrderStatus.Pending,
          totalCents: lines.reduce((sum, line) => sum + line.quantity * line.unitPriceCents, 0),
        }),
      );

      for (const line of lines) {
        await manager.save(OrderItem, { orderId: created.id, ...line });
        await this.inventoryService.reserve(manager, line.sku, line.quantity);
      }

      return created;
    });

    await this.paymentService.authorize(order);
    return order;
  }

  private async _priceLines(items: OrderItemDto[]): Promise<Pick<OrderItem, 'productId' | 'sku' | 'quantity' | 'unitPriceCents'>[]> {
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

Prices come from the product module, never from the request: the client sends only what it buys and how many (`OrderItemDto`), and an unknown sku is a 400. Every write goes through the transaction's manager. `manager.create()` builds an instance, so `@Exclude()` still applies to the response (see `api-use-dto-serialization`). Inventory items belong to the inventory module, so `InventoryService` writes them, inside the order's transaction. Other orders lock the same inventory items, so the transaction runs through the retry (below), and it takes them in `sku` order, the one order every transaction on them follows. The payment call comes after the commit: a failed charge leaves a pending order to retry or cancel, never half-written rows.

`InventoryService.reserve()` takes the caller's manager, so its write joins the caller's transaction. Its conditional update makes the stock check and the decrement one statement, so two buyers cannot both pass it, and its exception rolls back the caller's whole transaction:

```typescript
// inventory/services/inventory.service.ts
@Injectable()
export class InventoryService {
  async reserve(manager: EntityManager, sku: string, quantity: number): Promise<void> {
    const { affected } = await manager
      .createQueryBuilder()
      .update(InventoryItem)
      .set({ quantity: () => 'quantity - :quantity' })
      .where('sku = :sku AND quantity >= :quantity', { sku, quantity })
      .execute();
    if (!affected) {
      throw new BadRequestException(`Not enough stock for ${sku}`, { errorCode: 'INSUFFICIENT_STOCK' });
    }
  }
}
```

A redemption checks the balance between its steps, and other transactions change the same balance, so its steps run in one transaction through the retry:

```typescript
// gift-card/services/gift-card.service.ts
@Injectable()
export class GiftCardService {
  constructor(private readonly databaseTransactionService: DatabaseTransactionService) {}

  async redeem(giftCardId: string, orderId: string, amountCents: number): Promise<void> {
    await this.databaseTransactionService.retried(async (manager) => {
      // Raw SQL inside the builder names columns, not properties
      const debited = await manager
        .createQueryBuilder()
        .update(GiftCard)
        .set({ balanceCents: () => 'balance_cents - :amountCents' })
        .where('id = :giftCardId AND balance_cents >= :amountCents', { giftCardId, amountCents })
        .execute();
      if (!debited.affected) {
        throw new BadRequestException('The gift card balance is too low', { errorCode: 'INSUFFICIENT_GIFT_CARD_BALANCE' });
      }

      await manager.save(GiftCardRedemption, { giftCardId, orderId, amountCents });
    });
  }
}
```

**A state transition** puts the expected status in the criteria of one `update()` and acts on `affected`:

```typescript
// order/services/order.service.ts
async markAsPaid(id: string): Promise<void> {
  const { affected } = await this.orderRepository.update(
    { id, status: OrderStatus.Pending, deletedAt: IsNull() },
    { status: OrderStatus.Paid },
  );
  if (affected) {
    return;
  }

  const order = await this.orderRepository.findOne({ where: { id } });
  if (!order) {
    throw new NotFoundException(`Order #${id} not found`);
  }
  throw new BadRequestException(`An order in status ${order.status} cannot be paid`, { errorCode: 'ORDER_NOT_PAYABLE' });
}
```

Two payment callbacks for one order cannot both mark it paid. When no row changed, the order was either never pending or paid by a concurrent call.

**A plain update** is one `update()` with the request's fields, then a read for the response:

```typescript
// user/services/user.service.ts
async update(authUser: JwtUser, id: string, dto: UpdateUserDto): Promise<User> {
  this._assertVisible(authUser, id);
  const { affected } = await this.userRepository.update({ id, deletedAt: IsNull() }, dto);
  if (!affected) {
    throw new NotFoundException(`User #${id} not found`);
  }
  return this.findById(authUser, id);
}
```

It writes only the fields in the request, and only to a user that is not deleted and that the caller may see (`_assertVisible()`, see `error-throw-http-exceptions`). The read returns an instance, so `@Exclude()` still applies. An empty body still sets `updatedAt`: TypeORM adds the `@UpdateDateColumn` to every update.

**A row lock**, for a limit of this module checked by a count and then an insert. `lock: { mode: 'pessimistic_write' }` is `SELECT ... FOR UPDATE`, held until the transaction ends, so uploads to one product wait for each other on its row:

```typescript
// product/services/product-image.service.ts
private _saveWithinLimit(image: ProductImageDraft): Promise<ProductImage> {
  return this.dataSource.transaction(async (manager) => {
    const product = await manager.findOne(Product, {
      select: { id: true },
      where: { id: image.productId },
      lock: { mode: 'pessimistic_write' },
    });
    if (!product) {
      throw new NotFoundException(`Product #${image.productId} not found`);
    }
    const count = await manager.countBy(ProductImage, { productId: image.productId });
    if (count >= PRODUCT_IMAGE_MAX_COUNT) {
      throw new BadRequestException(`A product can have at most ${PRODUCT_IMAGE_MAX_COUNT} images`, {
        errorCode: 'PRODUCT_IMAGE_LIMIT_REACHED',
      });
    }
    return manager.save(manager.create(ProductImage, { ...image, position: count }));
  });
}
```

**An advisory lock**, for a limit no row of this module stands for. The limit is one clients see, so it lives in `constants/`. One customer's placements wait for each other on a lock held until the transaction ends, whose key names its purpose; the user row is the user module's, so it is not locked here:

```typescript
// order/constants/order-limits.constant.ts
export const ORDER_MAX_PENDING = 3;

// order/services/order.service.ts
async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
  return this.dataSource.transaction(async (manager) => {
    await manager.query('SELECT pg_advisory_xact_lock(hashtextextended($1, 0))', [`order-pending:${authUser.id}`]);
    const pendingOrders = await manager.countBy(Order, { userId: authUser.id, status: OrderStatus.Pending });
    if (pendingOrders >= ORDER_MAX_PENDING) {
      throw new BadRequestException(`A customer can have at most ${ORDER_MAX_PENDING} unpaid orders`, {
        errorCode: 'UNPAID_ORDER_LIMIT_REACHED',
      });
    }
    const order = await manager.save(manager.create(Order, { userId: authUser.id, status: OrderStatus.Pending }));
    await manager.save(OrderItem, dto.items.map((item) => ({ orderId: order.id, sku: item.sku, quantity: item.quantity })));
    return order;
  });
}
```

**A retry on a conflict**, for a transaction that locks rows other transactions also lock, and `SERIALIZABLE` for a check over rows that no one key names:

```typescript
// database/utils/transaction-conflict.ts
import { ConflictException } from '@nestjs/common';
import { QueryFailedError } from 'typeorm';

// Serialization failure, deadlock
const TRANSACTION_CONFLICT_CODES = new Set(['40001', '40P01']);

export function isTransactionConflict(error: unknown): boolean {
  return error instanceof QueryFailedError && TRANSACTION_CONFLICT_CODES.has((error.driverError as Error & { code?: string }).code ?? '');
}

export function transactionConflictException(): ConflictException {
  return new ConflictException('Too many changes at the same moment; try again', { errorCode: 'TRANSACTION_CONFLICT' });
}

// database/services/database-transaction.service.ts
const TRANSACTION_ATTEMPTS = 3;

@Injectable()
export class DatabaseTransactionService {
  constructor(private readonly dataSource: DataSource) {}

  // The work runs again on a conflict, so it only touches the database: a call to another system would be repeated
  retried<T>(work: (manager: EntityManager) => Promise<T>): Promise<T> {
    return this._retryOnConflict(() => this.dataSource.transaction(work));
  }

  serializable<T>(work: (manager: EntityManager) => Promise<T>): Promise<T> {
    return this._retryOnConflict(() => this.dataSource.transaction('SERIALIZABLE', work));
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

// database/database.module.ts
@Module({
  imports: [TypeOrmModule.forRootAsync({ /* ... */ })],
  providers: [DatabaseTransactionService],
  exports: [DatabaseTransactionService],
})
export class DatabaseModule {}
```

The `database` module owns the connection: it registers `TypeOrmModule.forRootAsync()` (its options are in `devops-use-config-module`), which `AppModule` imports through it, and holds everything that knows TypeORM's transactions and errors, so `shared/` never imports TypeORM (see `arch-shared-folder`). A module that retries a transaction imports `DatabaseModule` and injects `DatabaseTransactionService`; the global filter imports the conflict check from its `utils/`. `TRANSACTION_ATTEMPTS` is an implementation detail, so it stays in the file. A conflict that outlasts the attempts becomes a 409, not a 500. The whole transaction runs again on a conflict, so every check in it reads what the other one wrote.

Canceling an order locks gift cards and inventory items that other orders share, so it takes them in the app's one order of tables, as every other transaction on them does:

```typescript
// order/services/order.service.ts
const canceled = await this.databaseTransactionService.retried(async (manager) => {
  // One lock order for every transaction on these rows: the order, its gift cards, then its inventory items
  const order = await this._lock(manager, authUser, id);
  // ...
  await this.giftCardService.restore(manager, order.id);
  await this.inventoryService.release(manager, items);
  return order;
});
```

**A chunked bulk write** falls back to one row at a time for any error a row's own data causes:

```typescript
// database/utils/row-level-error.ts
export function isRowLevelError(error: unknown): boolean {
  const code = error instanceof QueryFailedError ? ((error.driverError as Error & { code?: string }).code ?? '') : '';
  // SQLSTATE classes 22 (data exception) and 23 (integrity constraint violation)
  return code.startsWith('22') || code.startsWith('23');
}

// product/services/product-import.service.ts
private async _importChunk(rows: ProductImportRow[]): Promise<void> {
  try {
    await this.dataSource.transaction((manager) => this._importRows(manager, rows));
  } catch (error) {
    if (!isRowLevelError(error)) {
      throw error;
    }
    for (const row of rows) {
      try {
        await this.dataSource.transaction((manager) => this._importRows(manager, [row]));
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

Reference: [TypeORM Transactions](https://typeorm.io/transactions)
