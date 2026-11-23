---
title: Mock External Services at the Module Boundary
impact: HIGH
impactDescription: Fast, deterministic stack-free tiers that still cover provider failures
tags: testing, mocking, external-services, vitest, jest
---

## Mock External Services at the Module Boundary

The unit and e2e tiers never call anything outside the process: third-party APIs, the database, queues, object storage (see `test-organize-by-tier`). Replace each at the provider the code injects (`HttpService`; an SDK's client by its class, `S3Client` or `Stripe`, or by a token only when the SDK exports no class to inject; TypeORM's repository token; or `PrismaService` with a `tx` of its own, see `test-use-testing-module`), so the code that uses it still runs. An SDK client is replaced by an object with the one method the code calls (`{ send: vi.fn() }` for `S3Client`), and the commands it receives are the assertion (`PutObjectCommand` and its `input`). The settings that make that client fail fast (its timeout, no retries) are checked once with the real SDK against a server in the test process (a `node:http` server answering 503, a socket that never answers): a mocked `send()` cannot show that a timeout is honored. Real databases belong to the database tier (`test-database-tier`), and real running systems to the integration tier (`test-integration-suites`).

- Drive the code through its route like any e2e test (see `test-use-testing-module`); the external client is the mock, not the service that calls it.
- The client is mocked in the module that owns it. When the provider sits behind another module (the order module charges through the payment module's `PaymentService`), the order module's route specs mock the service that module exports, as for any other module, with the outcomes its contract defines and errors built the way the real implementation builds them (a `ServiceUnavailableException` with its `retryAfterSeconds`, see `di-liskov-substitution`). The payment module's own specs mock the SDK and check that each answer of the provider comes out as the right outcome.
- Cover the provider's failure modes, not only success: timeout, rate limiting (429), server error, malformed body. Assert what the API returns for each.
- Mock data has the provider's real shape. A mock that returns `{}` passes tests the real response would fail. Failures too: an HTTP failure is an `AxiosError`, which the client recognizes (see `micro-use-http-clients`). And they fail the way the real one does: BullMQ's `add()` does not settle in time while Redis is unreachable (from startup on it never settles, after a lost connection it rejects only minutes later, see `micro-use-queues`), so a queue mock whose `add()` rejects at once tests a failure that never happens, and misses the hang.
- Freeze time with fake timers (`vi.useFakeTimers()` in Vitest, `jest.useFakeTimers()` in Jest) for expiry and scheduling logic; restore real timers afterwards.

**Incorrect (real calls, real database, happy path only):**

```typescript
describe('Shipments', () => {
  it('returns the tracking', async () => {
    const client = new CarrierClient(new HttpService(axios.create()), new ClsService(new AsyncLocalStorage())); // Real HTTP call, constructed by hand
    expect(await client.getTracking('1Z999AA10123456784')).toBeDefined(); // Slow, flaky, and only the success path
  });
});

beforeEach(async () => {
  await dataSource.query('DELETE FROM users'); // Unit/e2e test touching a real database
});
```

**Correct (client mocked at its provider, failures covered, driven over HTTP):**

```typescript
// test/e2e/shipment/shipment.spec.ts
import { AxiosError, AxiosResponse } from 'axios';
import request from 'supertest';

const ORDER_ID = '00000000-0000-4000-8000-000000000002';
const SHIPMENT_ID = '00000000-0000-4000-8000-000000000004';
const TRACKING_NUMBER = '1Z999AA10123456784';

describe('Shipments', () => {
  let app: INestApplication;
  const httpService = { get: jest.fn() };
  const shipmentRepository = { findOne: jest.fn() };
  const orderService = { findById: jest.fn() };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ApiModule.forRoot()],
      controllers: [ShipmentController],
      providers: [
        ShipmentService,
        CarrierClient,
        { provide: HttpService, useValue: httpService },
        { provide: getRepositoryToken(Shipment), useValue: shipmentRepository },
        { provide: OrderService, useValue: orderService },
        { provide: ClsService, useValue: { getId: () => 'test-request-id' } },
        { provide: APP_GUARD, useValue: testAuthGuard(() => AUTH_USER) },
      ],
    }).compile();
    app = await createTestApp(moduleRef);
  });

  afterAll(() => app.close());
  beforeEach(() => {
    jest.clearAllMocks();
    shipmentRepository.findOne.mockResolvedValue({ id: SHIPMENT_ID, orderId: ORDER_ID, trackingNumber: TRACKING_NUMBER });
    orderService.findById.mockResolvedValue({ id: ORDER_ID, userId: AUTH_USER.id, status: OrderStatus.Shipped });
  });

  describe('GET /shipments/by-tracking-number/:trackingNumber', () => {
    it('returns the tracking and passes the request id on', async () => {
      httpService.get.mockReturnValue(of({ status: 200, data: { status: 'in_transit', estimatedDeliveryAt: '2026-11-20T18:00:00.000Z' } }));

      const res = await request(app.getHttpServer()).get(`/shipments/by-tracking-number/${TRACKING_NUMBER}`);

      expect(res.status).toBe(200);
      expect(res.body.data).toEqual({ status: 'in_transit', estimatedDeliveryAt: '2026-11-20T18:00:00.000Z' });
      expect(httpService.get).toHaveBeenCalledWith(
        `/trackings/${TRACKING_NUMBER}`,
        expect.objectContaining({ headers: { 'x-request-id': 'test-request-id' } }),
      );
    });

    it("returns 404 for a tracking number of another customer's order", async () => {
      orderService.findById.mockRejectedValue(new NotFoundException(`Order #${ORDER_ID} not found`));

      const res = await request(app.getHttpServer()).get(`/shipments/by-tracking-number/${TRACKING_NUMBER}`);

      expect(res.status).toBe(404);
      expect(res.body.message).not.toContain(ORDER_ID);
      expect(orderService.findById).toHaveBeenCalledWith(AUTH_USER, ORDER_ID);
      expect(httpService.get).not.toHaveBeenCalled();
    });

    it('returns 404 for a tracking number the carrier does not know', async () => {
      httpService.get.mockReturnValue(of({ status: 404, data: { message: 'Unknown tracking number' } }));

      const res = await request(app.getHttpServer()).get(`/shipments/by-tracking-number/${TRACKING_NUMBER}`);

      expect(res.status).toBe(404);
    });

    it('returns 503 when the carrier times out', async () => {
      httpService.get.mockReturnValue(throwError(() => new AxiosError('timeout of 5000ms exceeded', 'ECONNABORTED')));

      const res = await request(app.getHttpServer()).get(`/shipments/by-tracking-number/${TRACKING_NUMBER}`);

      expect(res.status).toBe(503);
    });

    it('returns 503 with Retry-After when the carrier rate-limits us', async () => {
      const response = { status: 429, data: { message: 'Rate limited' }, headers: { 'retry-after': '30' } } as unknown as AxiosResponse;
      httpService.get.mockReturnValue(throwError(() => new AxiosError('Request failed with status code 429', 'ERR_BAD_REQUEST', undefined, undefined, response)));

      const res = await request(app.getHttpServer()).get(`/shipments/by-tracking-number/${TRACKING_NUMBER}`);

      expect(res.status).toBe(503);
      expect(res.headers['retry-after']).toBe('30');
    });
  });
});

// test/helpers/mock-stripe.ts
export function createMockStripe() {
  return { paymentIntents: { create: jest.fn(), retrieve: jest.fn() } };
}

// test/unit/auth/is-refresh-token-expired.spec.ts
describe('isRefreshTokenExpired', () => {
  beforeEach(() => {
    jest.useFakeTimers();
    jest.setSystemTime(new Date('2026-01-15T10:30:00.000Z'));
  });

  afterEach(() => jest.useRealTimers());

  it('is expired one second after expiresAt', () => {
    expect(isRefreshTokenExpired(new Date('2026-01-15T10:29:59.000Z'))).toBe(true);
  });
});
```

In the shipment spec, `ShipmentService` and `CarrierClient` are the real ones: the client's failure mapping is what these tests cover. The lookup is the caller's own: before calling the carrier, the service finds the shipment in its repository and reads its order through the order module's `OrderService`, mocked here, whose `findById(authUser, id)` answers 404 for another customer's order. The service answers that 404 without the order's id. `createMockStripe()` is a factory for a large SDK with only the methods the code uses, provided as `{ provide: Stripe, useValue: createMockStripe() }`. The last spec tests a pure function with frozen time.

Reference: [Vitest Mocking](https://vitest.dev/guide/mocking) · [Jest Mock Functions](https://jestjs.io/docs/mock-functions) · [NestJS Testing](https://docs.nestjs.com/fundamentals/testing)
