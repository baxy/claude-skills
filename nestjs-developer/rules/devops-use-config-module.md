---
title: Load Configuration Through Namespaced Config Files
impact: HIGH
impactDescription: Every setting has one typed home, and a misconfigured service refuses to start instead of failing at runtime
tags: devops, configuration, environment, validation, registerAs
---

## Load Configuration Through Namespaced Config Files

Use `@nestjs/config` with **one config file per concern**. Each file maps environment variables to one namespace with `registerAs`, and the rest of the code reads settings through that namespace. `process.env` is read in exactly one place: the files in `src/config/`.

**Files and namespaces:**

- `src/config/{concern}.config.ts` exports `{concern}Config = registerAs('{namespace}', () => ({ ... }))`. The folder is flat (see `arch-feature-modules`).
- Shared concerns use the same namespace and shape in every service: `app` (`env`, `port`, `host`, `corsOrigins`, `trustedProxies`, `logLevel`, `logPretty`, `shutdownDrainMs`), `auth`, `db` (`host`, `port`, `username`, `password`, `name`, `schema`, `logging`, `ssl`, `sslCa`, `poolSize`), `aws`, `eventBridge`, `redis` (`host`, `port`, `username`, `password`, `tls`, `tlsCa`). A setting that belongs to this service alone goes in a namespace named for the service or the concern (`studio`, `mail`, `platform`), never into `app`.
- Environment variables are grouped by prefix per concern (`APP_`, `DB_`, `AWS_`, `AUTH_`, `EVENT_BRIDGE_`, `REDIS_`). Every authentication setting uses `AUTH_` (`AUTH_JWT_SECRET`), whichever identity provider issues the tokens: a provider-specific prefix such as `AUTH0_` ties the names to one vendor. Defaults and parsing (`parseInt(..., 10)`, `=== 'true'`, splitting lists) happen in the config file, so consumers receive typed values.
- **An SDK's own variables are not app config.** The AWS SDK reads its credentials itself (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, a profile, or the role of the machine or task it runs on), so no config file reads them and `validate-env.ts` does not require them: on AWS, a role provides them. `.env.example` lists them, empty, for a local S3-compatible server, and its test names them as the SDK's.
- A variable that holds a secret ends in `_KEY`, `_SECRET`, `_PASSWORD` or `_TOKEN` (`MAIL_API_KEY`, `AUTH_JWT_SECRET`, `DB_PASSWORD`), and one holding a URL that may carry credentials ends in `_URL`. The redactor of Sentry events and log lines finds the secrets by these suffixes (see `error-report-to-sentry`), so a secret named otherwise reaches both unredacted.
- All files are loaded once, in `AppModule`: `ConfigModule.forRoot({ isGlobal: true, load: [...], validate: validateEnv })`. A command line's root module, `CliModule`, loads them the same way (see `arch-feature-modules`).

**`src/config/` is a folder, not a module:**

- There is no `config.module.ts` and no module of our own around configuration. The only configuration module is `@nestjs/config`'s `ConfigModule`, registered once in `AppModule` (the same way `src/shared/` holds files and no module, see `arch-shared-folder`).
- Every setting lives here, including the ones a single feature module uses: the encryption module's settings are `src/config/encryption.config.ts`, never `encryption/config/`. All environment variables are then found in one folder.
- No `ConfigModule.forFeature()` in feature modules: a namespace registered there skips the single load in `AppModule` and the `validate` hook.
- **A module meant for reuse** (in another app, as a library under `libs/`, or instantiated more than once) has its own configuration as a typed options contract, not as environment variables: it is a configurable module (see `arch-use-dynamic-modules`) and never imports from `src/config/`, injects `ConfigService` or reads `process.env`. Each app that uses it keeps the variables in its own `src/config/` and maps that namespace into the module's options in `forRootAsync`. Every variable stays in one folder, and the module works in any app and in tests with plain values.
- The folder holds only `{concern}.config.ts` files and `validate-env.ts`. A function that builds a module's options from a namespace (the TypeORM options, a client's settings) lives where it is used.
- One name in four forms: the file `event-bridge.config.ts` exports `eventBridgeConfig` for the namespace `eventBridge`, read from `EVENT_BRIDGE_*` variables.

**Required values are validated at load.** A config file that defaults a missing variable to `''` hides it: `getOrThrow` treats an empty string as defined, so the service starts, passes its health check and fails every request. `src/config/validate-env.ts` lists the variables the service cannot run without and refuses to start when one is missing or malformed, or outside the bounds security sets for it (the JWT key's length and the access token's lifetime, see `security-auth-jwt`). Because it is the `validate` hook of `ConfigModule`, it runs wherever `AppModule` is built: the app itself and the integration tier (e2e testing modules do not import `AppModule`, see `test-use-testing-module`).

**An optional setting is checked when it is set.** A default covers a missing value, not a wrong one: `parseInt('15s', 10)` is 15, so `APP_SHUTDOWN_DRAIN_MS=15s` drains for 15 milliseconds and `DB_LOCK_TIMEOUT_MS=2s` waits 2 milliseconds for a lock, and `DB_TRANSACTION_TIMEOUT_MS=five` gives `NaN`, with which a Prisma app started, answered 200 to readiness and failed every transaction. `validate-env.ts` refuses an optional number that is set but is not a whole number, and settings that only work together: a limit that must sit below another (the lock wait limit below the transaction timeout, see `db-prisma-schema-conventions`).

**A URL that receives credentials or personal data is `https` outside `dev`.** A provider's base URL that the app calls with an API key header, or sends phone numbers and order details to, hands both to anyone on the way when it is plain `http`. `validate-env.ts` refuses such a URL unless it is `https`, everywhere but `APP_ENV` `dev`, where a stand-in on the developer's machine may answer over `http`.

**The connections to the database and Redis use TLS outside `dev` for the same reason**: they carry the password and every row the app reads. `validate-env.ts` refuses `DB_SSL` other than `true`, and `REDIS_TLS` other than `true` when `REDIS_HOST` is set, everywhere but `APP_ENV` `dev`. A managed service often signs its certificate with a CA of its own, which Node's list does not hold: `DB_SSL_CA_FILE` and `REDIS_TLS_CA_FILE` name that CA's file, the config file reads it into `db.sslCa` and `redis.tlsCa`, and the clients pass it as `ssl: { ca }` and `tls: { ca }`, so the certificate is still checked. Turning the check off (`rejectUnauthorized: false`) instead lets anyone on the way present a certificate of their own.

**`APP_ENV` is required and takes one of three values: `dev`, `stage` or `prod`.** It has no default, and `validate-env.ts` refuses a missing or any other value. A default lets a missing variable pass for one environment, and a misspelled value (`production`) matches no comparison in the code, so the service would run with another environment's behavior. A local run uses `dev`.

What must differ on a developer's machine is a setting of its own, never derived from `APP_ENV`, which a local run shares with the deployed `dev`: `APP_LOG_PRETTY` turns on single-line pretty logs (see `devops-use-logging`), and `APP_SHUTDOWN_DRAIN_MS` sets the shutdown drain (see `devops-graceful-shutdown`). `.env.example` sets both for a local run (`true` and `0`); deployed environments leave them at their defaults, JSON logs and a 15-second drain.

**Every variable is listed in `.env.example`.** The project root has a committed `.env.example` and a local `.env` beside it, which git ignores. The file is named `.env.example`, never `.env.sample`.

- It lists every variable `src/config/` and `src/sentry/preload.ts` read, and no other apart from an SDK's own variables (above), which the test names as the SDK's. They are grouped by concern under a line that names the required ones (`# db (required: DB_HOST, DB_USERNAME, DB_PASSWORD, DB_NAME)`). Each variable sits under a comment line of its own saying what it is. Never put a comment after the value on the same line: Docker's `--env-file` keeps it as part of the value.
- Values: the config file's default where there is one, an example value where it is safe (`DB_HOST=localhost`), and empty for every secret, the local database's password included (`DB_PASSWORD=`, `MAIL_API_KEY=`). Examples point only at local services, never at a shared environment, so a copy never reaches shared data.
- It is an example, not a working setup. A copy runs once whoever runs the service has filled in the secrets and pointed the values at what they run. Providing every required value is that person's job, a developer's or the deployment's, and `validate-env.ts` names every missing one at startup. The README lists what must be provided (see `docs-keep-readmes-current`).
- A unit test keeps it complete: `test/unit/config/env-example.spec.ts` fails when the variables in `.env.example` and those `src/config/` and `src/sentry/preload.ts` read differ, apart from the SDK's variables it names, so a variable added to a config file without the example fails `test` in CI.
- Adding, renaming or removing a variable changes `src/config/`, `validate-env.ts` when the variable is required, and `.env.example` in the same change.

**Reading settings:**

- Required value: `configService.getOrThrow<string>('auth.jwt.secret')`.
- Optional value: `configService.get<boolean>('db.logging', false)`, always with the default at the call site.
- Several values of one namespace in a provider: inject the namespace itself, `@Inject(mailConfig.KEY) private readonly mailSettings: ConfigType<typeof mailConfig>`, which is fully typed. Import the type with `import type { ConfigType } from '@nestjs/config'`: the `tsconfig.json` that `nest new` generates (NestJS 11 and 12) enables `isolatedModules` with `emitDecoratorMetadata`, and a value import used in a decorated constructor fails with TS1272. Name the parameter `{namespace}Settings`, never the same as the config constant: `typeof mailConfig` would then refer to the parameter itself and fail to compile (TS2502).
- Module factories (`forRootAsync`, `registerAsync`) follow the same rule: `inject: [dbConfig.KEY]` with a `dbSettings` parameter, not `ConfigService` with one `getOrThrow` per key.
- Keys are always namespaced paths (`db.host`, `auth.jwt.issuer`). Never pass a raw environment variable name (`'DB_HOST'`, `'JWT_SECRET'`) to `ConfigService`.

**Exceptions to "process.env only in src/config/":**

- The migration `DataSource` (`migrations/type-orm/config.ts`), which the TypeORM CLI runs outside Nest (see `db-typeorm-schema-conventions`), or `prisma.config.ts`, which the Prisma CLI reads (see `db-prisma-schema-conventions`). Both read only the `DB_*` variables `db.config.ts` reads.
- `src/sentry/preload.ts`, which `node --import` (or `--require`) loads before the app to start Sentry (see `error-report-to-sentry`). It reads `SENTRY_DSN`, `APP_ENV` and `APP_RELEASE`, and hands `process.env` to the secret redactor.
- A value read before `ConfigModule` exists, or one that must not get the config file's default, with a comment saying why.
- Test code, for a variable only the tests read (`INTEGRATION_API_URL`). A test helper that reaches a server the app also uses (the database, Redis, S3) builds its connection from the app's own mapping instead: the `registerAs` factory called directly (`redisConnectionOptions(redisConfig())`), or the migration `DataSource` for the database. It never reads that server's variables with defaults of its own, which drift from the app's: a helper that rebuilds the Redis options leaves out the TLS CA the app sets, and falls back to `localhost` when a variable is missing.

**Incorrect (scattered env access, raw keys, silent empty defaults, an environment name with a default):**

```typescript
@Injectable()
export class MailService {
  private readonly apiKey = process.env.SENDGRID_API_KEY || 'default'; // Env read outside src/config, fake default
}

@Injectable()
export class TokenService {
  constructor(private readonly configService: ConfigService) {}

  sign(payload: object): string {
    const secret = this.configService.get('JWT_SECRET'); // Raw env name, no namespace, may be undefined
    // ...
  }
}

// app.module.ts - nothing checks that required variables are set
ConfigModule.forRoot({ isGlobal: true });

// src/config/app.config.ts - a missing APP_ENV becomes 'development', and nothing refuses 'production'
export const appConfig = registerAs('app', () => ({
  env: process.env.APP_ENV || 'development',
}));
const pretty = appSettings.env !== 'prod'; // Pretty logs in the deployed dev and stage too
```

**Correct (one file per concern, validated at load, namespaced reads):**

```typescript
// src/config/app.config.ts
export const APP_ENVS = ['dev', 'stage', 'prod'] as const;
export const LOG_LEVELS = ['trace', 'debug', 'info', 'warn', 'error', 'fatal'] as const;

export const appConfig = registerAs('app', () => ({
  env: process.env.APP_ENV as (typeof APP_ENVS)[number],
  port: parseInt(process.env.APP_PORT || '3000', 10),
  host: process.env.APP_HOST || '0.0.0.0',
  corsOrigins: (process.env.APP_CORS_ORIGINS || '').split(',').filter(Boolean),
  trustedProxies: (process.env.APP_TRUSTED_PROXIES || '').split(',').map((proxy) => proxy.trim()).filter(Boolean),
  logLevel: (process.env.APP_LOG_LEVEL || 'info') as (typeof LOG_LEVELS)[number],
  logPretty: process.env.APP_LOG_PRETTY === 'true',
  shutdownDrainMs: parseInt(process.env.APP_SHUTDOWN_DRAIN_MS || '15000', 10),
}));

// src/config/db.config.ts
export const dbConfig = registerAs('db', () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432', 10),
  username: process.env.DB_USERNAME || '',
  password: process.env.DB_PASSWORD || '',
  name: process.env.DB_NAME || '',
  schema: process.env.DB_SCHEMA || 'public',
  logging: process.env.DB_LOGGING === 'true',
  ssl: process.env.DB_SSL === 'true',
  // Unset, Node's own list of CAs checks the server's certificate
  sslCa: process.env.DB_SSL_CA_FILE ? readFileSync(process.env.DB_SSL_CA_FILE, 'utf8') : undefined,
  poolSize: parseInt(process.env.DB_POOL_SIZE || '10', 10),
}));

// src/config/mail.config.ts
export const mailConfig = registerAs('mail', () => ({
  apiKey: process.env.MAIL_API_KEY || '',
  from: process.env.MAIL_FROM || 'no-reply@example.com',
}));

// src/config/validate-env.ts
const REQUIRED = ['APP_ENV', 'DB_HOST', 'DB_USERNAME', 'DB_PASSWORD', 'DB_NAME', 'MAIL_API_KEY'] as const;
const HTTPS_URLS = ['CARRIER_API_BASE_URL'] as const;
// parseInt() reads '15s' as 15, so a value that is set must be a whole number
const WHOLE_NUMBERS = ['APP_PORT', 'APP_SHUTDOWN_DRAIN_MS', 'DB_PORT', 'DB_POOL_SIZE'] as const;

export function validateEnv(env: Record<string, unknown>): Record<string, unknown> {
  const missing = REQUIRED.filter((name) => !String(env[name] ?? '').trim());
  if (missing.length > 0) {
    throw new Error(`Refusing to start: ${missing.join(', ')} must be set`);
  }
  if (!APP_ENVS.some((appEnv) => appEnv === env.APP_ENV)) {
    throw new Error(`Refusing to start: APP_ENV must be one of ${APP_ENVS.join(', ')}`);
  }
  // Optional, but a typo (APP_LOG_LEVEL=verbose) must not start the service at the wrong level
  if (env.APP_LOG_LEVEL && !LOG_LEVELS.some((level) => level === env.APP_LOG_LEVEL)) {
    throw new Error(`Refusing to start: APP_LOG_LEVEL must be one of ${LOG_LEVELS.join(', ')}`);
  }
  const notWhole = WHOLE_NUMBERS.filter((name) => String(env[name] ?? '') !== '' && !/^\d+$/.test(String(env[name])));
  if (notWhole.length > 0) {
    throw new Error(`Refusing to start: ${notWhole.join(', ')} must be whole numbers`);
  }
  const plainHttp = HTTPS_URLS.filter((name) => env[name] && !String(env[name]).startsWith('https://'));
  if (env.APP_ENV !== 'dev' && plainHttp.length > 0) {
    throw new Error(`Refusing to start: ${plainHttp.join(', ')} must be https URLs outside APP_ENV dev`);
  }
  const plainConnections = [
    env.DB_SSL !== 'true' ? 'DB_SSL' : undefined,
    env.REDIS_HOST && env.REDIS_TLS !== 'true' ? 'REDIS_TLS' : undefined,
  ].filter(Boolean);
  if (env.APP_ENV !== 'dev' && plainConnections.length > 0) {
    throw new Error(`Refusing to start: ${plainConnections.join(', ')} must be true outside APP_ENV dev`);
  }
  return env;
}

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [appConfig, dbConfig, mailConfig],
      validate: validateEnv,
    }),
    DatabaseModule,
  ],
})
export class AppModule {}

// database/database.module.ts
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      inject: [dbConfig.KEY],
      useFactory: (dbSettings: ConfigType<typeof dbConfig>) => ({
        type: 'postgres',
        host: dbSettings.host,
        port: dbSettings.port,
        username: dbSettings.username,
        password: dbSettings.password,
        database: dbSettings.name,
        schema: dbSettings.schema,
        extra: { options: `-c search_path=${dbSettings.schema},public` },
        logging: dbSettings.logging,
        logger: new LoggerTypeOrmQueryLogger(),
        ssl: dbSettings.ssl ? { ca: dbSettings.sslCa } : false,
        poolSize: dbSettings.poolSize,
        autoLoadEntities: true,
        namingStrategy: new SnakeNamingStrategy(),
        uuidExtension: 'pgcrypto',
        installExtensions: false,
        synchronize: false,
      }),
    }),
  ],
  providers: [DatabaseTransactionService],
  exports: [DatabaseTransactionService],
})
export class DatabaseModule {}

@Injectable()
export class MailService {
  constructor(@Inject(mailConfig.KEY) private readonly mailSettings: ConfigType<typeof mailConfig>) {}

  async send(to: string, subject: string, body: string): Promise<void> {
    await this._client().send({ from: this.mailSettings.from, to, subject, body });
  }

  private _client(): MailClient { /* ... uses this.mailSettings.apiKey */ }
}
```

`APP_ENV` has no default, and `validateEnv()` refuses any value outside its list, as it does for a set `APP_LOG_LEVEL`. The pool size is set against PostgreSQL's `max_connections` (see `perf-optimize-database`). `mail` is a concern only this service has. `validateEnv()` runs as `ConfigModule` loads, before any provider is built. `HTTPS_URLS` lists the URLs the app sends a key or personal data to: the carrier's takes the key in a request header (see `micro-use-http-clients`). The TypeORM options are the `database` module's, which owns the connection (see `db-typeorm-use-transactions`). In them, the `search_path` is the one the migration DataSource sets, and `LoggerTypeOrmQueryLogger` sends the queries through pino without their values (see `devops-use-logging`). `SnakeNamingStrategy` is required, and `uuidExtension: 'pgcrypto'` goes with `installExtensions: false` (see `db-typeorm-schema-conventions`). `synchronize` is hardcoded to `false`, never read from config: schema changes go through migrations (see `db-typeorm-use-migrations`). `MailService` needs several values of one namespace, so it injects the namespace.

The matching `.env.example`:

```bash
# .env.example
# app (required: APP_ENV)
# Where the app runs: dev, stage or prod; a local run is dev
APP_ENV=dev
# The port the API listens on
APP_PORT=3000
# The address the API listens on; 0.0.0.0 also takes connections from outside a container
APP_HOST=0.0.0.0
# Origins allowed by CORS, comma-separated
APP_CORS_ORIGINS=http://localhost:5173
# Addresses and CIDRs of the load balancer and every proxy in front of it, comma-separated; empty trusts none
APP_TRUSTED_PROXIES=
# The lowest log level written: trace, debug, info, warn, error or fatal
APP_LOG_LEVEL=info
# Single-line pretty logs instead of JSON; deployments leave it unset
APP_LOG_PRETTY=true
# Milliseconds the API keeps serving after readiness fails on shutdown; deployments leave the 15 s default
APP_SHUTDOWN_DRAIN_MS=0
# The image tag, which Sentry reports as the release; the image sets it, so it stays empty locally
APP_RELEASE=

# sentry
# Where errors are reported; empty sends nothing to Sentry
SENTRY_DSN=

# db (required: DB_HOST, DB_USERNAME, DB_PASSWORD, DB_NAME)
# The PostgreSQL host
DB_HOST=localhost
# The PostgreSQL port
DB_PORT=5432
# The database user
DB_USERNAME=postgres
# The database user's password
DB_PASSWORD=
# The database name
DB_NAME=orders
# The schema the app's tables live in
DB_SCHEMA=public
# Log every query
DB_LOGGING=false
# Connect over TLS; required outside APP_ENV dev
DB_SSL=false
# The file of the CA that signed the database's certificate, when Node's own list does not hold it
DB_SSL_CA_FILE=
# Connections in the app's pool
DB_POOL_SIZE=10

# mail (required: MAIL_API_KEY)
# The mail provider's API key; set your own
MAIL_API_KEY=
# The sender address
MAIL_FROM=no-reply@example.com
```

```typescript
// test/unit/config/env-example.spec.ts
import { readdirSync, readFileSync } from 'node:fs';
import { join } from 'node:path';

const APP_DIR = join(__dirname, '../../..');
const CONFIG_DIR = join(APP_DIR, 'src/config');
const SOURCES = [
  ...readdirSync(CONFIG_DIR)
    .filter((file) => file.endsWith('.config.ts'))
    .map((file) => join(CONFIG_DIR, file)),
  join(APP_DIR, 'src/sentry/preload.ts'),
];

const namesIn = (text: string, pattern: RegExp): string[] => [...text.matchAll(pattern)].map((match) => match[1]);

describe('.env.example', () => {
  it('lists exactly the variables src/config/ and src/sentry/preload.ts read', () => {
    const read = SOURCES.flatMap((file) => namesIn(readFileSync(file, 'utf8'), /process\.env\.([A-Z0-9_]+)/g));
    const listed = namesIn(readFileSync(join(APP_DIR, '.env.example'), 'utf8'), /^([A-Z0-9_]+)=/gm);

    expect([...new Set(listed)].sort()).toEqual([...new Set(read)].sort());
  });
});
```

The spec checks that the variables the app reads are exactly those `.env.example` lists. It reads `src/sentry/preload.ts` with the config files, since that file reads its variables before `ConfigModule` exists (see `error-report-to-sentry`). In an ESM project, `APP_DIR` is built from `import.meta.dirname` instead of `__dirname`.

Reference: [NestJS Configuration](https://docs.nestjs.com/techniques/configuration) · [Configuration namespaces](https://docs.nestjs.com/techniques/configuration#configuration-namespaces)
