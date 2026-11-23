---
title: Use Proper Module Sharing Patterns
impact: CRITICAL
impactDescription: Prevents duplicate instances, memory leaks, and state inconsistency
tags: architecture, modules, sharing, exports
---

## Use Proper Module Sharing Patterns

NestJS modules are singletons by default. When a service is properly exported from a module and that module is imported elsewhere, the same instance is shared. However, providing a service in multiple modules creates separate instances, leading to memory waste, state inconsistency, and confusing behavior. Always encapsulate services in dedicated modules, export them explicitly, and import the module where needed.

**Incorrect (service provided in multiple modules):**

```typescript
// StorageService provided directly in multiple modules - WRONG
// storage.service.ts
@Injectable()
export class StorageService {
  private cache = new Map(); // Each instance has separate state!

  store(key: string, value: unknown): void {
    this.cache.set(key, value);
  }
}

// app.module.ts
@Module({
  providers: [StorageService], // Instance #1
  controllers: [AppController],
})
export class AppModule {}

// product.module.ts
@Module({
  providers: [StorageService], // Instance #2 - different from AppModule!
  controllers: [ProductController],
})
export class ProductModule {}

// Problems:
// 1. Two separate StorageService instances exist
// 2. cache.set() in ProductModule doesn't affect AppModule's cache
// 3. Memory wasted on duplicate instances
// 4. Debugging nightmares when state doesn't sync
```

**Correct (dedicated module with exports):**

```typescript
// storage/storage.module.ts
@Module({
  providers: [StorageService],
  exports: [StorageService],
})
export class StorageModule {}

// product/product.module.ts
@Module({
  imports: [StorageModule],
  controllers: [ProductController],
  providers: [ProductService],
})
export class ProductModule {}

// user/user.module.ts
@Module({
  imports: [StorageModule],
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}

// app.module.ts
@Module({
  imports: [
    ProductModule,
    UserModule,
  ],
})
export class AppModule {}
```

`StorageModule` exports `StorageService`. The modules that use it import `StorageModule` and never list `StorageService` in their own `providers`, so they all share the same instance. `AppModule` imports `StorageModule` only if it needs `StorageService` itself, and here it does not.

**A module's public face is a service, plus its pure functions and the types, enums and DTOs that describe its data** (see `arch-import-boundaries`). For anything with state or I/O, other modules inject the service a module exports (`PaymentService`, `StorageService`, `AuthTokenService`), never its token, its adapter or its client. A pure function another module needs, such as the database module's error checks (`isUniqueViolation()`, `isTransactionConflict()`), is imported from the owning module's `utils/` instead: a function with no state gains nothing from injection. The function still lives in the module that owns what it knows, so the ORM's error codes stay in the database module. The interface and its token stay inside the module, for swapping the implementation (see `di-use-interfaces-tokens`): `PaymentService` injects `PAYMENT_ADAPTER`, and the order module sees only `PaymentService`. A service that passes each call on is still the module's API: its consumers do not change when the module's insides do, and it is where logic of the module's own goes later (sending a refund to the provider that took the payment, once there are two). A module that exports a token, an adapter, a client or a third-party module it configured lets every consumer depend on its insides and read its config.

**One client per outside system.** Two modules that call the same outside system share one capability module for it, never a copy each: two copies duplicate the part that matters most, the mapping of the system's failures to 503 or 500 (see `micro-use-http-clients`), and a fix to one misses the other. A second copy of a client is a finding even when the two store different kinds of files. The capability module's service speaks its consumers' terms: `storageService.put(StorageFolder.ProductImages, key, file)`, and its adapter maps each folder to the system's names, the app's one bucket and the folder's prefix in it (see `micro-use-http-clients`). The same holds for a system the app needs several connections to: the `redis` module builds every Redis connection the app opens itself (the commands' client, the subscriber, the Socket.IO adapter's pair) with one logging of its availability and closes them all; BullMQ, which opens its own, gets its options from the same `redisConnectionOptions()` (see `micro-use-queues`), so the settings are mapped once; and a module that publishes or subscribes calls `RedisService`, never `new Redis()` of its own (see `api-real-time-updates`).

**The one exception is a connection a library needs.** An infrastructure module that hands a library its connection exports the connection: the `redis` module's `REDIS_CLIENT`, which the rate-limit store and the cache store take as a raw client (see `security-rate-limiting`, `perf-use-caching`), as TypeORM's `DataSource` is shared. A service could not stand in for it, since the library calls the client's own methods. The export is for those libraries alone: app code that publishes, subscribes or pings goes through `RedisService`.

**Global modules: only third-party infrastructure, registered once.** `ConfigModule.forRoot({ isGlobal: true })` (see `devops-use-config-module`), the logger module, `ClsModule`, `CacheModule` (see `perf-use-caching`) and `NestjsFormDataModule` (see `api-file-uploads`) are global. Register each in `AppModule`, or in the app's one module for that infrastructure, which `AppModule` imports: `LoggerModule` registers `ClsModule` and pino (see `devops-use-logging`), `SentryModule` the SDK's module (see `error-report-to-sentry`). Do not write your own `@Global()` modules or give your configurable modules an `isGlobal` option, and never a catch-all `CoreModule`/`CommonModule`/`SharedModule` that bundles unrelated services: it hides what each module depends on, and every consumer gets everything. Plain reusable files go in `src/shared/` and are imported directly (see `arch-shared-folder`); a reusable service gets its own module named for what it does.

```typescript
// Incorrect: a grab-bag module, global so nobody sees who uses what
@Global()
@Module({
  providers: [DateService, MailService, PdfService],
  exports: [DateService, MailService, PdfService],
})
export class CommonModule {}
```

Instead, one module per capability, imported where it is used:

```typescript
@Module({ providers: [MailService], exports: [MailService] })
export class MailModule {}

@Module({ imports: [MailModule], providers: [OrderService] })
export class OrderModule {}
```

**Export a service, not a third-party module you configured.** When another module needs what a configured third-party module does (`JwtModule`, `HttpModule`), the owning module wraps it in a service of its own instead of re-exporting the module. The app's own configurable module is the exception: it is configured once, in a module that re-exports it, so importers get its configured service, which is that module's public face (see `arch-use-dynamic-modules`).

A WebSocket gateway in another module checks handshake tokens through `AuthTokenService`, which verifies them as `AuthJwtStrategy` does; `JwtModule` and its options stay inside the auth module, which does not export `JwtModule`. The HTTP guards are global and need no import (see `security-use-guards`).

```typescript
// auth/auth.module.ts
@Module({
  imports: [UserModule, PassportModule, JwtModule.registerAsync({ /* ... */ })],
  providers: [AuthJwtStrategy, AuthSessionService, AuthTokenService],
  exports: [AuthSessionService, AuthTokenService],
})
export class AuthModule {}
```

Reference: [NestJS Modules](https://docs.nestjs.com/modules#shared-modules)
