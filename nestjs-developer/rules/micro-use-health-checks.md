---
title: Expose Liveness and Readiness Probes with Terminus
impact: MEDIUM-HIGH
impactDescription: The orchestrator restarts a stuck instance and routes traffic only to ready ones
tags: microservices, health-checks, terminus, kubernetes
---

## Expose Liveness and Readiness Probes with Terminus

Expose two probes with `@nestjs/terminus`, in a `health` module:

- **Liveness** (`GET /health/live`): is the process able to serve at all? Check nothing external. A database outage must not make the orchestrator restart every instance.
- **Readiness** (`GET /health/ready`): can this instance take traffic now? Check the dependencies a request cannot do without (the database), and fail while shutting down (see `devops-graceful-shutdown`). A dependency the app falls back from stays out, such as the rate-limit counters in Redis (see `security-rate-limiting`): every instance would report not ready at once and the load balancer would take them all out of service, for an outage that requests survive.

**The database ping uses a connection of its own, outside the app's pool.** A ping through the pool waits for a free connection like any query, so a pool full of transactions waiting on one hot row fails readiness, and since the same row is hot on every instance, every instance leaves the load balancer at once, for a state that requests survive with a 503 and `Retry-After` (see `error-use-exception-filters`). The same reasoning keeps Redis out of readiness in an app that falls back from it. `HealthDatabaseService` holds one connection (a `pg` pool of one, with the app's `db` settings) and answers whether the database itself answers.

The health controller opts out of the app's global behavior that does not apply to probes:

- `@Public()`: the orchestrator sends no token (see `security-use-guards`).
- `@SkipRateLimits()`: probes run every few seconds, and a probe that went through the rate limiter would also depend on the limiter's storage, so a Redis outage would fail liveness on every instance. It skips every throttler in `API_THROTTLERS`; a bare `@SkipThrottle()` skips only a throttler named `default`, which an app with named throttlers does not have (see `security-rate-limiting`). The probes' e2e spec builds `ThrottlerModule` from the same `API_THROTTLERS` and sends more probes than the shortest limit allows, so a skip that misses a throttler fails it.
- `version: VERSION_NEUTRAL` when URI versioning is enabled, so the probe path stays stable (see `api-versioning`).
- `@ApiExcludeController()`: the probes are not part of the API's contract (see `api-document-with-openapi`).
- `@UseFilters(HealthUnavailableFilter)`: a filter scoped to the controller answers its 503s. The global filter would also log every failing probe as a failed request with its stack, every few seconds and on every deploy's drain (see `error-use-exception-filters`), while Terminus already logs which check failed. A filter on the controller takes the exception before the global one.

A dependency without a built-in indicator gets a small service of the health module (`HealthRedisService`, see `arch-resource-naming`), built with `HealthIndicatorService` and returning `up()` or `down()`. So does the shutdown state (`HealthShutdownService.isServing()`), so `ready()` makes one call and the controller orchestrates nothing (see `arch-single-responsibility`). Terminus 12 removed the old `HealthIndicator` base class and `HealthCheckError`: an indicator no longer reports failure by throwing. Give a built-in check its timeout with `.withTimeout(ms)`; the `timeout` option is deprecated. On NestJS 11 (Terminus 11), checks return a promise with no `.withTimeout()`: pass `{ timeout: 1_000 }` as the second argument instead.

**Incorrect (static OK, or a probe that checks everything):**

```typescript
@Controller('health')
export class HealthController {
  @Get()
  check(): string {
    return 'OK'; // Healthy while the database is unreachable
  }

  @Get('live')
  async live(): Promise<string> {
    await this.externalApi.ping(); // A third-party outage restarts every instance
    return 'OK';
  }
}
```

**Correct (separate probes, dependency checks only on readiness):**

```typescript
// health/controllers/health.controller.ts
@Public()
@SkipRateLimits()
@ApiExcludeController()
@UseFilters(HealthUnavailableFilter)
@Controller({ path: 'health', version: VERSION_NEUTRAL })
export class HealthController {
  constructor(
    private readonly healthCheckService: HealthCheckService,
    private readonly healthDatabaseService: HealthDatabaseService,
    private readonly healthShutdownService: HealthShutdownService,
  ) {}

  @Get('live')
  @HealthCheck()
  live(): Promise<HealthCheckResult> {
    return this.healthCheckService.check([]);
  }

  @Get('ready')
  @HealthCheck()
  ready(): Promise<HealthCheckResult> {
    return this.healthCheckService.check([
      () => this.healthShutdownService.isServing('shutdown'),
      () => this.healthDatabaseService.isHealthy('database'),
    ]);
  }
}

// health/services/health-database.service.ts
const PING_TIMEOUT_MS = 1_000;

@Injectable()
export class HealthDatabaseService implements OnApplicationShutdown {
  private readonly logger = new Logger(HealthDatabaseService.name);
  private readonly pool: Pool;

  constructor(
    private readonly healthIndicatorService: HealthIndicatorService,
    @Inject(dbConfig.KEY) dbSettings: ConfigType<typeof dbConfig>,
  ) {
    this.pool = new Pool({
      host: dbSettings.host,
      port: dbSettings.port,
      user: dbSettings.username,
      password: dbSettings.password,
      database: dbSettings.name,
      ssl: dbSettings.ssl ? { ca: dbSettings.sslCa } : false,
      max: 1,
      connectionTimeoutMillis: PING_TIMEOUT_MS,
      query_timeout: PING_TIMEOUT_MS,
    });
    // An idle connection the database ends: without a listener, pg's error event ends the process
    this.pool.on('error', (error) => this.logger.warn({ err: error }, 'The database ended the health connection'));
  }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    const indicator = this.healthIndicatorService.check(key);
    try {
      await this.pool.query('SELECT 1');
      return indicator.up();
    } catch (error) {
      return indicator.down({ message: error instanceof Error ? error.message : 'Database unreachable' });
    }
  }

  async onApplicationShutdown(): Promise<void> {
    await this.pool.end();
  }
}

// health/services/health-redis.service.ts
@Injectable()
export class HealthRedisService {
  constructor(
    private readonly healthIndicatorService: HealthIndicatorService,
    private readonly redisService: RedisService,
  ) {}

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    const indicator = this.healthIndicatorService.check(key);
    try {
      await this.redisService.ping();
      return indicator.up();
    } catch (error) {
      return indicator.down({ message: error instanceof Error ? error.message : 'Redis unreachable' });
    }
  }
}

// health/filters/health-unavailable.filter.ts
@Catch(ServiceUnavailableException)
export class HealthUnavailableFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: ServiceUnavailableException, host: ArgumentsHost): void {
    // Terminus puts the failed checks under error: { shutdown: { status: 'down' }, database: { ... } }
    const failed = Object.keys((exception.getResponse() as { error?: object }).error ?? {});
    const message = failed.length > 0 ? `Not ready: ${failed.join(', ')}` : 'Not ready';

    this.httpAdapterHost.httpAdapter.reply(
      host.switchToHttp().getResponse(),
      {
        success: false,
        statusCode: HttpStatus.SERVICE_UNAVAILABLE,
        error: 'Service Unavailable',
        message,
        errorCode: 'SERVICE_UNAVAILABLE',
      },
      HttpStatus.SERVICE_UNAVAILABLE,
    );
  }
}
```

`live()` checks nothing: an answer from the process is all liveness means. `isServing('shutdown')` is down from the moment shutdown starts. `HealthDatabaseService` keeps its one connection outside the app's pool, for the reason given above; its one-second ping timeout is an example. `HealthRedisService` is the custom indicator for an app whose requests cannot do without Redis: `ready()` then adds `() => this.healthRedisService.isHealthy('redis')`, and `RedisService.ping()` sends `PING` on the client that runs the app's commands. `HealthUnavailableFilter` answers the probes' 503s, so the global filter never logs them.

The Kubernetes probes for these routes:

```yaml
livenessProbe:
  httpGet: { path: /health/live, port: 3000 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /health/ready, port: 3000 }
  periodSeconds: 5
  failureThreshold: 2
startupProbe:
  httpGet: { path: /health/live, port: 3000 }
  periodSeconds: 5
  failureThreshold: 30
```

With a global prefix (`app.setGlobalPrefix('api')`), either exclude the health routes from it or use the prefixed path in the probes.

### With Prisma

With the pg adapter, Prisma opens no connection at startup: `$connect()` resolves at once, and the first query fails with `P1001` while the database is down (see `db-prisma-schema-conventions`). The app therefore starts without a database, where TypeORM retried and then exited. That is the intended behavior: readiness checks the database and answers 503 until it answers, so the orchestrator routes no traffic to the instance, and liveness stays up, so it does not restart the instance in a loop while the database is away. Do not add a connection check to liveness or a startup query that exits to restore the old behavior.

Readiness checks the database with the same `HealthDatabaseService` (`pg` is a direct dependency with Prisma too, see `references/project-setup.md`), not with Terminus's `PrismaHealthIndicator`, which pings through the client it is given, so through the app's pool. A spec of code that still uses that indicator mocks it as it runs: Terminus 12.2.0 calls `$runCommandRaw({ ping: 1 })` first and falls back to `$queryRawUnsafe('SELECT 1')` only when that call fails with a message containing "Use the mongodb provider", which Prisma 7.10.1 throws on PostgreSQL (`PrismaClientValidationError: The postgresql provider does not support $runCommandRaw. Use the mongodb provider.`). A mock whose `$runCommandRaw` resolves, or rejects with any other error, never reaches `SELECT 1`.

Reference: [NestJS Terminus](https://docs.nestjs.com/recipes/terminus)
