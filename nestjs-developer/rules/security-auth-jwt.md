---
title: Authenticate with a Passport JWT Strategy That Builds JwtUser
impact: CRITICAL
impactDescription: Every handler receives a complete, verified caller, or the request is refused
tags: security, jwt, authentication, passport, tokens
---

## Authenticate with a Passport JWT Strategy That Builds JwtUser

Authentication has one mechanism: a `passport-jwt` strategy (`AuthJwtStrategy`) behind a global `AuthJwtGuard` (see `security-use-guards`). The strategy verifies the token and turns its claims into a `JwtUser`, the one type for the authenticated caller. Handlers receive it with `@AuthUser()` (see `api-custom-param-decorators`), and services take it as their first parameter (see `arch-service-conventions`).

**`JwtUser` has no optional fields, and `validate()` refuses a token that cannot fill one.** An `undefined` id in a query scoped to the caller fails one of two ways: on TypeORM 0.3, and on Prisma without `strictUndefinedChecks`, which skips every `undefined` value in a query, it drops the `WHERE` condition, so the query answers with every caller's rows; on TypeORM 1.x, whose `invalidWhereValuesBehavior` defaults to `throw`, and on Prisma with `strictUndefinedChecks` (see `db-prisma-schema-conventions`), every such request fails with a 500. Refusing the token is what makes the type true.

- Verify signature, expiry, issuer and audience, and pin the algorithm (`algorithms: ['HS256']` in the strategy, `algorithm: 'HS256'` in `JwtModule`'s sign options), so a token cannot choose its own. With your own signing key, read it from config (`auth.jwt.*`, see `devops-use-config-module`); with an external identity provider (Auth0, Cognito), use `secretOrKeyProvider` from `jwks-rsa`, the provider's issuer and audience, and the algorithm it signs with (usually `RS256`).
- **Build `JwtUser` from the claims while they stay fresh.** With a refresh exchange and access tokens of 15 minutes or less, each refresh rebuilds the roles from the user's row, so the claims are at most minutes old: `validate()` reads no database, and services that need more than the claims load it themselves. A deleted or deactivated user keeps access until their access token expires, which is why access tokens live minutes.
- **Without a refresh exchange, or with longer access tokens, load the caller on every request.** Nothing renews such a token's claims before it expires, so a role removed now would keep working for the token's whole life (an hour, at the 60-minute limit below). `validate()` then loads the user by the token's `sub` through `UserService` (one primary-key read) and builds `JwtUser` from the row: a deleted user gets a 401, and `roles` are the current ones, so `AuthRolesGuard` and every service that reads `authUser.roles` see a change at the next request. The token carries no roles then (see "An app without refresh tokens", below).
- **When the app can disable accounts**, what is done inside those minutes must not outlast them: the routes that change who may sign in or act (in the shop: deactivating or reactivating a user, changing an email, a password or roles) load the caller and refuse a deactivated one (403). That check is part of the change: it reads the caller's row inside the change's transaction, locked with the target's in id order, so two callers acting on each other at the same moment cannot both pass it (see `db-use-transactions`). A login refuses a deactivated user too, once the password has matched (see `security-hash-passwords`). When a deactivation must also end what the caller already holds, its event does it (the shop ends a deactivated admin's claims and live connections). Rules about acting on one's own account belong to the app (the shop refuses a caller acting on their own deactivation).
- Never put secrets or personal data beyond what `JwtUser` needs into a token you issue.
- Tokens you issue: short-lived access tokens (minutes), and refresh tokens that are random, stored hashed with SHA-256 (they are high-entropy, so a slow password hash adds nothing and prevents lookup), and revocable. Passwords are the opposite case: see `security-hash-passwords`.
- **`validate-env.ts` holds both values to their bounds** (see `devops-use-config-module`). It refuses an `AUTH_JWT_SECRET` under 64 characters: HS256 needs a key of at least 256 bits (RFC 7518, section 3.2), which is 64 characters as the hex that `openssl rand -hex 32` prints. It refuses an access-token TTL over 60 minutes, so "minutes" holds in every environment and a revoked session's access token dies soon.

**When the app issues its own refresh tokens**, a session is one row, and that row alone decides whether the session lives. (With an external identity provider, the provider issues and revokes them, and the app has no login route.) Each login creates a session row, whose id the access token carries (`JwtUser.sessionId`), and its first refresh token. Each refresh token works once.

- **The refresh exchange checks the user again.** It loads the user and refuses one that no longer exists (401) or, when the app can disable accounts, is deactivated (403 `USER_DEACTIVATED`, so the user is told why, as at login), checked only after the session and the token passed their own checks: a token of an ended session answers 401 whatever the account's state, and learns nothing about it. It builds the new access token from the user's current roles, so a role change takes effect at the next refresh. Deactivating or deleting a user revokes their sessions (each session's row, below).
- **Refresh** runs one transaction. It first updates the session row on the conditions it must meet (not revoked, not expired), which locks that row until the transaction ends. It then marks the presented token used, on the condition that it was still unused, and saves the next token. Either update changing nothing is a 401.
- **Logout, a reused token and a user's deletion revoke the session row.** A revocation that meets a refresh in progress waits for it on the row lock, and every later refresh finds the session revoked, so no token of the session survives. Revoking the session's tokens instead (`UPDATE … WHERE session_id = :id`) misses the token a refresh is inserting at that moment (see `db-use-transactions`).
- **A refresh token presented a second time revokes its session**: a copy of it exists, and neither holder should keep the session.
- **A session ends at a fixed time**, set once, at login, and refreshing never moves it, so a session cannot be kept alive forever, a stolen one included. Every refresh token of the session expires with it.
- An access token keeps working until it expires after the session is revoked, which is why access tokens live minutes.
- **Ended rows are deleted.** Every refresh adds a token row. A cleanup deletes the sessions that ended (revoked or expired) longer ago than their retention, their tokens with them, and the tokens used longer ago than theirs in live sessions, both retentions in the auth module's `constants/`. The `scheduler/` module calls it on an interval on every instance, which is safe because a second run finds nothing more to delete (see `arch-feature-modules`). A used token deleted this way is refused as unknown when it comes back, instead of ending its session: the token retention is how long a reuse is caught.
- **A login is limited per account too**, not only per address (see `security-rate-limiting`).

**Incorrect (hardcoded secret, untyped payload, partial users accepted):**

```typescript
@Module({
  imports: [JwtModule.register({ secret: 'my-secret-key', signOptions: { expiresIn: '7d' } })], // Secret in code, long-lived
})
export class AuthModule {}

@Injectable()
export class AuthJwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({ jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), secretOrKey: 'my-secret-key' });
  }

  async validate(payload: any): Promise<any> {
    return payload; // Raw claims; a token without `sub` yields a user with no id
  }
}
```

**Correct (configured strategy, complete JwtUser or refusal):**

```typescript
// shared/types/jwt-user.type.ts
export type JwtUser = {
  id: string;
  email: string;
  roles: Role[];
  sessionId: string;
};

// src/config/auth.config.ts
export const authConfig = registerAs('auth', () => ({
  jwt: {
    secret: process.env.AUTH_JWT_SECRET || '',
    issuer: process.env.AUTH_JWT_ISSUER || '',
    audience: process.env.AUTH_JWT_AUDIENCE || '',
    accessTokenTtl: process.env.AUTH_JWT_ACCESS_TOKEN_TTL || '15m',
  },
}));

// auth/strategies/auth-jwt.strategy.ts
@Injectable()
export class AuthJwtStrategy extends PassportStrategy(Strategy) {
  constructor(@Inject(authConfig.KEY) authSettings: ConfigType<typeof authConfig>) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: authSettings.jwt.secret,
      issuer: authSettings.jwt.issuer,
      audience: authSettings.jwt.audience,
      // A token cannot choose its own algorithm
      algorithms: ['HS256'],
      ignoreExpiration: false,
    });
  }

  validate(payload: AuthJwtPayload): JwtUser {
    if (!payload.sub || !payload.email || !Array.isArray(payload.roles) || !payload.sid) {
      throw new UnauthorizedException('Token is missing required claims');
    }
    return { id: payload.sub, email: payload.email, roles: payload.roles, sessionId: payload.sid };
  }
}

// auth/auth.module.ts
@Module({
  imports: [
    UserModule,
    PassportModule,
    JwtModule.registerAsync({
      inject: [authConfig.KEY],
      useFactory: (authSettings: ConfigType<typeof authConfig>) => ({
        secret: authSettings.jwt.secret,
        signOptions: {
          algorithm: 'HS256',
          expiresIn: authSettings.jwt.accessTokenTtl,
          issuer: authSettings.jwt.issuer,
          audience: authSettings.jwt.audience,
        },
      }),
    }),
  ],
  controllers: [AuthController],
  providers: [AuthJwtStrategy, AuthSessionService],
})
export class AuthModule {}

// auth/entities/auth-session.entity.ts
@Entity('auth_sessions')
@Index('IDX_auth_sessions_user_id', ['userId'])
export class AuthSession {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'uuid' })
  userId: string;

  @Column({ type: 'timestamptz' })
  expiresAt: Date;

  @Column({ type: 'timestamptz', nullable: true })
  revokedAt: Date | null;

  @Column({ type: 'timestamptz', nullable: true })
  lastRefreshedAt: Date | null;

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date;
}

// auth/constants/auth-session-lifetimes.constant.ts
export const AUTH_SESSION_TTL_MS = 30 * 24 * 60 * 60 * 1000;
export const AUTH_ENDED_SESSION_RETENTION_MS = 7 * 24 * 60 * 60 * 1000;
export const AUTH_USED_TOKEN_RETENTION_MS = 24 * 60 * 60 * 1000;

// auth/services/auth-session.service.ts

@Injectable()
export class AuthSessionService {
  constructor(
    @InjectRepository(AuthRefreshToken)
    private readonly authRefreshTokenRepository: Repository<AuthRefreshToken>,
    @InjectRepository(AuthSession)
    private readonly authSessionRepository: Repository<AuthSession>,
    private readonly jwtService: JwtService,
    private readonly userService: UserService,
    @Inject(getStorageToken()) private readonly throttlerStorage: ThrottlerStorage,
    private readonly dataSource: DataSource,
  ) {}

  async create(dto: CreateAuthSessionDto): Promise<AuthSessionResponseDto> {
    await this._countLoginAttempt(dto.email);
    const user = await this.userService.verifyCredentials(dto.email, dto.password);
    const { session, refreshToken } = await this.dataSource.transaction(async (manager) => {
      // A password change or reset committed since the check revoked every session but this one, not saved yet
      if (!(await this.userService.isCurrentPasswordHash(manager, user.id, user.passwordHash))) {
        throw new UnauthorizedException('Invalid email or password');
      }
      const session = await manager.save(
        manager.create(AuthSession, { userId: user.id, expiresAt: new Date(Date.now() + AUTH_SESSION_TTL_MS) }),
      );
      return { session, refreshToken: await this._saveRefreshToken(manager, session.id) };
    });
    return { accessToken: this._signAccessToken(user, session.id), refreshToken };
  }

  async refresh(dto: RefreshAuthSessionDto): Promise<AuthSessionResponseDto> {
    const presented = await this.authRefreshTokenRepository.findOne({
      where: { tokenHash: this._hash(dto.refreshToken) },
      relations: { session: true },
    });
    if (!presented) {
      throw this._invalidRefreshToken();
    }
    if (presented.usedAt) {
      // A used token came back: a copy exists
      await this._revoke(presented.sessionId);
      throw this._invalidRefreshToken();
    }
    const { user, refreshToken } = await this.dataSource.transaction(async (manager) => {
      const now = new Date();
      // The session row first: this update locks it, so a revocation waits for the transaction to end
      const session = await manager.update(
        AuthSession,
        { id: presented.sessionId, revokedAt: IsNull(), expiresAt: MoreThan(now) },
        { lastRefreshedAt: now },
      );
      const token = await manager.update(AuthRefreshToken, { id: presented.id, usedAt: IsNull() }, { usedAt: now });
      if (!session.affected || !token.affected) {
        throw this._invalidRefreshToken();
      }
      // The account last, so only a live session learns its state
      const user = await this.userService.findForAuthentication(presented.session.userId);
      if (!user) {
        throw this._invalidRefreshToken();
      }
      if (user.deactivatedAt) {
        throw new ForbiddenException('This account is deactivated', { errorCode: 'USER_DEACTIVATED' });
      }
      return { user, refreshToken: await this._saveRefreshToken(manager, presented.sessionId) };
    });
    return { accessToken: this._signAccessToken(user, presented.sessionId), refreshToken };
  }

  async delete(authUser: JwtUser): Promise<void> {
    await this._revoke(authUser.sessionId);
  }

  // A session created after this update fails at its next refresh, which checks the user again
  async revokeAllForUser(userId: string): Promise<void> {
    await this.authSessionRepository.update({ userId, revokedAt: IsNull() }, { revokedAt: new Date() });
  }

  async deleteEnded(): Promise<{ sessions: number; refreshTokens: number }> {
    const sessionsEndedBefore = new Date(Date.now() - AUTH_ENDED_SESSION_RETENTION_MS);
    const sessions = await this.authSessionRepository
      .createQueryBuilder()
      .delete()
      .where('revoked_at < :before OR expires_at < :before', { before: sessionsEndedBefore })
      .execute();
    const refreshTokens = await this.authRefreshTokenRepository.delete({
      usedAt: LessThan(new Date(Date.now() - AUTH_USED_TOKEN_RETENTION_MS)),
    });
    return { sessions: sessions.affected ?? 0, refreshTokens: refreshTokens.affected ?? 0 };
  }

  private async _revoke(sessionId: string): Promise<void> {
    await this.authSessionRepository.update({ id: sessionId, revokedAt: IsNull() }, { revokedAt: new Date() });
  }

  private async _saveRefreshToken(manager: EntityManager, sessionId: string): Promise<string> {
    const refreshToken = randomBytes(32).toString('hex');
    await manager.save(manager.create(AuthRefreshToken, { sessionId, tokenHash: this._hash(refreshToken) }));
    return refreshToken;
  }

  private _signAccessToken(user: User, sessionId: string): string {
    return this.jwtService.sign({ sub: user.id, email: user.email, roles: user.roles, sid: sessionId });
  }

  private _invalidRefreshToken(): UnauthorizedException {
    return new UnauthorizedException('Invalid refresh token');
  }

  private _hash(token: string): string {
    return createHash('sha256').update(token).digest('hex');
  }
}
```

`JwtUser.sessionId` is the login the token belongs to, so logout ends that one session. `validate-env.ts` requires `AUTH_JWT_SECRET`, `AUTH_JWT_ISSUER` and `AUTH_JWT_AUDIENCE`, so the empty defaults in `auth.config.ts` never reach the strategy (see `devops-use-config-module`). `AuthModule` imports `UserModule` for `UserService.verifyCredentials()` (see `security-hash-passwords`), and `JwtModule` signs with `HS256`, the one algorithm `AuthJwtStrategy` accepts. The session lifetimes live in the auth module's `constants/`, not private to the service, because clients see them.

`AuthSessionService` serves `POST /auth/login`, `POST /auth/refresh` and `POST /auth/logout`. `create()` counts the attempt against the per-account limit first, a 429 past it (see `security-rate-limiting`), and `verifyCredentials()` answers 401 on any mismatch and 403 for a deactivated user. `isCurrentPasswordHash()` locks the user's row in the session's transaction (see `security-hash-passwords`). In `refresh()`, the 401 after the two updates covers a revoked or expired session and a token a concurrent refresh used first; a deleted user is a 401 too. `revokeAllForUser()` serves a user's deletion. `deleteEnded()` may run on every instance, since a second run finds nothing more to delete, and the sessions it deletes take their refresh tokens with them through `ON DELETE CASCADE`.

The shop's choices, which another app makes its own way: the `AuthSession` row (table `auth_sessions`: `userId`, `expiresAt`, `revokedAt`, `lastRefreshedAt`) and the `AuthRefreshToken` row (table `auth_refresh_tokens`: `sessionId`, `tokenHash`, `usedAt`); the session's id in the `sid` claim; a refresh that updates the session on `revokedAt IS NULL` and `expiresAt` in the future, then the token on `usedAt IS NULL`; a session of 30 days; ended sessions kept 7 days and used tokens a day; and the `scheduler/` module calling `deleteEnded()` hourly.

**An app without refresh tokens** issues only access tokens, so its `validate()` loads the caller, as said above. Its token holds `sub` and `email`, and its `JwtUser` has no `sessionId`:

```typescript
// auth/strategies/auth-jwt.strategy.ts
async validate(payload: AuthJwtPayload): Promise<JwtUser> {
  if (!payload.sub) {
    throw new UnauthorizedException('Token is missing required claims');
  }
  // The row, not the claims: a removed role or a deleted user counts at the next request
  const user = await this.userService.findForAuthentication(payload.sub);
  if (!user) {
    throw new UnauthorizedException('Invalid token');
  }
  return { id: user.id, email: user.email, roles: user.roles };
}
```

`findForAuthentication()`, the refresh exchange's lookup too, reads the user by primary key, selecting only `id`, `email` and `roles` (and `deactivatedAt` when the app can disable accounts), and returns `null` for a missing or soft-deleted one; when the app can disable accounts, `validate()` also refuses a deactivated user with the 403 a login gives.

Reference: [NestJS Passport](https://docs.nestjs.com/recipes/passport) · [passport-jwt](https://github.com/mikenicholson/passport-jwt)
