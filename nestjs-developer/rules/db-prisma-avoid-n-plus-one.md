---
title: Avoid N+1 Query Problems with Prisma
impact: HIGH
impactDescription: Related rows load with the query that needs them, not one query per row
tags: database, n-plus-one, queries, performance, prisma
---

## Avoid N+1 Query Problems with Prisma

The Prisma form of `db-avoid-n-plus-one`. With Prisma the cause is almost always an `await` inside a loop. Load the module's own relations in the same call with `include` or a nested `select`, or load the children of many rows with one `in` query and group them in memory. Another module's rows come from one call to its service, never from an `include` of its model.

- By default, `include` loads each relation level with one more query (`WHERE order_id IN (...)`), however many rows the list has, so it is not an N+1. `relationLoadStrategy: 'join'` still needs the `relationJoins` preview feature in Prisma 7; do not enable it for this.
- Prisma batches `findUnique` calls made in the same tick with the same `where` fields and selection, but never a sequential `await` in a loop, `findFirst` or `findMany`. Do not rely on the batching: write the `in` query.
- Counts and sums run in the database: `_count` in a `select`, `groupBy` with `_sum`.
- `DB_LOGGING=true` logs every query `PrismaService` sends (`db.logging`, see `db-prisma-schema-conventions`).

**Incorrect (a query per row):**

```typescript
@Injectable()
export class OrderService {
  async findAllWithItems(authUser: JwtUser): Promise<OrderWithItems[]> {
    const orders = await this.prisma.order.findMany({ where: { userId: authUser.id, deletedAt: null } });
    // 1 query for the orders, then N more, one per order
    return Promise.all(
      orders.map(async (order) => ({
        ...order,
        items: await this.prisma.orderItem.findMany({ where: { orderId: order.id } }),
      })),
    );
  }
}
```

**Correct (relations in the same call, or one `in` query):**

```typescript
// order/types/order-with-items.type.ts
export type OrderWithItems = Prisma.OrderGetPayload<{ include: { items: true } }>;

// order/constants/order-summary-select.constant.ts
export const ORDER_SUMMARY_SELECT = {
  id: true,
  status: true,
  totalCents: true,
  items: { select: { id: true, quantity: true, unitPriceCents: true } },
  _count: { select: { items: true } },
} satisfies Prisma.OrderSelect;
// order/types/order-summary.type.ts
export type OrderSummary = Prisma.OrderGetPayload<{ select: typeof ORDER_SUMMARY_SELECT }>;

@Injectable()
export class OrderService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly productService: ProductService,
  ) {}

  findAllWithItems(authUser: JwtUser): Promise<OrderWithItems[]> {
    return this.prisma.order.findMany({
      where: { userId: authUser.id, deletedAt: null },
      include: { items: true },
    });
  }

  async findProductsOfOrders(orders: OrderWithItems[]): Promise<Map<string, Product>> {
    const productIds = [...new Set(orders.flatMap((order) => order.items.map((item) => item.productId)))];
    const products = await this.productService.findByIds(productIds);
    return new Map(products.map((product) => [product.id, product]));
  }

  findSummaries(authUser: JwtUser): Promise<OrderSummary[]> {
    return this.prisma.order.findMany({ where: { userId: authUser.id, deletedAt: null }, select: ORDER_SUMMARY_SELECT });
  }

  async findItemsByOrderIds(orderIds: string[]): Promise<Map<string, OrderItem[]>> {
    const items = await this.prisma.orderItem.findMany({ where: { orderId: { in: orderIds } } });
    const itemsByOrder = new Map<string, OrderItem[]>();
    for (const item of items) {
      itemsByOrder.set(item.orderId, [...(itemsByOrder.get(item.orderId) ?? []), item]);
    }
    return itemsByOrder;
  }
}

// product/services/product.service.ts
findByIds(ids: string[]): Promise<Product[]> {
  return this.prisma.product.findMany({ where: { id: { in: ids }, deletedAt: null } });
}
```

Result shapes come from the generated client (`Prisma.OrderGetPayload`), never a hand-written copy. `findAllWithItems()` loads the items with the list, one query per relation level whatever the row count; `findProductsOfOrders()` loads the products of every order in one call to `ProductService`, since the product module owns them (see `arch-data-access-in-services`); `findSummaries()` selects only the fields the response needs (see `perf-optimize-database`); `findItemsByOrderIds()` loads the children of many orders with one `in` query and groups them in memory.

Reference: [Prisma relation queries](https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries) · [Prisma query optimization](https://www.prisma.io/docs/orm/v7/prisma-client/queries/advanced/query-optimization-performance)
