---
title: Report Errors to Sentry with One SDK and an Allowlist
impact: HIGH
impactDescription: Every real failure reaches Sentry, and no secret, credential or personal data leaves with it
tags: error-handling, sentry, observability, security, secrets, logging
---

## Report Errors to Sentry with One SDK and an Allowlist

Sentry receives every real failure (a 5xx response, a failed job, a crash) and nothing else. Version 11 of the Sentry SDK collects rich context by default: request and response headers, bodies and cookies, query parameters, the bound parameters and results of database queries, user data and each stack frame's local variables. Its filtering of sensitive values works on keys (a header, a cookie, a query parameter), so a secret inside an error message, which has no key, goes out unscrubbed. Every event is therefore **rebuilt from an allowlist** of fields that are never sensitive, and every configured secret value is replaced wherever it appears, in Sentry events and log lines alike.

**One SDK, started before the app:**

- `@sentry/nestjs` is the only Sentry dependency, at one exact version, and only `src/sentry/preload.ts` and `src/sentry/` import it. Two SDKs of different majors make two clients, and an error captured through the one that was never initialized is dropped without a trace.
- `src/sentry/preload.ts` starts the SDK before the app's modules load: `node --import ./dist/sentry/preload.js` in an ESM project, `node --require ./dist/sentry/preload.js` in CommonJS, Sentry's documented methods. Sentry's docs call this file `instrument.ts` and put it at the top level; here it is named `preload.ts` for what Node does with it, and it lives in the Sentry module, at the module's root next to `sentry.module.ts`: everything Sentry needs is in one folder. It also installs the process crash handlers, which report through Sentry. Every start script passes the flag (`start:dev` through `nest start --exec`), and so does the image's `ENTRYPOINT`. The local scripts (`start`, `start:dev`, `start:debug`) also pass `--env-file-if-exists=.env`: `ConfigModule` reads `.env` only when `AppModule` loads, after `preload.ts` has run, so without the flag a `SENTRY_DSN` in `.env` never starts the SDK and the secrets in `.env` are never redacted. `start:prod` and the image leave it out: a deployment gets its variables from the environment. Importing it from `main.ts` is not equivalent in ESM, where the SDK can wrap the libraries it instruments only when it starts through `--import`. `preload.ts` imports only `@sentry/nestjs` and the Nest-free files of `src/sentry/`; the Nest parts of the SDK live in `@sentry/nestjs/setup`, which only `SentryModule` imports.
- `preload.ts` runs before `ConfigModule` exists, so it reads `SENTRY_DSN`, `APP_ENV` and `APP_RELEASE` from `process.env` (the exception in `devops-use-config-module`), and the `.env.example` sync test reads it too.
- Without `SENTRY_DSN`, `SentryService.init()` does not call `Sentry.init()`, so a local run sends nothing. It still loads the secrets, so log lines are redacted.
- `--enable-source-maps` makes stack traces point at the TypeScript source, with no source map upload.
- `SentryService` makes every call into the SDK, through static methods: none of its callers gets an instance from Nest (`preload.ts`, the crash handlers, pino's write hook, the CLS `setup` hook), and the global filter calls it the same way. It carries no Nest decorator, so `preload.ts` loads no Nest code before the SDK starts.
- `AppModule` imports `SentryModule` (`src/sentry/sentry.module.ts`). It imports the SDK's own `SentryModule.forRoot()` from `@sentry/nestjs/setup`, which names each event after the route pattern (`POST /orders/:id/cancel`), never the URL with its ids. It also flushes the queued events on shutdown: Nest ends the process with the signal after its hooks run, so the SDK's own flush on `beforeExit` never runs.

**Errors only, until a rule covers more.** No `tracesSampleRate` or `tracesSampler`, so no span is created, and no profiling; `beforeSendLog` and `beforeSendMetric` drop everything; `maxBreadcrumbs: 0`; every `dataCollection` category is off. No release-health sessions either: the default `ProcessSession` integration is removed and the HTTP integration's request sessions are off, since every probe would count as a session and every reported 5xx as a crash. That integration's spans are off too (`httpIntegration({ sessions: false, spans: false })`). No trace headers on outgoing requests (`tracePropagationTargets: []`): even without tracing, the SDK adds `sentry-trace` and `baggage` to every call, and `baggage` tells the provider the environment, the release and the DSN's public key. Client reports stay on (`sendClientReports: true`): they count the events the SDK dropped, by reason, without their content, and are the one sign in Sentry of an error that never arrived. Logs go to pino (see `devops-use-logging`): no `Sentry.logger`, and no `pinoIntegration` or `consoleLoggingIntegration`, which would send an unredacted copy of every line. Never set:

- `debug: true` in a deployed environment: the SDK prints the URL of every outgoing request, and a provider's URL can carry its key.
- `maxValueLength`: the SDK truncates before `beforeSend` runs, so a secret cut in half no longer matches the redactor.
- `includeLocalVariables`: every frame's local variables, secrets included.

**What reaches Sentry.** `beforeSend` returns a new event built from these fields, never the SDK's event with fields deleted, because a denylist misses the next field the SDK adds:

| Field | Sent |
|---|---|
| `event_id`, `timestamp`, `level`, `platform`, `environment`, `release`, `sdk` | as they are |
| `transaction` | the route pattern |
| `exception.values[]` | `type`, `mechanism`, `value` redacted, and each frame's `filename`, `function`, `lineno`, `colno` and `in_app` |
| `message` | redacted |
| `tags` | `request_id` only, to find the request's log lines |

Everything else is dropped: `request`, `user`, `contexts`, `breadcrumbs`, `extra`, `modules`, `server_name`, and each frame's source lines and variables. Every hook catches its own errors and returns `null`: when `beforeSend` throws, the SDK drops the event and sends a new one about the failure instead.

**Secrets.** The redactor replaces each configured secret with `[REDACTED]`, both the raw value and its JSON-escaped form, longest first. The secrets are the values of the environment variables whose names end in `_KEY`, `_SECRET`, `_PASSWORD`, `_TOKEN` or `_URL`, plus the password of every URL among them, which is why secrets are named that way (see `devops-use-config-module`). Values shorter than 8 characters are skipped, so a local password like `admin` does not blank out the word everywhere. The same redactor, `SentryService.redactSecrets()`, runs on every log line through pino's `hooks.streamWrite`, which receives each serialized line just before it is written, and on what the crash handlers print. Sentry's server-side data scrubbing stays on in the project's settings as a second layer, but nothing relies on it.

**What is captured, and where:**

- **HTTP:** the global exception filter (see `error-use-exception-filters`) reports every response of 500 or above with `SentryService.reportServerError()`, and nothing below 500. The event carries the request id: the CLS `setup` hook tags the request's Sentry scope with it (see `devops-use-logging`). Not `@SentryExceptionCaptured()`, which skips every `HttpException`: a 502 or 503 a service throws on purpose because a provider is down would never reach Sentry.
- **Jobs:** a worker throws to fail an attempt, so BullMQ retries it, and an attempt that will be retried is not a failure. The SDK's `Nest` integration wraps each `@Processor()`'s `process()` and captures what it throws, every attempt, and has no option to leave BullMQ out, so the init filters the `Nest` integration out (without tracing, wrapping handlers and capturing them is all it does). `@OnWorkerEvent('failed')` reports the job with `SentryService.reportError()` once the failure is final, which `micro-use-queues` reads from `job.finishedOn` (attempts spent, an `UnrecoverableError`, or a stalled job), and a failure that is the other side's to fix (a customer's webhook receiver refusing a delivery) is not reported at all.
- **Scheduled jobs and listeners:** with the `Nest` integration gone, nothing captures what `@Interval`, `@Cron` and `@OnEvent` methods throw, and by the rules they never throw (see `error-handle-async-errors`). Each catches, logs and calls `SentryService.reportError()`, which captures the error as handled. So does code that leaves something to a person, such as a payment whose outcome can no longer be learned (see `db-use-transactions`).
- **Crashes:** the SDK's `onUncaughtException` and `onUnhandledRejection` integrations capture. `onUnhandledRejection` runs with `mode: 'none'`: its default, `'warn'`, prints the rejection's stack with `console.error`, past the redactor. The crash handlers that `preload.ts` installs (see `devops-use-logging`) print a redacted line, flush and exit, for an uncaught exception and an unhandled rejection alike.
- **Shutdown:** `SentryModule` calls `SentryService.close()` in `onApplicationShutdown`.

**Testing.** The global filter's e2e spec does not drive the SDK: no DSN is set, so nothing is sent, and a spy on `SentryService.reportServerError` checks that a 5xx calls it and a 4xx does not. `SentryService` has a spec of its own on the real SDK, with a recording `transport` and a release set (the SDK sends sessions only for a release). Its secret is a marker built at runtime, so no source line or stack frame contains it; it goes into an environment variable named as a secret and into a thrown error's message. The spec checks that exactly one envelope is sent, an event with the error's type and the request id, that no envelope contains the marker, and that outgoing requests carry no Sentry header.

**Incorrect (two SDKs, rich defaults, a denylist, a late import):**

```typescript
// src/sentry/preload.ts
import * as SentryNode from '@sentry/node';          // 7.x initialized here...
import { SentryGlobalFilter } from '@sentry/nestjs'; // ...11.x used for the filter: it has no client and sends nothing

SentryNode.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 1.0,                 // Spans carry headers, queries and outgoing URLs
  maxValueLength: 250,                   // Truncates before beforeSend: half a secret no longer matches
  debug: true,                           // Prints every outgoing URL, API keys included
  beforeSend(event) {
    delete event.request?.headers?.authorization; // The cookies, the body and the message still go out
    return event;
  },
});

// main.ts
import './sentry/preload'; // In ESM, too late for the SDK to wrap what it instruments
```

**Correct (one package, an allowlist, the redactor on events and logs):**

```typescript
// src/sentry/preload.ts
import { SentryService } from './services/sentry.service';

SentryService.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.APP_ENV,
  release: process.env.APP_RELEASE,
  env: process.env,
});
// ... the crash handlers
```

`APP_RELEASE` is the image tag, empty locally. The redactor takes the secrets from `env`, the variables named as secrets. The crash handlers follow, before the app's first module loads (see `devops-use-logging`). The start scripts of an ESM project, where a CommonJS project passes `--require` instead of `--import`:

```jsonc
// package.json
"start:dev": "nest start --watch --exec \"node --env-file-if-exists=.env --enable-source-maps --import ./dist/sentry/preload.js\"",
"start:prod": "node --enable-source-maps --import ./dist/sentry/preload.js dist/main.js"
```

```typescript
// sentry/services/sentry.service.ts
import * as Sentry from '@sentry/nestjs';

export class SentryService {
  private static secrets: string[] = [];

  static init(options: SentryInitOptions): void {
    // Before the DSN check: log lines are redacted even when nothing is sent
    SentryService.secrets = secretValues(options.env);
    if (!options.dsn) {
      return;
    }
    Sentry.init({
      dsn: options.dsn,
      environment: options.environment,
      release: options.release,
      tracePropagationTargets: [],
      maxBreadcrumbs: 0,
      dataCollection: {
        userInfo: false,
        cookies: false,
        httpHeaders: false,
        httpBodies: [],
        urlQueryParams: false,
        databaseQueryData: false,
        queues: false,
        graphQL: { document: false, variables: false },
        genAI: { inputs: false, outputs: false },
        mcp: { inputs: false, outputs: false },
        stackFrameVariables: false,
        frameContextLines: 0,
      },
      beforeSend: (event) => SentryService._dropOnError(() => rebuildEvent(event, SentryService.redactSecrets)),
      beforeSendLog: () => null,
      beforeSendMetric: () => null,
      sendClientReports: true,
      integrations: (defaults) =>
        defaults
          // ProcessSession counts release-health sessions; Nest captures every failed attempt of a job
          .filter((integration) => !['ProcessSession', 'Nest'].includes(integration.name))
          .map((integration) => {
            if (integration.name === 'Http') {
              return Sentry.httpIntegration({ sessions: false, spans: false });
            }
            if (integration.name === 'OnUnhandledRejection') {
              return Sentry.onUnhandledRejectionIntegration({ mode: 'none' });
            }
            return integration;
          }),
      transport: options.transport,
    });
  }

  static redactSecrets(text: string): string {
    return SentryService.secrets.reduce(
      (redacted, secret) =>
        redacted.split(secret).join('[REDACTED]').split(JSON.stringify(secret).slice(1, -1)).join('[REDACTED]'),
      text,
    );
  }

  static setRequestId(requestId: string | undefined): void {
    Sentry.getIsolationScope().setTag('request_id', requestId);
  }

  static reportServerError(exception: unknown): void {
    Sentry.captureException(exception, { mechanism: { handled: false, type: 'auto.function.nestjs.exception_captured' } });
  }

  static reportError(exception: unknown): void {
    Sentry.captureException(exception, { mechanism: { handled: true, type: 'generic' } });
  }

  static async close(): Promise<void> {
    await Sentry.close(2_000);
  }

  // A hook that throws makes the SDK send a new event about the failure: drop the event instead
  private static _dropOnError<T>(build: () => T): T | null {
    try {
      return build();
    } catch {
      return null;
    }
  }
}

// sentry/utils/secret-values.ts
const SECRET_SUFFIXES = ['_KEY', '_SECRET', '_PASSWORD', '_TOKEN', '_URL'];
// Shorter values would blank out ordinary words
const MIN_SECRET_LENGTH = 8;

export function secretValues(env: Record<string, string | undefined>): string[] {
  const values = Object.entries(env).flatMap(([name, value]) => {
    if (!value || !SECRET_SUFFIXES.some((suffix) => name.endsWith(suffix))) {
      return [];
    }
    return name.endsWith('_URL') ? [value, urlPassword(value)] : [value];
  });
  return [...new Set(values)].filter((value) => value.length >= MIN_SECRET_LENGTH).sort((a, b) => b.length - a.length);
}

function urlPassword(url: string): string {
  try {
    return decodeURIComponent(new URL(url).password);
  } catch {
    return '';
  }
}

// sentry/utils/rebuild-event.ts
import type { ErrorEvent } from '@sentry/nestjs';

export function rebuildEvent(event: ErrorEvent, redact: (text: string) => string): ErrorEvent {
  const requestId = event.tags?.request_id;
  return {
    type: undefined,
    event_id: event.event_id,
    timestamp: event.timestamp,
    level: event.level,
    platform: event.platform,
    environment: event.environment,
    release: event.release,
    sdk: event.sdk,
    transaction: event.transaction,
    message: event.message && redact(event.message),
    tags: requestId === undefined ? undefined : { request_id: requestId },
    exception: {
      values: event.exception?.values?.map((exception) => ({
        type: exception.type,
        mechanism: exception.mechanism,
        value: exception.value && redact(exception.value),
        stacktrace: {
          frames: exception.stacktrace?.frames?.map(({ filename, function: fn, lineno, colno, in_app }) => ({
            filename,
            function: fn,
            lineno,
            colno,
            in_app,
          })),
        },
      })),
    },
  };
}

// sentry/sentry.module.ts
import { SentryModule as SentrySdkModule } from '@sentry/nestjs/setup';

@Module({ imports: [SentrySdkModule.forRoot()] })
export class SentryModule implements OnApplicationShutdown {
  // Nest ends the process with the signal after its hooks, so the SDK's own flush on beforeExit never runs
  async onApplicationShutdown(): Promise<void> {
    await SentryService.close();
  }
}

// api/filters/api-exception.filter.ts
if (statusCode >= 500) {
  this.logger.error({ err: exception }, requestLine);
  SentryService.reportServerError(exception);
}

// logger/utils/build-pino-params.ts
pinoHttp: { hooks: { streamWrite: (line: string) => SentryService.redactSecrets(line) } },
// logger/logger.module.ts
ClsModule.forRoot({ middleware: { mount: true, generateId: true, setup: (cls) => SentryService.setRequestId(cls.getId()) } }),

// test/e2e/api/api-exception.filter.spec.ts
const reportServerError = jest.spyOn(SentryService, 'reportServerError');

it('reports a 5xx and not a 4xx', async () => {
  await request(app.getHttpServer()).get('/probe/fail');
  await request(app.getHttpServer()).get('/probe/missing');

  expect(reportServerError).toHaveBeenCalledTimes(1);
});
```

Only tests set a `transport`. `reportError()` is for a failure outside a request that its code caught: a scheduled job, a listener, or work left to a person. `secretValues()` and `rebuildEvent()` are pure functions. `SentryModule` is imported by `AppModule`, and the SDK's module names each event after the route pattern. The filter's lines are its 5xx branch (see `error-use-exception-filters`); pino's write hook puts the redactor on every log line, and the request context's `setup` hook puts the request id on every event (see `devops-use-logging`). The filter's spec sets no DSN, so nothing is sent, and a spy watches what the filter reports.

### With Prisma

- A busy database (no connection within `maxWait`, a lock wait past `lock_timeout`) answers 503 without an event: the one 5xx the filter does not report, since it is load, not a bug (see `error-use-exception-filters`).
- A `PrismaClientValidationError`'s message prints every argument of the failed call, values included (see `devops-use-logging`). `rebuildEvent()` keeps its last paragraph, the argument and the reason, matched by the exception's `type`, which also covers it as a linked cause. In the mapping of each of `exception.values`:

```typescript
// sentry/utils/rebuild-event.ts
value:
  exception.value &&
  redact(exception.type === 'PrismaClientValidationError' ? (exception.value.trim().split('\n\n').pop() ?? '') : exception.value),
```

Reference: [Sentry for Nest.js](https://docs.sentry.io/platforms/javascript/guides/nestjs/) · [Installation methods (`--import`)](https://docs.sentry.io/platforms/javascript/guides/node/install/) · [Options](https://docs.sentry.io/platforms/javascript/guides/nestjs/configuration/options/) · [Scrubbing sensitive data](https://docs.sentry.io/platforms/javascript/guides/nestjs/data-management/sensitive-data/) · [pino `hooks.streamWrite`](https://github.com/pinojs/pino/blob/main/docs/api.md#streamwrite)
