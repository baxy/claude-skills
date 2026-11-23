---
title: Implement Rate Limiting
impact: HIGH
impactDescription: Protects against abuse and ensures fair resource usage
tags: security, rate-limiting, throttler, protection
---

## Implement Rate Limiting

Use `@nestjs/throttler` to limit request rates per client, with stricter limits on endpoints that check credentials, send messages or tell whether an account exists (logins, password resets, invitations, and sign-up, whose 409 says an email is taken). A route that checks a secret the caller supplies (a card number, a gift card code, a one-time code) is a credential check and gets a per-account limit like a login's (in the shop: the pay route, for example 5 in 10 minutes). Without one, a card thief tests stolen cards against it.

A route that lets a caller hold a scarce shared resource before committing to it (a reservation, a hold) is also limited by count per account (in the shop: at most 3 unpaid orders per customer, since an order reserves stock until it is paid; see `db-use-transactions` for counting them safely). A rate limit alone lets one account keep every unit held by placing new holds as the old ones expire.

Register `ThrottlerGuard` globally (an `APP_GUARD` after the authentication guards, see `security-use-guards`) so the tracker can use the caller's id. The limits are one list, `API_THROTTLERS` in the api module's `constants/`, which `ThrottlerModule` is built from. A provider's callback (Stripe's events, see `api-controller-conventions`) is limited too, with a limit of its own, generous enough for the provider's bursts (for example 50 a second and 1000 a minute per address): with none, anyone can send it bodies to parse before the signature check. A route that must never be limited, such as the health probes (see `micro-use-health-checks`), takes `@SkipRateLimits()`, which skips every throttler in that list by name. A bare `@SkipThrottle()` skips only a throttler named `default`, so with named throttlers it skips nothing, and listing the names by hand stops working the day a throttler is added. Behind a load balancer, the client IP is only correct once exactly the load balancer's addresses are trusted (`APP_TRUSTED_PROXIES`, see `api-platform-agnostic`): trusting every proxy lets a client pick its own address and its own limit.

A request with an invalid token gets its 401 from the authentication guard, which runs first, so the throttler never counts it. That is acceptable: refusing it costs one signature check and no query.

**The guard, `AuthThrottlerGuard`, overrides two methods of `ThrottlerGuard`, three in an app with a gateway or message handlers (`shouldSkip()`, which skips every context that is not HTTP, see `security-use-guards`), and nothing else:**

- **Its tracker** is the caller's id when there is one, and otherwise the library's own tracker (`super.getTracker()`), never the raw address. From 6.7.0 the library groups IPv6 addresses by their /64 (`normalizeIp()`), since one client holds a whole /64 and can send each request from another address in it; a tracker that returns `request.ip` gives each of those requests a fresh count. Checked on 6.8.0: six requests from six addresses of one /64 passed a limit of 3 with the raw address, and the fourth was refused with the library's tracker. A count kept outside the guard (the queue dashboard's, see `micro-use-queues`) keys an anonymous caller with the same `normalizeIp()`.
- **Its 429 carries the delay**, so the global filter writes `Retry-After` (see `error-use-exception-filters`). The library names the header after the throttler, `Retry-After-{name}`, for every throttler but one named `default`, and with named throttlers (`short`, `long`) no client reads it. `throwThrottlingException()` throws an `HttpException` with `retryAfterSeconds` from `timeToBlockExpire`, which both storages give in whole seconds, rounded up. Checked on 6.8.0: the 429 had `Retry-After-long` and no `Retry-After` before the override, and both after it.

**A login is also limited per account.** The limit per address does not stop someone with many addresses from guessing one account's password. `AuthSessionService.create()` therefore counts every attempt against the email, before checking the password, in the throttler's own storage (`getStorageToken()`, which `ThrottlerModule` provides globally): a limit per window from the auth module's `constants/` (10 attempts in 15 minutes in the example), keyed by the SHA-256 of the normalized email, so Redis holds no email. Past the limit it answers 429 with `Retry-After`, to the right password too. The block lasts the window: the Redis storage refuses a block duration of 0. The trade-off is that anyone can lock an account's logins for one window; that is temporary, while a guessed password is not.

**Asking for a reset link, where the app offers one, is counted per email the same way** (`auth-password-reset:{sha256(email)}`, with a limit of its own from the auth module's `constants/`), and past the limit it still answers 202, with no mail sent, so the answer stays the same for every email.

**The counters live in memory, and in Redis only when the spec requires it.** By default each instance counts in its own memory, the throttler's default storage, so with several instances every limit holds per instance, which the README states. When the spec requires the limits to hold across instances, the counters are kept in Redis (`@nest-lab/throttler-storage-redis`), and the example below is that form. Redis holds nothing but counters here, so its outage must not fail requests. The storage, `ApiThrottlerStore` in `api/stores/`, catches a failed Redis call and counts in the instance's memory instead, warning at most once a minute. After a failure it counts in memory for a few seconds (5 in the example) before it tries Redis again: a Redis that keeps the connection open and stops answering fails each call only after the client's command timeout, so trying it on every request would add that timeout to every request, once per throttler. The limits are then per instance for the length of the outage, which still slows guessing; refusing requests instead would turn a Redis outage into a full one. Where `REDIS_HOST` is not set (a local run of an app with no queues), the store counts in memory from the start. An app that has Redis for its queues keeps its counters in memory unless the spec requires them shared.

The Redis client comes from a `redis` module that provides `REDIS_CLIENT`. It fails a command at once while disconnected (`enableOfflineQueue: false`), so a request never waits on a Redis that is down, and it logs when Redis becomes unavailable and when it is back, not every reconnection attempt. That client serves the cache (see `perf-use-caching`) and, when they are in Redis, the counters; a BullMQ connection keeps its offline queue (see `micro-use-queues`). An app that also publishes or subscribes gets those connections from the same module, through `RedisService` (see `api-real-time-updates`).

**Every limit of its own has a test** in the route's e2e spec: the request one past the limit answers 429 with `Retry-After`. A limit raised by mistake from 5 to 50 passes every other test.

**Incorrect (no limits on credential endpoints, one limit for everything):**

```typescript
@Controller('auth')
export class AuthController {
  @Public()
  @HttpCode(HttpStatus.OK)
  @Post('login')
  login(@Body() dto: CreateAuthSessionDto): Promise<AuthSessionResponseDto> {
    return this.authSessionService.create(dto); // Unlimited: credentials can be brute-forced
  }

  @Public()
  @HttpCode(HttpStatus.ACCEPTED)
  @Post('forgot-password')
  forgotPassword(@Body() dto: CreateAuthPasswordResetDto): Promise<void> {
    return this.authPasswordResetService.create(dto); // Can be used to flood a user with emails
  }
}
```

**Correct (global throttler, stricter limits on credential endpoints):**

```typescript
// api/constants/api-throttlers.constant.ts
export const API_THROTTLERS: ThrottlerOptions[] = [
  { name: 'short', ttl: 1_000, limit: 10 },
  { name: 'long', ttl: 60_000, limit: 300 },
];

// api/decorators/skip-rate-limits.decorator.ts
// A bare @SkipThrottle() skips only a throttler named 'default'
export const SkipRateLimits = () =>
  SkipThrottle(Object.fromEntries(API_THROTTLERS.map(({ name }) => [name ?? 'default', true])));

// api/stores/api-throttler.store.ts
const REDIS_RETRY_DELAY_MS = 5_000;

export class ApiThrottlerStore implements ThrottlerStorage, OnApplicationShutdown {
  private readonly logger = new Logger(ApiThrottlerStore.name);
  private readonly memory = new ThrottlerStorageService();
  private readonly redis: ThrottlerStorageRedisService | null;
  private redisRetryAt = 0;
  private lastWarnedAt = Number.NEGATIVE_INFINITY;

  constructor(redisClient: Redis | null) {
    this.redis = redisClient ? new ThrottlerStorageRedisService(redisClient) : null;
  }

  async increment(key: string, ttl: number, limit: number, blockDuration: number, throttlerName: string) {
    if (this.redis && Date.now() >= this.redisRetryAt) {
      try {
        return await this.redis.increment(key, ttl, limit, blockDuration, throttlerName);
      } catch (error) {
        // A hung Redis costs one command timeout per delay
        this.redisRetryAt = Date.now() + REDIS_RETRY_DELAY_MS;
        this._warnFallback(error);
      }
    }
    return this.memory.increment(key, ttl, limit, blockDuration, throttlerName);
  }

  onApplicationShutdown(): void {
    // Not a provider, so Nest never clears its expiry timers
    this.memory.onApplicationShutdown();
  }

  private _warnFallback(error: unknown): void {
    if (Date.now() - this.lastWarnedAt < 60_000) {
      return;
    }
    this.lastWarnedAt = Date.now();
    this.logger.warn({ err: error }, 'Redis failed: rate limits are counted in this instance until it answers again');
  }
}

// redis/redis.module.ts
@Module({
  providers: [
    {
      provide: REDIS_CLIENT,
      inject: [redisConfig.KEY],
      useFactory: (redisSettings: ConfigType<typeof redisConfig>): Redis | null => {
        if (!redisSettings.host) {
          return null;
        }
        const client = new Redis({
          ...redisConnectionOptions(redisSettings),
          commandTimeout: 1_000,
          enableOfflineQueue: false,
        });
        // ioredis reports every reconnection attempt: log the change of state instead
        let available = true;
        client.on('error', (error: unknown) => {
          if (available) {
            available = false;
            logger.warn({ err: error }, 'Redis is unavailable');
          }
        });
        client.on('ready', () => {
          if (!available) {
            available = true;
            logger.log('Redis is available again');
          }
        });
        return client;
      },
    },
  ],
  exports: [REDIS_CLIENT],
})
export class RedisModule implements OnApplicationShutdown {
  constructor(@Inject(REDIS_CLIENT) private readonly redis: Redis | null) {}

  onApplicationShutdown(): void {
    this.redis?.disconnect();
  }
}

// app.module.ts
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [RedisModule],
      inject: [REDIS_CLIENT],
      useFactory: (redis: Redis | null) => ({ throttlers: API_THROTTLERS, storage: new ApiThrottlerStore(redis) }),
    }),
  ],
  providers: [
    { provide: APP_GUARD, useClass: AuthJwtGuard },
    { provide: APP_GUARD, useClass: AuthRolesGuard },
    { provide: APP_GUARD, useClass: AuthThrottlerGuard },
  ],
})
export class AppModule {}

// auth/guards/auth-throttler.guard.ts
@Injectable()
export class AuthThrottlerGuard extends ThrottlerGuard {
  protected async getTracker(request: ApiRequest): Promise<string> {
    // The library's tracker: an IPv6 address by its /64
    return request.user?.id ?? (await super.getTracker(request));
  }

  // The library sends only Retry-After-{throttler} for a named throttler
  protected async throwThrottlingException(context: ExecutionContext, detail: ThrottlerLimitDetail): Promise<void> {
    throw new HttpException(
      { message: 'Too many requests; try again later', retryAfterSeconds: detail.timeToBlockExpire },
      HttpStatus.TOO_MANY_REQUESTS,
    );
  }
}

// auth/constants/auth-rate-limits.constant.ts
export const AUTH_LOGIN_ADDRESS_LIMIT = 5;
export const AUTH_LOGIN_ADDRESS_WINDOW_MS = 60_000;
export const AUTH_LOGIN_ACCOUNT_LIMIT = 10;
export const AUTH_LOGIN_ACCOUNT_WINDOW_MS = 15 * 60 * 1000;
export const AUTH_PASSWORD_RESET_LIMIT = 3;
export const AUTH_PASSWORD_RESET_WINDOW_MS = 60 * 60 * 1000;

// auth/services/auth-session.service.ts
private async _countLoginAttempt(email: string): Promise<void> {
  const { isBlocked, timeToBlockExpire } = await this.throttlerStorage.increment(
    `auth-login:${this._hash(email)}`,
    AUTH_LOGIN_ACCOUNT_WINDOW_MS,
    AUTH_LOGIN_ACCOUNT_LIMIT,
    // The block duration: the Redis storage refuses 0
    AUTH_LOGIN_ACCOUNT_WINDOW_MS,
    'auth-login-account',
  );
  if (isBlocked) {
    throw new HttpException(
      { message: 'Too many login attempts for this account', retryAfterSeconds: timeToBlockExpire },
      HttpStatus.TOO_MANY_REQUESTS,
    );
  }
}

// user/constants/user-rate-limits.constant.ts
export const USER_SIGN_UP_LIMIT = 5;
export const USER_SIGN_UP_WINDOW_MS = 60_000;

// user/controllers/user.controller.ts
@Public()
@Throttle({ long: { limit: USER_SIGN_UP_LIMIT, ttl: USER_SIGN_UP_WINDOW_MS } })
@Post()
create(@Body() dto: CreateUserDto): Promise<User> {
  return this.userService.create(dto);
}

// auth/controllers/auth.controller.ts
@Controller('auth')
export class AuthController {
  @Public()
  @Throttle({ long: { limit: AUTH_LOGIN_ADDRESS_LIMIT, ttl: AUTH_LOGIN_ADDRESS_WINDOW_MS } })
  @HttpCode(HttpStatus.OK)
  @Post('login')
  login(@Body() dto: CreateAuthSessionDto): Promise<AuthSessionResponseDto> {
    return this.authSessionService.create(dto);
  }

  @Public()
  @Throttle({ long: { limit: AUTH_PASSWORD_RESET_LIMIT, ttl: AUTH_PASSWORD_RESET_WINDOW_MS } })
  @HttpCode(HttpStatus.ACCEPTED)
  @Post('forgot-password')
  forgotPassword(@Body() dto: CreateAuthPasswordResetDto): Promise<void> {
    return this.authPasswordResetService.create(dto);
  }
}
```

The numbers are examples. The `short` throttler limits bursts and `long` sustained use. `REDIS_RETRY_DELAY_MS` is how long the store counts in memory after a failure before it tries Redis again. `REDIS_CLIENT` is `null` without `REDIS_HOST`. Its `commandTimeout` bounds a connected but slow Redis, and `enableOfflineQueue: false` fails a command at once while Redis is disconnected, so the store falls back to memory; TLS is required outside `dev` (see `devops-use-config-module`). `RedisModule` disconnects the client after the HTTP server has closed (see `devops-graceful-shutdown`). The global exception filter writes the `Retry-After` header clients read from the guard's `retryAfterSeconds` (see `error-use-exception-filters`). The auth module keeps its limits in one place, its `constants/` (see `arch-feature-modules`), and `AuthController` serves the authentication flows (see `api-controller-conventions`). Sign-up's 409 says whether an email is taken, so it is limited like a login.

Reference: [NestJS Throttler](https://docs.nestjs.com/security/rate-limiting)
