---
title: Handle Async Errors Outside the Request Cycle
impact: HIGH
impactDescription: Prevents silent failures and process crashes from unhandled rejections
tags: error-handling, async, promises, events, cron
---

## Handle Async Errors Outside the Request Cycle

Nest catches errors thrown by route handlers and hands them to the exception filter. Nothing catches errors from work that runs outside a request: promises started without `await`, `@OnEvent` listeners, `@Cron` jobs. An unhandled rejection there is at best lost, at worst a crash.

- **Await every promise** inside a request. A promise you deliberately do not await gets a `.catch()` that logs it; work that must eventually succeed goes to a queue instead (see `micro-use-queues`).
- **Every listener and scheduled job catches its own errors**, logs them as pino's `err` field, `this.logger.error({ err: error }, message)`, which keeps the stack, every cause and a thrown value that is not an Error (see `error-use-exception-filters`), and reports them with `SentryService.reportError(error)`. It never rethrows: there is no caller to receive it, and a rejection from a timer or a listener reaches the crash handlers, which end the process. The Sentry SDK captures only what such code throws, so without the report a failed job never reaches Sentry (see `error-report-to-sentry`).
- **Queue processors are the exception:** BullMQ catches what a processor throws and retries the job, so a processor throws instead of catching (see `micro-use-queues`).
- **So are listeners of outbox events:** the outbox dispatches an event again until every listener succeeded, so they throw instead of catching, and do their work idempotently. They are declared with `@OnOutboxEvent()`, whose `promisify: true` hands their promise to the dispatcher; without it, their failure is an unhandled rejection (see `arch-use-events`).
- **Log the error, not the payload.** Identify the work by id (`order ${event.orderId}`), never by logging the whole event or DTO (see `devops-use-logging`).
- The process-level `uncaughtException`/`unhandledRejection` handlers are the last resort, installed once by `src/sentry/preload.ts` (see `devops-use-logging`).

**Incorrect (floating promises, unguarded listeners and jobs):**

```typescript
@Injectable()
export class UserService {
  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(await this._toEntity(dto));
    this.mailService.sendWelcome(user.email); // Not awaited, no catch: an unhandled rejection
    return user;
  }
}

@Injectable()
export class RecommendationOrderEventsListener {
  @OnEvent('order.created')
  onOrderCreated(event: OrderCreatedEvent): void {
    this._recordPurchase(event); // Promise dropped; its failure is unhandled
  }
}

@Injectable()
export class SchedulerService {
  @Interval(60 * 60_000)
  async cleanUpTempFiles(): Promise<void> {
    await this.tempFileService.deleteStale(); // A throw here is an unhandled rejection
  }
}
```

**Correct (every path ends in a handled outcome):**

```typescript
@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(await this._toEntity(dto));
    // Best effort: deliberately not awaited
    this.mailService.sendWelcome(user.email).catch((error: unknown) => {
      this.logger.error({ err: error }, `Welcome email failed for user ${user.id}`);
      SentryService.reportError(error);
    });
    return user;
  }
}

@Injectable()
export class RecommendationOrderEventsListener {
  private readonly logger = new Logger(RecommendationOrderEventsListener.name);

  constructor(private readonly recommendationService: RecommendationService) {}

  @OnEvent('order.created', { async: true })
  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    try {
      await this.recommendationService.recordPurchase(event.orderId);
    } catch (error) {
      this.logger.error({ err: error }, `Recording purchase failed for order ${event.orderId}`);
      SentryService.reportError(error);
    }
  }
}

// scheduler/services/scheduler.service.ts
@Injectable()
export class SchedulerService {
  private readonly logger = new Logger(SchedulerService.name);
  // An interval fires whether or not its last run has ended, so a slow run would overlap the next
  private readonly running = new Set<string>();

  constructor(private readonly tempFileService: TempFileService) {}

  @Interval(TEMP_FILE_CLEANUP_INTERVAL_MS)
  cleanUpTempFiles(): Promise<void> {
    return this._runAlone('Temp file cleanup', async () => {
      const deleted = await this.tempFileService.deleteStale();
      this.logger.log(`Deleted ${deleted} stale temp files`);
    });
  }

  private async _runAlone(job: string, work: () => Promise<void>): Promise<void> {
    if (this.running.has(job)) {
      this.logger.warn(`${job}: the last run has not ended, this one is skipped`);
      return;
    }
    this.running.add(job);
    try {
      await work();
    } catch (error) {
      this.logger.error({ err: error }, `${job} failed`);
      SentryService.reportError(error);
    } finally {
      this.running.delete(job);
    }
  }
}
```

A welcome email that must arrive would be queued instead of sent unawaited. The temp file cleanup is per-instance housekeeping (this instance's temp directory), so the app's schedule fits it; a job that must run once across all instances is a BullMQ job scheduler (see `micro-use-queues`). `TEMP_FILE_CLEANUP_INTERVAL_MS` lives in the feature's `constants/`, and the interval counts from this instance's start.

Reference: [Node.js unhandledRejection](https://nodejs.org/api/process.html#event-unhandledrejection) · [NestJS Events](https://docs.nestjs.com/techniques/events)
