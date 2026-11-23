---
title: Run Background Jobs on BullMQ Queues
impact: MEDIUM-HIGH
impactDescription: Slow or must-succeed work is retried, visible and off the request path
tags: microservices, queues, bullmq, background-jobs, scheduling
---

## Run Background Jobs on BullMQ Queues

Use `@nestjs/bullmq` for work that is slow (report generation, file processing) or must eventually succeed (emails, webhooks to other systems). The request records the work, enqueues a job and returns (`202 Accepted` with the record's id and status, see `api-controller-conventions`); a worker processes it with retries.

- The Redis connection comes from config (`redis.*`, see `devops-use-config-module`) through `BullModule.forRootAsync`. An app with queues requires `REDIS_HOST`: `validate-env.ts` refuses to start without it, and no code keeps a branch for running without Redis. The fallbacks for a Redis that fails while the app runs stay (the rate limits in memory, the catalog from the database, see `security-rate-limiting` and `perf-use-caching`).
- Each feature module registers the queues it owns (`BullModule.registerQueue({ name: 'report' })`) and keeps its workers in `processors/` (see `arch-feature-modules`). Queue names are singular, kebab-case resource or concern names.
- A worker extends `WorkerHost` and implements `process(job)`, branching on `job.name`. It throws to fail a job, so BullMQ retries it with backoff; it never swallows an error. A failed attempt that will be retried is not a failure for Sentry: `@OnWorkerEvent('failed')` reports the job, and marks its record failed, once the failure is final (see `error-report-to-sentry`). A final failure is the one where `job.finishedOn` is set, which BullMQ does only when it moves the job to the failed set. Counting attempts (`job.attemptsMade >= job.opts.attempts`) misses the failures BullMQ makes final early: a processor that throws `UnrecoverableError`, and a job that stalled more than `maxStalledCount` times (a job that crashes its worker twice, such as an import that runs out of memory). Checked on BullMQ 6.7.8: both reached `failed` after 1 attempt of 3, with `finishedOn` set, and a retried failure had none. `@OnWorkerEvent('error')` logs when the worker loses Redis and when it is back, not every reconnection attempt.
- Jobs carry ids, not entities: the worker loads current data when it runs.
- **The record is saved before its job is added, and no transaction spans the two.** Add the job with `jobId: record.id`, so adding it twice queues it once (a custom job id cannot contain `:`, so a UUID fits). When `add()` fails, mark the record failed and answer 503 with the error as the `cause` (see `error-throw-http-exceptions`): a record left queued with no job would never change.
- **A claim tells a run in progress from one that stopped.** A worker that claims its record before an outside call (`queued` to `sending`, before the mail goes to SES) may meet its own job again while it is still calling: BullMQ takes a stalled job back and runs it on another worker. The second run that finds a fresh claim fails, for the queue to try again later, and leaves the outcome to the run that holds the claim; only a claim older than any call could take (5 minutes, read from the record's `updatedAt`) counts as abandoned, and goes to a person as unresolved. Treating every claim found as abandoned records a mail that went out as unresolved.
- **`add()` is bounded by a time limit.** BullMQ's `add()` does not fail while it cannot reach Redis: from startup on it neither resolves nor rejects, and after a lost connection it rejects only minutes later, so the request would hang. Every `add()` goes through `addJobInTime()`, which treats 2 seconds without an answer as a failure. It lives in the queue module, `queue/utils/add-job-in-time.ts`, with the rest of what knows BullMQ, and every module that adds jobs imports it from there. A job that BullMQ still adds after that runs only if its record still wants it, since every worker first claims its record by the record's status: a report marked failed is left alone, a mail still queued is sent once.
- **A job added for an event goes through the outbox.** A listener that records work and adds its job runs after the change committed, so a crash in between, or a Redis failure while adding, loses the work, with no request to answer 503. Work that must follow an event is recorded through the outbox (see `arch-use-events`), whose dispatch retries until the consumer has recorded it.
- **Work recorded in the change's own transaction.** When the work must be recorded together with the change that causes it (a refund owed by a canceled order, written in the cancellation's transaction), its row is the record, and a scheduled job of the `scheduler/` module sends what is due. The row then keeps what a queue job would: its attempts, the time of the next one, each wait longer than the last, and a failed state that is reported to Sentry for a person once the attempts run out. The job takes the rows by their next attempt, so rows that keep failing never hold up the others, and every instance may run it, so each send is idempotent (a key the provider deduplicates). This form also works where Redis is optional.
- **Scheduling:** a recurring job that must run once across all instances is a BullMQ job scheduler. `@nestjs/schedule` runs on every instance and starts its intervals over at each restart, so use it only for housekeeping that may run there and may be skipped now and then: work on the instance's own state (its temp directory), or work on shared state whose result is the same whichever instance runs it and however often (deleting rows past their retention). It is an `@Interval` method of the `scheduler/` module (see `arch-feature-modules` and `error-handle-async-errors`).
- **A new job kind gets a queue of its own while a release that does not know it may still run.** During a rollout, and after a rollback, the previous release's workers take jobs from every queue they know and fail a job name they have no branch for. A new kind of job goes on a new queue, which only the release that knows it has a worker for; after a rollback, its scheduler is removed by hand (`removeJobScheduler(id)`).

**A job scheduler:**

- **An upsert replaces the scheduler's pending run.** `upsertJobScheduler()` removes the delayed job of the next run and works out a new one from now. A run that came due while no worker took it (every instance stopped at that hour, every worker busy) is dropped by the next upsert: the startup of the first instance back, or an upsert repeated on an interval. So read the scheduler first with `getJobScheduler(id)`, and upsert only when it is missing or its `pattern`, `tz`, job name, data or options differ. BullMQ stores the template's options merged with the queue's default job options (`{ ...queue.defaultJobOptions, ...template.opts }`, checked on 6.7.8), so the comparison merges them the same way: compared with the template's own options, the stored ones always differ, and every start upserts and drops the due run. They come back in another key order, so compare them deeply (`isDeepStrictEqual` from `node:util`).
- **Registered in `onApplicationBootstrap`, bounded in time, and checked again.** The read and the upsert do not settle in time while Redis is down, as `add()` does not, so they run within a time limit (`addJobInTime()`). A failure is logged and reported, never thrown: the instance starts without Redis and serves what does not need it. A job of the `scheduler/` module runs the same check on an interval, so a scheduler missed at startup, or removed by hand, comes back.
- **Set `tz`.** Without it the pattern runs in the time zone of the host that registered it, and moves with that host's daylight saving time.
- **A job that reports on a period starts with a margin after the period ends.** A row stamped just before the cutoff by a transaction that commits just after it is not there yet when the run starts at the cutoff, and the next run covers the next period: start a few minutes after the period ends.

**A queue dashboard** (Bull Board) uses the adapter for your HTTP platform. The routes a library mounts on the platform itself are outside Nest: no global guard, filter, interceptor or throttler reaches them, so the dashboard's scope brings its own.

- **Authentication on every request** of the scope, the page, its files and its API alike, in a hook of the scope.
- **An error handler** that answers in the API's error shape, logs the error as `err` and reports a 5xx to Sentry (see `error-use-exception-filters`), so a malformed cookie is a 401, not the library's own 500 with the raw message.
- **A log line per request**, with the method, the path without its query string, the status, and the caller's id and address as fields, which records who did what (see `api-use-interceptors`).
- **A rate limit** of the scope (see `security-rate-limiting`), counted per caller id, or per `normalizeIp(request.ip)` from `@nestjs/throttler` for an anonymous request, so an IPv6 client counts once per /64 as it does for the throttler.
- **A dashboard opened in a browser needs a credential the browser sends by itself.** A browser navigation carries no `Authorization` header, so the hook reads a credential such as a session cookie (`HttpOnly`, `Secure`, `SameSite=Strict`, its `Path` the dashboard's). In the shop, a credentialed call from the front end sets the cookie. That call needs CORS with `credentials: true`, the origins still an allowlist (see `api-platform-agnostic`), and a `SameSite=Strict` cookie is stored and sent only when the front end and the API are on the same site. The API docs say so, and an integration test sends the credentialed preflight.
- **Read-only when the workers claim records by status** (`new BullMQAdapter(queue, { readOnlyMode: true })`). A retry from the dashboard runs a worker that finds its record already `failed`, claims nothing and completes, so the job turns green and the record stays failed. A retry goes through a route of the record's own module, which sets the record back and queues its job again.
- **Every dashboard request ends in time.** BullMQ commands do not settle in time while Redis is down, so a dashboard request would wait with no answer, and one still pending holds the shutdown (see `devops-graceful-shutdown`). Two checks cover the two ways Redis fails:
  - **503 while the queues' connection is not ready** (`queue.getBackend().connection.status` is not `'ready'`). A Redis never reached leaves it `'initializing'`, and BullMQ waits for the connection before it sends anything, so no command timeout applies.
  - **A command timeout on the dashboard's own connection.** A Redis that keeps the connection open and stops answering leaves the status `'ready'`, so the first check lets the request through. The dashboard registers its queues on a connection of its own with ioredis's `commandTimeout` (for example 2 seconds), so each command fails in time and the error handler answers. Not on the queue module's shared connection: the workers' blocking commands wait on purpose. Checked on BullMQ 6.7.8 with Redis hung behind a proxy: the status stayed `'ready'`, `getJobCounts()` had no answer after 6 seconds without the option and failed after 2 seconds with it.

**Incorrect (slow work in the request, no retries, timers):**

```typescript
@Controller('reports')
export class ReportController {
  @Post()
  async create(@AuthUser() authUser: JwtUser, @Body() dto: CreateReportDto): Promise<Report> {
    const report = await this.reportService.render(authUser, dto); // Minutes inside a request: the client times out
    await this.mailService.send(authUser.email, report);            // A failure here loses the report
    return report;
  }
}

setInterval(() => this.authSessionService.deleteExpired(), 60_000); // Runs on every instance, errors unhandled
```

**Correct (enqueue in the request, process in a worker, schedule once):**

```typescript
// redis/utils/redis-connection-options.ts
export function redisConnectionOptions(redisSettings: ConfigType<typeof redisConfig>): RedisOptions {
  return {
    host: redisSettings.host,
    port: redisSettings.port,
    username: redisSettings.username,
    password: redisSettings.password,
    // ioredis: an options object turns TLS on
    tls: redisSettings.tls ? { ca: redisSettings.tlsCa } : undefined,
  };
}

// queue/queue.module.ts
// Set before any module registers a queue or a processor
BullModule.workerClass = QueueWorker;
BullModule.queueClass = QueueBoundedQueue;

@Module({
  imports: [
    BullModule.forRootAsync({
      inject: [redisConfig.KEY],
      useFactory: (redisSettings: ConfigType<typeof redisConfig>) => ({
        connection: redisConnectionOptions(redisSettings),
        defaultJobOptions: {
          attempts: 3,
          backoff: { type: 'exponential', delay: 1_000 },
          // BullMQ keeps every finished job unless told
          removeOnComplete: 1_000,
          removeOnFail: 5_000,
        },
      }),
    }),
  ],
})
export class QueueModule {}

// report/report.module.ts
@Module({
  imports: [BullModule.registerQueue({ name: 'report' })],
  controllers: [ReportController],
  providers: [ReportService, ReportRenderService, ReportProcessor],
})
export class ReportModule {}

// report/services/report.service.ts
@Injectable()
export class ReportService {
  constructor(
    @InjectRepository(Report)
    private readonly reportRepository: Repository<Report>,
    @InjectQueue('report') private readonly reportQueue: Queue,
  ) {}

  async create(authUser: JwtUser, dto: CreateReportDto): Promise<Report> {
    const report = await this.reportRepository.save(
      this.reportRepository.create({ ...dto, userId: authUser.id, status: ReportStatus.Queued }),
    );
    try {
      await addJobInTime('report', () => this.reportQueue.add('render', { reportId: report.id }, { jobId: report.id }));
    } catch (error) {
      await this.reportRepository.update({ id: report.id, status: ReportStatus.Queued }, { status: ReportStatus.Failed });
      throw new ServiceUnavailableException('Reports cannot be queued right now', { cause: error });
    }
    return report;
  }
}

// report/processors/report.processor.ts
@Processor('report')
export class ReportProcessor extends WorkerHost {
  private readonly logger = new Logger(ReportProcessor.name);

  constructor(
    private readonly reportService: ReportService,
    private readonly reportRenderService: ReportRenderService,
  ) {
    super();
  }

  async process(job: Job<{ reportId: string }>): Promise<void> {
    switch (job.name) {
      case 'render':
        await this.reportRenderService.render(job.data.reportId);
        return;
      default:
        throw new Error(`Unknown job ${job.name}`);
    }
  }

  @OnWorkerEvent('failed')
  async onFailed(job: Job<{ reportId: string }>, error: Error): Promise<void> {
    this.logger.error({ err: error }, `Job ${job.name} ${job.id} failed (attempt ${job.attemptsMade})`);
    // BullMQ sets finishedOn only once the job is in the failed set: attempts spent, an UnrecoverableError, or stalled
    // past maxStalledCount
    if (!job.finishedOn) {
      return;
    }
    SentryService.reportError(error);
    try {
      await this.reportService.markFailed(job.data.reportId);
    } catch (markError) {
      this.logger.error({ err: markError }, `Report ${job.data.reportId} could not be marked failed`);
    }
  }
}

// report/constants/report-nightly.constant.ts
export const REPORT_NIGHTLY_QUEUE = 'report-nightly';
export const REPORT_NIGHTLY_SCHEDULER_ID = 'report-nightly';
export const REPORT_NIGHTLY_SCHEDULE = { pattern: '15 0 * * *', tz: 'Europe/Berlin' };

// report/services/report-nightly.service.ts
@Injectable()
export class ReportNightlyService implements OnApplicationBootstrap {
  private readonly logger = new Logger(ReportNightlyService.name);

  constructor(@InjectQueue(REPORT_NIGHTLY_QUEUE) private readonly reportNightlyQueue: Queue) {}

  // After every onModuleInit, so the workers exist
  async onApplicationBootstrap(): Promise<void> {
    await this.ensureSchedule();
  }

  async ensureSchedule(): Promise<void> {
    try {
      await addJobInTime(REPORT_NIGHTLY_QUEUE, () => this._ensureSchedule());
    } catch (error) {
      this.logger.error({ err: error }, 'Could not check the nightly report schedule');
      SentryService.reportError(error);
    }
  }

  private async _ensureSchedule(): Promise<void> {
    const template = { name: 'build', data: {}, opts: { attempts: 5, backoff: { type: 'exponential', delay: 60_000 } } };
    const existing = await this.reportNightlyQueue.getJobScheduler(REPORT_NIGHTLY_SCHEDULER_ID);
    // BullMQ stores the template's options over the queue's default job options
    const storedOpts = { ...this.reportNightlyQueue.defaultJobOptions, ...template.opts };
    const unchanged =
      existing?.pattern === REPORT_NIGHTLY_SCHEDULE.pattern &&
      existing.tz === REPORT_NIGHTLY_SCHEDULE.tz &&
      existing.name === template.name &&
      isDeepStrictEqual(existing.template?.data ?? {}, template.data) &&
      isDeepStrictEqual(existing.template?.opts ?? {}, storedOpts);
    if (!unchanged) {
      // upsertJobScheduler() replaces the pending run, dropping one that came due
      await this.reportNightlyQueue.upsertJobScheduler(REPORT_NIGHTLY_SCHEDULER_ID, REPORT_NIGHTLY_SCHEDULE, template);
    }
  }
}
```

BullMQ opens its Redis connections itself, from options: they start from the redis module's `redisConnectionOptions()`, the one mapping of the `redis` settings that every connection of the app starts from (see `arch-module-sharing`), the queue module's and the dashboard's alike. The queue module holds the root config and the queue and worker classes, which close within a time limit (see `devops-graceful-shutdown`), and `AppModule` imports it. The numbers of finished jobs kept for inspection (the last 1000 completed, 5000 failed) are examples.

`ReportService` is the producer. The controller answers 202 with the report's id and status. When the job cannot be added, the report is marked failed, since a queued report that no job will render would never change. `ReportProcessor` is the worker: `ReportRenderService` (in `report/services/report-render.service.ts`) throws to fail the job, and BullMQ retries it. A failure that is not final returns early, since a retry follows. `markFailed()` changes the report only from the status it expects, as every claim does.

The nightly report's schedule is in `constants/`, since operators see when it runs. The pattern and the time zone are examples: 00:15 leaves a margin after midnight. Its queue is a new one, since the release before has no worker for it. `ReportNightlyService` keeps one run a night across all instances. A job of the `scheduler/` module also calls `ensureSchedule()` on an interval, and a failed check is logged and reported while the instance still starts.

**The dashboard's queues on a connection of their own**, whose commands fail in time:

```typescript
// queue-dashboard/queue-dashboard.module.ts
const QUEUE_DASHBOARD_COMMAND_TIMEOUT_MS = 2_000;

@Module({
  imports: [
    BullModule.registerQueueAsync(
      ...['report', REPORT_NIGHTLY_QUEUE].map((name) => ({
        name,
        inject: [redisConfig.KEY],
        useFactory: (redisSettings: ConfigType<typeof redisConfig>) => ({
          connection: { ...redisConnectionOptions(redisSettings), commandTimeout: QUEUE_DASHBOARD_COMMAND_TIMEOUT_MS },
        }),
      })),
    ),
  ],
  providers: [QueueDashboardService],
})
export class QueueDashboardModule {}

// queue-dashboard/services/queue-dashboard.service.ts
if (this.queues.some((queue) => queue.getBackend().connection.status !== 'ready')) {
  reply.header('Retry-After', '5');
  reply.code(HttpStatus.SERVICE_UNAVAILABLE).send({
    success: false,
    statusCode: HttpStatus.SERVICE_UNAVAILABLE,
    error: 'Service Unavailable',
    message: 'The queues are unavailable',
    errorCode: 'SERVICE_UNAVAILABLE',
  });
}
```

A queue's own connection replaces the root's connection options, while the root's default job options still apply, so the dashboard starts from `redisConnectionOptions()` and adds `commandTimeout`. The status check runs in the scope's `onRequest` hook, after authentication: it covers a Redis never reached, where no command timeout applies. The 2-second command timeout and the `Retry-After` of 5 seconds are examples.

Reference: [NestJS Queues (BullMQ)](https://docs.nestjs.com/techniques/queues) · [BullMQ Job Schedulers](https://docs.bullmq.io/guide/job-schedulers)
