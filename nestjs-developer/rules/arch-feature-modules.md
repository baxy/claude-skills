---
title: Organize by Feature Modules
impact: CRITICAL
impactDescription: Any module's layout is known before opening it, and every new file has one obvious home
tags: architecture, modules, organization, folders
---

## Organize by Feature Modules

Organize the application into feature modules that encapsulate related functionality. Each feature module owns its controllers, services, entities, DTOs and helpers. Never organize the whole app by technical layer (all controllers together, all services together).

Inside a module, **every file lives in a folder named for its type**. Only `{module}.module.ts` stays at the module root. The exceptions are two entry points that Node runs, named for what Node does with them: `sentry/preload.ts`, which `node --import` runs before `main.ts` to start Sentry and install the crash handlers (see `error-report-to-sentry`), and `cli/main.ts`, the command line's counterpart of `src/main.ts` (below). A configurable module has no `{module}.module-definition.ts`: its options type goes in `types/`, its options token in `constants/`, and the builder in `{module}.module.ts` (see `arch-use-dynamic-modules`). Type folders: `controllers/`, `services/`, `entities/`, `dtos/`, `types/`, `interfaces/`, `constants/`, `utils/`, `validators/`, `guards/`, `strategies/`, `decorators/`, `interceptors/`, `middlewares/` (Nest middleware, `.middleware.ts`), `filters/`, `errors/`, `events/` (event payload classes), `listeners/` (`@OnEvent` handlers), `processors/` (queue workers), `commands/` (command-line commands, `.command.ts`), `stores/` (storage adapters a library is handed), `workers/` and `queues/` (subclasses of a queue library's classes), `gateways/` (WebSocket gateways), `clients/` (calls to outside systems), `adapters/` (one outside system behind a module's interface), `proto/` (another service's gRPC contract). There is no `repositories/` folder: services use TypeORM's repositories directly (see `arch-data-access-in-services`).

- This is unconditional. A folder holding one file is not redundant: it makes the home of the next file obvious, and the layout of a module you have never opened is known before you look. A threshold ("add a folder once there are three controllers") stops being applied the moment nobody notices the crossing.
- `utils/` holds the module's pure logic: functions and rules that belong to this module, nothing with a side effect (an SDK call, I/O) and no state. The module's own files and tests import them, and so does another module that needs one, directly and without injection (the database module's error checks, the queue module's `addJobInTime()`, see `arch-module-sharing`). A function that only wraps a call it is handed (a time limit around it, such as the queue module's `addJobInTime()`, see `micro-use-queues`) does no I/O of its own and may live here. A function that only its own file uses stays in that file, unexported, as a private method or a file-local function. What may move to `src/shared/` is in `arch-shared-folder`.
- `types/` holds the module's exported types, one per file named after it: a `type` alias in a `.type.ts` file (`types/order-summary.type.ts` exports `type OrderSummary = { ... }`), an enum in an `.enum.ts` file (`types/order-status.enum.ts`, see `arch-resource-naming`). A type that only its own file uses stays in that file, unexported.
- `interfaces/` holds the module's interfaces, one per file with the `.interface.ts` suffix that the Nest CLI generates: `interfaces/payment-adapter.interface.ts`. An `interface` is only for a contract a class implements, an abstraction behind an injection token (`PaymentAdapter`, see `di-use-interfaces-tokens`); the data it passes around is a `type`. A second `interface` of the same name merges into the first without a word, while a second `type` is a compile error.
- `constants/` holds exported constants and injection tokens, one concern per file with the `.constant.ts` suffix: `constants/auth-rate-limits.constant.ts`, `constants/mail-options.constant.ts`. A named value that sets behavior the API's clients or operators see (a rate limit and its window, a session lifetime, a retention period) lives here, even while one file uses it, so that a module's limits are found in one place and not across its controllers and services. The test: the README or the API docs state the value. A value that only serves one file's implementation (a byte count, a pattern, an index name, an internal timeout) stays in that file (see `arch-resource-naming`). `src/config/` is no module and has no `constants/`, so the bounds `validate-env.ts` checks stay in that file.
- `validators/` holds custom `class-validator` decorators (`is-postal-code.validator.ts`).
- `stores/` holds the storage adapters a library is handed (a cache store, the throttler's storage), with the `.store.ts` suffix: `cache/stores/cache-redis.store.ts`, `api/stores/api-throttler.store.ts`. They keep state and are built with `new` in the module's options factory (see `di-prefer-constructor-injection`), so they are neither `utils/` nor providers.
- `workers/` holds the app's subclasses of a library's worker that the library builds itself (`queue/workers/queue.worker.ts`, the BullMQ `Worker` that `@nestjs/bullmq` builds for every processor, set through `BullModule.workerClass`, see `devops-graceful-shutdown`), with the `.worker.ts` suffix; the processors that do the jobs stay in their features' `processors/`. `queues/` holds the same for the library's queue (`queue/queues/queue-bounded.queue.ts`, set through `BullModule.queueClass`), with the `.queue.ts` suffix.
- `clients/` holds the classes that call an outside system: a provider's API, another service over HTTP or gRPC. A client owns the transport only (the requests, the key, the timeouts and the raw answers); what an answer means is not its business. It is named for the system it calls, not for the module: `StripeClient` in `payment/clients/stripe.client.ts` (see `arch-resource-naming`). An AWS service's client also names the vendor, `AwsS3Client` in `storage/clients/aws-s3.client.ts`, since the SDK's own client already has the plain name (`S3Client`).
- `adapters/` holds the classes that put one outside system behind an interface the module owns: one adapter per system, on top of that system's client. An adapter builds the system's requests from the module's types, checks every answer field by field and returns the module's shapes; its checks are its private methods, not `utils/`. It is named for the system too, `{System}Adapter` in `adapters/{system}.adapter.ts` (`StripeAdapter` in `payment/adapters/stripe.adapter.ts`, `AwsS3Adapter` in `storage/adapters/aws-s3.adapter.ts`), implements the module's interface (`PaymentAdapter`) and is bound to the module's token (`PAYMENT_ADAPTER`, see `di-use-interfaces-tokens`). A system behind two of the module's interfaces gets one adapter for each, named for the system and the role (`TwilioSmsAdapter`, `TwilioVoiceAdapter`). The next system adds a client and an adapter beside these, not a module. The api module's `adapters/` also holds the app's subclasses of Nest's own adapters (`ApiRedisIoAdapter extends IoAdapter`, see `api-real-time-updates`): the same word, the same folder.
- `proto/` holds a byte-identical copy of another service's `.proto` contract, at the path its owner keeps it, pinned by a unit test of its SHA-256. The build copies it into `dist/` (an `assets` entry in `nest-cli.json`), since `tsc` copies only TypeScript.
- Two exceptions, both files that belong to no module: `src/config/` stays flat (one `{concern}.config.ts` per concern, see `devops-use-config-module`; `config/` is already the type folder), and domain-neutral files used by several modules live under `src/shared/`, themselves in type subfolders (see `arch-shared-folder`).
- Top-level files in `src/`: `main.ts` and `app.module.ts`, nothing else. A command line is a module of its own, `src/cli/` (below). The global setup that `main.ts` and the tests share is the api module's (`src/api/`, see `test-use-testing-module`).
- One module holds the app's schedule: `src/scheduler/`, imported only by `AppModule`, next to `ScheduleModule.forRoot()` from `@nestjs/schedule`. Its `SchedulerService` holds only the timing: one `@Interval` method per recurring job, each calling the method of the feature's service that does the work and catching its errors (see `error-handle-async-errors`). The work, its state and its tests stay in the feature. Use `@Interval`, not `@Cron`: a cron expression fires on the wall clock, so every instance would run the job at the same second, while an interval counts from each instance's start. The interval starts over when the instance restarts, so an instance that lives less than the interval never runs the job: keep the interval well below how long an instance usually lives. Every instance runs every job, so a job here either works on the instance's own state or gives the same result however many instances run it (a delete of rows past their retention, a refund sent again with its idempotency key). An interval also fires while its last run is still going, so `SchedulerService` skips a run of a job whose previous run has not ended. A job that must not be skipped by restarts, or must run once across all instances, such as sending a digest, is a BullMQ job scheduler instead (see `micro-use-queues`).
- `src/shared/` is a folder of plain files, **not a Nest module**: no `shared.module.ts`, no services. Something that needs dependency injection and is used by several modules gets its own module named for what it does (`mail/`, `payment/`). See `arch-shared-folder`.

**A command-line entry point** (an operator's task with no route, such as creating an administrator in an app where no route grants a role) is a second entry point, in the `cli` module:

- `cli/main.ts` picks the command from the arguments and runs it. `CliModule` is the command line's root module, as `AppModule` is the API's: `ConfigModule.forRoot()` with the same `load` and `validate` as `AppModule`, `LoggerModule`, and the feature modules its commands call, without the HTTP server, the queue workers or the schedule. It runs on `NestFactory.createApplicationContext(CliModule)`, closed in a `finally`, and starts like the API, through Sentry's preload (`node --import ./dist/sentry/preload.js dist/cli/main.js <command>`, see `error-report-to-sentry`), as a `package.json` script.
- Each command is a provider in `commands/`, named for what it does (`CliCreateAdminCommand` in `cli/commands/cli-create-admin.command.ts`). The work is a method of the feature's service, which the command calls, as a controller would. Its input is validated with the request DTO the route would use, with the same validator options, so a command cannot store what the API refuses.
- A secret comes on standard input, piped, never as an argument: other users of the machine can read a process's arguments, and a terminal would show it as it is typed.
- **An operator's mistake is a message and an exit code, not a crash.** A usage error, input the DTO refuses, or a 4xx the service throws (a taken email) prints its message on stderr and sets `process.exitCode = 1`. Thrown out of `cli/main.ts`, it would reach the crash handlers of `preload.ts`, which print the alert marker and send a Sentry event for a typo (see `devops-use-logging`). Anything else still reaches them: that one is a crash.
- Whether a command may run more than once, and what it refuses, is the app's choice, stated in its README.
- It has a test: the service method in the e2e tier, named for its consumer (`describe('UserService.createAdmin (used by the command line)')`, see `test-use-testing-module`), and the built command in the integration tier (see `test-integration-suites`).

The example is ESM, for its top-level `await`, and runs as `node --import ./dist/sentry/preload.js dist/cli/main.js create-admin --email <email> --name <name>`:

```typescript
// cli/main.ts
const USAGE = 'Usage: printf %s "$PASSWORD" | pnpm cli create-admin --email <email> --name <name>';

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
    // Ends the pool, so the process exits
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

// cli/commands/cli-create-admin.command.ts
@Injectable()
export class CliCreateAdminCommand {
  constructor(private readonly userService: UserService) {}

  async run(args: string[]): Promise<void> {
    const { values } = parseArgs({ args, options: { email: { type: 'string' }, name: { type: 'string' } } });
    const dto = plainToInstance(CreateUserDto, { email: values.email, name: values.name, password: await this._readStdin() });
    const errors = await validate(dto, API_VALIDATION_PIPE_OPTIONS);
    if (errors.length > 0) {
      throw new CliUsageError(errors.flatMap((error) => Object.values(error.constraints ?? {})).join('\n'));
    }
    const user = await this.userService.createAdmin(dto);
    process.stdout.write(`Created the administrator ${user.email}\n`);
  }

  private async _readStdin(): Promise<string> {
    if (process.stdin.isTTY) {
      // Typed at a terminal, the password would show
      throw new CliUsageError('Pipe the password on standard input: printf %s "$PASSWORD" | pnpm cli create-admin ...');
    }
    const chunks: Buffer[] = [];
    for await (const chunk of process.stdin) {
      chunks.push(chunk as Buffer);
    }
    return Buffer.concat(chunks).toString('utf8').trim();
  }
}

// cli/errors/cli-usage.error.ts
export class CliUsageError extends Error {}

// cli/cli.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [appConfig, authConfig, dbConfig], validate: validateEnv }),
    LoggerModule,
    UserModule,
  ],
  providers: [CliCreateAdminCommand],
})
export class CliModule {}
```

`API_VALIDATION_PIPE_OPTIONS` are the options of the route's `ValidationPipe` (see `security-validate-all-input`). `_readStdin()` stays a private method: it reads the process's input, so it is no `utils/` function. An error rethrown at the end is a crash, which the handlers of `preload.ts` report. The next command is another provider in `commands/` and another name in `main.ts`.

**Names** of the module and of everything in it (singular resource names, the module-name prefix, capability modules) are in `arch-resource-naming`.

**Incorrect (technical layers at the app root, or loose files at the module root):**

```typescript
// Technical layer organization (anti-pattern)
src/
├── controllers/
│   ├── user.controller.ts
│   └── order.controller.ts
├── services/
│   ├── user.service.ts
│   └── order.service.ts
└── app.module.ts

// Feature modules, but files loose at the module root
src/
└── user/
    ├── dto/
    │   └── create-user.dto.ts
    ├── user.controller.ts        // loose
    ├── user-admin.controller.ts   // second controller, still loose
    ├── user.service.ts           // loose
    ├── user.entity.ts             // loose
    ├── slug.ts                    // helper with no obvious home
    └── user.module.ts
```

**Correct (feature modules with type folders):**

```typescript
src/
├── user/
│   ├── controllers/
│   │   ├── user.controller.ts
│   │   └── user-admin.controller.ts
│   ├── services/
│   │   └── user.service.ts
│   ├── entities/
│   │   └── user.entity.ts
│   ├── dtos/
│   │   ├── create-user.dto.ts
│   │   ├── update-user.dto.ts
│   │   ├── find-all-users.query.dto.ts
│   │   └── user-id.param.dto.ts
│   ├── types/
│   │   └── user-summary.type.ts
│   ├── utils/
│   │   └── build-display-name.ts
│   └── user.module.ts            // the only file at the module root
├── order/
│   ├── controllers/
│   ├── services/
│   ├── entities/
│   ├── dtos/
│   └── order.module.ts
├── payment/
│   ├── adapters/
│   │   └── stripe.adapter.ts      // StripeAdapter implements PaymentAdapter
│   ├── clients/
│   │   └── stripe.client.ts       // StripeClient: transport only
│   ├── constants/
│   │   └── payment-adapter.constant.ts
│   ├── interfaces/
│   │   └── payment-adapter.interface.ts
│   ├── types/
│   │   └── payment-result.type.ts
│   └── payment.module.ts
├── scheduler/
│   ├── services/
│   │   └── scheduler.service.ts   // timing only, calls the features' services
│   └── scheduler.module.ts
├── auth/
│   ├── controllers/
│   ├── guards/
│   │   ├── auth-jwt.guard.ts
│   │   └── auth-roles.guard.ts
│   ├── strategies/
│   │   └── auth-jwt.strategy.ts
│   └── auth.module.ts
├── sentry/
│   ├── services/
│   │   └── sentry.service.ts
│   ├── preload.ts                 // an entry point Node runs, allowed at a module root: node --import runs it
│   └── sentry.module.ts
├── cli/                           // only in an app with a command line
│   ├── commands/
│   │   └── cli-create-admin.command.ts
│   ├── errors/
│   │   └── cli-usage.error.ts
│   ├── main.ts                    // the other entry point at a module root: node runs dist/cli/main.js
│   └── cli.module.ts
├── api/                           // the HTTP layer: global setup, and what every controller uses
│   ├── constants/
│   │   └── api-validation-pipe-options.constant.ts
│   ├── decorators/
│   │   └── auth-user.decorator.ts
│   ├── dtos/
│   │   └── api-pagination.query.dto.ts
│   ├── filters/
│   │   └── api-exception.filter.ts
│   ├── interceptors/
│   │   ├── api-logger.interceptor.ts
│   │   └── api-response.interceptor.ts
│   ├── types/
│   │   └── api-request.type.ts
│   └── api.module.ts              // forRoot(), setup(), createFastifyAdapter()
├── shared/
│   ├── types/
│   │   └── jwt-user.type.ts
│   └── utils/
│       └── slug-generator.ts          // no shared.module.ts: plain reusable files
├── config/
│   ├── app.config.ts
│   ├── auth.config.ts
│   ├── db.config.ts
│   └── validate-env.ts
├── app.module.ts
└── main.ts

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
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [appConfig, authConfig, dbConfig], validate: validateEnv }),
    DatabaseModule,
    UserModule,
    OrderModule,
  ],
})
export class AppModule {}
```

A module exports only what other modules consume (see `arch-module-sharing`). See `arch-import-boundaries` for how files in these folders import each other.

Reference: [NestJS Modules](https://docs.nestjs.com/modules)
