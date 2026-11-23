---
title: Follow Service Conventions
impact: MEDIUM
impactDescription: Services read the same way everywhere and own all domain rules
tags: architecture, services, dependency-injection, conventions
---

## Follow Service Conventions

Services hold all business logic and domain validation. Shape validation (types, formats, required fields) belongs to DTOs and the global `ValidationPipe`; services enforce what only the domain knows: whether a resource exists, whether the caller owns it, whether a state transition is legal.

- **Naming**: the service is **singular**, named for its resource: `OrderService` in `order/services/order.service.ts`, injected as `private readonly orderService: OrderService`. A second service in the module starts with the module name too (`OrderItemService`, `OrderPricingService`). See `arch-resource-naming`.
- **Constructor injection order**: `@InjectRepository()` repositories first (own module's entities only, see `arch-data-access-in-services`), then other services and token-injected abstractions, then configuration (`ConfigService` or a namespace injected with `@Inject(xConfig.KEY)`), then `DataSource`. All injected dependencies are `private readonly`, without a `_` prefix (see `style-private-underscore-prefix`).
- **Method parameter order**: the authenticated user (`authUser: JwtUser`, see `security-auth-jwt`) first when the method acts on behalf of a caller, then resource identifiers, then the data DTO, then optional parameters. A method no caller scopes (an admin delete, a lookup another module makes by id) starts with the identifiers.
- **Private methods use the `_` prefix** (see `style-private-underscore-prefix`).
- **Constants live outside the class**, in `UPPER_SNAKE_CASE`, never as `static readonly` members: a value that sets behavior clients see is exported from the module's `constants/` with the prefix (`ORDER_CANCELABLE_STATUSES`, `ORDER_MAX_PENDING`), and an implementation detail is a `const` at the top of the service's file (see `arch-feature-modules`).
- **Exceptions**: `NotFoundException` for a missing resource, `BadRequestException` for a broken domain rule (including an illegal state transition), `ConflictException` for a clash with existing data such as a duplicate (see `error-throw-http-exceptions`).
- **Transactions**: `dataSource.transaction(manager => ...)` for writes that span several entities, through `DatabaseTransactionService.retried()` when the transaction locks rows other requests also lock (see `db-use-transactions`, `db-typeorm-use-transactions`).
- **Pagination** returns `PaginatedResult<T>` (`{ result, totalCount, limit, offset }`, see `perf-optimize-database`).

**Incorrect (mixed order, mutable deps, shape checks in the service):**

```typescript
@Injectable()
export class OrderService {
  constructor(
    private config: ConfigService,                                  // Not readonly, wrong position
    private dataSource: DataSource,
    private paymentService: PaymentService,
    @InjectRepository(Order) private orderRepository: Repository<Order>,
  ) {}

  static cancelable = ['pending', 'paid'];                         // Not UPPER_SNAKE_CASE

  async cancel(id: string, dto: CancelOrderDto, authUser: JwtUser) { // authUser last
    if (typeof dto.reason !== 'string' || dto.reason.length > 500) {  // Shape validation belongs in the DTO
      throw new BadRequestException('Invalid reason');
    }
    // ...
  }

  private checkStatus(order: Order) { /* ... */ }                  // Missing _ prefix
}
```

**Correct (consistent order, domain rules only):**

```typescript
// order/constants/order-limits.constant.ts
export const ORDER_CANCELABLE_STATUSES: readonly OrderStatus[] = [OrderStatus.Pending, OrderStatus.Paid];

// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    @InjectRepository(OrderItem)
    private readonly orderItemRepository: Repository<OrderItem>,
    private readonly databaseTransactionService: DatabaseTransactionService,
    private readonly inventoryService: InventoryService,
    private readonly paymentService: PaymentService,
    private readonly configService: ConfigService,
  ) {}

  async findById(authUser: JwtUser, id: string): Promise<Order> {
    const order = await this.orderRepository.findOne({ where: { id, userId: authUser.id } });
    if (!order) {
      throw new NotFoundException(`Order #${id} not found`);
    }
    return order;
  }

  async cancel(authUser: JwtUser, id: string, dto: CancelOrderDto): Promise<Order> {
    const order = await this.findById(authUser, id);
    this._assertCancelable(order);

    return this.databaseTransactionService.retried(async (manager) => {
      const { affected } = await manager.update(
        Order,
        { id: order.id, status: In(ORDER_CANCELABLE_STATUSES), deletedAt: IsNull() },
        { status: OrderStatus.Canceled, cancelReason: dto.reason },
      );
      if (!affected) {
        throw new BadRequestException(`Order #${id} changed status and can no longer be canceled`, {
          errorCode: 'ORDER_NOT_CANCELABLE',
        });
      }
      const orderItems = await manager.find(OrderItem, { where: { orderId: order.id }, order: { sku: 'ASC' } });
      await this.inventoryService.release(manager, orderItems);
      return manager.findOneByOrFail(Order, { id: order.id });
    });
  }

  private _assertCancelable(order: Order): void {
    if (!ORDER_CANCELABLE_STATUSES.includes(order.status)) {
      throw new BadRequestException(`An order in status ${order.status} cannot be canceled`, { errorCode: 'ORDER_NOT_CANCELABLE' });
    }
  }
}
```

The status in the update's criteria makes it match nothing when a request shipped the order meanwhile, and the 400 then says the status changed (see `db-use-transactions`). `update()` does not skip soft-deleted rows as `findOne()` does, so the criteria name `deletedAt` too. Inventory items belong to the inventory module, so `InventoryService` puts the units back, inside the order's transaction. Other orders lock the same inventory items, so the items are released in `sku` order and the transaction runs through the retry (see `db-use-transactions`). This service injects no `DataSource`; one that does puts it last.

### With Prisma

- **Constructor injection order**: `private readonly prisma: PrismaService` first, in place of the repositories, then other services and token-injected abstractions, then configuration. There is no `DataSource` to inject last (see `arch-data-access-in-services`).
- **Transactions**: `this.prisma.$transaction(async (tx) => ...)`, or `PrismaTransactionService`'s retries, with every query inside on `tx` (see `db-prisma-use-transactions`).

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly prismaTransactionService: PrismaTransactionService,
    private readonly inventoryService: InventoryService,
    private readonly paymentService: PaymentService,
    private readonly configService: ConfigService,
  ) {}

  async cancel(authUser: JwtUser, id: string, dto: CancelOrderDto): Promise<Order> {
    const order = await this.findById(authUser, id);
    this._assertCancelable(order);

    return this.prismaTransactionService.retried(async (tx) => {
      const { count } = await tx.order.updateMany({
        where: { id: order.id, status: { in: [...ORDER_CANCELABLE_STATUSES] }, deletedAt: null },
        data: { status: OrderStatus.Canceled, cancelReason: dto.reason ?? Prisma.skip },
      });
      if (count === 0) {
        throw new BadRequestException(`Order #${id} changed status and can no longer be canceled`, {
          errorCode: 'ORDER_NOT_CANCELABLE',
        });
      }
      const orderItems = await tx.orderItem.findMany({ where: { orderId: order.id }, orderBy: { sku: 'asc' } });
      await this.inventoryService.release(tx, orderItems);
      return tx.order.findFirstOrThrow({ where: { id: order.id } });
    });
  }
}
```

Reference: [NestJS Providers](https://docs.nestjs.com/providers)
