---
title: Use Structured Logging
impact: MEDIUM-HIGH
impactDescription: Structured logging enables effective debugging and monitoring
tags: devops, logging, structured-logs, pino, correlation
---

## Use Structured Logging

Log structured JSON through `nestjs-pino`, and never use `console.log`. Every line should carry the request's correlation fields automatically, so a request can be followed across log lines and services by filtering on a field.

- **Hand Nest the pino logger at bootstrap**: create the app with `bufferLogs: true`, then call `app.useLogger(app.get(Logger))`. Without this, lines logged during startup (module init, dependency injection errors) are printed by Nest's default text logger before pino takes over, and those are exactly the lines you need when a deploy fails.
- **Application code keeps using `new Logger(MyService.name)`** from `@nestjs/common`. It delegates to pino once `useLogger` is set, so services never import pino.
- **One `LoggerModule`** in `src/logger/logger.module.ts` registers the request context (`ClsModule`) and pino (nestjs-pino's own `LoggerModule`, imported as `PinoLoggerModule`), and `AppModule` imports it (see `arch-module-sharing`). The functions that build their options live in `logger/utils/`, where unit tests import them.
- **Correlation IDs are fields, not text.** pino-http binds the request's id to the request's logger as `requestId` (`customAttributeKeys` renames its `reqId`), so every line of the request carries it. pino-http reuses `req.id` when the adapter has set it (Fastify), and otherwise calls its `genReqId`, which returns `cls.getId()` (Express). Without that, pino-http adds its own `reqId`, a counter per process that repeats across instances and restarts. This needs the request context to exist before pino-http runs, which holds because `LoggerModule` registers `ClsModule` first; an e2e spec of the log line guards it. Never append them to the message string; filter on the field instead (`filter requestId = "..."`). The id is the caller's `x-request-id` when it is a short plain token and a new UUID otherwise (`resolveRequestId()`), since it lands in every log line and Sentry event of the request. On Fastify the adapter assigns it (`genReqId` in `ApiModule.createFastifyAdapter()`, see `api-platform-agnostic`) and the request context takes it from `req.id`; on Express nothing sets `req.id`, so the request context resolves the header itself. Its `setup` hook returns the id to clients in the `x-request-id` response header, so they can quote it. On Express, a request the body parser rejects (malformed JSON) is answered before the request context exists, so its 400 carries no id, and a README that documents the header says so. Fastify parses the body after the middleware, so every response carries it.
- **One threshold, `APP_LOG_LEVEL`** (`app.logLevel`, see `devops-use-config-module`): pino writes that level and every level above it, in the order `trace`, `debug`, `info`, `warn`, `error`, `fatal`. The default is `info`. Each environment sets its own without a code change: `error` for errors only, `debug` for the query logs below, `trace` for everything, including Nest's `logger.verbose()`. `validateEnv` refuses any other value, `silent` included, so a deployment cannot turn its logs off by mistake.
- **Database queries go through the same logger, without their values.** With `db.logging` on, TypeORM's default logger prints every query and its bound values with `console`, past pino and its redaction: a sign-up would print the password hash. The TypeORM options set `logger: new LoggerTypeOrmQueryLogger()` (below), which writes each query at `debug`; the Prisma client does the same in `PrismaService` (see `db-prisma-schema-conventions`). So `DB_LOGGING=true` shows queries only with `APP_LOG_LEVEL=debug` or lower.
- **JSON by default, pretty output only when asked**: single-line `pino-pretty` only when `app.logPretty` is on (`APP_LOG_PRETTY=true`, which `.env.example` sets for a local run, see `devops-use-config-module`). Every deployed environment leaves it unset and logs JSON. Never derive it from `APP_ENV`: a local run and the deployed `dev` share that value.
- **Errors go in pino's `err` field**: `this.logger.error({ err: error }, message)`. Nest's `Logger` hands an object first argument to pino as the line's fields, so pino's error serializer writes the stack with every cause (see `error-use-exception-filters`).
- **Request lines record who, not what was asked.** Each request's line carries the caller's id and the client address as fields, and the path without its query string, whose search terms and emails would land in every line (see `api-use-interceptors`).
- **Redact secrets** (authorization headers, passwords, tokens) in the pino config, and never log request bodies wholesale. A logged `QueryFailedError` carries the query's bound values (`parameters`) and PostgreSQL's `detail`, which quotes row values: redact those too. An HTTP exception keeps its constructor options, its `cause` included, so `err.options` would print a wrapped error's own fields (an axios request's headers and body, a query's bound values): redact it, and the cause's message and stack still reach `err.message` and `err.stack`. A secret inside a message or a stack has no path, so every line also passes through `SentryService.redactSecrets()` in pino's `hooks.streamWrite` (see `error-report-to-sentry`).
- **The one `console` exception: process crash handlers.** `src/sentry/preload.ts`, which starts Sentry before the app (see `error-report-to-sentry`), installs the `uncaughtException` and `unhandledRejection` handlers before the app's first module loads. They write a fixed marker with `console.error`, on purpose: they must work even when the logger is what failed, and alerting can match the literal marker. They print the error through `SentryService.redactSecrets()`, then flush Sentry with `SentryService.close()` and exit with code 1. Both exit: since Node 15 an unhandled rejection ends the process, and a handler that only logs would turn that off and keep a process running in an unknown state. Do not convert them to the structured logger.

**Incorrect (console logging, ids inside messages, logger set up too late):**

```typescript
@Injectable()
export class UserService {
  async create(dto: CreateUserDto): Promise<User> {
    console.log('Creating user:', dto);                       // Unstructured, logs the whole body
    const user = await this.userRepository.save(await this._toEntity(dto));
    this.logger.log(`[${this.cls.getId()}] User ${user.id} created`); // requestId baked into text
    return user;
  }
}

console.log('Login attempt:', { email, password });          // Secret in the logs

// main.ts - no bufferLogs: startup lines bypass pino
const app = await NestFactory.create(AppModule);
app.useLogger(app.get(Logger));
```

**Correct (pino bound to the request context, Nest Logger in app code):**

```typescript
// logger/logger.module.ts
@Module({
  imports: [
    ClsModule.forRoot({
      global: true,
      middleware: {
        mount: true,
        generateId: true,
        // Fastify sets req.id before this middleware runs; Express does not
        idGenerator: (req: IncomingMessage & { id?: string }) => req.id ?? resolveRequestId(req.headers['x-request-id']),
        setup: (cls: ClsService, req: IncomingMessage, res: ServerResponse) => {
          SentryService.setRequestId(cls.getId());
          res.setHeader('x-request-id', cls.getId());
        },
      },
    }),
    PinoLoggerModule.forRootAsync({
      inject: [ClsService, appConfig.KEY],
      useFactory: buildPinoParams,
    }),
  ],
})
export class LoggerModule {}

// logger/utils/build-pino-params.ts
export function buildPinoParams(cls: ClsService, appSettings: ConfigType<typeof appConfig>): Params {
  return {
    pinoHttp: {
      autoLogging: false,
      base: undefined,
      quietReqLogger: true,
      customAttributeKeys: { reqId: 'requestId' },
      genReqId: () => cls.getId(),
      redact: [
        'req.headers.authorization',
        'req.headers.cookie',
        '*.password',
        '*.token',
        // A QueryFailedError copies the query's bound values and PostgreSQL's detail onto itself
        'err.parameters',
        'err.detail',
        'err.driverError.detail',
        // An HttpException keeps its options, its cause whole
        'err.options',
      ],
      hooks: { streamWrite: (line: string) => SentryService.redactSecrets(line) },
      transport: appSettings.logPretty ? { target: 'pino-pretty', options: { singleLine: true } } : undefined,
      level: appSettings.logLevel,
    },
  };
}

// logger/utils/logger-typeorm-query-logger.ts
export class LoggerTypeOrmQueryLogger implements TypeOrmLogger {
  private readonly logger = new Logger('TypeORM');

  logQuery(query: string): void {
    this.logger.debug({ query });
  }

  logQueryError(error: string | Error, query: string): void {
    this.logger.debug({ query, error: error instanceof Error ? error.message : error });
  }

  logQuerySlow(durationMs: number, query: string): void {
    this.logger.warn({ query, durationMs });
  }

  logSchemaBuild(message: string): void {
    this.logger.debug(message);
  }

  logMigration(message: string): void {
    this.logger.log(message);
  }

  log(level: 'log' | 'info' | 'warn', message: unknown): void {
    if (level === 'warn') {
      this.logger.warn(message);
    } else {
      this.logger.log(message);
    }
  }
}

// logger/utils/resolve-request-id.ts
const REQUEST_ID_PATTERN = /^[\w.-]{1,128}$/;

export function resolveRequestId(header: string | string[] | undefined): string {
  return typeof header === 'string' && REQUEST_ID_PATTERN.test(header) ? header : randomUUID();
}

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [appConfig, dbConfig], validate: validateEnv }),
    LoggerModule,
    // ...
  ],
})
export class AppModule {}

// src/sentry/preload.ts
process.on('uncaughtException', (error) => {
  // console, not the logger: this must work when the logger is what failed
  console.error('[users-api-alert] uncaught exception:', SentryService.redactSecrets(String(error.stack ?? error)));
  void SentryService.close().finally(() => process.exit(1));
});
process.on('unhandledRejection', (reason) => {
  console.error('[users-api-alert] unhandled rejection:', SentryService.redactSecrets(String(reason instanceof Error ? reason.stack : reason)));
  // Exits as Node does by default: the process is in an unknown state
  void SentryService.close().finally(() => process.exit(1));
});

// main.ts

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(AppModule, ApiModule.createFastifyAdapter(appConfig().trustedProxies), {
    bufferLogs: true,
  });
  app.useLogger(app.get(Logger));
  // ... rest of bootstrap
}

@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(await this._toEntity(dto));
    this.logger.log(`User ${user.id} created`);
    return user;
  }
}
```

In `LoggerModule`, the request context's `setup` hook puts the same id on the request's Sentry events. In `buildPinoParams()`, `autoLogging` is off because `ApiLoggerInterceptor` logs successful requests and the exception filter logs failures; `base: undefined` drops pino's `pid` and `hostname` fields; `quietReqLogger` makes the request's logger bind only its id, not the whole request. `LoggerTypeOrmQueryLogger` logs a failed query at `debug` only: the query also reaches the exception filter, which logs it once with its stack when it is a 5xx. In `src/sentry/preload.ts`, the crash handlers come after `SentryService.init()` (see `error-report-to-sentry`). In `main.ts`, `Logger` is nestjs-pino's, the rest of the bootstrap is in `api-platform-agnostic`, and an Express app is created with `NestFactory.create(AppModule, { bufferLogs: true })`. Application code logs the message only, with Nest's `Logger`: `requestId` comes from the request's logger. `UserService._toEntity()` hashes the password (see `security-hash-passwords`).

To follow one request, filter on its id in CloudWatch Logs Insights or any JSON log search:

```text
fields @timestamp, level, msg
| filter requestId = "3f0c..."
| sort @timestamp asc
```

### With Prisma

Prisma's errors carry values in other places than TypeORM's:

- **PostgreSQL's `detail`** (a CHECK violation's "Failing row contains (...)", every column of the row) is under `meta.driverAdapterError.cause.detail`. pino's serializer writes each error of an `AggregateError` under `aggregateErrors`, with its own fields, where the path for the top-level error does not reach: redact both paths. A wrapped cause's fields are not written, only its message and stack.
- **A `PrismaClientValidationError`** (an argument of the wrong type, or an `undefined` that `strictUndefinedChecks` refused) prints every argument of the call in its message and stack, values included: a failed `user.updateMany()` printed both password hashes, and `errorFormat: 'minimal'` prints the same. No redact path reaches text inside a message, so the `err` serializer keeps its last paragraph, which names the argument and the reason (``Argument `email`: Invalid value provided. Expected String, provided Int.``), and the stack's frames. Sentry gets the same (see `error-report-to-sentry`). A validation error is a bug in the call: let it reach the global filter as it is, never wrapped as another error's cause, whose message and stack the serializer appends whole.

```typescript
// logger/utils/build-pino-params.ts
redact: [
  'req.headers.authorization',
  'req.headers.cookie',
  '*.password',
  '*.token',
  'err.meta.driverAdapterError.cause.detail',
  'err.aggregateErrors[*].meta.driverAdapterError.cause.detail',
  'err.options',
],
serializers: { err: serializeLogError },

// logger/utils/serialize-log-error.ts
// pino-http hands its err serializer the error pino already serialized
export function serializeLogError(serialized: SerializedError): SerializedError {
  if (serialized.type !== 'PrismaClientValidationError') {
    return serialized;
  }
  const reason = serialized.message.trim().split('\n\n').pop() ?? '';
  const frames = serialized.stack.split('\n').filter((line) => line.startsWith('    at '));
  return { ...serialized, message: reason, stack: [`${serialized.type}: ${reason}`, ...frames].join('\n') };
}
```

Reference: [NestJS Logger](https://docs.nestjs.com/techniques/logger) · [nestjs-pino](https://github.com/iamolegga/nestjs-pino)
