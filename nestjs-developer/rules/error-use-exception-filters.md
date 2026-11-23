---
title: Format Every Error in One Global Exception Filter
impact: HIGH
impactDescription: Consistent error responses, with internals logged and never sent
tags: error-handling, exception-filters, consistency, platform-agnostic
---

## Format Every Error in One Global Exception Filter

Never catch exceptions and format error responses in controllers. Services throw typed exceptions (see `error-throw-http-exceptions`), and **one** global filter, `ApiExceptionFilter`, turns every error into the error half of the response envelope (see `api-use-interceptors`):

```json
{ "success": false, "statusCode": 404, "error": "Not Found", "message": "User #00000000-0000-0000-0000-000000000000 not found", "errorCode": "NOT_FOUND" }
{ "success": false, "statusCode": 409, "error": "Conflict", "message": "Email already registered", "errorCode": "EMAIL_TAKEN" }
```

- It is a provider of the api module, so Nest builds it with its dependencies, and `ApiModule.setup()` registers it (`app.useGlobalFilters(app.get(ApiExceptionFilter))`, see `test-use-testing-module`), so e2e tests get the same error shape. Do not add a second filter per error type; extend this one. Three filters are scoped to their controller or gateway instead, each for a case this one does not serve: the health controller's `HealthUnavailableFilter`, which answers the probes' 503s (see `micro-use-health-checks`), a gateway's `ApiWsExceptionFilter`, since global filters do not run on gateways (see `api-real-time-updates`), and a message controller's own filter, since the HTTP app's global filter does not reach message handlers (see `micro-use-patterns`).
- Write it with `HttpAdapterHost` (`httpAdapter.reply(...)`) instead of Express's `response.status().json()`, so it works on Express and Fastify (see `api-platform-agnostic`).
- Read the message from `exception.getResponse()`, not `exception.message`: for `ValidationPipe` errors it holds the per-field messages, while `exception.message` is just "Bad Request Exception". Keep `ValidationPipe`'s default `errorFormat: 'list'` (NestJS 12): `'grouped'` turns `message` into an object keyed by field, which the envelope's `string | string[]` does not allow.
- Translate known non-HTTP errors here: a unique violation (`QueryFailedError` with PostgreSQL code `23505`) becomes 409, a transaction conflict no helper retried (`40001`, `40P01`) becomes the 409 the retry helper answers once its attempts run out (`TRANSACTION_CONFLICT`, see `db-use-transactions`), text PostgreSQL cannot store (a NUL character: `22021` in a text column, `22P05` as `\u0000` in `jsonb`) becomes 400, since validation lets it through and it is the client's input, a domain error class shared with non-HTTP consumers becomes its status. Match on error codes, never on message text. A service that wants a more specific 409 ("Email already registered") catches the violation of its own index by name, with the same `isUniqueViolation()` helper, instead of looking the value up first: two requests could both pass the lookup.
- **Every error carries an `errorCode`.** It is a stable identifier clients branch on instead of the message text, which may be reworded, so a client can always read the same field.
  - The exception's own code comes first (NestJS 12, `new ConflictException('Email already registered', { errorCode: 'EMAIL_TAKEN' })`). Read it from `exception.getResponse()`, where both versions put it: NestJS 12 adds the option's value to the response body, and on NestJS 11, which has no such option, services pass it in the response object (`new ConflictException({ message: 'Email already registered', errorCode: 'EMAIL_TAKEN' })`). Without the copy, the global filter drops it.
  - Otherwise the status gives it: the status's name in `HttpStatus` (`BAD_REQUEST`, `NOT_FOUND`, `TOO_MANY_REQUESTS`, `INTERNAL_SERVER_ERROR`), and `ERROR` for a status `HttpStatus` does not name.
- **A known retry delay becomes a `Retry-After` header.** An exception that carries `retryAfterSeconds` in its response object (see `error-throw-http-exceptions`) gets the header, in whole seconds, through `httpAdapter.setHeader()`; the field stays out of the body, and a value that is not a positive integer sends no header.
- Unknown errors become a generic 500. Log the stack and the `cause`; never send either to the client. Log the error as pino's `err` field, `this.logger.error({ err: exception }, requestLine)`: pino's error serializer writes the stack with every cause appended, and keeps a thrown value that is not an Error as it is. Passed as an argument after the message, an error object or a thrown value never reaches the line, and `error.stack` alone loses the causes. A `QueryFailedError` carries the query's bound values, so the pino config redacts them (see `devops-use-logging`).
- Every 5xx is also reported to Sentry here, with `SentryService.reportServerError(exception)`, and nothing below 500 (see `error-report-to-sentry`).
- Every failed request gets one log line here (a failing probe, which `HealthUnavailableFilter` answers, gets none from this filter): a 5xx at error level with the stack, any other status at warn level, each with the method, the path without its query string and the status, and the caller's id and address as fields (see `api-use-interceptors`). Guards reject a request before any interceptor runs, so this filter is the only place that sees a 401, 403 or 429.
- When translating a lower-level error yourself, keep it as the `cause` so logs show the root cause.

**Incorrect (manual error handling in controllers, Express-only filters):**

```typescript
@Controller('users')
export class UserController {
  @Get(':id')
  async findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto, @Res() res: Response) {
    try {
      const user = await this.userService.findById(authUser, params.id);
      return res.json(user);
    } catch (error) {
      this.logger.error(error);
      return res.status(500).json({ statusCode: 500, message: 'Internal server error' });
    }
  }
}

// Express-only and loses validation details
@Catch()
export class ApiExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse<Response>();
    const status = exception instanceof HttpException ? exception.getStatus() : 500;
    const message = exception instanceof HttpException ? exception.message : 'Internal server error';
    response.status(status).json({ statusCode: status, message }); // Breaks on Fastify
  }
}
```

**Correct (one platform-agnostic global filter with the envelope shape):**

```typescript
// payment/services/payment.service.ts
@Injectable()
export class PaymentService {
  constructor(@Inject(PAYMENT_ADAPTER) private readonly paymentAdapter: PaymentAdapter) {}

  async charge(order: Order): Promise<PaymentResult> {
    try {
      return await this.paymentAdapter.charge(order.totalCents, order.currency);
    } catch (error) {
      if (error instanceof PaymentDeclinedException) {
        throw new BadRequestException('The payment was declined', { cause: error, errorCode: 'PAYMENT_DECLINED' });
      }
      throw error;
    }
  }
}

// api/types/api-error-response.type.ts
export type ApiErrorResponse = {
  success: false;
  statusCode: number;
  error: string;
  message: string | string[];
  errorCode: string;
};

// database/utils/unique-violation.ts
const PG_UNIQUE_VIOLATION = '23505';

export function isUniqueViolation(error: unknown, constraint?: string): boolean {
  if (!(error instanceof QueryFailedError)) {
    return false;
  }
  // TypeORM types driverError as Error; PostgreSQL's error carries the SQLSTATE in code and the index in constraint
  const driverError = error.driverError as Error & { code?: string; constraint?: string };
  return driverError.code === PG_UNIQUE_VIOLATION && (constraint === undefined || driverError.constraint === constraint);
}

// api/filters/api-exception.filter.ts
@Catch()
export class ApiExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(ApiExceptionFilter.name);

  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(caught: unknown, host: ArgumentsHost): void {
    // A conflict no transaction retried answers as one that ran out of retries: the client may send it again
    const exception = isTransactionConflict(caught) ? transactionConflictException() : caught;
    const { httpAdapter } = this.httpAdapterHost;
    const ctx = host.switchToHttp();
    const request = ctx.getRequest<ApiRequest>();
    const statusCode = this._status(exception);
    const requestLine = `${httpAdapter.getRequestMethod(request)} ${apiRequestPath(httpAdapter.getRequestUrl(request))} ${statusCode}`;
    // No user on a 401: the guard refused the token
    const caller = { userId: request.user?.id, ip: request.ip };

    if (statusCode >= 500) {
      this.logger.error({ err: exception, ...caller }, requestLine);
      SentryService.reportServerError(exception);
    } else {
      this.logger.warn(caller, requestLine);
    }

    const body: ApiErrorResponse = {
      success: false,
      statusCode,
      error: this._errorName(statusCode),
      message: this._message(exception, statusCode),
      errorCode: this._errorCode(exception, statusCode),
    };

    const retryAfterSeconds = this._responseObject(exception)?.retryAfterSeconds;
    if (typeof retryAfterSeconds === 'number' && Number.isInteger(retryAfterSeconds) && retryAfterSeconds > 0) {
      httpAdapter.setHeader(ctx.getResponse(), 'Retry-After', String(retryAfterSeconds));
    }

    httpAdapter.reply(ctx.getResponse(), body, statusCode);
  }

  private _status(exception: unknown): number {
    if (exception instanceof HttpException) return exception.getStatus();
    if (exception instanceof EntityNotFoundException) return HttpStatus.NOT_FOUND;
    if (isUniqueViolation(exception)) return HttpStatus.CONFLICT;
    if (isUnstorableText(exception)) return HttpStatus.BAD_REQUEST;
    return HttpStatus.INTERNAL_SERVER_ERROR;
  }

  // getResponse() keeps ValidationPipe's per-field messages; .message would not
  private _message(exception: unknown, statusCode: number): string | string[] {
    if (exception instanceof HttpException) {
      const response = exception.getResponse();
      if (typeof response === 'string') return response;
      return (response as { message?: string | string[] }).message ?? exception.message;
    }
    if (exception instanceof EntityNotFoundException) return exception.message;
    if (statusCode === HttpStatus.CONFLICT) return 'Resource already exists';
    if (statusCode === HttpStatus.BAD_REQUEST) return 'The request contains a character that cannot be stored';
    return 'Internal server error';
  }

  // NestJS 12's errorCode option and a NestJS 11 response object both land in getResponse()
  private _errorCode(exception: unknown, statusCode: number): string {
    const errorCode = this._responseObject(exception)?.errorCode;
    return typeof errorCode === 'string' ? errorCode : (HttpStatus[statusCode] ?? 'ERROR');
  }

  private _responseObject(exception: unknown): Record<string, unknown> | undefined {
    if (!(exception instanceof HttpException)) return undefined;
    const response = exception.getResponse();
    return typeof response === 'object' && response !== null ? (response as Record<string, unknown>) : undefined;
  }

  private _errorName(statusCode: number): string {
    const name = HttpStatus[statusCode];
    return name ? name.toLowerCase().split('_').map((word) => word[0].toUpperCase() + word.slice(1)).join(' ') : 'Error';
  }
}
```

`PaymentService` translates the one error it knows, a declined payment, and keeps the adapter's error as the `cause`, for the logs; any other error goes on unchanged, since the client has already mapped the provider's failures to a 503 or a 500 (see `micro-use-http-clients`). `errorCode` is the exception's own, or the name of its status. The filter and the services that name the clash share `isUniqueViolation()`, in the `database` module's `utils/`, with the rest of what knows TypeORM's errors (see `db-typeorm-use-transactions`). `isUnstorableText()` (`database/utils/unstorable-text.ts`) lives there too, though only the filter calls it: it reads PostgreSQL's codes from TypeORM's error.

### With Prisma

The filter maps Prisma's codes on `Prisma.PrismaClientKnownRequestError` instead of `QueryFailedError`: `P2002`, a unique violation, becomes 409, and `P2025`, a record that `update()`, `delete()` or a `*OrThrow` query did not find, becomes 404. A `P2025` message names the Prisma call and the source file, so the client gets the generic message for the status, as for a 409.

```typescript
const PRISMA_UNIQUE_VIOLATION = 'P2002';
const PRISMA_RECORD_NOT_FOUND = 'P2025';

private _status(exception: unknown): number {
  if (exception instanceof HttpException) return exception.getStatus();
  if (exception instanceof EntityNotFoundException) return HttpStatus.NOT_FOUND;
  if (exception instanceof Prisma.PrismaClientKnownRequestError) {
    if (exception.code === PRISMA_UNIQUE_VIOLATION) return HttpStatus.CONFLICT;
    if (exception.code === PRISMA_RECORD_NOT_FOUND) return HttpStatus.NOT_FOUND;
  }
  return HttpStatus.INTERNAL_SERVER_ERROR;
}

if (statusCode === HttpStatus.NOT_FOUND) return 'Resource not found';
```

The last line goes in `_message()`, after the `EntityNotFoundException` line.

A service that gives one constraint its own `errorCode` catches the `P2002` and reads the constraint's name from `meta.driverAdapterError.cause.constraint.index`: with a driver adapter, Prisma 7 sets no `meta.target`.

**A busy database answers 503, not 500** (see `db-prisma-use-transactions`): a transaction that found no free connection within `maxWait` (`P2028` without `meta.timeout`), and a lock wait past `lock_timeout` (SQLSTATE `55P03`, read with `postgresErrorCode()`). Nothing was written and the same request may succeed a moment later, so the answer carries `Retry-After` (1 second here, an example), and the filter logs it at warn without a Sentry event: it is load, not a bug, and while the pool stays busy every request would otherwise add an event. A `P2028` with `meta.timeout` and `meta.timeTaken` is a transaction that outlived its timeout, which with lock waits bounded means the work is too slow: it stays a 500 reported to Sentry.

```typescript
// prisma/utils/database-busy.ts
const PRISMA_TRANSACTION_API_ERROR = 'P2028';
const PG_LOCK_NOT_AVAILABLE = '55P03';

export function isDatabaseBusy(error: unknown): boolean {
  if (postgresErrorCode(error) === PG_LOCK_NOT_AVAILABLE) {
    return true;
  }
  // P2028 also reports an expired transaction, which carries meta.timeout
  return error instanceof Prisma.PrismaClientKnownRequestError && error.code === PRISMA_TRANSACTION_API_ERROR && error.meta?.timeout === undefined;
}

export function databaseBusyException(): ServiceUnavailableException {
  return new ServiceUnavailableException({ message: 'The database is busy; try again', errorCode: 'DATABASE_BUSY', retryAfterSeconds: 1 });
}

// api/filters/api-exception.filter.ts
catch(caught: unknown, host: ArgumentsHost): void {
  const busy = isDatabaseBusy(caught);
  const exception = isTransactionConflict(caught) ? transactionConflictException() : busy ? databaseBusyException() : caught;
  // ...
  if (statusCode >= 500 && !busy) {
    this.logger.error({ err: exception, ...caller }, requestLine);
    SentryService.reportServerError(exception);
  } else {
    this.logger.warn(caller, requestLine);
  }
  // ...
}
```

Reference: [NestJS Exception Filters](https://docs.nestjs.com/exception-filters) · [Catch everything](https://docs.nestjs.com/exception-filters#catch-everything) · [PostgreSQL error codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
