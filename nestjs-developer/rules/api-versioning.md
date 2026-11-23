---
title: Version Breaking Changes with Nest's URI Versioning
impact: MEDIUM
impactDescription: Existing clients keep working while a new contract ships next to the old one
tags: api, versioning, breaking-changes, compatibility
---

## Version Breaking Changes with Nest's URI Versioning

Add a version only for a **breaking** change (a removed or renamed field, a changed type or meaning, such as a 200 from a pay route that meant "paid" and may now mean "waiting for the customer to authenticate"). Additive changes (a new optional field, a new endpoint) ship unversioned.

Whether the API is versioned from its first route is the spec's decision (`docs/specs/`, see `docs-specs-plans-and-notes`). When the spec does not say, start unversioned: versioning is turned on at the first breaking change, without moving any existing path.

- Use Nest's built-in versioning with one strategy for the whole API: URI versioning (`/v1/users`), enabled in `ApiModule.setup()`, so e2e tests call the same paths as clients (see `test-use-testing-module`). Do not hand-write `v1/` into controller paths, and do not mix strategies.
- **Versioned from the first route:** `defaultVersion: '1'`, so every route is `/v1/...`.
- **Turned on later:** `defaultVersion: [VERSION_NEUTRAL, '1']`. Every existing route then answers at its old path (`/users`) and at `/v1/users`, so turning it on breaks no client. With `defaultVersion: '1'` alone, `/users` would return 404 from that release on.
- **When a new version ships, it joins `defaultVersion`** (`['1', '2']`, or `[VERSION_NEUTRAL, '1', '2']` for an API that started unversioned), so every route answers under `/v2` without listing versions, controllers added later included. Only the routes that changed are pinned: the new version's controller declares `version: '2'`, and each changed route's old handler keeps the versions it still serves (`@Version('1')`, or `@Version([VERSION_NEUTRAL, '1'])`). A `@Version` on a handler or a controller replaces the default instead of adding to it: `@Version(['1', '2'])` on an unchanged route of an API that started unversioned drops its old path, and a controller left out of the new default answers 404 under `/v2`.
- A new version gets its own controller next to the old one (`UserV2Controller` in `controllers/user-v2.controller.ts`), holding only the routes that changed, and its own response DTOs (`user-v2-response.dto.ts`). Services stay version-agnostic: versions differ in the controller and the response DTO, never in an `if (version === ...)` in a service. A paginated route whose versions return different response DTOs is the one place a controller converts: the service returns what every version's response is built from (the entities, or a type that adds computed fields such as ratings), and each version's controller maps `result` to its own DTO with `plainToInstance` (see `api-use-dto-serialization`).
- Routes that are not part of the client contract are `VERSION_NEUTRAL`: health probes (see `micro-use-health-checks`) and webhooks keep one stable path.
- Mark a version as deprecated once its removal date is set, with three headers. `Deprecation` takes RFC 9745's date form, `@` and the seconds since the epoch (`@1795132800`), never `true`, which a client reading the header by that RFC drops. `Sunset` (RFC 8594) is the announced removal, as an HTTP date. A `Link` points at a real URI, `rel="deprecation"` for the migration notes; a target such as `</v2/users/{id}>` is a template, not a URI, and no client can follow it. Until the date is set, the older version is simply supported: shipping v2 does not deprecate v1 by itself.
- **`@Header()` sets headers on the route's own answers only.** Nest applies it after the guards have run, so a request a guard refuses (401, 403, 429) or the body parser refuses (400) gets none. A deprecation must reach every answer of the deprecated version, so its headers are set before the guards, by a middleware of the api module for that version's paths: `/v1/...`, and for an API that started unversioned its unversioned paths too, the version-neutral routes left out. Nest middleware gets Node's request and response on both platforms (see `api-platform-agnostic`), where `request.url` is relative to where the middleware is mounted and `originalUrl` holds the whole path, the global prefix included (`/api/v1/users`), so the middleware matches the version after the prefix. The operations' docs also get `deprecated: true`.
- A cached response changes shape with its version: the cache key names the shape (see `perf-use-caching`).

**Incorrect (breaking change in place, versions in paths and services):**

```typescript
@Controller('users')
export class UserController {
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id); // `name` was split into firstName/lastName: old clients break
  }
}

@Controller('v2/users') // Hand-written version segment
export class UsersV2Controller {
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto, @Headers('X-API-Version') version: string) {
    return this.userService.findById(authUser, params.id, version); // Service branches on the API version
  }
}
```

**Correct (one strategy, a controller and response DTO per version):**

```typescript
// src/api/api.module.ts
app.setGlobalPrefix(API_GLOBAL_PREFIX);
app.enableVersioning({ type: VersioningType.URI, defaultVersion: ['1', '2'] });

// user/controllers/user.controller.ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Version('1')
  @Get(':id')
  @SerializeOptions({ type: UserV1ResponseDto, excludeExtraneousValues: true })
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id);
  }

  @Roles(Role.Admin)
  @HttpCode(HttpStatus.NO_CONTENT)
  @Delete(':id')
  delete(@Param() params: UserIdParamDto): Promise<void> {
    return this.userService.delete(params.id);
  }
}

// user/controllers/user-v2.controller.ts
@Controller({ path: 'users', version: '2' })
export class UserV2Controller {
  constructor(private readonly userService: UserService) {}

  @Get(':id')
  @SerializeOptions({ type: UserV2ResponseDto, excludeExtraneousValues: true })
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id);
  }
}

// user/dtos/user-v1-response.dto.ts
export class UserV1ResponseDto {
  @ApiProperty({ description: 'UUID of the user', example: '00000000-0000-0000-0000-000000000000' })
  @Expose()
  id: string;

  @ApiProperty({ description: 'Full name', example: 'Jane Doe' })
  @Expose()
  @Transform(({ obj }) => `${obj.firstName} ${obj.lastName}`)
  name: string;
}

// api/constants/api-global-prefix.constant.ts
export const API_GLOBAL_PREFIX = 'api';

// api/constants/api-v1-deprecation.constant.ts
export const API_V1_DEPRECATION_HEADERS: Record<string, string> = {
  // RFC 9745: since when, in seconds since the epoch (2026-11-20)
  Deprecation: '@1795132800',
  // RFC 8594: the announced removal, as an HTTP date
  Sunset: 'Wed, 30 Jun 2027 00:00:00 GMT',
  Link: '<https://shop.example.com/docs/api-v2-migration>; rel="deprecation"; type="text/html"',
};

// api/middlewares/api-v1-deprecation.middleware.ts
@Injectable()
export class ApiV1DeprecationMiddleware implements NestMiddleware {
  use(request: IncomingMessage & { originalUrl?: string }, response: ServerResponse, next: () => void): void {
    // url is relative to where the middleware is mounted; originalUrl is the whole path, the global prefix included
    if ((request.originalUrl ?? request.url ?? '').startsWith(`/${API_GLOBAL_PREFIX}/v1/`)) {
      for (const [name, value] of Object.entries(API_V1_DEPRECATION_HEADERS)) {
        response.setHeader(name, value);
      }
    }
    next();
  }
}

// api/api.module.ts
export class ApiModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer.apply(ApiV1DeprecationMiddleware).forRoutes({ path: '*path', method: RequestMethod.ALL });
  }
}

@ApiOperation({ summary: 'Get a user', description: 'Deprecated: use version 2.', deprecated: true })
```

The example is an API versioned from its first route, with v2 shipped; an API that started unversioned uses `[VERSION_NEUTRAL, '1', '2']` instead, and pins its changed handlers to `[VERSION_NEUTRAL, '1']`. `UserController` has no version of its own, so its routes serve every version but the one a handler pins: `findById()` changed in v2, so it keeps `/v1` while `UserV2Controller` serves `/v2`, and the unchanged `delete()` answers at `/v1` and `/v2` through the default. `UserV2Controller` holds version 2 of the changed routes only, on the same service, and `UserV1ResponseDto` is the old contract, derived from the same entity. The deprecation headers exist once v1's removal date is set. The global prefix is one constant, which `setup()` passes to `app.setGlobalPrefix()` and the middleware matches after. The middleware runs before the guards and the body parser, so a 401, 403, 429 or 400 carries them too, and it is applied to every route, a path no route matches included. Each v1 operation in the docs gets `deprecated: true`.

Reference: [NestJS Versioning](https://docs.nestjs.com/techniques/versioning) · [RFC 9745 Deprecation header](https://www.rfc-editor.org/rfc/rfc9745) · [RFC 8594 Sunset header](https://www.rfc-editor.org/rfc/rfc8594)
