---
title: Use Constructor Injection, Never new or Property Injection
impact: CRITICAL
impactDescription: Every dependency is visible in one place, shared through the container, and replaceable in tests
tags: dependency-injection, constructor, testing
---

## Use Constructor Injection, Never new or Property Injection

Every dependency of a provider, controller, guard or interceptor arrives through its constructor as `private readonly`. Two things bypass this and are not allowed:

- **`new` for an injectable class** (`new MailService()`). The instance is outside the container: it is not shared, its own dependencies are not injected, its lifecycle hooks never run, and tests cannot replace it. Five uses of `new` are fine: `new Logger(Class.name)` (the standard Nest pattern; it delegates to the app logger), the global pipes, filter and interceptors that `ApiModule.setup()` registers with `app.useGlobal*()` (see `test-use-testing-module`), a per-route enhancer that takes constructor arguments, such as `@UseInterceptors(new ApiTimeoutInterceptor(10_000))` (see `api-use-interceptors`), a library object a module's options take as an instance, built in that module's `useFactory` from injected values, such as the throttler's storage (`storage: new ApiThrottlerStore(redis)`, see `security-rate-limiting`), and a provider built in a factory from an optional provider the factory injects. The notification module's map of channels (see `di-avoid-service-locator`) injects `{ token: SmsService, optional: true }` and adds `new NotificationSmsChannelAdapter(smsService)` only when the SMS module is there: the class exists only when its dependency does, that dependency is injected, and a test still replaces `SmsService`.
- **Property injection** (`@Inject()` on a field). The dependency is hidden from the constructor signature and unset if the class is constructed any other way.

An optional dependency is still a constructor parameter, marked `@Optional()`. Since NestJS 12 a subclass does not inherit the marker: give the subclass its own constructor that declares `@Optional()` again and passes the value to `super()`, or Nest throws `UnknownDependenciesException`.

Optional infrastructure (an SMS provider that only some deployments have) is a module imported with `ConditionalModule.registerWhen(SmsModule.forRootAsync({ ... }), isSmsApiConfigured)` from `@nestjs/config`. When the condition is false the module is not imported, so its service is simply missing for `@Optional()` or an optional `inject` entry. The condition reads the environment, so it lives in `src/config/` beside its namespace (`isSmsApiConfigured(env)` in `sms-api.config.ts`, see `devops-use-config-module`).

**Incorrect (manual instantiation and property injection):**

```typescript
@Injectable()
export class OrderService {
  private readonly mailService = new MailService(); // Outside the container: not shared, not mockable

  @Inject(PaymentService)
  private paymentService: PaymentService;           // Hidden dependency

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    // ...
  }
}
```

**Correct (everything through the constructor):**

```typescript
@Injectable()
export class OrderService {
  private readonly logger = new Logger(OrderService.name);

  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    private readonly mailService: MailService,
    private readonly paymentService: PaymentService,
    @Optional() private readonly analyticsService?: AnalyticsService,
  ) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const order = await this.orderRepository.save(this.orderRepository.create({ ...dto, userId: authUser.id }));
    await this.analyticsService?.trackPurchase(order.id, order.totalCents);
    return order;
  }
}
```

`new Logger` is one of the allowed uses of `new`. `PaymentService` and `AnalyticsService` are their modules' services, never a token or a client (see `arch-module-sharing`). The optional `AnalyticsService` comes from a module only some deployments import, and the service works without it.

Tests replace these dependencies in the testing module (see `test-use-testing-module`); nothing constructs the service by hand. A unit spec of a provider with no injected dependency is the one exception (see `test-organize-by-tier`): it calls `new`, and then the lifecycle hooks Nest would have called, such as `onModuleInit()`.

Reference: [NestJS Providers](https://docs.nestjs.com/providers) · [Optional providers](https://docs.nestjs.com/providers#optional-providers)
