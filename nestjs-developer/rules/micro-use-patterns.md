---
title: Use Message and Event Patterns Correctly
impact: MEDIUM
impactDescription: Proper patterns ensure reliable microservice communication
tags: microservices, message-pattern, event-pattern, communication
---

## Use Message and Event Patterns Correctly

NestJS microservices support two communication patterns: request-response (`@MessagePattern`) and event-based (`@EventPattern`). Use a message pattern when the caller needs the answer to proceed, and an event pattern to announce something that happened. On a transport without a broker (TCP), an event emitted while the receiver is down is lost, so an announcement the receiver must record (the warehouse telling the shop an order is packed) is a command, sent again until it is answered, with a handler that answers a repeat as it answered the first.

- **Commands** (message patterns) are named `{object}.{verb}` in the imperative: `{ cmd: 'inventory.reserve' }`.
- **Events** follow the event naming of `arch-use-events`: `{object}.{action}` in the past tense (`order.created`).
- Event payloads carry ids and the fields consumers need, never whole entities.
- **The transport authenticates the connection, not the message.** A TCP or broker listener has no HTTP guards: the global `APP_GUARD`s skip every context that is not HTTP (`context.getType() !== 'http'`, see `api-real-time-updates`), which is safe only because the connection is authenticated instead. Between services over TCP, that is mutual TLS: the listener presents its certificate and requires one from the caller, signed by the services' own CA (`requestCert: true`, `rejectUnauthorized: true`), and the client presents its own. On a broker (Redis, RabbitMQ, NATS), each service connects over TLS with credentials of its own, allowed only the queues or subjects it uses. Never a credential inside the payload: a message is logged, retried and dead-lettered, and the secret goes everywhere it does. The certificates and broker credentials are settings, read from files or the environment like every secret (see `devops-use-config-module`). "Only the internal network reaches it" is a network policy to keep, not a check.
- **A critical step on another service is undone through the outbox.** The other service cannot join the transaction, so the undo is recorded before the call: an outbox event (`order_reservation.canceled`, see `arch-use-events`) written in its own short transaction, due only after the call's deadline plus a margin, which the transaction that completes the local write withdraws; a refusal or no answer dispatches it at once. The commit then decides whether the undo ever goes out, and a crash, a cut request or a lost commit between the call and the commit still releases what the other service holds. No answer in time is an unknown outcome, not a refusal: the undo goes out, and it may reach the other service before the late request does, so the contract says a cancel for an id the service has not seen yet is remembered and refuses that id when it comes.
- **Message handlers get their own pipes, filters and logging.** Global pipes, filters and interceptors registered on the HTTP app do not reach a hybrid app's message handlers: the controller declares its `ValidationPipe` and an exception filter that answers with the exception's status and message, and an interceptor logs each message with its pattern, outcome, duration and a correlation id the message carries or one made for it (see `devops-use-logging`).

**Incorrect (using wrong pattern for use case):**

```typescript
// Use @MessagePattern for fire-and-forget
@Controller()
export class NotificationController {
  @MessagePattern('user.created')
  async handleUserCreated(data: UserCreatedEvent) {
    // This WAITS for response, blocking the sender
    await this.emailService.sendWelcome(data.email);
    // If email fails, sender gets an error (coupling!)
  }
}

// Use @EventPattern expecting a response
@Controller()
export class OrderController {
  @EventPattern('inventory.check')
  async checkInventory(data: CheckInventoryDto) {
    const available = await this.inventory.check(data);
    return available; // This return value is IGNORED with @EventPattern!
  }
}

// Tight coupling in client
@Injectable()
export class UserService {
  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(await this._toEntity(dto));

    // Blocks until notification service responds
    await this.client.send('user.created', user).toPromise();
    // If notification service is down, user creation fails!

    return user;
  }
}
```

**Correct (use MessagePattern for request-response, EventPattern for fire-and-forget):**

A message pattern serves a caller that needs the answer to proceed: the handler's return value is sent back, and `OrderService` waits for it before it creates the order.

```typescript
// The inventory service's listener: only callers with a certificate from the services' CA connect
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
  options: {
    host: '0.0.0.0',
    port: inventorySettings.port,
    tlsOptions: {
      key: inventorySettings.tls.key,
      cert: inventorySettings.tls.cert,
      ca: inventorySettings.tls.ca,
      requestCert: true,
      rejectUnauthorized: true,
    },
  },
});

@Controller()
@UsePipes(new ValidationPipe(API_VALIDATION_PIPE_OPTIONS))
@UseFilters(ApiRpcExceptionFilter)
@UseInterceptors(ApiRpcLoggerInterceptor)
export class InventoryController {
  constructor(private readonly inventoryService: InventoryService) {}

  @MessagePattern({ cmd: 'inventory.check' })
  checkInventory(@Payload() data: CheckInventoryDto): Promise<InventoryResult> {
    return this.inventoryService.check(data.items);
  }
}

@Injectable()
export class OrderService {
  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const inventory = await firstValueFrom(
      this.inventoryClient.send<InventoryResult>({ cmd: 'inventory.check' }, { items: dto.items }),
    );

    if (!inventory.available) {
      throw new BadRequestException('Insufficient inventory', { errorCode: 'WAREHOUSE_OUT_OF_STOCK' });
    }

    return this.orderRepository.save(this.orderRepository.create({ ...dto, userId: authUser.id }));
  }
}
```

The listener accepts only connections that present a certificate from the services' CA, so no guard on the controller checks the caller, and the payload carries only the command's data. `inventoryClient` is registered with the shop's own certificate and the same CA in its `tlsOptions`. The controller declares what the HTTP app's globals do not give it: the `ValidationPipe` checks the payload against `CheckInventoryDto`, `ApiRpcExceptionFilter` answers with the exception's status and message, and `ApiRpcLoggerInterceptor` logs each message.

An event pattern is fire-and-forget, for notifications and side effects. The handler returns nothing, since nobody waits for it, and handles its own errors: an event has no caller to receive them (see `error-handle-async-errors`). `emit()` neither blocks nor waits, so the user is created whatever happens to the event.

```typescript
@Controller()
export class NotificationController {
  @EventPattern('user.created')
  async handleUserCreated(data: UserCreatedEvent): Promise<void> {
    try {
      await this.emailService.sendWelcome(data.email);
    } catch (error) {
      this.logger.error({ err: error }, `Welcome email failed for user ${data.userId}`);
      SentryService.reportError(error);
    }
  }
}

@Injectable()
export class UserService {
  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(await this._toEntity(dto));

    this.eventClient.emit('user.created', {
      userId: user.id,
      email: user.email,
      timestamp: new Date(),
    });

    return user;
  }
}
```

A critical step on another service combines both. Inventory is a separate service, so it cannot join the order's transaction, as an in-process module would (see `db-use-transactions`).

```typescript
@Injectable()
export class OrderPlacementService {
  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const orderId = randomUUID();
    // Recorded before the call, due after its deadline: a crash before the commit still releases the items
    const undo = await this.dataSource.transaction((manager) =>
      this.outboxService.add(manager, 'order_reservation.canceled', new OrderReservationCanceledEvent(orderId), {
        delayMs: ORDER_RESERVATION_TIMEOUT_MS + ORDER_RESERVATION_UNDO_MARGIN_MS,
      }),
    );
    const reserved = await firstValueFrom(
      this.inventoryClient
        .send({ cmd: 'inventory.reserve' }, { orderId, items: dto.items })
        .pipe(timeout(ORDER_RESERVATION_TIMEOUT_MS)),
    ).catch(() => null);
    if (!reserved?.success) {
      this.outboxService.dispatchNow([undo]);
    }
    if (!reserved) {
      throw new ServiceUnavailableException('The warehouse did not answer; try again', { errorCode: 'WAREHOUSE_UNAVAILABLE' });
    }
    if (!reserved.success) {
      throw new BadRequestException('Could not reserve inventory', { errorCode: 'WAREHOUSE_OUT_OF_STOCK' });
    }
    const order = await this.databaseTransactionService.retried(async (manager) => {
      if (!(await this.outboxService.withdraw(manager, undo))) {
        throw transactionConflictException();
      }
      return manager.save(manager.create(Order, { id: orderId, userId: authUser.id, status: OrderStatus.Pending }));
    });

    this.eventClient.emit('order.created', { orderId: order.id, userId: authUser.id, totalCents: order.totalCents });
    return order;
  }
}
```

No answer (`null`) is an unknown outcome, so the recorded undo goes out at once, as it does for a refusal. The commit decides the rest: the order and the withdrawn undo land together, or neither does. When the dispatcher took the undo first, the items are already released, and the transaction ends in a conflict. The `order.created` notification is not critical, so it goes out as an event.

Errors: an `RpcException` thrown by a message handler is received by the caller, while an event handler logs its error and does not throw, retrying where it can. The message handler reads through the service (see `arch-data-access-in-services`) and returns a response DTO: `ClassSerializerInterceptor` is one of the global interceptors that do not reach it, so an entity would go out with `passwordHash` (see `api-use-dto-serialization`).

```typescript
@MessagePattern({ cmd: 'user.find' })
async getUser(@Payload() data: FindUserDto): Promise<UserResponseDto> {
  const user = await this.userService.findOne(data.userId);
  if (!user) {
    throw new RpcException('User not found');
  }
  return plainToInstance(UserResponseDto, user, { excludeExtraneousValues: true });
}

@EventPattern('order.created')
async handleOrderCreated(data: OrderCreatedEvent): Promise<void> {
  try {
    await this._processOrder(data);
  } catch (error) {
    this.logger.error({ err: error }, `Failed to process order.created for order ${data.orderId}`);
    SentryService.reportError(error);
  }
}
```

Reference: [NestJS Microservices](https://docs.nestjs.com/microservices/basics)
