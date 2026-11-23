---
title: Avoid Circular Dependencies
impact: CRITICAL
impactDescription: Prevents undefined providers at startup and modules that cannot be understood or tested alone
tags: architecture, modules, dependencies
---

## Avoid Circular Dependencies

Circular dependencies occur when Module A imports Module B, and Module B imports Module A (directly or transitively). Nest fails at startup with an undefined dependency, or resolves the cycle only through `forwardRef()`, which hides a missing boundary. Break the cycle instead: extract what both modules need into a third module, or let one side react to an event from the other. When the cycle comes from an inbound controller (the warehouse module calls `WarehouseService.reserve()` for checkout and also serves the warehouse's calls into orders), move the controller to the module whose service it calls (`OrderInternalController` in `order/`), as a provider's callback lives with the module it serves; an event cannot replace a call whose answer the caller waits for. `forwardRef()` is a last resort, with a comment explaining why the cycle cannot be broken.

**Incorrect (circular module imports):**

```typescript
// user.module.ts
@Module({
  imports: [OrderModule], // Orders needs Users, Users needs Orders = circular
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}

// order.module.ts
@Module({
  imports: [UserModule], // Circular dependency!
  providers: [OrderService],
  exports: [OrderService],
})
export class OrderModule {}
```

**Correct (extract shared logic or use events):**

The first way out extracts the logic both modules need into a third module named for that concern, never a catch-all `SharedModule` (`src/shared/` holds plain reusable files, see `arch-feature-modules`):

```typescript
// pricing/pricing.module.ts
@Module({
  providers: [PricingService],
  exports: [PricingService],
})
export class PricingModule {}

// user.module.ts
@Module({
  imports: [PricingModule],
  providers: [UserService],
})
export class UserModule {}

// order.module.ts
@Module({
  imports: [PricingModule],
  providers: [OrderService],
})
export class OrderModule {}
```

The second uses events for decoupled communication (see `arch-use-events`): the order module reacts to the user module's event without importing `UserModule`.

```typescript
// user/services/user.service.ts
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(await this._toEntity(dto));
    this.eventEmitter.emit('user.created', new UserCreatedEvent(user.id, user.email));
    return user;
  }
}

// order/listeners/order-user-events.listener.ts
@Injectable()
export class OrderUserEventsListener {
  private readonly logger = new Logger(OrderUserEventsListener.name);

  constructor(private readonly orderService: OrderService) {}

  @OnEvent('user.created', { async: true })
  async onUserCreated(event: UserCreatedEvent): Promise<void> {
    try {
      await this.orderService.createWelcomeCart(event.userId);
    } catch (error) {
      this.logger.error({ err: error }, `Welcome cart failed for user ${event.userId}`);
      SentryService.reportError(error);
    }
  }
}
```

`_toEntity()` hashes the password (see `security-hash-passwords`). The listener catches and reports its own errors (see `error-handle-async-errors`).

Reference: [NestJS Circular Dependency](https://docs.nestjs.com/fundamentals/circular-dependency)
