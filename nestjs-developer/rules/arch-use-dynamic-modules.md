---
title: Make Reusable Modules Configurable
impact: MEDIUM
impactDescription: Reusable modules without hardcoded config or hand-rolled forRoot boilerplate
tags: architecture, modules, dynamic-modules, configuration
---

## Make Reusable Modules Configurable

When a module wraps infrastructure that needs configuration (mail, storage, an API client) and is reused across apps or instantiated more than once, expose it as a dynamic module with `forRoot`/`forRootAsync` instead of reading `process.env` inside it or hardcoding values. A module used by one app only can inject its config namespace directly (see `devops-use-config-module`). Use `ConfigurableModuleBuilder` to generate the static methods, the options token, and the async variant instead of writing them by hand. Always provide the async variant so consumers can build options from their config namespace.

The options type is the module's whole configuration contract. The module never imports from the app's `src/config/`, injects `ConfigService` or reads `process.env`: those tie it to one app's variable names. The app maps its own namespace into the options (see `devops-use-config-module`). A module shared by several apps in a monorepo lives as a library under `libs/` (`nest g library`), where it cannot reach an app's `src/config/` at all.

**What the module needs from the app comes through its options too**, never through a library the app happens to use. A client that sends the request id with each call (see `micro-use-http-clients`) takes a `getRequestId` function in its options, which the app maps to its own request context in `forRootAsync`. Injecting `ClsService` inside the module imports `nestjs-cls`, so the module no longer loads in an app without that package: `@Optional()` covers a missing provider, not a missing package.

**Incorrect (config read inside the module, or a hand-rolled sync-only forRoot):**

```typescript
// Reads env directly - can't be configured per consumer or in tests
@Injectable()
export class MailService {
  private readonly apiKey = process.env.MAIL_API_KEY!;
  private readonly from = 'noreply@example.com';
}

// Hand-rolled forRoot - no forRootAsync, so consumers can't use ConfigService
@Module({})
export class MailModule {
  static forRoot(options: MailModuleOptions): DynamicModule {
    return {
      module: MailModule,
      providers: [{ provide: 'MAIL_OPTIONS', useValue: options }, MailService],
      exports: [MailService],
    };
  }
}
```

**Correct (ConfigurableModuleBuilder with async configuration):**

```typescript
// mail/types/mail-module-options.type.ts
export type MailModuleOptions = {
  apiKey: string;
  from: string;
  getRequestId?: () => string | undefined;
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

// mail/services/mail.service.ts
@Injectable()
export class MailService {
  constructor(@Inject(MAIL_OPTIONS) private readonly options: MailModuleOptions) {}
}

// notification/notification.module.ts
@Module({
  imports: [
    MailModule.forRootAsync({
      inject: [mailConfig.KEY, ClsService],
      useFactory: (mailSettings: ConfigType<typeof mailConfig>, clsService: ClsService) => ({
        apiKey: mailSettings.apiKey,
        from: mailSettings.from,
        getRequestId: () => clsService.getId(),
      }),
    }),
  ],
  providers: [NotificationService],
  exports: [NotificationService, MailModule],
})
export class NotificationModule {}

// order/order.module.ts
@Module({ imports: [NotificationModule], providers: [OrderService] })
export class OrderModule {}
```

The request id that `getRequestId` returns is sent with each call; the app provides the function, since the app knows where its request context lives. The services take the token from `constants/`, so no file imports `mail.module.ts` back and no module-definition file is needed (see `arch-feature-modules`). The builder adds no `isGlobal` extra: consumers import the module where they use it (see `arch-module-sharing`). `NotificationModule` configures `MailModule` once, mapping the app's mail namespace (`src/config/mail.config.ts`, see `devops-use-config-module`) onto its options, and re-exports it, so its importers get the configured `MailService`; `forRootAsync` also accepts `useClass` and `useExisting`. A module that sends mail, like `OrderModule`, imports the configuring module. Tests pass plain options:

```typescript
MailModule.forRoot({ apiKey: 'test', from: 'test@example.com' });
```

Configure the module in exactly one place, in a module that re-exports it, and import that module wherever the service is needed. Do not configure it in `AppModule`: feature modules do not see `AppModule`'s imports, and importing plain `MailModule` in a feature module creates a second instance without options, so Nest fails at startup with "can't resolve dependencies of the MailService" (`MAIL_OPTIONS`).

Naming convention: `forRoot` configures a module once for the whole app, `forFeature` configures it per consuming module (like `TypeOrmModule.forFeature`), and `register` creates a separately configured instance for each import.

Reference: [NestJS Dynamic Modules](https://docs.nestjs.com/fundamentals/dynamic-modules#configurable-module-builder)
