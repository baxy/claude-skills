# NestJS Anti-Patterns

Rule IDs in brackets refer to `../nestjs-developer/rules/<rule-id>.md`, relative to the `nestjs-reviewer` skill folder.

## Contents
- Controller: fat controllers, direct repository access, a hand-built reply for a status that varies, version lists on unchanged routes, one module's DTO in the api module
- Service: god service, tight coupling to infrastructure, a custom repository class, a must-happen side effect emitted after the commit
- Dependency injection: manual instantiation and property injection, circular dependencies, a service locator, per-request state in a singleton, a string token or a double chosen by `NODE_ENV`, a fake that breaks the contract
- Module: monolithic AppModule, over-exporting, a module-definition file and interfaces for data, side effects in `utils/`, services without the module name, clients and adapters named for the module, module code in `shared/`, a command line outside the `cli` module
- Data: multi-entity write without a transaction, check-then-write race, saving a loaded entity to update it, a retry loop of its own, unsafe migrations, unescaped search wildcards; with Prisma: `this.prisma` inside a transaction, an `undefined` value in a query, a query without `deletedAt: null`, Prisma data at the API boundary, a page and its count in a sequential `$transaction`
- Error handling: swallowing errors, unhandled errors outside a request, exposing internal errors, detecting a duplicate by lookup or by message, losing validation details, losing the error in the log line, reporting to Sentry with the SDK's defaults
- Testing: mocking the service under test, interaction-only tests, allow-path-only guard tests, bad tokens that change two things, an override of an undeclared provider, a spec that reads the developer's `.env`, a queue mock that fails at once, a database tier on the API's database, wrong tier, no isolation
- Configuration: hardcoded configuration, a config module of our own, an environment name with a default, an optional number without a check, an unchecked database or Redis connection, a comment after a value in `.env.example`

## Controller Anti-Patterns

### Fat Controllers [api-controller-conventions]
All business logic and domain validation live in services: a controller binds and validates input, calls one service (its module's, or the user module's for `GET /auth/me`) and returns the result. Logic in a controller is out of reach of the service's other callers, and decides error statuses away from the rule it enforces.

```typescript
// ❌ Anti-pattern: user/controllers/user.controller.ts
@Public()
@Post()
async create(@Body() dto: CreateUserDto) {
  // A lookup first: two sign-ups with one email both pass it
  const existing = await this.userRepository.findOne({ where: { email: dto.email } });
  if (existing) throw new ConflictException('Email already registered');

  const passwordHash = await argon2.hash(dto.password);
  const user = await this.userRepository.save({ ...dto, passwordHash }); // A plain object: @Exclude() is skipped

  this.mailService.sendWelcome(user.email); // Not awaited, no catch: an unhandled rejection
  return { data: user }; // An ad-hoc wrapper, with the password hash in it
}

// ✅ Fix: the controller delegates and returns the service's result
@Public()
@Post()
create(@Body() dto: CreateUserDto): Promise<User> {
  return this.userService.create(dto);
}
```

**Fix**: `UserService.create()` owns the logic: it hashes through `UserPasswordHasherService`, builds the row with `create()`, lets the unique index decide the 409 (see "Detecting a Duplicate by Lookup or by Message"), and queues the welcome mail or sends it with a `.catch()` that logs and reports. The entity is returned as it is, its `passwordHash` hidden by `@Exclude()`, and the envelope comes from `ApiResponseInterceptor`.

### Direct Repository Access in Controllers [api-controller-conventions, arch-data-access-in-services]
A controller never injects a repository or the `DataSource`: the queries belong to its module's service.

```typescript
// ❌ Anti-pattern
@Controller('products')
export class ProductController {
  constructor(
    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>,
  ) {}
}

// ✅ Fix: inject the service instead
@Controller('products')
export class ProductController {
  constructor(private readonly productService: ProductService) {}
}
```

### A Hand-Built Reply for a Status That Varies [api-platform-agnostic, api-controller-conventions]
A creating `PUT` answers 201 when it created and 200 when it replaced. Replying through `@Res()` ties the handler to one adapter, and the reply skips every interceptor: no envelope, and no serializer to apply `@Exclude()`.

```typescript
// ❌ Anti-pattern: Express-only, and the interceptors never see the value
@Put(':id/shipping-address')
async replaceShippingAddress(
  @AuthUser() authUser: JwtUser,
  @Param() params: OrderIdParamDto,
  @Body() dto: OrderAddressDto,
  @Res() res: Response,
) {
  const { address, created } = await this.orderService.replaceShippingAddress(authUser, params.id, dto);
  res.status(created ? 201 : 200).json(address);
}

// ✅ Fix: the status through the adapter, the value returned
@Put(':id/shipping-address')
async replaceShippingAddress(
  @AuthUser() authUser: JwtUser,
  @Param() params: OrderIdParamDto,
  @Body() dto: OrderAddressDto,
  @Res({ passthrough: true }) response: unknown,
): Promise<OrderShippingAddress> {
  const { address, created } = await this.orderService.replaceShippingAddress(authUser, params.id, dto);
  this.httpAdapterHost.httpAdapter.status(response, created ? HttpStatus.CREATED : HttpStatus.OK);
  return address;
}
```

The controller injects `HttpAdapterHost`. A status that never varies is `@HttpCode()`, and a file is a `StreamableFile`.

### Version Lists on Unchanged Routes [api-versioning]
A `@Version` on a handler or a controller replaces `defaultVersion` instead of adding to it. Listing versions on unchanged routes drops the old unversioned path, and every controller without a list answers 404 under a version the default does not hold.

```typescript
// ❌ Anti-pattern: an API that started unversioned ships v2
// api/api.module.ts, in setup()
app.enableVersioning({ type: VersioningType.URI, defaultVersion: [VERSION_NEUTRAL, '1'] }); // v2 left out

@Controller('users')
export class UserController {
  @Version(['1', '2']) // Unchanged route: /users stops answering
  @HttpCode(HttpStatus.NO_CONTENT)
  @Delete(':id')
  delete(@Param() params: UserIdParamDto): Promise<void> { /* ... */ }
}

// ✅ Fix: the new version joins the default, and only the changed routes are pinned
app.enableVersioning({ type: VersioningType.URI, defaultVersion: [VERSION_NEUTRAL, '1', '2'] });

@Controller('users')
export class UserController {
  @Version([VERSION_NEUTRAL, '1']) // Changed in v2: UserV2Controller serves /v2
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> { /* ... */ }

  @Roles(Role.Admin)
  @HttpCode(HttpStatus.NO_CONTENT)
  @Delete(':id')
  delete(@Param() params: UserIdParamDto): Promise<void> { /* ... */ }
}

@Controller({ path: 'users', version: '2' })
export class UserV2Controller { /* the changed routes only */ }
```

An API versioned from its first route uses `['1', '2']` and pins a changed handler to `'1'`.

### One Module's DTO in the API Module [api-dto-naming, arch-shared-folder]
The api module holds only what works the same for every module (`ApiPaginationQueryDto`, `@IsStorableDate()`), and `shared/` holds no DTOs. A shape of one module's data stays in that module when a second module starts to nest it.

```typescript
// ❌ Anti-pattern: the order's address moved to the api module once shipments nested it
// api/dtos/api-address.dto.ts
export class ApiAddressDto { /* ... */ }

// ✅ Fix: it stays in the module whose data it describes, and other modules import it from there
// order/dtos/order-address.dto.ts
export class OrderAddressDto { /* ... */ }

// shipment/dtos/create-shipment.dto.ts
import { OrderAddressDto } from '@/order/dtos/order-address.dto';
```

## Service Anti-Patterns

### God Service [arch-single-responsibility]
A service that handles several unrelated concerns: a name with "And" in it, a constructor with dependencies from many modules, methods that share no state. Orchestration in the controller is the same problem from the other side.

```typescript
// ❌ Anti-pattern: users, orders, stats and payments in one service
@Injectable()
export class UserAndOrderService {
  async createUser(dto: CreateUserDto) { /* ... */ }
  async createOrder(userId: string, dto: CreateOrderDto) { /* ... */ }
  async calculateOrderStats(userId: string) { /* ... */ }
  async validatePayment(orderId: string) { /* ... */ }
}

// ❌ Anti-pattern: the controller runs the use case
@Post()
async create(@AuthUser() authUser: JwtUser, @Body() dto: CreateOrderDto) {
  const order = await this.orderService.create(authUser, dto);
  await this.paymentService.charge(order);
  await this.notificationService.sendOrderConfirmation(order);
  return order;
}
```

**Fix**: one concern per service, each in its module (`UserService`, `OrderService`, `OrderStatsService`, `PaymentService`). The service that owns the use case orchestrates it: `OrderService.create()` calls `PaymentService` and leaves the confirmation mail to the `order.created` event, written to the outbox in the order's transaction, since the mail must eventually go out (`arch-use-events`). The controller makes one call and returns its result.

### Tight Coupling to Infrastructure [di-use-interfaces-tokens, arch-feature-modules]
A service that builds and calls an outside system's client itself cannot be tested without that system, and the next provider means rewriting the service.

```typescript
// ❌ Anti-pattern: notification/services/notification.service.ts calls the provider's SDK itself
@Injectable()
export class NotificationService {
  private readonly ses = new SESv2Client({ region: process.env.AWS_REGION });

  async sendOrderConfirmation(email: string, orderId: string): Promise<void> {
    await this.ses.send(new SendEmailCommand({ /* ... */ }));
  }
}

// ✅ Fix: the module's interface behind its token, one adapter per system on top of that system's client
// notification/interfaces/notification-email-adapter.interface.ts
export interface NotificationEmailAdapter {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

// notification/constants/notification-email-adapter.constant.ts
export const NOTIFICATION_EMAIL_ADAPTER = Symbol('NOTIFICATION_EMAIL_ADAPTER');

// notification/clients/aws-ses.client.ts
@Injectable()
export class AwsSesClient {
  constructor(private readonly sesClient: SESv2Client) {}

  async send(input: SendEmailCommandInput): Promise<void> { /* ... */ }
}

// notification/adapters/aws-ses.adapter.ts
@Injectable()
export class AwsSesAdapter implements NotificationEmailAdapter {
  constructor(private readonly awsSesClient: AwsSesClient) {}

  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    await this.awsSesClient.send({ /* ... */ });
  }
}

// notification/notification.module.ts
@Module({
  providers: [
    AwsSesClient,
    AwsSesAdapter,
    { provide: NOTIFICATION_EMAIL_ADAPTER, useExisting: AwsSesAdapter },
    NotificationService,
  ],
  exports: [NotificationService],
})
export class NotificationModule {}

// notification/services/notification.service.ts
@Injectable()
export class NotificationService {
  constructor(
    @Inject(NOTIFICATION_EMAIL_ADAPTER) private readonly notificationEmailAdapter: NotificationEmailAdapter,
  ) {}

  async sendOrderConfirmation(email: string, orderId: string): Promise<void> {
    await this.notificationEmailAdapter.sendEmail(email, 'Order Confirmed', `Your order ${orderId} has been confirmed.`);
  }
}
```

The SDK's own client (`SESv2Client`) is a provider of the module, built in a factory from its config namespace (`micro-use-http-clients`). `AwsSesClient` owns the transport; `AwsSesAdapter` builds the requests from the module's values and checks the answers. Other modules inject `NotificationService`, never the token, the adapter or the client (`arch-module-sharing`), and a test replaces the binding with `FakeNotificationEmailAdapter` through `overrideProvider(NOTIFICATION_EMAIL_ADAPTER)`.

### A Custom Repository Class [arch-data-access-in-services]
A class between the service and TypeORM splits each feature across two layers, and in the e2e tier its mock replaces the whole query, so a dropped ownership filter passes every test.

```typescript
// ❌ Anti-pattern: order/repositories/order.repository.ts, a layer the service only forwards to
@Injectable()
export class OrderRepository {
  constructor(@InjectRepository(Order) private readonly repository: Repository<Order>) {}

  findPageForUser(userId: string, query: FindAllOrdersQueryDto): Promise<[Order[], number]> {
    return this.repository.findAndCount({ where: { userId }, skip: query.offset, take: query.limit });
  }
}

@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    @InjectRepository(User) private readonly userRepository: Repository<User>, // another module's entity
  ) {}
}

// ✅ Fix: the service injects TypeORM's repository for its own entity and keeps its queries
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    private readonly userService: UserService,
  ) {}

  async findAll(authUser: JwtUser, query: FindAllOrdersQueryDto): Promise<PaginatedResult<Order>> {
    const [result, totalCount] = await this.orderRepository.findAndCount({
      where: { userId: authUser.id },
      order: { createdAt: 'DESC', id: 'DESC' },
      skip: query.offset,
      take: query.limit,
    });
    return { result, totalCount, limit: query.limit, offset: query.offset };
  }
}
```

Users are read through `UserService`. The e2e test mocks `getRepositoryToken(Order)`, so the `userId` filter the service builds is checked. With Prisma the same holds: no class wrapping `this.prisma.order`, no `this.prisma.user` in the order module, no `PrismaClient` created by hand.

### A Must-Happen Side Effect Emitted After the Commit [arch-use-events]
An in-process event emitted after the commit is lost when the process stops in between, and `@nestjs/event-emitter` logs and drops a listener's error, so the work is never tried again.

```typescript
// ❌ Anti-pattern: the cancellation mail depends on the process surviving past the commit
const order = await this.dataSource.transaction(async (manager) => { /* the conditional update */ });
this.eventEmitter.emit('order.canceled', new OrderCanceledEvent(order.id));

// mail/listeners/mail-order-events.listener.ts
@OnEvent('order.canceled', { async: true })
async onOrderCanceled(event: OrderCanceledEvent): Promise<void> {
  await this.mailService.sendOrderCanceled(event.orderId);
}

// ✅ Fix: the event written in the change's transaction, dispatched after the commit
const { order, event } = await this.dataSource.transaction(async (manager) => {
  /* the conditional update */
  const event = await this.outboxService.add(manager, 'order.canceled', new OrderCanceledEvent(id));
  return { order: await manager.findOneByOrFail(Order, { id }), event };
});
this.outboxService.dispatchNow([event]);

// mail/listeners/mail-order-events.listener.ts
@OnOutboxEvent('order.canceled')
async onOrderCanceled(event: OrderCanceledEvent): Promise<void> {
  await this.mailService.sendOrderCanceled(event.orderId);
}
```

`@OnOutboxEvent()` is `@OnEvent(name, { async: true, promisify: true, suppressErrors: false })`: the listener throws on failure, so the event is dispatched again, and it records its work under a unique key (one mail per kind and order), so a second dispatch sends nothing more. A scheduled job dispatches what is still due, claiming only the event names this release has listeners for. A listener whose work may be lost (a live update) keeps a plain `@OnEvent` and catches its own errors.

## Dependency Injection Anti-Patterns

### Manual Instantiation and Property Injection [di-prefer-constructor-injection]
`new` for an injectable class bypasses the container: the instance is not shared, its own dependencies and lifecycle hooks are skipped, and tests cannot replace it. `@Inject()` on a field hides the dependency from the constructor.

```typescript
// ❌ Anti-pattern
@Injectable()
export class OrderService {
  private readonly mailService = new MailService(); // outside the container

  @Inject(PaymentService)
  private paymentService: PaymentService; // hidden from the constructor
}

// ✅ Fix
@Injectable()
export class OrderService {
  private readonly logger = new Logger(OrderService.name);

  constructor(
    private readonly mailService: MailService,
    private readonly paymentService: PaymentService,
  ) {}
}
```

`new` is fine in five places, none of them a finding: `new Logger(Class.name)`, the global pipes, filter and interceptors `ApiModule.setup()` registers, a per-route enhancer with constructor arguments (`new ApiTimeoutInterceptor(...)`), a library object a module's options take as an instance, built in its `useFactory` from injected values (`new ApiThrottlerStore(redis)`), and a provider built in a factory from an optional provider the factory injects. An optional dependency is a constructor parameter with `@Optional()`, which a subclass repeats in its own constructor since NestJS 12.

### Circular Dependencies [arch-avoid-circular-deps]
Two modules that import each other fail at startup with an undefined dependency, or load only through `forwardRef()`, which hides a missing boundary.

```typescript
// ❌ Anti-pattern: each module imports the other
@Module({ imports: [OrderModule], providers: [UserService], exports: [UserService] })
export class UserModule {}

@Module({ imports: [UserModule], providers: [OrderService], exports: [OrderService] })
export class OrderModule {}

// ✅ Fix: the order module reacts to the user module's event without importing UserModule
// user/services/user.service.ts
async create(dto: CreateUserDto): Promise<User> {
  const user = await this.userRepository.save(await this._toEntity(dto));
  this.eventEmitter.emit('user.created', new UserCreatedEvent(user.id, user.email));
  return user;
}

// order/listeners/order-user-events.listener.ts
@OnEvent('user.created', { async: true })
async onUserCreated(event: UserCreatedEvent): Promise<void> {
  try {
    await this.orderService.createWelcomeCart(event.userId);
  } catch (error) {
    this.logger.error({ err: error }, `Welcome cart failed for user ${event.userId}`);
    SentryService.reportError(error);
  }
}

// ⚠️ Last resort only
@Injectable()
export class OrderService {
  constructor(
    // forwardRef: <why this cycle cannot be broken>
    @Inject(forwardRef(() => UserService))
    private readonly userService: UserService,
  ) {}
}
```

The other ways out: extract what both modules need into a third module named for that concern (`PricingModule`), never a catch-all `SharedModule`; and when an inbound controller causes the cycle (the warehouse module serves the warehouse's calls into orders while orders call `WarehouseService`), move that controller to the module whose service it calls (`OrderInternalController` in `order/`), since an event cannot replace a call whose answer the caller waits for.

### Service Locator [di-avoid-service-locator]
A dependency resolved at runtime disappears from the constructor, so neither the reader nor the testing module sees it.

```typescript
// ❌ Anti-pattern
@Injectable()
export class OrderService {
  constructor(private readonly moduleRef: ModuleRef) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const paymentService = this.moduleRef.get(PaymentService, { strict: false });
    // ...
  }
}

// ✅ Fix
@Injectable()
export class OrderService {
  constructor(private readonly paymentService: PaymentService) {}
}
```

`ModuleRef` stays for two cases, by a token or a class, never an interface: `moduleRef.resolve(token, contextId)` for a request-scoped or transient provider, and `moduleRef.get()` on a lazily loaded module. An implementation chosen per call (a channel kind) comes from a map of every implementation, built in a factory provider of the module from injected providers, behind a token of its own; a missing one then fails at startup, not on the first call that asks for it.

### Per-Request State in a Singleton [di-scope-awareness]
Providers are singletons: a field set per request is shared by every concurrent request, and `Scope.REQUEST` makes every consumer request-scoped too.

```typescript
// ❌ Anti-pattern
@Injectable()
export class RequestContextService {
  private userId: string;

  setUser(userId: string): void {
    this.userId = userId; // request B overwrites request A's caller
  }
}

@Injectable({ scope: Scope.REQUEST }) // every consumer becomes request-scoped
export class AuditService {
  constructor(@Inject(REQUEST) private readonly request: ApiRequest) {}
}

// ✅ Fix: the caller as a parameter, cross-cutting values from CLS
@Injectable()
export class AuditService {
  constructor(
    @InjectRepository(AuditEntry)
    private readonly auditEntryRepository: Repository<AuditEntry>,
    private readonly clsService: ClsService,
  ) {}

  async record(authUser: JwtUser, action: string): Promise<void> {
    await this.auditEntryRepository.save({ userId: authUser.id, action, requestId: this.clsService.getId() });
  }
}
```

`Scope.REQUEST` is only for a provider that wraps a per-request resource and cannot work otherwise, such as a GraphQL DataLoader.

### A String Token, or a Double Chosen by `NODE_ENV` [di-use-interfaces-tokens]
An interface does not exist at runtime, a string token can collide or be mistyped, and a double picked by `NODE_ENV` ships test code in production.

```typescript
// ❌ Anti-pattern
@Injectable()
export class PaymentService {
  constructor(private readonly paymentAdapter: PaymentAdapter) {} // an interface: nothing to inject at runtime
}

@Module({
  providers: [
    {
      provide: 'PAYMENT_ADAPTER',
      useClass: process.env.NODE_ENV === 'test' ? FakePaymentAdapter : StripeAdapter,
    },
  ],
})
export class PaymentModule {}

// ✅ Fix: a Symbol token in constants/, the binding in the owning module, the double swapped in by the test
// payment/constants/payment-adapter.constant.ts
export const PAYMENT_ADAPTER = Symbol('PAYMENT_ADAPTER');

// payment/payment.module.ts
@Module({
  providers: [StripeClient, StripeAdapter, { provide: PAYMENT_ADAPTER, useExisting: StripeAdapter }, PaymentService],
  exports: [PaymentService],
})
export class PaymentModule {}

// payment/services/payment.service.ts
@Injectable()
export class PaymentService {
  constructor(@Inject(PAYMENT_ADAPTER) private readonly paymentAdapter: PaymentAdapter) {}
}

// In the spec
const moduleRef = await Test.createTestingModule({ imports: [PaymentModule] })
  .overrideProvider(PAYMENT_ADAPTER)
  .useValue(new FakePaymentAdapter())
  .compile();
```

An abstract class is the other option: it is its own token, and consumers need no `@Inject()`. When the implementation depends on the deployment, it is chosen from config, never from `NODE_ENV`; `FakePaymentAdapter` lives in `test/helpers/fake-payment-adapter.ts`.

### A Fake That Breaks the Contract [di-liskov-substitution]
A double kinder than the implementation it stands for passes tests that production then fails, and an implementation that reports a refusal like any other failure makes the caller treat it as unknown and send it again forever.

```typescript
// ❌ Anti-pattern: test/helpers/fake-payment-adapter.ts
export class FakePaymentAdapter implements PaymentAdapter {
  async charge(amountCents: number, currency: string): Promise<PaymentResult> {
    return { success: true } as PaymentResult; // no transactionId, and no currency check the real adapter makes
  }
}

// ✅ Fix: the fake checks and answers as the real adapter does, and runs the shared contract suite
export class FakePaymentAdapter implements PaymentAdapter {
  private sequence = 0;

  async charge(amountCents: number, currency: string): Promise<PaymentResult> {
    if (!PAYMENT_SUPPORTED_CURRENCIES.includes(currency)) {
      throw new PaymentInvalidCurrencyException(`Unsupported currency: ${currency}`);
    }
    if (amountCents === 99_999) {
      throw new PaymentDeclinedException('Card declined (test scenario)');
    }
    if (amountCents === 99_998) {
      throw new PaymentRefusedException('Request refused (test scenario)');
    }
    if (amountCents === 99_997) {
      throw new ServiceUnavailableException('No answer from the provider (test scenario)');
    }
    return { success: true, transactionId: `fake_${++this.sequence}`, amountCents };
  }

  async refund(transactionId: string, amountCents?: number): Promise<PaymentRefundResult> { /* ... */ }
}

// test/unit/payment/fake-payment-adapter.spec.ts
describe('FakePaymentAdapter', () => {
  testPaymentAdapterContract(() => new FakePaymentAdapter());
});
```

The contract names every outcome its caller acts on. For a call that changes something in an outside system (a charge, a transfer, a message sent), those are done, rejected for this input (a declined card, for a payment), refused and unknown, and every implementation, fakes included, reports each one the same way. A property only some implementations have (delivered once when sent twice with one key) is declared by each, and the contract suite tests what is declared; a fake declares what the implementation it stands for declares. `StripeAdapter` has an injected client, `StripeClient`, which alone calls the SDK, so the payment module's e2e suite covers it with the SDK mocked by its class (`Stripe`) instead of constructing it.

## Module Anti-Patterns

### Monolithic AppModule [arch-feature-modules]
Every controller, service and entity in `AppModule` leaves no module boundary to read, test or change on its own.

```typescript
// ❌ Anti-pattern
@Module({
  imports: [TypeOrmModule.forFeature([User, Order, Product, Shipment, Report])],
  controllers: [UserController, OrderController, ProductController, ShipmentController, ReportController],
  providers: [UserService, OrderService, ProductService, ShipmentService, ReportService],
})
export class AppModule {}

// ✅ Fix: one module per feature, each owning its controllers, services and entities
// user/user.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UserController, UserAdminController],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}

// app.module.ts
@Module({
  imports: [ConfigModule.forRoot({ /* ... */ }), DatabaseModule, UserModule, OrderModule],
})
export class AppModule {}
```

Inside each module every file sits in a type folder, and only `{module}.module.ts` sits at its root; `src/` holds only `main.ts` and `app.module.ts`.

### Over-Exporting [arch-module-sharing]
A module that exports its token, its adapter, its client or a third-party module it configured lets every consumer depend on its insides and read its config.

```typescript
// ❌ Anti-pattern: the insides exported with the service
@Module({
  providers: [StripeClient, StripeAdapter, { provide: PAYMENT_ADAPTER, useExisting: StripeAdapter }, PaymentService],
  exports: [PaymentService, PAYMENT_ADAPTER, StripeAdapter, StripeClient],
})
export class PaymentModule {}

// ❌ Anti-pattern: a configured third-party module re-exported
@Module({
  imports: [UserModule, PassportModule, JwtModule.registerAsync({ /* ... */ })],
  providers: [AuthJwtStrategy, AuthSessionService, AuthTokenService],
  exports: [AuthSessionService, JwtModule],
})
export class AuthModule {}

// ✅ Fix: the module's public face is its services
@Module({
  providers: [StripeClient, StripeAdapter, { provide: PAYMENT_ADAPTER, useExisting: StripeAdapter }, PaymentService],
  exports: [PaymentService],
})
export class PaymentModule {}

@Module({
  imports: [UserModule, PassportModule, JwtModule.registerAsync({ /* ... */ })],
  providers: [AuthJwtStrategy, AuthSessionService, AuthTokenService],
  exports: [AuthSessionService, AuthTokenService],
})
export class AuthModule {}
```

A gateway in another module checks tokens through `AuthTokenService`. Two exports are not findings: the app's own configurable module, re-exported by the one module that configures it, and a connection a library takes raw (`REDIS_CLIENT` for the cache and rate-limit stores), which app code never injects.

### A Module-Definition File and Interfaces for Data [arch-feature-modules, arch-use-dynamic-modules]
The NestJS docs put a configurable module's builder in a `{module}.module-definition.ts` of its own, a second file at the module root, to avoid a circular import; a token in `constants/` avoids it too. Data is a `type`: a second `interface` of the same name merges into the first without a word.

```typescript
// ❌ Anti-pattern: mail/mail.module-definition.ts
export interface MailModuleOptions {
  apiKey: string;
  from: string;
}

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN: MAIL_OPTIONS } =
  new ConfigurableModuleBuilder<MailModuleOptions>().setClassMethodName('forRoot').build();

// ✅ Fix: the options type in types/, the token in constants/, the builder in the module file
// mail/types/mail-module-options.type.ts
export type MailModuleOptions = {
  apiKey: string;
  from: string;
};

// mail/constants/mail-options.constant.ts
export const MAIL_OPTIONS = Symbol('MAIL_OPTIONS');

// mail/mail.module.ts
const { ConfigurableModuleClass } = new ConfigurableModuleBuilder<MailModuleOptions>({
  optionsInjectionToken: MAIL_OPTIONS,
})
  .setClassMethodName('forRoot')
  .build();

@Module({
  providers: [MailService],
  exports: [MailService],
})
export class MailModule extends ConfigurableModuleClass {}
```

### Side Effects in `utils/` [arch-feature-modules]
`utils/` holds pure functions only. A helper there that calls an SDK or keeps state is service code in a folder that says otherwise, and nothing can replace it in a test.

```typescript
// ❌ Anti-pattern: payment/utils/ calls the provider and keeps a client
let client: Stripe | undefined;

export async function chargeCard(apiKey: string, amountCents: number): Promise<string> {
  client ??= new Stripe(apiKey);
  return (await client.paymentIntents.create({ amount: amountCents, currency: 'usd' })).id;
}

// ✅ Fix: the call in an injectable client named for the system, utils/ keeps the pure parts
// payment/clients/stripe.client.ts
@Injectable()
export class StripeClient {
  constructor(private readonly stripe: Stripe) {} // the SDK, a provider of the module by its class

  async createPaymentIntent(amountCents: number): Promise<string> { /* ... */ }
}

// payment/utils/to-amount-cents.ts
export const toAmountCents = (amount: number): number => Math.round(amount * 100);
```

### Services Without the Module Name [arch-resource-naming]
The services of a capability module follow the resource rule: they start with the module name, end in `Service` and sit in `services/{name}.service.ts`.

```typescript
// ❌ Anti-pattern: health/services/
export class ShutdownDrain {}         // shutdown-drain.ts
export class RedisHealthIndicator {}  // redis-health.indicator.ts

// ✅ Fix
export class HealthShutdownService {} // health-shutdown.service.ts
export class HealthRedisService {}    // health-redis.service.ts
```

### Clients and Adapters Named for the Module [arch-resource-naming, arch-feature-modules]
A client or adapter stands for an outside system and is named for it, the way the third-party code is; the interface it implements is the module's contract, with the module prefix and the `Adapter` suffix. The older `{Variant}{Abstraction}` names (`StripePaymentGateway`) and module-prefixed clients no longer apply.

```typescript
// ❌ Anti-pattern
export class PaymentStripeClient {}                              // payment/clients/payment-stripe.client.ts
export interface PaymentGateway {}                               // payment/interfaces/payment-gateway.interface.ts
export const PAYMENT_GATEWAY = Symbol('PAYMENT_GATEWAY');
export class StripePaymentGateway implements PaymentGateway {}   // payment/adapters/stripe.adapter.ts
export class S3Client {}                                         // storage/clients/s3.client.ts: the SDK's own name
export class FakePaymentGateway implements PaymentGateway {}     // test/helpers/fake-payment-gateway.ts
export const STRIPE_CLIENT = Symbol('STRIPE_CLIENT');            // a token, though the SDK's client is a class (Stripe)

// ✅ Fix
export class StripeClient {}                                     // payment/clients/stripe.client.ts
export interface PaymentAdapter {}                               // payment/interfaces/payment-adapter.interface.ts
export const PAYMENT_ADAPTER = Symbol('PAYMENT_ADAPTER');        // payment/constants/payment-adapter.constant.ts
export class StripeAdapter implements PaymentAdapter {}          // payment/adapters/stripe.adapter.ts
export class AwsS3Client {}                                      // storage/clients/aws-s3.client.ts
export class FakePaymentAdapter implements PaymentAdapter {}     // test/helpers/fake-payment-adapter.ts
```

The interface is `{Module}{Role}Adapter` behind `{MODULE}_{ROLE}_ADAPTER`, and a test double is `Fake{Interface}`. A system behind two of the module's interfaces gets one adapter per role (`TwilioSmsAdapter`, `TwilioVoiceAdapter`). The SDK's own client is a provider of the module injected by its class (`Stripe`, `S3Client`); a token only when the SDK exports no class to inject, so a token is a finding only for an SDK whose client is a class. Only the system's client calls the SDK; the adapter calls the client. An app has one client per outside system, so the name cannot clash with another module's.

### Module Code in `shared/` [arch-shared-folder]
`src/shared/` holds plain, domain-neutral files that several modules import. A provider listed there by several modules gets one instance per module, and a file that imports a module or the ORM makes every consumer depend on it.

```text
❌ Anti-pattern
src/shared/
├── services/
│   └── storage.service.ts             // a provider: one instance per module that lists it
├── dtos/
│   └── order-address.dto.ts           // the order module's data
├── decorators/
│   └── auth-user.decorator.ts         // the HTTP layer
├── utils/
│   └── unique-violation.ts            // imports TypeORM's QueryFailedError
└── stock-availability.ts              // one module's rules, loose at the root

✅ Fix
src/
├── api/decorators/auth-user.decorator.ts
├── database/utils/unique-violation.ts // the module that owns the connection
├── inventory/services/inventory-availability.service.ts
├── order/dtos/order-address.dto.ts    // other modules import it from here
├── storage/
│   ├── services/storage.service.ts
│   └── storage.module.ts              // provides and exports StorageService once
└── shared/
    ├── types/
    │   ├── jwt-user.type.ts           // app-wide, with the Role enum
    │   └── paginated-result.type.ts   // app-wide
    └── utils/
        └── slug-generator.ts          // domain-neutral, two modules
```

### A Command Line Outside the `cli` Module [arch-feature-modules]
A script beside `main.ts` takes its secret where other users of the machine can read it, starts the whole API module tree, validates nothing, and turns an operator's typo into a crash.

```typescript
// ❌ Anti-pattern: src/create-admin.ts
const [email, password] = process.argv.slice(2); // the password in the process's arguments
const app = await NestFactory.createApplicationContext(AppModule); // the queue workers and the schedule start too
await app.get(UserService).createAdmin({ email, name: email, password }); // input no DTO checked
await app.close(); // skipped when createAdmin throws; a taken email reaches the crash handlers

// ✅ Fix: cli/main.ts, run as node --import ./dist/sentry/preload.js dist/cli/main.js create-admin --email <email> --name <name>
try {
  const [command, ...args] = process.argv.slice(2);
  if (command !== 'create-admin') {
    throw new CliUsageError(USAGE);
  }
  const app = await NestFactory.createApplicationContext(CliModule, { bufferLogs: true });
  app.useLogger(app.get(Logger));
  try {
    await app.get(CliCreateAdminCommand).run(args);
  } finally {
    await app.close();
  }
} catch (error) {
  const operatorError = error instanceof CliUsageError || (error instanceof HttpException && error.getStatus() < 500);
  if (!operatorError) {
    throw error;
  }
  process.stderr.write(`${error.message}\n`);
  process.exitCode = 1;
}
```

`CliModule` loads `ConfigModule.forRoot()` with `AppModule`'s `load` and `validate`, `LoggerModule` and the feature modules its commands call, without the HTTP server, the queue workers or the schedule. `CliCreateAdminCommand` (in `cli/commands/cli-create-admin.command.ts`) reads the password piped on standard input, validates its input with the route's `CreateUserDto` and `API_VALIDATION_PIPE_OPTIONS`, and calls `UserService.createAdmin()`. Any error that is not an operator's mistake is rethrown, so the crash handlers report it.

## Data Anti-Patterns

### Multi-Entity Write Without a Transaction [db-use-transactions]
Every query of the transaction goes through it: an injected repository or `this.dataSource` inside `dataSource.transaction()` commits on its own. A call to another system inside the transaction is the opposite mistake: it cannot be rolled back and holds the transaction and its locks open for a network round trip, so the transaction commits first and the call runs after it.

```typescript
// ❌ Anti-pattern: a failure after the first save leaves an order without items
const order = await this.orderRepository.save(this.orderRepository.create({ userId: authUser.id }));
await this.orderItemRepository.save(items.map((item) => ({ ...item, orderId: order.id })));

// ✅ Fix (TypeORM): one transaction, every query through its manager
return this.dataSource.transaction(async (manager) => {
  const order = await manager.save(manager.create(Order, { userId: authUser.id }));
  await manager.save(OrderItem, items.map((item) => ({ ...item, orderId: order.id })));
  return order;
});

// ✅ Fix (Prisma): a nested create is one transaction by itself
return this.prisma.order.create({ data: { userId: authUser.id, items: { create: items } } });
```

### Check-Then-Write Race [db-use-transactions]
Two concurrent requests both read the row, both pass the check in code, and both write: an order is paid twice, or a shipped order is canceled. A transaction under `READ COMMITTED` does not prevent it. The same race hides in a limit checked by a count and then an insert: lock the row the limit belongs to (`FOR UPDATE`), or take an advisory lock keyed by the set, before counting.

```typescript
// ❌ Anti-pattern: the status checked in code, then updated by id
const order = await this.orderRepository.findOneByOrFail({ id });
if (order.status !== OrderStatus.Pending) throw new BadRequestException('Order is not pending');
await this.orderRepository.update(id, { status: OrderStatus.Paid });

// ✅ Fix (TypeORM): the expected state in the update's criteria, the outcome read from affected
const { affected } = await this.orderRepository.update(
  { id, status: OrderStatus.Pending, deletedAt: IsNull() },
  { status: OrderStatus.Paid },
);
if (!affected) {
  // Read the row only now, to answer 404 for a missing order and 400 for one in another status
}

// ✅ Fix (Prisma): updateMany, the outcome read from count
const { count } = await this.prisma.order.updateMany({
  where: { id, status: OrderStatus.Pending, deletedAt: null },
  data: { status: OrderStatus.Paid },
});
```

### Saving a Loaded Entity to Update It [db-typeorm-use-transactions]
`save()` reloads the row, soft-deleted rows included, and writes every column whose value in memory differs from the reloaded one: a concurrent change to another column is overwritten with the stale value, and a soft delete that landed between the read and the save is undone. `save()` is for inserts.

```typescript
// ❌ Anti-pattern: a stale copy written back
const user = await this.userRepository.findOneByOrFail({ id });
Object.assign(user, dto);
return this.userRepository.save(user);

// ✅ Fix: one update with the request's fields, only on a live row, then a read for the response
this._assertVisible(authUser, id);
const { affected } = await this.userRepository.update({ id, deletedAt: IsNull() }, dto);
if (!affected) {
  throw new NotFoundException(`User #${id} not found`);
}
return this.findById(authUser, id);
```

### A Retry Loop of Its Own [db-use-transactions, db-prisma-use-transactions]
Every module that retries uses the transaction service of the module that owns the connection. A loop written in a service repeats that work and easily checks the wrong error: with Prisma, `P2034` covers a conflict on a model query only, while a deadlock on a raw `FOR UPDATE` comes as `P2010` and a conflict at `COMMIT` as the driver adapter's own error, rethrown unconverted. Its last conflict then reaches the client as a 500 instead of a 409.

```typescript
// ❌ Anti-pattern: a deadlock on a row lock and a conflict at COMMIT are never retried, and the last attempt's error is a 500
for (let attempt = 1; ; attempt += 1) {
  try {
    return await this.prisma.$transaction(work);
  } catch (error) {
    const conflict = error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2034';
    if (!conflict || attempt === 3) throw error;
  }
}

// ✅ Fix: the transaction service of the module that owns the connection, which reads PostgreSQL's code
// (40001, 40P01) and answers 409 TRANSACTION_CONFLICT once its attempts run out
return this.prismaTransactionService.retried(work);   // Prisma
return this.databaseTransactionService.retried(work); // TypeORM
```

The work runs again on a conflict, so it only touches the database: a call to another system inside it would be repeated.

### Unsafe Migrations [db-use-migrations, db-typeorm-use-migrations, db-prisma-use-migrations]
- A hand-written structural migration instead of one the ORM generated from the model: `migration:create` filled with DDL, a hand-written Prisma `migration.sql`, or a stand-in labeled "regenerate this".
- An applied migration edited in place: the databases where it already ran never get the change, so environments diverge without a warning. With Prisma, not even a migration only the author's database has run: `migrate dev` notices the edit and asks to reset the database.
- The schema changed outside migrations: `synchronize: true` or `synchronize` read from config, `prisma db push`, DDL run by the app.
- Migrations run at startup (`migrationsRun: true`, a start script that runs `prisma migrate deploy`), from every replica at once.
- A column or table dropped or renamed in one release while the previous release still reads it, a rename the generator wrote as a drop plus an add included.
- A column dropped while the model still maps it, or a property or field removed to unmap it, which generates the drop a release early.
- A migration that adds an enum value and also uses it.

**Fix**: generate the migration from the model, fix forward with a new migration, split destructive changes into expand and contract (unmapping the column first: TypeORM `select: false, insert: false, update: false`, Prisma `@ignore` on an optional field), use a new enum value in the next migration, and apply with `migration:run` or `prisma migrate deploy` as a deploy step of its own.

### Unescaped Search Wildcards [perf-optimize-database, db-prisma-schema-conventions]
TypeORM's `Like()` and `ILike()` and Prisma's `contains`, `startsWith` and `endsWith` put the value into a `LIKE` pattern as it is, so a `%` or `_` in the client's text matches anything. A substring or case-insensitive search also needs a trigram index, or it reads the whole table.

```typescript
// ❌ Anti-pattern: ?search=% matches every row
where: { name: ILike(`%${query.search}%`) }                                  // TypeORM
where: { name: { contains: query.search, mode: 'insensitive' }, deletedAt: null } // Prisma

// ✅ Fix: the client's \, % and _ escaped, so its text matches itself
where: { name: ILike(`%${escapeLike(query.search)}%`) }
where: { name: { contains: escapeLike(query.search), mode: 'insensitive' }, deletedAt: null }

// shared/utils/escape-like.ts
export function escapeLike(value: string): string {
  return value.replace(/[\\%_]/g, (character) => `\\${character}`);
}
```

### `this.prisma` Inside a Transaction [db-prisma-use-transactions]
`this.prisma` inside `$transaction()` runs outside the transaction and commits on its own; a method that joins the caller's transaction takes `tx: Prisma.TransactionClient` as its first parameter.

```typescript
// ❌ Anti-pattern: the delete commits on its own, and the refund holds the transaction open
await this.prisma.$transaction(async (tx) => {
  await tx.order.update({ where: { id }, data: { status: OrderStatus.Canceled } });
  await this.prisma.orderItem.deleteMany({ where: { orderId: id } });
  await this.paymentService.refund(id);
});

// ✅ Fix: every query on tx, the state transition conditional, the outside call after the commit
await this.prisma.$transaction(async (tx) => {
  const { count } = await tx.order.updateMany({
    where: { id, status: { in: [OrderStatus.Pending, OrderStatus.Paid] }, deletedAt: null },
    data: { status: OrderStatus.Canceled },
  });
  if (count === 0) {
    throw new BadRequestException(`Order #${id} can no longer be canceled`, { errorCode: 'ORDER_NOT_CANCELABLE' });
  }
  await tx.orderItem.deleteMany({ where: { orderId: id } });
});
await this.paymentService.refund(id);
```

### An `undefined` Value in a Prisma Query [db-prisma-schema-conventions, db-prisma-use-transactions]
Prisma drops a `where` key whose value is `undefined`. With `strictUndefinedChecks`, an explicit `undefined` anywhere in a query throws instead, and a DTO instance holds `undefined` for every optional field the request left out.

```typescript
// ❌ Anti-pattern: without strictUndefinedChecks, a job whose orderId is missing updates every order;
// with it, data: dto throws for every optional field the request left out
await this.prisma.order.updateMany({ where: { id: job.data.orderId }, data: { status: OrderStatus.Canceled } });
return this.prisma.user.update({ where: { id, deletedAt: null }, data: dto });

// ✅ Fix: strictUndefinedChecks in previewFeatures, Prisma.skip for an optional filter, only the DTO's defined fields
const where: Prisma.OrderWhereInput = { userId: authUser.id, status: query.status ?? Prisma.skip, deletedAt: null };
return this.prisma.user.update({ where: { id, deletedAt: null }, data: definedFields(dto) });
```

### A Prisma Query Without `deletedAt: null` [db-prisma-schema-conventions]
Prisma has no soft delete: nothing filters for you, and every test passes when a filter is dropped unless a test names it. A partial unique index makes its column a `findUnique` key without its condition.

```typescript
// ❌ Anti-pattern: both return deleted rows
const user = await this.prisma.user.findUnique({ where: { email } });
const orders = await this.prisma.order.findMany({ where: { userId: authUser.id } });

// ✅ Fix: every query on a soft-deletable model filters deletedAt: null, and a test names each filter
const user = await this.prisma.user.findFirst({ where: { email, deletedAt: null } });
const orders = await this.prisma.order.findMany({ where: { userId: authUser.id, deletedAt: null } });
```

An e2e assertion that matches the `where` with `expect.objectContaining()` on the id passes with the filter gone; the expected `where` includes `deletedAt: null`.

### Prisma Data at the API Boundary [api-use-dto-serialization, db-prisma-schema-conventions]
Prisma returns plain objects, so `@Exclude()` has nothing to act on. Two layers are required: `PRISMA_OMIT`, the global `omit` of `PrismaService`, keeps sensitive fields out of every result, and a response DTO converted with `excludeExtraneousValues: true` keeps a field added later private. A Prisma input type as the body skips validation and lets a client set any column, relations included.

```typescript
// ❌ Anti-pattern: a token hash added to the schema but not to PRISMA_OMIT, the row returned as Prisma gives it,
// and the body bound to Prisma's input type
@Post()
create(@Body() data: Prisma.UserCreateInput): Promise<User> {
  return this.userService.create(data);
}

// ✅ Fix: the field in the global omit in the same change, a request DTO, a response DTO with excludeExtraneousValues
// prisma/constants/prisma-omit.constant.ts
export const PRISMA_OMIT = {
  user: { passwordHash: true, resetTokenHash: true },
} as const;

// user/controllers/user.controller.ts
@Post()
@SerializeOptions({ type: UserResponseDto, excludeExtraneousValues: true })
create(@Body() dto: CreateUserDto): Promise<UserRecord> {
  return this.userService.create(dto);
}
```

`UserRecord` is `Prisma.UserGetPayload<{ omit: typeof PRISMA_OMIT.user }>`. Inside a transaction the result types still list the omitted fields, which are `undefined` at runtime: only the owning module's service reads one, from a query that opts back in with `omit: { passwordHash: false }`.

### A Page and Its Count in a Sequential `$transaction` [perf-optimize-database, db-prisma-use-transactions]
Under `READ COMMITTED` each statement of a sequential transaction reads what is committed when it starts, so the pair gets no common snapshot, and it holds a connection for its whole length, failing with `P2028` after `maxWait` while the pool is busy, where plain queries wait their turn.

```typescript
// ❌ Anti-pattern: no consistency gained, and a busy pool turns the list into an error
const [orders, totalCount] = await this.prisma.$transaction([
  this.prisma.order.findMany({ where, orderBy: [{ createdAt: 'desc' }, { id: 'desc' }], skip: query.offset, take: query.limit }),
  this.prisma.order.count({ where }),
]);

// ✅ Fix: two plain queries
const [orders, totalCount] = await Promise.all([
  this.prisma.order.findMany({ where, orderBy: [{ createdAt: 'desc' }, { id: 'desc' }], skip: query.offset, take: query.limit }),
  this.prisma.order.count({ where }),
]);
```

Where the total must match the page, the pair runs in a transaction with `isolationLevel: Prisma.TransactionIsolationLevel.RepeatableRead`.

## Error Handling Anti-Patterns

### Swallowing Errors [error-throw-http-exceptions, error-use-exception-filters]
A catch that turns every error into an ordinary answer hides the failure: nothing is logged, nothing reaches Sentry, and a database outage looks like a missing user.

```typescript
// ❌ Anti-pattern
async findById(authUser: JwtUser, id: string): Promise<User | null> {
  this._assertVisible(authUser, id);
  try {
    return await this.userRepository.findOne({ where: { id } });
  } catch (error) {
    return null; // A database outage now looks like "user not found"
  }
}

// ✅ Fix: a missing row is a 404; any other error reaches the global filter
async findById(authUser: JwtUser, id: string): Promise<User> {
  this._assertVisible(authUser, id);
  const user = await this.userRepository.findOne({ where: { id } });
  if (!user) {
    throw new NotFoundException(`User #${id} not found`);
  }
  return user;
}
```

**Fix**: catch only an error you turn into a defined outcome (a unique violation into a 409), keeping the original as the `cause`. The filter logs everything else with its causes, answers a generic 500 and reports it to Sentry.

### Unhandled Errors Outside a Request [error-handle-async-errors, error-report-to-sentry]
Nothing catches what work outside a request throws: an unawaited promise, an `@OnEvent` listener, an `@Interval` job. Its rejection reaches the crash handlers, which end the process.

```typescript
// ❌ Anti-pattern
this.mailService.sendWelcome(user.email); // Not awaited, no catch

@OnEvent('order.created')
onOrderCreated(event: OrderCreatedEvent): void {
  this.recommendationService.recordPurchase(event.orderId); // Promise dropped; its failure is unhandled
}

// ✅ Fix: the listener catches, logs the error as err, reports it and never rethrows
@OnEvent('order.created', { async: true })
async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
  try {
    await this.recommendationService.recordPurchase(event.orderId);
  } catch (error) {
    this.logger.error({ err: error }, `Recording purchase failed for order ${event.orderId}`);
    SentryService.reportError(error);
  }
}
```

**Fix**: a promise left unawaited on purpose gets a `.catch()` that logs and reports it, and work that must eventually succeed goes to a queue. Queue processors and outbox listeners (`@OnOutboxEvent()`) are the exception: they throw, so the job or the event is tried again.

### Exposing Internal Errors [error-use-exception-filters]
Sending raw error messages or stack traces to clients leaks internals.

```typescript
// ❌ Anti-pattern
catch (error) {
  throw new HttpException(error.stack, 500); // Stack trace exposed
}

// ✅ Fix: a message of your own for the error you know, the original kept as the cause for the logs
catch (error) {
  if (error instanceof PaymentDeclinedException) {
    throw new BadRequestException('The payment was declined', { cause: error, errorCode: 'PAYMENT_DECLINED' });
  }
  throw error;
}
```

An error you do not translate goes on as it is, rethrown or never caught: the filter answers an unknown one with a generic 500.

### Detecting a Duplicate by Lookup or by Message [error-use-exception-filters, error-throw-http-exceptions]
A lookup before the insert lets two concurrent requests both pass it. The unique index decides, recognized by its SQLSTATE code and its name, never by the message text.

```typescript
// ❌ Anti-pattern: a lookup first
const existing = await this.userRepository.findOne({ where: { email: dto.email } });
if (existing) throw new ConflictException('Email already registered');

// ❌ Anti-pattern: the driver's message, matched in the filter
if (exception instanceof QueryFailedError && exception.message.includes('duplicate key')) { /* 409 */ }

// ✅ Fix: database/utils/unique-violation.ts, used by the filter and by the services that name the clash
const PG_UNIQUE_VIOLATION = '23505';

export function isUniqueViolation(error: unknown, constraint?: string): boolean {
  if (!(error instanceof QueryFailedError)) {
    return false;
  }
  const driverError = error.driverError as Error & { code?: string; constraint?: string };
  return driverError.code === PG_UNIQUE_VIOLATION && (constraint === undefined || driverError.constraint === constraint);
}

// user/services/user.service.ts
try {
  return await this.userRepository.save(await this._toEntity(dto));
} catch (error) {
  if (isUniqueViolation(error, 'UQ_users_email_active')) {
    throw new ConflictException('Email already registered', { cause: error, errorCode: 'EMAIL_TAKEN' });
  }
  throw error;
}
```

The checks that know the ORM's errors (`isUniqueViolation()`, `isTransactionConflict()`, `isUnstorableText()`) live in the `database` module's `utils/`, or the `prisma` module's with Prisma (`P2002`, `isDatabaseBusy()`), never in the api module or in `shared/`.

### Losing Validation Details in a Global Filter [error-use-exception-filters]

```typescript
// ❌ Anti-pattern: every validation error becomes "Bad Request Exception"
const message = exception instanceof HttpException ? exception.message : 'Internal server error';

// ✅ Fix: the message from getResponse(), which holds ValidationPipe's per-field messages
if (exception instanceof HttpException) {
  const response = exception.getResponse();
  if (typeof response === 'string') return response;
  return (response as { message?: string | string[] }).message ?? exception.message;
}
```

`ValidationPipe` keeps its default `errorFormat: 'list'`: `'grouped'` turns `message` into an object keyed by field, which the envelope's `string | string[]` does not allow.

### Losing the Error in the Log Line [error-use-exception-filters]
With `nestjs-pino`, an argument after the message never reaches the line unless it is a stack string, so an error object, its `cause` and a thrown value that is not an Error are lost.

```typescript
// ❌ Anti-pattern: the cause is dropped, and a thrown string disappears
this.logger.error(`Recording purchase failed for order ${orderId}`, error instanceof Error ? error.stack : error);
this.logger.error('Caused by', error.cause);

// ✅ Fix: the err field, which pino serializes with the stack and every cause
this.logger.error({ err: error }, `Recording purchase failed for order ${orderId}`);
```

### Reporting to Sentry With the SDK's Defaults [error-report-to-sentry]
Version 11 of the SDK sends request headers, bodies, cookies, query data, database query data and frame variables by default, and its filtering works on keys, so a secret inside an error message goes out as it is. A second Sentry package makes a second client, and what is captured through the one never initialized is dropped without a trace.

```typescript
// ❌ Anti-pattern: src/sentry/preload.ts, imported from main.ts
import * as Sentry from '@sentry/node'; // A second Sentry package beside @sentry/nestjs

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 1.0, // Spans carry headers, queries and outgoing URLs
  beforeSend(event) {
    delete event.request?.headers?.authorization; // A denylist: the cookies, the body and the message still go out
    return event;
  },
});

// main.ts
import './sentry/preload'; // In ESM, too late for the SDK to wrap what it instruments
```

**Fix**: `@sentry/nestjs` alone, at one exact version. Every start script loads `src/sentry/preload.ts` with `node --enable-source-maps --import ./dist/sentry/preload.js` (`--require` in CommonJS), and it calls `SentryService.init()`: no tracing, every `dataCollection` category off, and a `beforeSend` that rebuilds each event from the allowlist and redacts the secrets. `AppModule` imports `SentryModule`, and the global filter reports each 5xx with `SentryService.reportServerError()`.

## Testing Anti-Patterns

### Mocking the Service Under Test [test-use-testing-module]
A suite that mocks the service it is named after only proves the controller calls it, and passes while the service is broken.

```typescript
// ❌ Anti-pattern: the service is mocked in its own suite
providers: [{ provide: OrderService, useValue: { findById: jest.fn() } }],

// ✅ Fix: real service, mocks at the repository boundary and for other modules' services, driven over HTTP
const moduleRef = await Test.createTestingModule({
  imports: [ApiModule.forRoot()],
  controllers: [OrderController],
  providers: [
    OrderService,
    { provide: getRepositoryToken(Order), useValue: orderRepository },
    { provide: DataSource, useValue: dataSource },
    { provide: PaymentService, useValue: paymentService },
    { provide: APP_GUARD, useValue: testAuthGuard(() => AUTH_USER) },
  ],
}).compile();
app = await createTestApp(moduleRef);

const res = await request(app.getHttpServer()).get(`/orders/${ORDER_ID}`);
expect(res.status).toBe(200);
expect(orderRepository.findOne).toHaveBeenCalledWith({ where: { id: ORDER_ID, userId: USER_ID } });
```

### Only Testing Interactions [test-use-testing-module]
Tests that only assert which mock was called don't prove the result is right. Assert the status and body, and the repository call where it is the observable effect.

```typescript
// ❌ Anti-pattern: only the interaction is checked
it('creates a user', async () => {
  await request(app.getHttpServer()).post('/users').send(dto);
  expect(userRepository.save).toHaveBeenCalled();
});

// ✅ Fix: positive and negative paths, outcome asserted
describe('POST /users', () => {
  it('creates a user', async () => {
    userRepository.save.mockResolvedValue({ id: USER_ID, ...dto });

    const res = await request(app.getHttpServer()).post('/users').send(dto);

    expect(res.status).toBe(201);
    expect(res.body.data).toMatchObject({ id: USER_ID, email: dto.email });
  });

  it('rejects a duplicate email', async () => {
    userRepository.save.mockRejectedValue(
      new QueryFailedError('INSERT', [], Object.assign(new Error('duplicate key'), { code: '23505', constraint: 'UQ_users_email_active' })),
    );

    const res = await request(app.getHttpServer()).post('/users').send(dto);

    expect(res.status).toBe(409);
  });
});
```

### Only Testing the Allow Path of Guards [test-guards-and-interceptors]
A guard test that only checks that authorized users get in will never catch an authorization bypass, and a hand-built `ExecutionContext` skips the real metadata lookup and order. Drive the guard through a probe route and test that unauthorized callers are rejected.

```typescript
// ❌ Anti-pattern: mocked metadata, a mocked context, the allow path only
const guard = new AuthRolesGuard(new Reflector());
const context = { switchToHttp: () => ({ getRequest: () => ({ user: { roles: ['admin'] } }) }) } as any;
jest.spyOn(Reflector.prototype, 'getAllAndOverride').mockReturnValue(['admin']);
expect(guard.canActivate(context)).toBe(true);

// ✅ Fix: a probe route on createTestApp(), the caller picked per request by a test guard, the deny path too
it('rejects a user without the admin role', async () => {
  const res = await request(app.getHttpServer()).get('/probe/admin').set('x-test-user', 'user');
  expect(res.status).toBe(403);
});
```

### Bad Tokens That Change Two Things [test-guards-and-interceptors]
The real `AuthJwtGuard` suite is the only one that runs `AuthJwtStrategy`. A bad token that differs from a valid one in two ways is refused for one of them, so the check for the other can break unnoticed.

```typescript
// ❌ Anti-pattern: another secret and no audience; refused on the audience, it passes with a broken signature check
['a token signed with another secret', jwtService.sign(VALID_CLAIMS, { secret: 'another-secret', issuer: 'test-issuer', expiresIn: '5m' })],

// ✅ Fix: one change from a valid token per case
['a token signed with another secret', sign(VALID_CLAIMS, { secret: 'another-secret' })],
['a token from another issuer', sign(VALID_CLAIMS, { issuer: 'another-issuer' })],
```

### An Override of a Provider No Module Declares [test-use-testing-module]
`overrideProvider()` of a provider that no module provides is ignored without a word, so the spec keeps passing after the provider is dropped from its module.

```typescript
// ❌ Anti-pattern: still green once ReportProcessor leaves ReportModule
const moduleRef = await Test.createTestingModule({ imports: [ReportModule] })
  .overrideProvider(ReportProcessor)
  .useValue(reportProcessor)
  .compile();

// ✅ Fix: get each replaced provider from the compiled module, which throws for one no module declares
expect(moduleRef.get(ReportProcessor)).toBe(reportProcessor);
```

### A Spec That Reads the Developer's `.env` [test-use-testing-module]
`AppModule`'s `ConfigModule` fills every unset variable from `.env` and never overrides one already set. A spec that imports `AppModule` without setting what it depends on passes on a laptop and fails in CI, which has no `.env`.

```typescript
// ❌ Anti-pattern: a shutdown spec drains for the 0 ms a laptop's .env sets, and for the 15 s default in CI
const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();

// ✅ Fix: every setting the spec depends on is set first (vi.stubEnv() under Vitest)
process.env.APP_SHUTDOWN_DRAIN_MS = '0';
const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();

// A testing module built for one feature reads no .env at all
ConfigModule.forRoot({ ignoreEnvFile: true, load: [mailConfig] });
```

### A Queue Mock That Fails at Once [test-mock-external-services]
While Redis is unreachable, BullMQ's `add()` does not settle in time: from startup on it neither resolves nor rejects, and after a lost connection it rejects only minutes later. A mock that rejects at once tests a failure that never happens and misses the hang that `addJobInTime()` exists for.

```typescript
// ❌ Anti-pattern
reportQueue.add.mockRejectedValue(new Error('Connection is closed'));

// ✅ Fix: an add() that never settles; the test expects the record marked failed and a 503
reportQueue.add.mockReturnValue(new Promise(() => {}));
```

### A Database Tier on the API's Database [test-database-tier]
The database tier connects with the `.env` settings, which point at the database the running API uses. Its specs that commit rows hand their events to that API's outbox dispatcher, whose own rows then refuse the spec's cleanup.

```typescript
// ❌ Anti-pattern: the db project inherits DB_NAME from .env
{ extends: true, test: { name: 'db', include: ['test/db/**/*.spec.ts'] } },

// ✅ Fix (Vitest): the tier names a database of its own, set before any spec loads
{ extends: true, test: { name: 'db', include: ['test/db/**/*.spec.ts'], env: { DB_NAME: 'shop_test' } } },

// ✅ Fix (Jest): test/jest-db.json loads "setupFiles": ["<rootDir>/test/db/env.ts"]; Jest gives each spec a copy of
// process.env that the DataSource's process.loadEnvFile() never reaches
if (existsSync('.env')) {
  for (const [name, value] of Object.entries(parseEnv(readFileSync('.env', 'utf8')))) {
    process.env[name] ??= value;
  }
}
process.env.DB_NAME = 'shop_test';
```

### Tests in the Wrong Tier [test-organize-by-tier, test-integration-suites]
A spec or a test helper in `src/`, a "unit" test that needs a database, Redis or a network service, a service tested without its route, two spec files over the same routes, or an integration spec for what `ApiModule.setup()` registers (the security headers and CORS belong to the api module's e2e spec, and `main.ts` needs no spec). Each test belongs to exactly one tier under `test/`, decided by what it needs to run; a loopback server the spec starts and a repository file it reads do not count as outside the process.

### No Test Isolation [test-integration-suites]
Integration specs share a database across specs and across runs. A test that depends on another having run first, on the position of a row someone else created, or on a page's length as a total passes or fails by the order and the history of the runs.

```typescript
// ❌ Anti-pattern: order-dependent, position-based, a page length used as a total
describe('Orders', () => {
  let orderId: string;

  it('creates an order', async () => {
    const res = await caller.post('/orders', { items: [{ sku: 'SKU-001', quantity: 1 }] });
    orderId = res.body.data.id; // The next test depends on this one running first
  });

  it('lists orders', async () => {
    const res = await caller.get('/orders?limit=10');
    expect(res.body.data[0].status).toBe('shipped'); // The position of a row another run left
    expect(res.body.data.length).toBe(1);             // Bounded by limit, not a total
  });
});

// ✅ Fix: each test creates what it asserts on, checks the status first and finds its row by id
it('includes a just-created order on the first page', async () => {
  const created = await createOrder();
  const res = await caller.get<Order[]>('/orders?limit=100');

  expect(res.status).toBe(200);
  expect(ok(res).data.some((order) => order.id === created.id)).toBe(true);
});
```

## Configuration Anti-Patterns

### Hardcoded Configuration [devops-use-config-module]
Hardcoded values can't change per environment and leak secrets into source control. Settings come from `src/config/{concern}.config.ts` namespaces, the only place that reads `process.env`: inject the namespace for several values, or read a single value with a namespaced key.

```typescript
// ❌ Anti-pattern
@Injectable()
export class DatabaseService {
  private readonly host = 'localhost';
  private readonly password = 'secretpassword';
}

@Injectable()
export class MailService {
  private readonly apiKey = process.env.SENDGRID_API_KEY || 'default'; // process.env outside src/config/, a fake default
}

const secret = this.configService.get('JWT_SECRET'); // A raw variable name, no namespace, maybe undefined

// ✅ Fix: inject the config namespace (src/config/db.config.ts)
@Injectable()
export class DatabaseService {
  constructor(@Inject(dbConfig.KEY) private readonly dbSettings: ConfigType<typeof dbConfig>) {}

  getConfig() {
    return { host: this.dbSettings.host, port: this.dbSettings.port };
  }
}

// One required value: its namespaced key
const secret = this.configService.getOrThrow<string>('auth.jwt.secret');
```

### A Config Module of Our Own [devops-use-config-module]
`src/config/` is a folder, not a module. A namespace registered with `ConfigModule.forFeature()`, or a config file inside a feature module, skips the one load in `AppModule` and its `validate` hook, and scatters the environment variables over the code. A module meant for reuse takes typed options through `forRootAsync` instead, and the app maps its namespace into them.

```typescript
// ❌ Anti-pattern: encryption/config/encryption.config.ts, registered by the feature
@Module({ imports: [ConfigModule.forFeature(encryptionConfig)], providers: [EncryptionService] })
export class EncryptionModule {}

// ❌ Anti-pattern: config/config.module.ts, a module around configuration
@Module({ providers: [AppConfigService], exports: [AppConfigService] })
export class AppConfigModule {}

// ✅ Fix: src/config/encryption.config.ts, loaded with the others, once, in AppModule
ConfigModule.forRoot({ isGlobal: true, load: [appConfig, dbConfig, encryptionConfig], validate: validateEnv });

// encryption/services/encryption.service.ts injects the namespace
constructor(@Inject(encryptionConfig.KEY) private readonly encryptionSettings: ConfigType<typeof encryptionConfig>) {}
```

### An Environment Name With a Default [devops-use-config-module]
A default lets a missing `APP_ENV` pass for one environment, and a comparison with a value `APP_ENV` never takes matches nothing.

```typescript
// ❌ Anti-pattern
export const appConfig = registerAs('app', () => ({
  env: process.env.APP_ENV || 'development', // A missing value becomes 'development', and nothing refuses a misspelled one
}));
const pretty = appSettings.env !== 'prod'; // Pretty logs in the deployed dev and stage too

// ✅ Fix: required, one of three values, checked at load by validate-env.ts; local behavior is a setting of its own
export const APP_ENVS = ['dev', 'stage', 'prod'] as const;

export const appConfig = registerAs('app', () => ({
  env: process.env.APP_ENV as (typeof APP_ENVS)[number], // validateEnv refuses any other value
  logPretty: process.env.APP_LOG_PRETTY === 'true',
}));
```

### An Optional Number Without a Check [devops-use-config-module]
A default covers a missing value, not a wrong one. `parseInt('15s', 10)` is 15, so `APP_SHUTDOWN_DRAIN_MS=15s` drains for 15 milliseconds, and a value that parses to `NaN` starts a service that fails later.

```typescript
// ❌ Anti-pattern: the config file parses, and validate-env.ts checks only the required variables
shutdownDrainMs: parseInt(process.env.APP_SHUTDOWN_DRAIN_MS || '15000', 10),

// ✅ Fix: validate-env.ts refuses an optional number that is set but is not a whole number
const WHOLE_NUMBERS = ['APP_PORT', 'APP_SHUTDOWN_DRAIN_MS', 'DB_PORT', 'DB_POOL_SIZE'] as const;

const notWhole = WHOLE_NUMBERS.filter((name) => String(env[name] ?? '') !== '' && !/^\d+$/.test(String(env[name])));
if (notWhole.length > 0) {
  throw new Error(`Refusing to start: ${notWhole.join(', ')} must be whole numbers`);
}
```

### An Unchecked Database or Redis Connection [devops-use-config-module]
The connections to the database and Redis carry the password and every row the app reads. A plain connection outside `dev`, or TLS without the certificate check, hands both to anyone on the way.

```typescript
// ❌ Anti-pattern: TLS turned on, the certificate check turned off
ssl: dbSettings.ssl ? { rejectUnauthorized: false } : false,

// ✅ Fix: validate-env.ts refuses DB_SSL, and REDIS_TLS when REDIS_HOST is set, other than 'true' outside APP_ENV dev;
// a managed service's own CA is read from a file, and the certificate is still checked
// src/config/db.config.ts
sslCa: process.env.DB_SSL_CA_FILE ? readFileSync(process.env.DB_SSL_CA_FILE, 'utf8') : undefined,

// database/database.module.ts
ssl: dbSettings.ssl ? { ca: dbSettings.sslCa } : false,
```

### A Comment After a Value in `.env.example` [devops-use-config-module]
Docker's `--env-file` keeps everything after the `=` as the value, comment included.

```bash
# ❌ Anti-pattern: under Docker, DB_PORT is "5432   # the PostgreSQL port"
DB_PORT=5432   # the PostgreSQL port

# ✅ Fix: the comment on a line of its own, above the variable
# The PostgreSQL port
DB_PORT=5432
```
