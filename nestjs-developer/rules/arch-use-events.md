---
title: Use Event-Driven Architecture for Decoupling
impact: MEDIUM-HIGH
impactDescription: Enables async processing and modularity
tags: architecture, events, decoupling
---

## Use Event-Driven Architecture for Decoupling

Use `@nestjs/event-emitter` for intra-service events and message brokers for inter-service communication. Events allow modules to react to changes without direct dependencies, improving modularity and enabling async processing.

**Event names** are lowercase and dot-separated: `{object}.{action}`, with the action in the past tense because the event reports something that already happened - `order.created`, `order.shipped`, `user.registered`. A multi-word object uses an underscore (`order_item.added`, `payment_method.removed`). The same convention applies to in-process events and to events sent through a message broker, so a name reads the same wherever it is consumed.

```typescript
this.eventEmitter.emit('order.created', event);
this.eventEmitter.emit('order_item.added', event);
```

Not these:

```typescript
this.eventEmitter.emit('OrderCreated', event);          // PascalCase, no separator
this.eventEmitter.emit('order_created', event);         // object and action not dot-separated
this.eventEmitter.emit('order.create', event);          // imperative, not past tense
this.eventEmitter.emit('order-item.added', event);       // hyphen in a multi-word object
```

**Incorrect (direct service coupling):**

```typescript
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    private readonly recommendationService: RecommendationService,
    private readonly mailService: MailService,
    private readonly analyticsService: AnalyticsService,
  ) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const order = await this.orderRepository.save(this.orderRepository.create({ ...dto, userId: authUser.id }));

    // OrderService knows every consumer; adding one means changing this method
    await this.recommendationService.recordPurchase(order.id);
    await this.mailService.sendConfirmation(order);
    await this.analyticsService.trackPurchase(order.id, order.totalCents);

    return order;
  }
}
```

**Correct (event-driven decoupling):**

```typescript
// order/events/order-created.event.ts
export class OrderCreatedEvent {
  constructor(
    public readonly orderId: string,
    public readonly userId: string,
    public readonly totalCents: number,
  ) {}
}

// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const order = await this.orderRepository.save(this.orderRepository.create({ ...dto, userId: authUser.id }));
    this.eventEmitter.emit('order.created', new OrderCreatedEvent(order.id, order.userId, order.totalCents));
    return order;
  }
}

// recommendation/listeners/recommendation-order-events.listener.ts
@Injectable()
export class RecommendationOrderEventsListener {
  private readonly logger = new Logger(RecommendationOrderEventsListener.name);

  constructor(private readonly recommendationService: RecommendationService) {}

  @OnEvent('order.created', { async: true })
  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    try {
      await this.recommendationService.recordPurchase(event.orderId);
    } catch (error) {
      this.logger.error({ err: error }, `Could not record purchase for order ${event.orderId}`);
      SentryService.reportError(error);
    }
  }
}
```

`OrderService` emits the event and knows no consumer. Each consumer listens in its own module, in a listener named for the source module (see `arch-resource-naming`), and catches its errors, since a listener's failure must not reach the emitter or crash the process (see `error-handle-async-errors`).

In-process events are lost if the process stops before a listener finishes, and an event emitted after the commit is lost if the process stops in between. A side effect that must eventually happen (an email, a webhook delivery, telling another service) therefore goes through the outbox:

- **The producer writes the event in its own transaction.** `OutboxService.add(manager, name, payload)` inserts it into `outbox_events`, so the event exists exactly when the change does; the producer does not emit it itself. After the commit, it asks the outbox to dispatch the new events at once. An undo that must go out unless a later step commits (releasing a reservation another service holds) is added with a delay (`{ delayMs }`), before the call; the later step's transaction withdraws it (`OutboxService.withdraw(manager, event)`, `false` when the dispatcher took it first), and a failed step sends it at once (`dispatchNow()`), see `micro-use-patterns`.
- **A scheduled job dispatches what is still due**, so every event is delivered at least once: each row is claimed, so two instances rarely take the same one; it keeps its attempts, the time of the next one with backoff, and a failed state reported to Sentry once the attempts run out (see `micro-use-queues`, "Work recorded in the change's own transaction"). Dispatched rows are deleted after a retention.
- **Dispatching emits the event in process** with `emitAsync()`, and marks it dispatched once every listener has succeeded. A listener of an outbox event throws on failure, so the event is dispatched again. It is declared with `@OnOutboxEvent(name)`, which is `@OnEvent(name, { async: true, promisify: true, suppressErrors: false })`: `@nestjs/event-emitter` logs and swallows a listener's error unless `suppressErrors` is false, and EventEmitter2 hands an async listener's promise to `emitAsync()` only with `promisify`; without it, `emitAsync()` resolves at once and the listener's failure becomes an unhandled rejection, which ends the process. Its work is idempotent by the event's id: it records its work under a unique key (one mail per kind and order, one delivery per webhook and event), and adds the job with that record's id. A listener whose work may be lost, such as a live update pushed to a browser, still catches its errors.
- **The outbox is a capability module** (`outbox/`, see `arch-resource-naming`), which every producer imports without a cycle, and consumers keep listening by event name: adding a consumer still changes no producer.
- **Releases run side by side during a deploy, and after a rollback.** An instance of the previous release dispatches what is due like any other, and `emitAsync()` resolves for an event no listener takes, so an older instance would mark a newer event dispatched with no consumer: its work is lost during the rollout, and after a rollback for every event of that name still due. The dispatcher claims only the event names this release has listeners for, collected at startup from the `@OnOutboxEvent()` handlers (`name = ANY(...)` in the claim), and leaves the rest due for an instance that knows them.
- **A side effect that moves to another consumer keeps one duplicate key across the release that moves it.** While both releases dispatch, one event can reach the old consumer on an old instance and the new consumer on a new one. Each records its work under its own key (a mail per kind and order, a delivery per channel and order), the two never meet, and the customer gets the mail twice. For that release, the new consumer also records the old consumer's key, inserting it only if it is not there (`ON CONFLICT DO NOTHING`) and acting only when it inserted it, so whichever release comes second does nothing.

Work that must succeed or fail together with the write, such as reserving stock or counting the units sold, is not a side effect at all: the owning service calls it inside its transaction (see `db-use-transactions`). Listeners also run in no guaranteed order, each in its own transaction, so a counter kept by listeners of two events (`order.paid` adds, `order.canceled` subtracts) drifts when the events of one order race.

Reference: [NestJS Events](https://docs.nestjs.com/techniques/events)
