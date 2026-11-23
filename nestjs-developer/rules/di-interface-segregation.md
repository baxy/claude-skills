---
title: Apply Interface Segregation Principle
impact: HIGH
impactDescription: Consumers depend only on what they call, and test doubles stay one method long
tags: dependency-injection, interfaces, solid, isp
---

## Apply Interface Segregation Principle

Clients should not be forced to depend on interfaces they don't use. In NestJS, this means keeping interfaces small and focused on specific capabilities rather than creating "fat" interfaces that bundle unrelated methods. When a service only needs to send emails, it shouldn't depend on an interface that also includes SMS, push notifications, and logging. Split large interfaces into role-based ones.

**Incorrect (fat interface forcing unused dependencies):**

```typescript
// Fat interface - forces all consumers to depend on everything
interface NotificationAdapter {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
  sendSms(phone: string, message: string): Promise<void>;
  sendPush(userId: string, notification: NotificationPushPayload): Promise<void>;
  sendSlack(channel: string, message: string): Promise<void>;
  logNotification(type: string, payload: unknown): Promise<void>;
  getDeliveryStatus(id: string): Promise<NotificationDeliveryStatus>;
  retryFailed(id: string): Promise<void>;
  scheduleNotification(dto: ScheduleNotificationDto): Promise<string>;
}

// Consumer only needs email, but must mock everything for tests
@Injectable()
export class OrderService {
  constructor(
    private readonly notifications: NotificationAdapter, // Depends on 8 methods, uses 1
  ) {}

  async confirmOrder(user: User, order: Order): Promise<void> {
    await this.notifications.sendEmail(
      user.email,
      'Order Confirmed',
      `Your order ${order.id} has been confirmed.`,
    );
  }
}

// Testing is painful - must mock unused methods
const mockNotificationAdapter = {
  sendEmail: jest.fn(),
  sendSms: jest.fn(),           // Never used, but required
  sendPush: jest.fn(),          // Never used, but required
  sendSlack: jest.fn(),         // Never used, but required
  logNotification: jest.fn(),   // Never used, but required
  getDeliveryStatus: jest.fn(), // Never used, but required
  retryFailed: jest.fn(),       // Never used, but required
  scheduleNotification: jest.fn(), // Never used, but required
};
```

**Correct (segregated interfaces by capability):**

The small interfaces and their tokens belong to the module that owns the capability, and stay inside it (see `di-use-interfaces-tokens`): the module's own services inject the ones they call, and other modules inject those services (see `arch-module-sharing`).

```typescript
// notification/constants/notification-email-adapter.constant.ts
export const NOTIFICATION_EMAIL_ADAPTER = Symbol('NOTIFICATION_EMAIL_ADAPTER');

// notification/constants/notification-sms-adapter.constant.ts
export const NOTIFICATION_SMS_ADAPTER = Symbol('NOTIFICATION_SMS_ADAPTER');

// notification/interfaces/notification-email-adapter.interface.ts
export interface NotificationEmailAdapter {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

// notification/interfaces/notification-sms-adapter.interface.ts
export interface NotificationSmsAdapter {
  sendSms(phone: string, message: string): Promise<void>;
}

// notification/interfaces/notification-push-adapter.interface.ts
export interface NotificationPushAdapter {
  sendPush(userId: string, notification: NotificationPushPayload): Promise<void>;
}

// notification/adapters/aws-ses.adapter.ts
@Injectable()
export class AwsSesAdapter implements NotificationEmailAdapter {
  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    // SES-specific implementation
  }
}

// notification/adapters/twilio.adapter.ts
@Injectable()
export class TwilioAdapter implements NotificationSmsAdapter {
  async sendSms(phone: string, message: string): Promise<void> {
    // Twilio-specific implementation
  }
}

// notification/services/notification.service.ts
@Injectable()
export class NotificationService {
  constructor(
    @Inject(NOTIFICATION_EMAIL_ADAPTER) private readonly notificationEmailAdapter: NotificationEmailAdapter,
  ) {}

  async sendOrderConfirmation(email: string, orderId: string): Promise<void> {
    await this.notificationEmailAdapter.sendEmail(email, 'Order Confirmed', `Your order ${orderId} has been confirmed.`);
  }
}

// notification/notification.module.ts
@Module({
  providers: [
    { provide: NOTIFICATION_EMAIL_ADAPTER, useClass: AwsSesAdapter },
    { provide: NOTIFICATION_SMS_ADAPTER, useClass: TwilioAdapter },
    NotificationService,
    NotificationAlertService,
  ],
  exports: [NotificationService, NotificationAlertService],
})
export class NotificationModule {}

// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(private readonly notificationService: NotificationService) {}
}

// test/e2e/notification/notification.service.spec.ts
const moduleRef = await Test.createTestingModule({ imports: [NotificationModule] })
  .overrideProvider(NOTIFICATION_EMAIL_ADAPTER)
  .useValue({ sendEmail: jest.fn() } satisfies NotificationEmailAdapter)
  .compile();
```

There is one token per capability, injected only inside the notification module, and one capability per interface, one interface per file (see `arch-feature-modules`). An outside system gets one adapter for each interface it can honor: a system behind two gets two, named for the system and the role (`TwilioSmsAdapter`, `TwilioVoiceAdapter`). `NotificationService` depends only on what it calls. The module keeps the tokens inside and exports its services, so another module, like the order module, injects `NotificationService`, never a token. The test replaces only what is used, one method long.

**Combining interfaces when needed:**

A service that needs two channels injects each one:

```typescript
// notification/services/notification-alert.service.ts
@Injectable()
export class NotificationAlertService {
  constructor(
    @Inject(NOTIFICATION_EMAIL_ADAPTER) private readonly notificationEmailAdapter: NotificationEmailAdapter,
    @Inject(NOTIFICATION_SMS_ADAPTER) private readonly notificationSmsAdapter: NotificationSmsAdapter,
  ) {}

  async sendCriticalAlert(user: User, message: string): Promise<void> {
    await Promise.all([
      this.notificationEmailAdapter.sendEmail(user.email, 'Critical Alert', message),
      this.notificationSmsAdapter.sendSms(user.phone, message),
    ]);
  }
}
```

Reference: [Interface Segregation Principle](https://en.wikipedia.org/wiki/Interface_segregation_principle)
