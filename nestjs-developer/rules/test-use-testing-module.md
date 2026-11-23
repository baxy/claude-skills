---
title: Test Services Through Their Routes
impact: HIGH
impactDescription: One request covers routing, validation, guards, controller, service and the response envelope together
tags: testing, e2e, testing-module, mocking, supertest
---

## Test Services Through Their Routes

In the e2e tier (see `test-organize-by-tier`), build the **real** controller and service with `Test.createTestingModule`, mock at the module boundary (TypeORM's repository tokens and `DataSource`, plus services that belong to other modules), and drive the app over HTTP in-process with `supertest`. One request then exercises routing, `ValidationPipe`, param binding, guards, the controller, the service, the exception filter and the response envelope in a single pass.

- **Never mock the service the suite is named after.** A suite that mocks its own service tests the controller and nothing else, and passes while the service is broken.
- Assert observable behavior: the status code, the response body (`res.body.data` inside the envelope, see `api-use-interceptors`), and what reached the repository boundary.
- **Same global setup as production.** The api module (`src/api/`) holds the API's global behavior. `ApiModule.forRoot()` provides the exception filter and the global interceptors, so Nest builds them with their dependencies, and `ApiModule.setup(app, corsOrigins)` registers them with the global `ValidationPipe`, after the security headers and CORS; both `main.ts` and `createTestApp()` call it, so e2e specs see the same headers as clients. `AppModule` imports `ApiModule.forRoot()`, and so does every e2e testing module, next to the feature it tests: without it, `ApiModule.setup()` fails on its first `app.get()`, so a forgotten import is an error, never a test that runs a different app. Do not register them as `APP_PIPE`/`APP_FILTER`/`APP_INTERCEPTOR` providers instead: a testing module that does not import their module runs without them, and its tests still pass. Guards stay `APP_GUARD` providers; tests replace authentication with a test guard. Route-shaping setup (`app.setGlobalPrefix()`, `app.enableVersioning()`, see `api-versioning`) also goes in `ApiModule.setup()`, or e2e tests call different paths than clients.
- **An override applies only to a provider some module declares.** `overrideProvider()` of a provider no module provides is ignored without a word, so a spec that replaces the job processors still passes once a processor is dropped from its module. Such a spec also gets each one it replaces from the compiled module (`moduleRef.get(ReportProcessor)`), which throws for a provider no module declares.
- **A spec that imports `AppModule` sets every setting it depends on**, with `vi.stubEnv()` (`process.env` under Jest). `AppModule`'s `ConfigModule` reads the developer's `.env` and never overrides a variable that is already set, so an unset value comes from that file: the spec passes on a laptop and fails in CI, which has no `.env`. A testing module built for one feature sets `ignoreEnvFile: true` on its own `ConfigModule` instead.
- **One helper knows the HTTP adapter.** `createTestApp()` creates the app for Express or Fastify; specs never touch adapter APIs, so they run unchanged on either (see `api-platform-agnostic`).
- Provide the caller with a test guard registered as `APP_GUARD` that sets a `JwtUser`; add `AuthRolesGuard` after it when the suite covers role rules.
- Repositories are plain objects of mock functions (`vi.fn()` in Vitest, `jest.fn()` in Jest). A service that uses a query builder gets a chainable mock (every builder method returns the builder) so the test can assert the conditions it received.
- A service method with **no route** gets a direct `describe('ServiceName.method')` block only when it has a non-HTTP consumer (another module calls it, or a command-line command, see `arch-feature-modules`); name the consumer in the block title, `describe('UserService.verifyCredentials (used by AuthSessionService)')`, not in a comment. A method with no caller at all gets no test. A lifecycle hook's caller is Nest: a startup hook that fills something a route serves (a warm-up, a list) is tested through `app.init()` alone, without calling the method, with its failure path too.
- Reset mocks between tests (`beforeEach(() => vi.clearAllMocks())`, or `jest.clearAllMocks()`); put shared constants (UUIDs, mock objects) at the top of the file. Test UUIDs must pass `@IsUUID()`, which checks the version digit: use `00000000-0000-4000-8000-000000000001` and so on. `00000000-0000-0000-0000-000000000001` is rejected with a 400, so every route test using it fails before reaching the service.

**Incorrect (mocking the service under test, or bypassing the HTTP layer):**

```typescript
// The service is mocked: this only proves the controller calls it
describe('Orders', () => {
  const orderService = { findById: jest.fn() };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [OrderController],
      providers: [{ provide: OrderService, useValue: orderService }],
    }).compile();
    // ...
  });
});

// Controller constructed by hand: no routing, no ValidationPipe, no guards, no envelope
it('returns the order', async () => {
  const controller = new OrderController(service as any);
  expect(await controller.findById(AUTH_USER, { id: ORDER_ID })).toBeDefined();
});
```

**Correct (real service, mocked repositories, in-process HTTP):**

```typescript
// api/constants/api-validation-pipe-options.constant.ts
export const API_VALIDATION_PIPE_OPTIONS: ValidationPipeOptions = {
  whitelist: true,
  forbidNonWhitelisted: true,
  // Nest defaults it to false, and a nested object whose @Type is missing then passes unvalidated
  forbidUnknownValues: true,
  transform: true,
};

// api/api.module.ts
@Module({})
export class ApiModule {
  static forRoot(): DynamicModule {
    return { module: ApiModule, providers: [ApiExceptionFilter, ApiLoggerInterceptor, ApiResponseInterceptor] };
  }

  // ... createFastifyAdapter()

  static async setup(app: NestFastifyApplication, corsOrigins: string[]): Promise<void> {
    await app.register(helmet);
    app.enableCors({ origin: corsOrigins, methods: ['GET', 'HEAD', 'POST', 'PUT', 'PATCH', 'DELETE'] });
    app.useGlobalPipes(new ValidationPipe(API_VALIDATION_PIPE_OPTIONS));
    app.useGlobalFilters(app.get(ApiExceptionFilter));
    // The envelope must wrap the serialized body, so the serializer comes last
    app.useGlobalInterceptors(
      app.get(ApiLoggerInterceptor),
      app.get(ApiResponseInterceptor),
      new ClassSerializerInterceptor(app.get(Reflector)),
    );
  }
}

// test/helpers/create-test-app.ts
export const TEST_CORS_ORIGIN = 'https://app.example.com';

export async function createTestApp(moduleRef: TestingModule): Promise<INestApplication<App>> {
  const app = moduleRef.createNestApplication<NestFastifyApplication>(ApiModule.createFastifyAdapter(), { logger: false });
  await ApiModule.setup(app, [TEST_CORS_ORIGIN]);
  await app.init();
  // Fastify adds its routes only once ready: supertest would get 404s
  await app.getHttpAdapter().getInstance().ready();
  return app;
}

// test/helpers/test-auth.guard.ts
export const testAuthGuard = (resolveUser: (request: ApiRequest) => JwtUser | undefined): CanActivate => ({
  canActivate: (context: ExecutionContext) => {
    if (context.getType() !== 'http') {
      return true;
    }
    const request = context.switchToHttp().getRequest<ApiRequest>();
    request.user = resolveUser(request);
    return true;
  },
});

// test/e2e/order/order.spec.ts
import request from 'supertest';

const USER_ID = '00000000-0000-4000-8000-000000000001';
const ORDER_ID = '00000000-0000-4000-8000-000000000002';
const SESSION_ID = '00000000-0000-4000-8000-000000000003';
const AUTH_USER: JwtUser = { id: USER_ID, email: 'jane@example.com', roles: [Role.User], sessionId: SESSION_ID };

describe('Orders', () => {
  let app: INestApplication;
  const orderRepository = { findOne: jest.fn(), save: jest.fn(), update: jest.fn(), create: jest.fn((value) => value) };
  const dataSource = { transaction: jest.fn((run) => run({ save: orderRepository.save })) };
  const paymentService = { authorize: jest.fn() };

  beforeAll(async () => {
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
  });

  afterAll(() => app.close());
  beforeEach(() => jest.clearAllMocks());

  describe('GET /orders/:id', () => {
    it('returns the order', async () => {
      orderRepository.findOne.mockResolvedValue({ id: ORDER_ID, userId: USER_ID, status: OrderStatus.Pending });

      const res = await request(app.getHttpServer()).get(`/orders/${ORDER_ID}`);

      expect(res.status).toBe(200);
      expect(res.body.data).toMatchObject({ id: ORDER_ID, status: OrderStatus.Pending });
      expect(orderRepository.findOne).toHaveBeenCalledWith({ where: { id: ORDER_ID, userId: USER_ID } });
    });

    it('returns 404 when the order does not exist', async () => {
      orderRepository.findOne.mockResolvedValue(null);

      const res = await request(app.getHttpServer()).get(`/orders/${ORDER_ID}`);

      expect(res.status).toBe(404);
      expect(res.body).toMatchObject({ success: false, statusCode: 404 });
    });

    it('returns 400 for a malformed id without reaching the repository', async () => {
      const res = await request(app.getHttpServer()).get('/orders/not-a-uuid');

      expect(res.status).toBe(400);
      expect(orderRepository.findOne).not.toHaveBeenCalled();
    });
  });

  describe('POST /orders', () => {
    const payload = { items: [{ sku: 'SKU-001', quantity: 2 }] };

    it('creates an order for the caller', async () => {
      orderRepository.save.mockImplementation(async (value) => ({ id: ORDER_ID, ...value }));

      const res = await request(app.getHttpServer()).post('/orders').send(payload);

      expect(res.status).toBe(201);
      expect(orderRepository.save).toHaveBeenCalledWith(
        expect.objectContaining({ userId: USER_ID, status: OrderStatus.Pending }),
      );
    });

    it('rejects unknown properties', async () => {
      const res = await request(app.getHttpServer()).post('/orders').send({ ...payload, status: OrderStatus.Paid });

      expect(res.status).toBe(400);
    });
  });

  describe('OrderService.markAsPaid (used by OrderPaymentEventsListener)', () => {
    it('updates only a pending order', async () => {
      const service = app.get(OrderService);
      orderRepository.update.mockResolvedValue({ affected: 1 });

      await service.markAsPaid(ORDER_ID);

      expect(orderRepository.update).toHaveBeenCalledWith(
        { id: ORDER_ID, status: OrderStatus.Pending, deletedAt: IsNull() },
        { status: OrderStatus.Paid },
      );
    });

    it('rejects an order that is already canceled', async () => {
      const service = app.get(OrderService);
      orderRepository.update.mockResolvedValue({ affected: 0 });
      orderRepository.findOne.mockResolvedValue({ id: ORDER_ID, userId: USER_ID, status: OrderStatus.Canceled });

      await expect(service.markAsPaid(ORDER_ID)).rejects.toThrow(BadRequestException);
    });
  });
});
```

`ApiModule` also holds `createFastifyAdapter()` (see `api-platform-agnostic`). `createTestApp()` is the only test file that knows the HTTP adapter: on Express it calls `moduleRef.createNestApplication()` and skips `ready()`. `testAuthGuard()` stands in for `AuthJwtGuard`, and skips every context that is not HTTP as `AuthJwtGuard` does, since a gateway or a message handler authenticates its caller itself. In the order spec, `OrderService` is the real one, `PaymentService` belongs to another module and is mocked, and the testing module imports `ApiModule.forRoot()` for the filter and the interceptors `ApiModule.setup()` registers.

On Fastify, `app.inject()` is a native alternative to `supertest`, but it ties the spec to the adapter; keep `supertest` so specs stay portable.

### With Prisma

Mock `PrismaService` instead of the repository tokens and `DataSource`: a plain object with one entry per delegate the module uses, each a plain object of mock functions. The assertions then check the `where` and `data` the service built. A typed deep mock (`mockDeep<PrismaService>()`) is not needed: the plain object fails loudly when the service calls a delegate the test did not expect.

- **`$transaction` hands the work a `tx` object of its own**, with the delegates the transaction uses, and the assertions inside a transaction name `tx`. A query the service sends to `this.prisma` inside the transaction, which in production runs outside it and commits on its own, then misses the assertion on `tx` and fails the test. With one mock for both, such a query passes. The array form resolves its queries.
- **Every soft-delete filter has an assertion that names it**: the expected `where` includes `deletedAt: null`. Prisma filters nothing by itself, so a mock that answers on `where.id` alone, or an `expect.objectContaining()` on the id, passes when the filter is gone.

```typescript
const tx = {
  order: { updateMany: jest.fn(), findFirstOrThrow: jest.fn() },
  orderItem: { deleteMany: jest.fn() },
};
const prisma = {
  order: { findFirst: jest.fn(), create: jest.fn(), updateMany: jest.fn() },
  $transaction: jest.fn((arg) => (typeof arg === 'function' ? arg(tx) : Promise.all(arg))),
};

// In the providers
{ provide: PrismaService, useValue: prisma },

it('returns the order', async () => {
  prisma.order.findFirst.mockResolvedValue({ id: ORDER_ID, userId: USER_ID, status: OrderStatus.Pending });

  const res = await request(app.getHttpServer()).get(`/orders/${ORDER_ID}`);

  expect(res.status).toBe(200);
  expect(prisma.order.findFirst).toHaveBeenCalledWith({ where: { id: ORDER_ID, userId: USER_ID, deletedAt: null } });
});

// describe('POST /orders/:id/cancel')
it('cancels the order inside its transaction', async () => {
  prisma.order.findFirst.mockResolvedValue({ id: ORDER_ID, userId: USER_ID, status: OrderStatus.Pending });
  tx.order.updateMany.mockResolvedValue({ count: 1 });
  tx.order.findFirstOrThrow.mockResolvedValue({ id: ORDER_ID, userId: USER_ID, status: OrderStatus.Canceled });

  const res = await request(app.getHttpServer()).post(`/orders/${ORDER_ID}/cancel`).send({});

  expect(res.status).toBe(200);
  expect(tx.order.updateMany).toHaveBeenCalledWith({
    where: { id: ORDER_ID, status: { in: ORDER_CANCELABLE_STATUSES }, deletedAt: null },
    data: { status: OrderStatus.Canceled },
  });
});

// describe('OrderService.markAsPaid (used by OrderPaymentEventsListener)')
it('updates only a pending order', async () => {
  prisma.order.updateMany.mockResolvedValue({ count: 1 });

  await app.get(OrderService).markAsPaid(ORDER_ID);

  expect(prisma.order.updateMany).toHaveBeenCalledWith({
    where: { id: ORDER_ID, status: OrderStatus.Pending, deletedAt: null },
    data: { status: OrderStatus.Paid },
  });
});
```

`tx` is the transaction's own client: the service must send every query inside the transaction to it, as the cancel case does.

Reference: [NestJS Testing](https://docs.nestjs.com/fundamentals/testing) · [supertest](https://github.com/ladjs/supertest)
