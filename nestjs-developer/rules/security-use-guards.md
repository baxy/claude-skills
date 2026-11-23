---
title: Use Global Guards for Authentication and Authorization
impact: HIGH
impactDescription: Every route is protected by default, and access rules are declared, not hand-coded
tags: security, guards, authentication, authorization
---

## Use Global Guards for Authentication and Authorization

Register two guards globally, in this order, as `APP_GUARD` providers in `AppModule`:

1. `AuthJwtGuard` (passport `AuthGuard('jwt')`, see `security-auth-jwt`): every route requires a valid token unless it is marked `@Public()`.
2. `AuthRolesGuard`: a route or controller marked `@Roles(...)` requires one of those roles.

Routes are then protected by default; forgetting a decorator fails closed. Never check `request.user` or roles inside a handler, and never repeat `@UseGuards(AuthJwtGuard, AuthRolesGuard)` on routes (the global guards already run; a second registration runs them twice).

- `@Public()` and `@Roles()` live in `api/decorators/`, the guards in `auth/guards/`. Metadata keys are exported constants in `api/constants/` (`API_IS_PUBLIC_KEY`, `API_ROLES_KEY`), never string literals repeated in two files.
- `AuthRolesGuard` must handle a public route and a missing user without throwing a `TypeError`.
- **In an app with a gateway or message handlers, the global guards skip every context that is not HTTP.** `APP_GUARD`s also run on `@SubscribeMessage`, `@MessagePattern` and `@EventPattern` handlers, which have no HTTP request to read, so both guards return `true` first thing when `context.getType() !== 'http'`, and so does the throttler guard, in `shouldSkip()` (see `security-rate-limiting`). The skip is safe only because those handlers check their callers themselves: a gateway authenticates its client when it connects and checks each message's sender in a guard of its own (see `api-real-time-updates`), and a message controller has a guard that checks every message's credential (see `micro-use-patterns`).
- Ownership ("is this the caller's order?") is a domain rule: the service scopes its query by `authUser.id` and answers 404 (see `arch-service-conventions`). Guards decide who may call a route, not which rows they see.

**Incorrect (manual checks in handlers):**

```typescript
@Controller('users')
export class UserController {
  @Delete(':id')
  async delete(@Req() req: ApiRequest, @Param() params: UserIdParamDto): Promise<void> {
    if (!req.user) {
      throw new UnauthorizedException();
    }
    if (!req.user.roles.includes(Role.Admin)) {
      throw new ForbiddenException(); // Repeated in every handler, forgotten in the next one
    }
    return this.userService.delete(params.id);
  }
}
```

**Correct (global guards, declarative decorators):**

```typescript
// api/constants/api-route-access.constant.ts
export const API_IS_PUBLIC_KEY = 'isPublic';
export const API_ROLES_KEY = 'roles';

// api/decorators/public.decorator.ts
export const Public = () => SetMetadata(API_IS_PUBLIC_KEY, true);

// api/decorators/roles.decorator.ts
export const Roles = (...roles: Role[]) => SetMetadata(API_ROLES_KEY, roles);

// auth/guards/auth-jwt.guard.ts
@Injectable()
export class AuthJwtGuard extends AuthGuard('jwt') {
  constructor(private readonly reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // A gateway or a message controller checks its callers itself
    if (context.getType() !== 'http') {
      return true;
    }
    const isPublic = this.reflector.getAllAndOverride<boolean>(API_IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    return isPublic ? true : super.canActivate(context);
  }
}

// auth/guards/auth-roles.guard.ts
@Injectable()
export class AuthRolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    if (context.getType() !== 'http') {
      return true;
    }
    const requiredRoles = this.reflector.getAllAndOverride<Role[] | undefined>(API_ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles?.length) {
      return true;
    }

    const user = context.switchToHttp().getRequest<ApiRequest>().user;
    return !!user && requiredRoles.some((role) => user.roles.includes(role));
  }
}

// app.module.ts
@Module({
  providers: [
    // In this order: the roles guard reads the user the JWT guard sets
    { provide: APP_GUARD, useClass: AuthJwtGuard },
    { provide: APP_GUARD, useClass: AuthRolesGuard },
  ],
})
export class AppModule {}

// user/controllers/user.controller.ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get(':id')
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
```

`GET /users/:id` is open to any authenticated caller, and its service answers 404 for an account that is not the caller's unless the caller is an admin (ownership, above); `DELETE /users/:id` is for admins only.

Reference: [NestJS Guards](https://docs.nestjs.com/guards) · [Enable authentication globally](https://docs.nestjs.com/security/authentication#enable-authentication-globally)
