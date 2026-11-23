---
title: Shut Down Gracefully with Nest's Lifecycle Hooks
impact: MEDIUM-HIGH
impactDescription: Deployments finish in-flight requests and jobs instead of cutting them off
tags: devops, graceful-shutdown, lifecycle, kubernetes
---

## Shut Down Gracefully with Nest's Lifecycle Hooks

The drain comes first, while every part of the app still works, and `app.close()` runs after it. Nest's own `app.enableShutdownHooks()` does it the other way round: on `SIGTERM`, `app.close()` runs `onModuleDestroy` first, where `@nestjs/bullmq` closes the queue workers, so readiness would fail only after the longest job in progress ended, and never while Redis is down, since a worker that cannot reach Redis never finishes closing. So `main.ts` calls `HealthModule.listenForShutdown(app)` instead. On `SIGTERM` or `SIGINT` it:

1. **Drains.** `HealthShutdownService.drain()` fails readiness and waits `app.shutdownDrainMs` while the load balancer stops routing to this instance; the HTTP server still serves.
2. **Closes the app** with `app.close()`, whose hooks run in this order, each step in its hook:
   - `onModuleDestroy`: background work stops. The queue workers close within a time limit (`QueueWorker`, see below).
   - `beforeApplicationShutdown`: anything that must still reach a dependency before the server closes, and the long-lived connections end: SSE streams complete and Socket.IO clients are disconnected (see `api-real-time-updates`).
   - Nest closes the HTTP server, which stops accepting connections and lets in-flight requests finish. Never create the app with `forceCloseConnections: true`: it destroys every open connection, a request still running included (a checkout waiting on another service, which then never records its outcome). The server's close waits for every connection, and Node and the adapters close only the idle ones, once, when it starts, so a keep-alive connection whose request finishes during the close stays open until its timeout (72 s on Fastify): `HealthModule.listenForShutdown()` calls `server.closeIdleConnections()` every 100 ms while `app.close()` runs.
   - `onApplicationShutdown`: the server is closed. Release what is left (connections, clients); TypeORM closes its own pool.
3. **Ends the process with the signal** (`process.kill(process.pid, signal)`, as Nest does once its hooks have run), never `process.exit(0)`. Its listeners are gone by then, so a second signal ends the process at once.

The crash handlers for `uncaughtException` (see `devops-use-logging`) are a different case: the process is already broken, and exiting is their job.

**In a container, step 3 needs an init.** The process an image starts runs as PID 1, and the kernel delivers to PID 1 no signal it has no handler for. The re-raised signal is then ignored, so the process ends only when its event loop empties, with status 0, and a second signal does not end it at once. Run Node under an init that forwards signals (tini, or the container runtime's `--init`).

**Queues close within a time limit too.** `@nestjs/bullmq` closes every queue once the HTTP server is closed, and BullMQ's close waits for the answer to `QUIT`, which a Redis that keeps the connection open and stops answering never sends: the shutdown then never ends. The queue module sets `BullModule.queueClass` to `QueueBoundedQueue` (in `queue/queues/`, see `arch-feature-modules`), whose `close()` disconnects past 1 s. Both classes are set when the queue module is loaded, and each feature module builds its queues and workers when it is imported, so `app.module.ts` imports the queue module before every feature module, with a comment saying why.

**Queues log their Redis errors.** A BullMQ queue with no `error` listener writes every Redis error with `console.error` (checked on 6.7.8: five stacks in 5 seconds against a closed port, none once a listener was added), past pino, the request id and the redactor. `QueueBoundedQueue` listens in its constructor and logs when the queue loses Redis and when it is back, as the workers do (see `micro-use-queues`), not every reconnection attempt.

**Queue workers close within a time limit.** BullMQ's graceful close waits for the jobs in progress, and never ends while it cannot reach Redis. `@nestjs/bullmq` builds every processor's worker from `BullModule.workerClass`, so the queue module sets it to `QueueWorker`, a `Worker` whose `close()` waits `QUEUE_WORKER_CLOSE_TIMEOUT_MS` (10 s in the example) and then disconnects. A second `close(true)` would not end it: BullMQ 6.7.8 keeps the first close's promise and hands it to every later call, so the forced close waits for the same `QUIT` (checked: still closing after 15 s against a Redis that stopped answering, where `disconnect()` ended it at the limit, 2 s in the check). A worker that never reached Redis has taken no job and closes at once. A job left in progress keeps its lock until the lock expires, and the stalled-job check of another instance queues it again, so jobs stay idempotent (see `micro-use-queues`).

The orchestrator's grace period (`terminationGracePeriodSeconds`, 30 s by default on Kubernetes) must exceed the drain delay, the workers' close limit, the longest request and Sentry's flush (2 s): 40 s with the defaults.

The drain delay is a setting, `app.shutdownDrainMs` (`APP_SHUTDOWN_DRAIN_MS`, 15 s by default, see `devops-use-config-module`), so a local run can turn it off: `.env.example` sets it to `0`. No load balancer runs locally, and in watch mode the Nest CLI starts the new process only after the old one exits, so every restart would wait out the full drain.

**Incorrect (Nest's order, an exit that cuts the hooks, work in the wrong hook):**

```typescript
// main.ts
app.enableShutdownHooks();           // Workers close before readiness fails: the drain waits for every job
process.on('SIGTERM', async () => {  // Races Nest's own handler
  await app.close();
  process.exit(0);                   // Cuts off any hook still running
});

@Injectable()
export class HealthShutdownService implements OnApplicationShutdown {
  async onApplicationShutdown(): Promise<void> {
    this.shuttingDown = true;                           // Too late: the HTTP server is already closed
    await new Promise((resolve) => setTimeout(resolve, 5_000));
  }
}
```

**Correct (the drain first, then Nest's hooks, each step in its place):**

```typescript
// main.ts
HealthModule.listenForShutdown(app);

// health/health.module.ts
const SHUTDOWN_SIGNALS: NodeJS.Signals[] = ['SIGTERM', 'SIGINT'];
const CLOSE_IDLE_CONNECTIONS_INTERVAL_MS = 100;

@Module({ imports: [TerminusModule], controllers: [HealthController], providers: [HealthShutdownService] })
export class HealthModule {
  static listenForShutdown(app: INestApplication): void {
    const onSignal = (signal: NodeJS.Signals): void => {
      for (const each of SHUTDOWN_SIGNALS) {
        // A second signal ends the process at once
        process.removeListener(each, onSignal);
      }
      HealthModule._shutDown(app, signal).catch((error: unknown) => {
        new Logger(HealthModule.name).error({ err: error }, 'The shutdown failed');
        process.exit(1);
      });
    };
    for (const signal of SHUTDOWN_SIGNALS) {
      process.on(signal, onSignal);
    }
  }

  private static async _shutDown(app: INestApplication, signal: NodeJS.Signals): Promise<void> {
    await app.get(HealthShutdownService).drain(signal);
    // The server closes only the connections idle when its close starts: one whose request ends later stays open
    const closingIdle = setInterval(() => app.getHttpServer().closeIdleConnections(), CLOSE_IDLE_CONNECTIONS_INTERVAL_MS);
    try {
      await app.close();
    } finally {
      clearInterval(closingIdle);
    }
    process.kill(process.pid, signal);
  }
}

// health/services/health-shutdown.service.ts
@Injectable()
export class HealthShutdownService {
  private readonly logger = new Logger(HealthShutdownService.name);
  private shuttingDown = false;

  constructor(
    private readonly healthIndicatorService: HealthIndicatorService,
    @Inject(appConfig.KEY) private readonly appSettings: ConfigType<typeof appConfig>,
  ) {}

  isServing(key: string): HealthIndicatorResult {
    const indicator = this.healthIndicatorService.check(key);
    return this.shuttingDown ? indicator.down({ message: 'Shutting down' }) : indicator.up();
  }

  async drain(signal: string): Promise<void> {
    const drainMs = this.appSettings.shutdownDrainMs;
    this.logger.log(`Received ${signal}, draining for ${drainMs} ms`);
    this.shuttingDown = true;
    await new Promise((resolve) => setTimeout(resolve, drainMs));
  }
}

// queue/workers/queue.worker.ts
export class QueueWorker extends Worker {
  private reachedRedis = false;

  constructor(name: string, processor: Processor, opts: WorkerOptions) {
    super(name, processor, opts);
    this.getBackend().once('ready', () => {
      this.reachedRedis = true;
    });
  }

  override async close(force = false): Promise<void> {
    if (force || !this.reachedRedis) {
      // A worker that never reached Redis has taken no job
      return super.close(true);
    }
    const closing = super.close();
    if (!(await settlesWithin(closing, QUEUE_WORKER_CLOSE_TIMEOUT_MS))) {
      // close(true) would return the pending close, still waiting for QUIT. Its jobs in progress are taken again by
      // another instance once their locks expire
      await this.disconnect();
    }
  }
}

// queue/queues/queue-bounded.queue.ts
export class QueueBoundedQueue extends Queue {
  private readonly logger = new Logger(QueueBoundedQueue.name);
  private available = true;

  constructor(name: string, opts?: QueueOptions) {
    super(name, opts);
    // Without an error listener, BullMQ writes each Redis error with console.error
    this.on('error', (error: Error) => {
      if (this.available) {
        this.available = false;
        this.logger.warn({ err: error }, `The ${name} queue lost Redis`);
      }
    });
    this.getBackend().on('ready', () => {
      if (!this.available) {
        this.available = true;
        this.logger.log(`The ${name} queue reached Redis again`);
      }
    });
  }

  override async close(): Promise<void> {
    if (!(await settlesWithin(super.close(), QUEUE_CLOSE_TIMEOUT_MS))) {
      await this.disconnect();
    }
  }
}

// search/services/search-index.service.ts
@Injectable()
export class SearchIndexService implements OnApplicationShutdown {
  constructor(@Inject(SEARCH_CLIENT) private readonly client: SearchClient) {}

  async onApplicationShutdown(): Promise<void> {
    await this.client.close();
  }
}
```

`main.ts` calls `HealthModule.listenForShutdown(app)` in place of `app.enableShutdownHooks()`, and the process ends with the signal, as Nest ends it. The readiness probe runs `isServing()` as one of its checks (see `micro-use-health-checks`). The drain's default of 15 s is an example, longer than the readiness probe's `periodSeconds` x `failureThreshold` (5 s x 2 = 10 s in the example). `@nestjs/bullmq` builds every processor's worker as a `QueueWorker` (`BullModule.workerClass`) and every registered queue as a `QueueBoundedQueue` (`BullModule.queueClass`). A client the app owns, like the search client here, is closed in `onApplicationShutdown`, after the server.

Reference: [NestJS Lifecycle Events](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)
