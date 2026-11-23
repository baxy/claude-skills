---
title: Single Responsibility for Services
impact: CRITICAL
impactDescription: Services stay small enough to read, test and change without touching unrelated features
tags: architecture, services, single-responsibility
---

## Single Responsibility for Services

Each service owns one domain concern. Avoid "god services" that handle several unrelated concerns: a name with "And" in it, a constructor with dependencies from many modules, or methods that share no state are the signs.

When a use case spans several concerns (create an order, charge it, notify the customer), the service that owns the use case orchestrates it: it calls the other modules' services, or emits an event for side effects that need not succeed together; a side effect that must eventually happen goes through the outbox, written in the use case's transaction (see `arch-use-events`). Controllers never orchestrate; they call one service method and return its result (see `api-controller-conventions`).

**Incorrect (god service, or orchestration in the controller):**

```typescript
@Injectable()
export class UserAndOrderService {
  async createUser(dto: CreateUserDto) { /* ... */ }
  async createOrder(userId: string, dto: CreateOrderDto) { /* ... */ }
  async calculateOrderStats(userId: string) { /* ... */ }
  async validatePayment(orderId: string) { /* ... */ }
}

@Controller('orders')
export class OrderController {
  @Post()
  async create(@AuthUser() authUser: JwtUser, @Body() dto: CreateOrderDto) {
    const order = await this.orderService.create(authUser, dto);
    await this.paymentService.charge(order);                    // Business flow in the controller
    await this.notificationService.sendOrderConfirmation(order);
    return order;
  }
}
```

**Correct (focused services; the owning service orchestrates):**

```typescript
// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    private readonly paymentService: PaymentService,
    private readonly outboxService: OutboxService,
    private readonly dataSource: DataSource,
  ) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const { order, event } = await this.dataSource.transaction(async (manager) => {
      const created = await manager.save(manager.create(Order, { ...dto, userId: authUser.id, status: OrderStatus.Pending }));
      const added = await this.outboxService.add(
        manager,
        'order.created',
        new OrderCreatedEvent(created.id, created.userId, created.totalCents),
      );
      return { order: created, event: added };
    });
    this.outboxService.dispatchNow([event]);
    await this.paymentService.authorize(order);
    return order;
  }
}

// order/services/order-stats.service.ts
@Injectable()
export class OrderStatsService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
  ) {}

  async calculateForUser(authUser: JwtUser): Promise<OrderStats> { /* ... */ }
}

// order/controllers/order.controller.ts
@Controller('orders')
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  @Post()
  create(@AuthUser() authUser: JwtUser, @Body() dto: CreateOrderDto): Promise<Order> {
    return this.orderService.create(authUser, dto);
  }
}
```

`OrderService` owns the "place an order" use case. It authorizes the payment after the save, so a failed authorization leaves the order pending (see `db-use-transactions`), and leaves the confirmation mail to the `order.created` event, written to the outbox in the order's transaction: the mail must eventually go out (see `arch-use-events`). `OrderStatsService` is a separate concern of the same module, and the controller makes one call and returns its result.

Reference: [NestJS Providers](https://docs.nestjs.com/providers)
