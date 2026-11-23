---
title: Shape Responses and Cross-Cutting Concerns with Interceptors
impact: MEDIUM-HIGH
impactDescription: Every success response has one shape, and handlers stay free of logging and wrapping code
tags: api, interceptors, response-envelope, logging, timeout
---

## Shape Responses and Cross-Cutting Concerns with Interceptors

Controllers return the service result as-is. Everything that applies to every route (the response envelope, request logging) lives in an interceptor registered once in `ApiModule.setup()` (see `test-use-testing-module`), so tests run it too. A timeout is the exception: it goes on the routes that need one (below).

### The response envelope

Every successful JSON response has the same shape, produced by `ApiResponseInterceptor`. Errors get the matching shape from the global exception filter (see `error-use-exception-filters`):

```json
{ "success": true,  "statusCode": 200, "data": { "id": "..." } }
{ "success": true,  "statusCode": 200, "data": [ ... ], "meta": { "totalCount": 42, "limit": 20, "offset": 0 } }
{ "success": false, "statusCode": 404, "error": "Not Found", "message": "Order #... not found", "errorCode": "NOT_FOUND" }
```

- A paginated service result (`PaginatedResult<T>`, `{ result, totalCount, limit, offset }`, see `perf-optimize-database`) becomes `data` + `meta`.
- `StreamableFile` (downloads, see `api-controller-conventions`) passes through unwrapped, and so does an empty body (`204`).
- Never build `{ data: ... }` or `{ success: ... }` in a controller or service.

### Where interceptors sit in a request

Nest runs every request in this order: middleware, guards, interceptors (before the handler), pipes, the handler, interceptors (after, innermost first), exception filters. Within each kind, global runs before controller-level, which runs before route-level. Other rules rely on this:

- Guards run before pipes, so a guard sees raw, unvalidated params and body. It decides from the caller and the route metadata, never from the input (see `security-use-guards`).
- An exception thrown by a guard skips every interceptor, and a request to an unknown route never reaches one, so `ApiLoggerInterceptor` never sees a 401, 403, 429 (the throttler is a guard) or a 404 for a missing route. The exception filter is the one place that sees every error, so it writes the log line for every failed request and `ApiLoggerInterceptor` logs successes only (see `error-use-exception-filters`).
- Pipes run after interceptors start, so an interceptor sees the request before `ValidationPipe` has transformed it.

### What a request log line holds

The interceptor's line and the filter's line have the same shape: the method, the path, the status (and the duration for a success) in the message, and the caller's id and the client address as fields (`userId`, `ip`), so the logs record who created an account or deactivated one, and a burst of failed logins can be traced to its address. The path is `request.url` without its query string (`apiRequestPath()`): a query string carries search terms, emails and card numbers into every line, at info level. The address is `request.ip`, which is the client's own only when exactly the proxies in front of the API are trusted (`APP_TRUSTED_PROXIES`, see `api-platform-agnostic`).

### Order of global interceptors

`ApiModule.setup()` lists them outermost first: `ApiLoggerInterceptor`, then `ApiResponseInterceptor`, then `ClassSerializerInterceptor` (see `api-use-dto-serialization`). The serializer, innermost, turns the entity or response DTO into a plain object first; the envelope then wraps that.

### A timeout stops the wait, not the work

`ApiTimeoutInterceptor` answers 408 when the handler has not answered in time, but rxjs's `timeout` only stops listening: the handler's promise, its queries and its calls go on and finish. So its message says the answer was not ready in time, never that the request was stopped, and it goes only on routes that read (a PDF rendered on download), applied per route or controller. On a route that writes, a 408 tells the client its write failed while it goes on to complete, and the client's retry writes it twice.

**Incorrect (logging and wrapping repeated in handlers):**

```typescript
@Controller('users')
export class UserController {
  private readonly logger = new Logger(UserController.name);

  @Get(':id')
  async findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto) {
    const start = Date.now();
    const user = await this.userService.findById(authUser, params.id);
    this.logger.log(`findById completed in ${Date.now() - start}ms`); // Repeated in every handler
    return { data: user, meta: { timestamp: new Date() } };            // Ad-hoc wrapper, different per endpoint
  }
}
```

**Correct (one interceptor per concern, registered in `ApiModule.setup()`):**

```typescript
// api/types/api-response-meta.type.ts
export type ApiResponseMeta = {
  totalCount: number;
  limit?: number;
  offset?: number;
  totalCountCapped?: true;
};

// api/types/api-success-response.type.ts
export type ApiSuccessResponse<T> = {
  success: true;
  statusCode: number;
  data: T;
  meta?: ApiResponseMeta;
};

// api/interceptors/api-response.interceptor.ts
@Injectable()
export class ApiResponseInterceptor implements NestInterceptor {
  constructor(private readonly reflector: Reflector) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (context.getType() !== 'http' || this.reflector.get<boolean>(SSE_METADATA, context.getHandler())) {
      return next.handle();
    }
    return next.handle().pipe(
      map((body: unknown) => {
        if (body instanceof StreamableFile || body === undefined) {
          return body;
        }

        const statusCode: number = context.switchToHttp().getResponse().statusCode;
        if (this._isPaginated(body)) {
          const { totalCount, limit, offset, totalCountCapped } = body;
          return {
            success: true,
            statusCode,
            data: body.result,
            meta: { totalCount, limit, offset, ...(totalCountCapped ? { totalCountCapped } : {}) },
          } satisfies ApiSuccessResponse<unknown[]>;
        }
        return { success: true, statusCode, data: body } satisfies ApiSuccessResponse<unknown>;
      }),
    );
  }

  private _isPaginated(body: unknown): body is PaginatedResult<unknown> {
    const candidate = body as PaginatedResult<unknown> | null;
    return typeof candidate === 'object' && candidate !== null
      && Array.isArray(candidate.result) && typeof candidate.totalCount === 'number';
  }
}

// api/interceptors/api-logger.interceptor.ts
@Injectable()
export class ApiLoggerInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (context.getType() !== 'http') {
      return next.handle();
    }
    const request = context.switchToHttp().getRequest<ApiRequest>();
    const startedAt = Date.now();

    return next.handle().pipe(
      tap(() => {
        const { statusCode } = context.switchToHttp().getResponse();
        this.logger.log(
          { userId: request.user?.id, ip: request.ip },
          `${request.method} ${apiRequestPath(request.url)} ${statusCode} ${Date.now() - startedAt}ms`,
        );
      }),
    );
  }
}

// api/utils/api-request-path.ts
export function apiRequestPath(url: string): string {
  return url.split('?', 1)[0];
}

// api/interceptors/api-timeout.interceptor.ts
@Injectable()
export class ApiTimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeoutMs: number) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      timeout(this.timeoutMs),
      catchError((error: unknown) => {
        throw error instanceof TimeoutError
          ? new RequestTimeoutException('The answer was not ready in time; try again later')
          : error;
      }),
    );
  }
}

// order/controllers/order.controller.ts
@UseInterceptors(new ApiTimeoutInterceptor(10_000))
@Get(':id/download')
download(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto): Promise<StreamableFile> { /* ... */ }
```

`totalCountCapped` marks a public search's total that stopped at its cap (see `perf-optimize-database`). Files and empty `204` responses pass through unwrapped. Both interceptors return `next.handle()` unchanged for a context that is not HTTP, and the envelope also for an `@Sse()` handler, whose events it would break (see `api-real-time-updates`). `ApiLoggerInterceptor` writes one line per successful request, and the exception filter logs the failed ones, guard rejections included; the `requestId` on the lines comes from the request logger (see `devops-use-logging`), never from the message text. Both log the path from `apiRequestPath()`, never the query string. `ApiTimeoutInterceptor` goes only on routes that read, since the handler goes on after the 408: the invoice download is a read, so nothing is left half done when the client stops waiting. Its 10 seconds to render an invoice is an example.

Error translation (database errors to 409, domain errors to 404) belongs in the exception filter, not in an interceptor. Response caching uses the built-in `CacheInterceptor` (see `perf-use-caching`).

Reference: [NestJS Interceptors](https://docs.nestjs.com/interceptors)
