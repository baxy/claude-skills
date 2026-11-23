---
title: Keep HTTP Code Platform-Agnostic
impact: MEDIUM-HIGH
impactDescription: The same controllers, guards, filters and tests run on Express and Fastify
tags: api, platform, express, fastify, http-adapter, cors
---

## Keep HTTP Code Platform-Agnostic

NestJS runs on Express (`@nestjs/platform-express`), Fastify (`@nestjs/platform-fastify`) or, from 12.3, its own adapter on `node:http` (`@nestjs/platform-node`). A new project uses Fastify unless the user or the spec asks for another adapter (see `references/project-setup.md`, which says why not the Node adapter); an existing project keeps the adapter in its `package.json`. The adapter is created in one function, `ApiModule.createFastifyAdapter()` in `src/api/api.module.ts`, which `main.ts` and the test app helper both call (see `test-use-testing-module`), so its options are the same in tests. Keep every controller, guard, interceptor, filter, decorator and test free of adapter-specific APIs. Code written against Express APIs compiles on Fastify but fails at runtime, and the reverse is also true.

- **No platform response API in handlers.** Do not use `@Res()` with `res.status().json()` (Express) or `reply.code().send()` (Fastify). Return values; use `@HttpCode()`/`@Header()` for static status and headers, and `StreamableFile` for file content. A status that depends on the call (a creating `PUT`: 201 when it created, 200 when it replaced) is set through the adapter, with the response injected in passthrough mode, and the handler still returns its value, so the interceptors and the serializer run as on every route: `@Res({ passthrough: true }) response: unknown` and `this.httpAdapterHost.httpAdapter.status(response, created ? HttpStatus.CREATED : HttpStatus.OK)`. Nest sets the route's static status before the handler runs and sets none when it replies, on both platforms.
- **`HttpAdapterHost` in filters and interceptors.** `httpAdapter.reply()`, `httpAdapter.setHeader()`, `httpAdapter.getRequestUrl()` and `httpAdapter.getRequestMethod()` work on both (see `error-use-exception-filters`).
- **Controller paths use plain segments and `:param`s.** Express 5 reads wildcards and optional parts with `path-to-regexp` 8 (`{*path}`, `{.:ext}`), Fastify with its own router (`*`, `:param?`), and Nest rewrites the old forms only on Express (see `references/project-setup.md`). Check a param's shape in its DTO, not with a pattern in the path (see `api-use-pipes`).
- **Type requests with a neutral shape.** Use a small `ApiRequest` type with what both platforms provide (`method`, `url`, `headers`, `ip`, `query`, plus your `user`), never `Request` from `express` or `FastifyRequest`.
- **Middleware works on raw Node objects.** On Fastify, Nest middleware receives Node's `IncomingMessage`/`ServerResponse`; Express's objects extend them. Write middleware against Node's types (`res.statusCode`, `res.end()`), or prefer guards and interceptors.
- **Plugins differ per platform.** Register them in `ApiModule.setup()`, which `main.ts` and `createTestApp()` both call, with the package for the adapter in use. What the Fastify adapter takes as an option (`trustProxy`, `multipart`) goes where the adapter is built, `ApiModule.createFastifyAdapter()`, so the test apps get it too:

  | Concern | Express | Fastify |
  |---|---|---|
  | Security headers | `helmet` via `app.use()` | `@fastify/helmet` via `app.register()` |
  | Compression | `compression` via `app.use()` | `@fastify/compress` via `app.register()` |
  | Cookies | `cookie-parser` | `@fastify/cookie` |
  | Multipart uploads | `nestjs-form-data` (see `api-file-uploads`) | `nestjs-form-data`, with `multipart: true` on the `FastifyAdapter`, which registers `@fastify/multipart` |
  | Trust the proxy (real client IP behind a load balancer) | `app.set('trust proxy', appSettings.trustedProxies)` | `new FastifyAdapter({ trustProxy: trustedProxies })` |
  | The Swagger UI (`SwaggerModule.setup()`) | served as is | needs `@fastify/static`; without it the app exits right after starting |

- **Trust only your own proxies.** Behind a load balancer, the connection comes from the load balancer, and the client's address is the last entry the load balancer appends to `X-Forwarded-For`. A client can also send that header itself, and the load balancer keeps what it sent. So trust exactly the load balancer's addresses: `APP_TRUSTED_PROXIES`, a comma-separated list of addresses and CIDRs (see `devops-use-config-module`), set in every deployed environment to the range the load balancer connects from, and every layer in front of it (a CDN). The adapter then reads the header from the right and stops at the first address it does not trust, which is the client. Never `trustProxy: true`, which takes the leftmost entry, the one the client wrote, so a client chooses its own address and gets past every per-address rate limit. Never a hop count either: Fastify 5 treats a number as trusting nothing, and on Express it trusts whatever connects directly. Empty, as in a local run, trusts nothing and uses the connection's address.
- **On the Node adapter**, a project keeps what this rule says for every adapter: no platform response API in handlers, the `HttpAdapterHost` methods, middleware written against Node's objects, and the adapter built in one place that `main.ts` and the test app share. Express middleware packages (`helmet`, `compression`, `cookie-parser`) run through `app.use()`, as in the Express column above. What differs from Fastify:
  - Its keep-alive timeout is 5 seconds (Fastify's is 72), shorter than most load balancers' idle timeout. Set `app.getHttpServer().keepAliveTimeout` above the load balancer's: a connection the server closes while the load balancer still reuses it fails that request with a 502.
  - Its body parsers stop at 100 KB (Fastify's limit is 1 MiB), so a route that takes more sets its own limit.
  - `trustProxy` takes the proxies' addresses (the `APP_TRUSTED_PROXIES` list, above), never a hop count.
  - It sends no ETag and no 304 for a handler's response.
  - Some libraries accept only Express or Fastify (`@nestjs/mercurius`, `@graphql-hive/nestjs`): check one before adding it.
- **List CORS methods explicitly.** `app.enableCors()` works on both, but the defaults differ: `@fastify/cors` allows only `GET,HEAD,POST`, so a `PUT`, `PATCH` or `DELETE` endpoint fails the browser preflight. Listing `methods` makes both behave the same. Verify a new method with an `OPTIONS` request carrying `Access-Control-Request-Method` and the test app's origin (`TEST_CORS_ORIGIN`, see `test-use-testing-module`), in the api module's e2e spec, which also checks the security headers: `ApiModule.setup()` registers both for the test apps as for `main.ts`.
- **Listen on all interfaces**: `app.listen(port, host)` with `app.host` defaulting to `'0.0.0.0'` (see `devops-use-config-module`). Fastify defaults to `127.0.0.1`, which is unreachable from outside a container; on Express it is harmless.

**Incorrect (platform APIs in handlers, interceptors and decorators):**

```typescript
import { Request, Response } from 'express';

@Controller('users')
export class UserController {
  @Get(':id')
  async findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto, @Res() res: Response) {
    res.status(200).json(await this.userService.findById(authUser, params.id)); // Express-only, breaks on Fastify
  }
}

@Injectable()
export class ApiNoStoreInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    context.switchToHttp().getResponse<FastifyReply>().header('Cache-Control', 'no-store'); // Fastify-only
    return next.handle();
  }
}

export const AuthUser = createParamDecorator((data: unknown, ctx: ExecutionContext) => {
  return ctx.switchToHttp().getRequest<Request>().user;   // Express type in shared code
});
```

**Correct (adapter and plugins in the api module, neutral code everywhere else):**

```typescript
// api/types/api-request.type.ts
import { IncomingHttpHeaders } from 'node:http';

export type ApiRequest = {
  method: string;
  url: string;
  headers: IncomingHttpHeaders;
  ip?: string;
  query?: unknown;
  user?: JwtUser;
};

// api/api.module.ts
export class ApiModule {
  static createFastifyAdapter(trustedProxies: string[] = []): FastifyAdapter {
    return new FastifyAdapter({
      trustProxy: trustedProxies.length > 0 ? trustedProxies : false,
      genReqId: (req: IncomingMessage) => resolveRequestId(req.headers['x-request-id']),
    });
  }
}

// main.ts
async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(AppModule, ApiModule.createFastifyAdapter(appConfig().trustedProxies), {
    bufferLogs: true,
  });
  app.useLogger(app.get(Logger));
  const configService = app.get(ConfigService);

  await ApiModule.setup(app, configService.getOrThrow<string[]>('app.corsOrigins'));
  HealthModule.listenForShutdown(app);
  await app.listen(configService.getOrThrow<number>('app.port'), configService.getOrThrow<string>('app.host'));
}

@Controller('users')
export class UserController {
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id);
  }
}

// api/interceptors/api-no-store.interceptor.ts
@Injectable()
export class ApiNoStoreInterceptor implements NestInterceptor {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  intercept(context: ExecutionContext, next: CallHandler) {
    const response = context.switchToHttp().getResponse();
    this.httpAdapterHost.httpAdapter.setHeader(response, 'Cache-Control', 'no-store');
    return next.handle();
  }
}

export const AuthUser = createParamDecorator((data: unknown, ctx: ExecutionContext): JwtUser | undefined => {
  return ctx.switchToHttp().getRequest<ApiRequest>().user;
});
```

`ApiRequest` is what code may rely on, on any adapter; `AuthJwtStrategy` sets its `user` (see `security-auth-jwt`). `createFastifyAdapter()` creates the adapter the same way for `main.ts` and `createTestApp()`, and `main.ts` passes it `appConfig().trustedProxies`, since the app's `ConfigService` does not exist before the app does. `ApiModule.forRoot()` and `ApiModule.setup()`, which registers helmet, CORS, the pipes, the filter and the interceptors, are in `test-use-testing-module`; `genReqId` and the nestjs-pino logger are in `devops-use-logging`, and `HealthModule.listenForShutdown()` in `devops-graceful-shutdown`. On Express, `main.ts` calls `NestFactory.create(AppModule, { bufferLogs: true })`, and `setup()` takes an `INestApplication` and registers `app.use(helmet())` instead. Below `main.ts`, handlers return values and send files through `StreamableFile`, interceptors and filters go through `HttpAdapterHost`, and decorators and guards read the neutral request type (see `api-custom-param-decorators`).

```bash
# Verify the preflight after adding a PATCH endpoint
curl -i -X OPTIONS https://api.example.com/api/users/00000000-0000-0000-0000-000000000000 \
  -H 'Origin: https://app.example.com' \
  -H 'Access-Control-Request-Method: PATCH'
# access-control-allow-methods must include PATCH
```

Reference: [NestJS HTTP adapter](https://docs.nestjs.com/faq/http-adapter) · [NestJS Fastify](https://docs.nestjs.com/techniques/performance) · [@fastify/cors](https://github.com/fastify/fastify-cors)
