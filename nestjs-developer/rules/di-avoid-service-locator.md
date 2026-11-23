---
title: Avoid the Service Locator Anti-Pattern
impact: HIGH
impactDescription: Dependencies stay visible in the constructor and replaceable in tests
tags: dependency-injection, anti-patterns, testing
---

## Avoid the Service Locator Anti-Pattern

Do not resolve dependencies at runtime with `ModuleRef.get()` or a hand-written global container. The dependency disappears from the constructor, so readers and the testing module cannot see it. Inject it in the constructor instead (see `di-prefer-constructor-injection`).

`ModuleRef` is legitimate in exactly two cases, and resolves by a token or a class, never by an interface, which does not exist at runtime (see `di-use-interfaces-tokens`):

- **A request-scoped or transient provider** that a singleton needs per call: `moduleRef.resolve(TOKEN, contextId)` (see `di-scope-awareness`).
- **Lazy-loaded modules** (see `perf-lazy-loading`), whose providers do not exist at startup: `moduleRef.get(TOKEN)` on the loaded module's reference.

**A choice made per call needs no `ModuleRef`.** When the implementation depends on a value only known per call (a channel kind, a provider name), the module builds a map of every implementation in a factory provider, from injected providers, and the service picks from the map. Every candidate is then a visible dependency of that factory, a test replaces one like any provider, and a missing one fails at startup, not on the first call that asks for it.

**Incorrect (dependencies hidden behind a locator):**

```typescript
@Injectable()
export class OrderService {
  constructor(private readonly moduleRef: ModuleRef) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const inventoryService = this.moduleRef.get(InventoryService, { strict: false }); // Invisible dependency
    const paymentService = this.moduleRef.get(PaymentService, { strict: false });
    // ...
  }
}

// Global singleton container
export class ServiceContainer {
  private static instance: ServiceContainer;
  private readonly services = new Map<string, unknown>();
  // ...
}
```

**Correct (constructor injection; a per-call choice from a map the module builds):**

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly inventoryService: InventoryService,
    private readonly paymentService: PaymentService,
  ) {}
}

// notification/interfaces/notification-channel-adapter.interface.ts
export interface NotificationChannelAdapter {
  send(user: User, message: NotificationMessage): Promise<void>;
}

// notification/adapters/notification-email-channel.adapter.ts
@Injectable()
export class NotificationEmailChannelAdapter implements NotificationChannelAdapter {
  constructor(@Inject(NOTIFICATION_EMAIL_ADAPTER) private readonly notificationEmailAdapter: NotificationEmailAdapter) {}

  send(user: User, message: NotificationMessage): Promise<void> {
    return this.notificationEmailAdapter.sendEmail(user.email, message.subject, message.body);
  }
}

// notification/types/notification-channels.type.ts
export type NotificationChannels = ReadonlyMap<NotificationChannelKind, NotificationChannelAdapter>;

// notification/constants/notification-channels.constant.ts
export const NOTIFICATION_CHANNELS = Symbol('NOTIFICATION_CHANNELS');

// notification/notification.module.ts
@Module({
  imports: [ConditionalModule.registerWhen(SmsModule.forRootAsync({ /* ... */ }), isSmsApiConfigured)],
  providers: [
    AwsSesClient,
    AwsSesAdapter,
    { provide: NOTIFICATION_EMAIL_ADAPTER, useExisting: AwsSesAdapter },
    NotificationEmailChannelAdapter,
    {
      provide: NOTIFICATION_CHANNELS,
      inject: [NotificationEmailChannelAdapter, { token: SmsService, optional: true }],
      useFactory: (emailChannel: NotificationEmailChannelAdapter, smsService?: SmsService): NotificationChannels =>
        new Map<NotificationChannelKind, NotificationChannelAdapter>([
          [NotificationChannelKind.Email, emailChannel],
          ...(smsService ? [[NotificationChannelKind.Sms, new NotificationSmsChannelAdapter(smsService)] as const] : []),
        ]),
    },
    NotificationDeliveryService,
  ],
  exports: [NotificationDeliveryService],
})
export class NotificationModule {}

// notification/services/notification-delivery.service.ts
@Injectable()
export class NotificationDeliveryService {
  constructor(@Inject(NOTIFICATION_CHANNELS) private readonly notificationChannels: NotificationChannels) {}

  async send(user: User, kind: NotificationChannelKind, message: NotificationMessage): Promise<void> {
    const channel = this.notificationChannels.get(kind);
    if (!channel) {
      throw new Error(`No ${kind} notification channel`);
    }
    await channel.send(user, message);
  }
}
```

`NotificationChannels` holds every channel of the module by kind, and `NOTIFICATION_CHANNELS` is injected only inside the module. The factory names every implementation it puts in the map; the SMS channel exists only in a deployment with the SMS module (see `di-prefer-constructor-injection`). A channel adapts what it sends through to the one shape the map needs. `NotificationEmailChannelAdapter` sends through the module's email role, `NotificationEmailAdapter`, which `AwsSesAdapter` implements, named for its outside system (as in `di-interface-segregation`); `NotificationSmsChannelAdapter` calls another module's service, so its class takes the notification prefix, then the name of the module it calls (see `arch-resource-naming`). Neither the map nor its caller knows a provider. `NotificationDeliveryService` picks the channel, which is known only per call.

Reference: [NestJS Module Reference](https://docs.nestjs.com/fundamentals/module-ref)
