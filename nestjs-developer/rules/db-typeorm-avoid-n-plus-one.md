---
title: Avoid N+1 Query Problems with TypeORM
impact: HIGH
impactDescription: Related rows load with the query that needs them, not one query per row
tags: database, n-plus-one, queries, performance, typeorm
---

## Avoid N+1 Query Problems with TypeORM

The TypeORM form of `db-avoid-n-plus-one`. Load related rows of the same module with the `relations` find option or a query builder join (`leftJoinAndSelect`), and add `select` when only some columns are needed; another module's rows come from one call to its service, whose query is `In(ids)`. A paged list loads them separately: page the parent rows alone (`skip`, `take`, `order` on their own columns), then load each relation for that page with one `In(ids)` query. With `relations` and `skip`/`take`, TypeORM first runs a `DISTINCT` query over the whole join to find the page's ids, which reads every matching row, and once `order` names a relation's column it pages the joined rows instead of the parents: pages come out short, repeat parents, and `totalCount` is wrong. A lazy relation (a `Promise<T>` property) runs a query on each access, so a list that serializes one runs N queries. `DB_LOGGING=true` turns on TypeORM's query logging locally (`db.logging`, see `devops-use-config-module`).

**Incorrect (lazy loading in loops causes N+1):**

```typescript
// Lazy loading in loops causes N+1
@Injectable()
export class OrderService {
  async getOrdersWithItems(userId: string): Promise<Order[]> {
    const orders = await this.orderRepository.find({ where: { userId } });
    // 1 query for orders

    for (const order of orders) {
      // N additional queries - one per order!
      order.items = await this.orderItemRepository.find({ where: { orderId: order.id } });
    }

    return orders;
  }
}

// Accessing lazy relations without loading
@Controller('users')
export class UserController {
  @Get()
  async findAll(): Promise<User[]> {
    const users = await this.userRepository.find();
    // If User.orders is lazy-loaded, serializing triggers N queries
    return users; // Each user.orders access = 1 query
  }
}
```

**Correct (use relations for eager loading):**

The `relations` find option loads the items in the same query, with a join. The products belong to the product module, so they load in one batch through its service:

```typescript
// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    private readonly productService: ProductService,
  ) {}

  async getOrdersWithItems(userId: string): Promise<Order[]> {
    return this.orderRepository.find({
      where: { userId },
      relations: { items: true },
    });
  }

  async getProductsOfOrders(orders: Order[]): Promise<Map<string, Product>> {
    const productIds = [...new Set(orders.flatMap((order) => order.items.map((item) => item.productId)))];
    const products = await this.productService.findByIds(productIds);
    return new Map(products.map((product) => [product.id, product]));
  }
}

// product/services/product.service.ts
findByIds(ids: string[]): Promise<Product[]> {
  return this.productRepository.findBy({ id: In(ids) });
}
```

The query builder suits more complex joins, such as a count per row or a condition on the join:

```typescript
// order/services/order.service.ts
@Injectable()
export class OrderService {
  async getOrdersWithItemCounts(userId: string): Promise<OrderWithItemCount[]> {
    return this.orderRepository
      .createQueryBuilder('order')
      .leftJoin('order.items', 'item')
      .select('order.id', 'id')
      .addSelect('order.status', 'status')
      .addSelect('COUNT(item.id)', 'itemCount')
      .where('order.userId = :userId', { userId })
      .groupBy('order.id')
      .getRawMany();
  }

  async getOrdersContainingSku(userId: string, sku: string): Promise<Order[]> {
    return this.orderRepository
      .createQueryBuilder('order')
      .innerJoinAndSelect('order.items', 'item', 'item.sku = :sku', { sku })
      .where('order.userId = :userId', { userId })
      .getMany();
  }
}
```

Find options with `select` load only the columns needed, a relation's included:

```typescript
async getOrderSummaries(userId: string): Promise<OrderSummary[]> {
  return this.orderRepository.find({
    where: { userId },
    relations: { items: true },
    select: {
      id: true,
      totalCents: true,
      status: true,
      items: {
        id: true,
        quantity: true,
        unitPriceCents: true,
      },
    },
  });
}
```

Reference: [TypeORM Relations](https://typeorm.io/relations)
