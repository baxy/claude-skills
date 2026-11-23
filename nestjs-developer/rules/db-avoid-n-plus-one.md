---
title: Avoid N+1 Query Problems
impact: HIGH
impactDescription: N+1 queries are one of the most common performance killers
tags: database, n-plus-one, queries, performance
---

## Avoid N+1 Query Problems

N+1 queries occur when you fetch a list of rows, then make an additional query for each row to load related data. Load the related data in the same query (a join) or in one batched query for all the rows (`WHERE order_id IN (...)`), never with a query per row in a loop. A join stays inside one module (an order with its items); another module's rows load in one batched call to that module's service (one `productService.findByIds(ids)` for the products of every item), since that module owns them (see `arch-data-access-in-services`). A relation that loads itself on access causes the same N+1 when a response serializes it. This rule says what every ORM must do; the forms are in `db-typeorm-avoid-n-plus-one` for TypeORM and `db-prisma-avoid-n-plus-one` for Prisma.

To find one locally, turn on the ORM's query logging and look for the same query repeated with different ids.

**GraphQL field resolvers** run once per parent, so a field that queries per parent is an N+1 that no join fixes. A request-scoped DataLoader collects the parent ids of one request and loads them in one batched query (see `di-scope-awareness`):

```typescript
import DataLoader from 'dataloader';

@Injectable({ scope: Scope.REQUEST })
export class OrderLoader {
  constructor(private readonly orderService: OrderService) {}

  readonly byUserId = new DataLoader<string, Order[]>(async (userIds) => {
    const orders = await this.orderService.findByUserIds([...userIds]);

    const ordersByUser = new Map<string, Order[]>();
    for (const order of orders) {
      const userOrders = ordersByUser.get(order.userId) || [];
      userOrders.push(order);
      ordersByUser.set(order.userId, userOrders);
    }

    // DataLoader expects one result per key, in the order of the keys
    return userIds.map((id) => ordersByUser.get(id) || []);
  });
}
```

The loader reads the orders of every user in the batch with one query, then groups them by `userId`. The `User` resolver's `orders` field calls the loader, and DataLoader batches the calls of one request into that single query:

```typescript
@ResolveField()
async orders(@Parent() user: User): Promise<Order[]> {
  return this.orderLoader.byUserId.load(user.id);
}
```

Reference: [DataLoader](https://github.com/graphql/dataloader)
