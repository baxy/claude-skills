---
title: Test Guards and Interceptors Through Probe Routes
impact: MEDIUM-HIGH
impactDescription: Catches auth bypasses and broken request handling that service tests never exercise
tags: testing, guards, interceptors, pipes, authorization
---

## Test Guards and Interceptors Through Probe Routes

Guards and interceptors have injected dependencies (`Reflector`, services), so they belong in the e2e tier (see `test-organize-by-tier`). Test them the way they run: register them on a small **probe controller** that exists only in the test, and drive it over HTTP with `supertest` through the shared `createTestApp()` helper (see `test-use-testing-module`). This exercises the real metadata lookup, the real execution order and the real error responses, which hand-built `ExecutionContext` mocks do not.

- Cover the **deny and error paths**, not only the happy path: an authorization test that only proves admins get in never catches a bypass.
- Vary the caller per request: a test authentication guard, registered before the guards under test, picks the user from a test-only header.
- Add one check against a real protected route that it rejects an unauthorized caller, to prove the guard is registered where it should be.
- **Test the real authentication once.** Every other suite replaces `AuthJwtGuard` with the test guard, so a broken `AuthJwtStrategy` (wrong secret, unchecked issuer, accepted expired token) would pass them all. One suite runs the real `AuthJwtGuard` and `AuthJwtStrategy` with a test secret: 401 for a missing, malformed, expired, wrongly signed, wrong-issuer, wrong-audience or incomplete token, 200 for a valid one. Each bad token changes one thing from a valid one: a token signed with another secret but also lacking its audience is refused on the audience, and passes with a broken signature check. A WebSocket gateway, which checks the token itself (see `api-real-time-updates`), gets the same cases.
- Pure pieces stay in the unit tier: DTO validation rules are tested by running `ValidationPipe` on the DTO directly.

**Incorrect (hand-built contexts, allow path only):**

```typescript
describe('AuthRolesGuard', () => {
  it('allows access', () => {
    const guard = new AuthRolesGuard(new Reflector());
    const context = { switchToHttp: () => ({ getRequest: () => ({ user: { roles: ['admin'] } }) }) } as any;
    jest.spyOn(Reflector.prototype, 'getAllAndOverride').mockReturnValue(['admin']);
    // Mocked metadata, mocked context, and nobody checks that non-admins are blocked
    expect(guard.canActivate(context)).toBe(true);
  });
});
```

**Correct (probe controller, identity per request, deny paths covered):**

```typescript
// test/e2e/auth/auth-roles.guard.spec.ts
import request from 'supertest';

const USERS: Record<string, JwtUser> = {
  admin: { id: '00000000-0000-4000-8000-000000000001', email: 'admin@example.com', roles: [Role.Admin], sessionId: '00000000-0000-4000-8000-000000000011' },
  user: { id: '00000000-0000-4000-8000-000000000002', email: 'jane@example.com', roles: [Role.User], sessionId: '00000000-0000-4000-8000-000000000012' },
};

@Controller('probe')
class ProbeController {
  @Get('public')
  @Public()
  open() {
    return { ok: true };
  }

  @Get('admin')
  @Roles(Role.Admin)
  admin() {
    return { ok: true };
  }

  @Get('missing')
  missing() {
    throw new NotFoundException('Probe resource not found');
  }
}

describe('AuthRolesGuard', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ApiModule.forRoot()],
      controllers: [ProbeController],
      providers: [
        { provide: APP_GUARD, useValue: testAuthGuard((req) => USERS[req.headers['x-test-user'] as string]) },
        { provide: APP_GUARD, useClass: AuthRolesGuard },
      ],
    }).compile();

    app = await createTestApp(moduleRef);
  });

  afterAll(() => app.close());

  describe('GET /probe/admin', () => {
    it('allows an admin', async () => {
      const res = await request(app.getHttpServer()).get('/probe/admin').set('x-test-user', 'admin');
      expect(res.status).toBe(200);
    });

    it('rejects a user without the admin role', async () => {
      const res = await request(app.getHttpServer()).get('/probe/admin').set('x-test-user', 'user');
      expect(res.status).toBe(403);
    });
  });

  describe('GET /probe/public', () => {
    it('allows an anonymous caller', async () => {
      const res = await request(app.getHttpServer()).get('/probe/public');
      expect(res.status).toBe(200);
    });
  });
});

// test/e2e/api/api-response.interceptor.spec.ts
describe('ApiResponseInterceptor', () => {
  it('wraps a successful response', async () => {
    const res = await request(app.getHttpServer()).get('/probe/public');
    expect(res.body).toEqual({ success: true, statusCode: 200, data: { ok: true } });
  });

  it('formats an error through the filter', async () => {
    const res = await request(app.getHttpServer()).get('/probe/missing').set('x-test-user', 'admin');
    expect(res.status).toBe(404);
    expect(res.body).toMatchObject({ success: false, statusCode: 404, message: 'Probe resource not found' });
  });
});

// test/e2e/auth/auth-jwt.guard.spec.ts
const AUTH_SETTINGS = { jwt: { secret: 'test-secret', issuer: 'test-issuer', audience: 'test-audience', accessTokenTtl: '5m' } };
const jwtService = new JwtService();
const sign = (claims: object, options: JwtSignOptions = {}): string =>
  jwtService.sign(claims, {
    secret: AUTH_SETTINGS.jwt.secret,
    issuer: AUTH_SETTINGS.jwt.issuer,
    audience: AUTH_SETTINGS.jwt.audience,
    expiresIn: '5m',
    ...options,
  });
const VALID_CLAIMS = { sub: USERS.user.id, email: USERS.user.email, roles: USERS.user.roles, sid: USERS.user.sessionId };

describe('AuthJwtGuard', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ApiModule.forRoot(), PassportModule],
      controllers: [ProbeController],
      providers: [
        AuthJwtStrategy,
        { provide: authConfig.KEY, useValue: AUTH_SETTINGS },
        { provide: APP_GUARD, useClass: AuthJwtGuard },
      ],
    }).compile();

    app = await createTestApp(moduleRef);
  });

  afterAll(() => app.close());

  describe('GET /probe/admin', () => {
    it('accepts a valid token', async () => {
      const res = await request(app.getHttpServer()).get('/probe/admin').set('Authorization', `Bearer ${sign(VALID_CLAIMS)}`);
      expect(res.status).toBe(200);
    });

    it.each([
      ['no token', undefined],
      ['a malformed token', 'not-a-jwt'],
      ['an expired token', sign(VALID_CLAIMS, { expiresIn: -60 })],
      ['a token signed with another secret', sign(VALID_CLAIMS, { secret: 'another-secret' })],
      ['a token from another issuer', sign(VALID_CLAIMS, { issuer: 'another-issuer' })],
      ['a token for another audience', sign(VALID_CLAIMS, { audience: 'another-audience' })],
      ['a token missing required claims', sign({ sub: USERS.user.id })],
    ])('rejects %s with 401', async (description, token) => {
      const req = request(app.getHttpServer()).get('/probe/admin');
      const res = token ? await req.set('Authorization', `Bearer ${token}`) : await req;
      expect(res.status).toBe(401);
    });
  });
});

// test/e2e/user/user.spec.ts
describe('DELETE /users/:id', () => {
  it('rejects a caller without the admin role', async () => {
    const res = await request(app.getHttpServer()).delete(`/users/${USER_ID}`).set('x-test-user', 'user');
    expect(res.status).toBe(403);
  });
});

// test/unit/user/create-user.dto.spec.ts
describe('CreateUserDto', () => {
  const pipe = new ValidationPipe(API_VALIDATION_PIPE_OPTIONS);
  const metadata: ArgumentMetadata = { type: 'body', metatype: CreateUserDto };

  it('rejects unknown properties', async () => {
    await expect(
      pipe.transform({ name: 'Jane Doe', email: 'jane@example.com', password: 'correct-horse-battery', role: 'admin' }, metadata),
    ).rejects.toThrow(BadRequestException);
  });
});
```

The roles guard spec mirrors `src/auth/guards/auth-roles.guard.ts` (see `test-organize-by-tier`). Its test guard picks the caller from a request header the spec sets and is registered before `AuthRolesGuard`. `createTestApp()` runs `ApiModule.setup()`, so the probe routes get the envelope, the serializer, the filter and the `ValidationPipe`; the interceptor spec uses the same probe controller and setup. The `AuthJwtGuard` spec runs the real guard and strategy, with tokens signed with a test secret, and registers no `AuthRolesGuard`, so `/probe/admin` needs only a token. It uses the same `ProbeController`, which moves to `test/helpers/` once two specs use it. The user spec registers `AuthRolesGuard` too, so it proves the real route is protected. The DTO spec needs no Nest container: its pipe takes `API_VALIDATION_PIPE_OPTIONS` from `api/constants/`, the options production uses.

Reference: [NestJS Testing](https://docs.nestjs.com/fundamentals/testing) · [Guards](https://docs.nestjs.com/guards)
