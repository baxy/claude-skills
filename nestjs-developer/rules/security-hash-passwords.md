---
title: Hash Passwords with Argon2id in One Provider
impact: CRITICAL
impactDescription: A leaked database does not reveal passwords, and login does not reveal which accounts exist
tags: security, passwords, hashing, argon2, bcrypt, authentication
---

## Hash Passwords with Argon2id in One Provider

A password is stored only as a slow, salted hash. New code uses Argon2id from Node's built-in `crypto.argon2()` (Node 24.7 and later), so no native package is installed or built; a project that already uses the `argon2` package may keep it, and one that already stores bcrypt hashes may keep bcrypt (below). Never store the plain text, never use a fast hash (SHA-256, MD5), and never encrypt a password: one leaked key would reveal every password. High-entropy tokens are a different case and are hashed with SHA-256 (see `security-auth-jwt`).

- **One provider knows the algorithm.** `UserPasswordHasherService` in the user module hashes, verifies and decides when to rehash. It is not exported (see `arch-module-sharing`): other modules go through `UserService`.
- **Hash before the entity exists.** The service turns `CreateUserDto.password` into `passwordHash` before calling `repository.create()`. No column is named `password`, and `passwordHash` carries `@Exclude()` + `@ApiHideProperty()` (see `api-use-dto-serialization`).
- **Parameters are explicit**: Argon2id with `memory: 19_456` (19 MiB), `passes: 2` and `parallelism: 1`, the OWASP minimum (the `argon2` package names the first two `memoryCost` and `timeCost`), a random salt of 16 bytes and a tag of 32. Raise them, never lower them. When they change, `needsRehash()` upgrades each stored hash at the user's next successful login.
- **bcrypt is accepted where it is already in use**, never chosen for new code. Use the native `bcrypt` package (not `bcryptjs`, which hashes on the main thread) with cost 12 or more, and cap the password at 72 **bytes** with `@IsByteLength(1, 72)`. bcrypt silently ignores everything after byte 72, so two passwords that share those bytes both log in, and multi-byte UTF-8 reaches 72 bytes well before 72 characters.
- **Moving from bcrypt to Argon2id needs no migration job.** `verify()` accepts both formats (`$2b$...` and `$argon2id$...`) and `needsRehash()` is true for every bcrypt hash, so each user moves to Argon2id at their next successful login. The rehash updates the row only while it still holds the hash that was just checked (`passwordHash` in the update's criteria): a password changed between the check and the write keeps its new hash, instead of the old password coming back (see `db-use-transactions`).
- **Always use the async API**: `crypto.argon2()`, never `argon2Sync()`. Hashing takes milliseconds to tens of milliseconds and runs on libuv's thread pool; a sync call blocks the event loop for every request.
- **The service writes and reads the hash string itself.** `crypto.argon2()` returns the raw key, so the service stores the PHC string other Argon2 libraries write, `$argon2id$v=19$m=…,t=…,p=…$<salt>$<tag>` with unpadded base64, and verifies by deriving the key again from the stored salt and parameters and comparing it with `timingSafeEqual()`, never `===`. The format is the `argon2` package's, so a project moving from the package keeps every stored hash: each verifies the other's.
- **Emails are compared in one normalized form, the same on write and on login.** `CreateUserDto` and `CreateAuthSessionDto` apply the same `trim().toLowerCase()` transform (see `security-validate-all-input`), so the stored value and the lookup always match. The `email` column and its unique index are case-sensitive and only work because every write stores the normalized form. Adding normalization to the login of a table that already has rows locks out every user stored with capitals or spaces: count them first (`SELECT count(*) FROM users WHERE email <> lower(trim(email))`), resolve any two that normalize to the same address, and normalize the stored rows in a data migration (see `db-use-migrations`) that ships with the login change.
- **A stored hash that cannot be parsed is a failed login, not a crash.** The hasher's parser throws on a hash it cannot read (a bad import, a manual edit, a truncated column), as `argon2.verify` does; `bcrypt.compare` returns `false` instead. `verify()` catches the error and returns `false`, so the client gets the same `401` as for a wrong password instead of a 500 that tells it this account is different. It logs a fixed message at error level, without the error: a parse error can quote parts of the hash.
- **Login reveals nothing.** An unknown email and a wrong password get the same `401 Invalid email or password`, and an unknown email is still verified against a dummy hash so both paths take the same time. Rate-limit the endpoint (see `security-rate-limiting`). When the app can disable accounts, a login refuses a deactivated user with 403 `USER_DEACTIVATED`, checked only once the password has matched, so only someone who knows the password learns the account's state (see `security-auth-jwt`).
- **Never transform a password.** No `trim()` or lowercasing in the DTO; length limits are 12 to 128 characters (see `security-validate-all-input`), plus 72 bytes on bcrypt, and the upper limit also caps the hashing cost.
- **When a user can change their password**, the change requires the current password (`ChangeUserPasswordDto`, see `api-use-mapped-types`) and revokes the user's other sessions in its own transaction, not through an event that runs after it: a refresh in between would still pass. A login re-checks, in the transaction that creates its session and with the user's row locked, that the hash it verified is still the user's, so a login racing a change or a reset keeps no session.
- **When a user can change their email, the change is a credential change.** Whoever changes the email receives what the app mails to the account, a reset link included, and can take it: the change requires the current password, notifies the owner through a channel the change does not control, and is rate-limited like sign-up, since its 409 says an email is taken.
- **If the app offers a password reset**, the link's token is at least 32 random bytes, stored only as its SHA-256 hash, works once and only within a short lifetime, and never travels in a query string, which servers log; the reset page's URL is `https` outside local development. The reset sets the password and revokes every session in one transaction. A reset, an email change and a password change use up the user's open links. The mail job's attempts all fall within the link's lifetime: a link waiting in a queue for hours sits there in plain text. Asking for a link answers 202 whether or not the email exists (see `api-controller-conventions`) and is rate-limited per address and per email (see `security-rate-limiting`).
- Passwords never appear in logs (see `devops-use-logging`) or error messages.

**Incorrect (plain text, fast or reversible hashes, a login that leaks accounts):**

```typescript
await this.userRepository.save(this.userRepository.create(dto)); // dto.password stored as-is

passwordHash: createHash('sha256').update(dto.password).digest('hex'); // Fast: billions of guesses per second on a GPU
passwordHash: encrypt(dto.password, key);                              // Reversible: one leaked key reveals every password

const user = await this.userRepository.findOne({ where: { email } });
if (!user) {
  throw new NotFoundException('No account with this email');          // Tells an attacker which emails exist
}
if (user.passwordHash !== sha256(password)) {                         // Fast hash, non-constant-time comparison
  throw new UnauthorizedException('Wrong password');
}
```

**Correct (one hashing provider, uniform login failure):**

```typescript
// user/services/user-password-hasher.service.ts
import { argon2, randomBytes, timingSafeEqual } from 'node:crypto';
import { promisify } from 'node:util';
// Only while bcrypt hashes from before the switch remain
import * as bcrypt from 'bcrypt';

const argon2Async = promisify(argon2);
const HASH_COST = { memory: 19_456, passes: 2, parallelism: 1 } as const;
const SALT_BYTES = 16;
const TAG_BYTES = 32;
const PHC_ARGON2ID = /^\$argon2id\$v=19\$m=(\d+),t=(\d+),p=(\d+)\$([A-Za-z0-9+/]+)\$([A-Za-z0-9+/]+)$/;
const BCRYPT_PREFIX = /^\$2[aby]\$/;

type HashCost = { memory: number; passes: number; parallelism: number };

// The PHC format's base64 has no padding
function toPhcBase64(bytes: Buffer): string {
  return bytes.toString('base64').replace(/=+$/, '');
}

@Injectable()
export class UserPasswordHasherService implements OnModuleInit {
  private readonly logger = new Logger(UserPasswordHasherService.name);
  private dummyHash = '';

  async onModuleInit(): Promise<void> {
    // Verified against when the email is unknown, so both login paths take the same time
    this.dummyHash = await this.hash(randomBytes(32).toString('hex'));
  }

  async hash(password: string): Promise<string> {
    const salt = randomBytes(SALT_BYTES);
    const tag = await this._derive(password, salt, HASH_COST, TAG_BYTES);
    const { memory, passes, parallelism } = HASH_COST;
    return `$argon2id$v=19$m=${memory},t=${passes},p=${parallelism}$${toPhcBase64(salt)}$${toPhcBase64(tag)}`;
  }

  async verify(hash: string | null, password: string): Promise<boolean> {
    try {
      if (hash !== null && BCRYPT_PREFIX.test(hash)) {
        return await bcrypt.compare(password, hash);
      }
      const stored = this._parse(hash ?? this.dummyHash);
      const tag = await this._derive(password, stored.salt, stored.cost, stored.tag.length);
      return hash !== null && timingSafeEqual(tag, stored.tag);
    } catch {
      // Not the error itself: a parse error can quote parts of the hash
      this.logger.error('A stored password hash could not be parsed');
      return false;
    }
  }

  needsRehash(hash: string): boolean {
    if (BCRYPT_PREFIX.test(hash)) {
      return true;
    }
    const { cost } = this._parse(hash);
    return cost.memory !== HASH_COST.memory || cost.passes !== HASH_COST.passes || cost.parallelism !== HASH_COST.parallelism;
  }

  private _derive(password: string, salt: Buffer, cost: HashCost, tagLength: number): Promise<Buffer> {
    return argon2Async('argon2id', { message: password, nonce: salt, ...cost, tagLength });
  }

  private _parse(hash: string): { cost: HashCost; salt: Buffer; tag: Buffer } {
    const match = PHC_ARGON2ID.exec(hash);
    if (!match) {
      throw new Error('Not an Argon2id hash');
    }
    const [, memory, passes, parallelism, salt, tag] = match;
    return {
      cost: { memory: Number(memory), passes: Number(passes), parallelism: Number(parallelism) },
      salt: Buffer.from(salt, 'base64'),
      tag: Buffer.from(tag, 'base64'),
    };
  }
}

// user/services/user.service.ts
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly userPasswordHasherService: UserPasswordHasherService,
  ) {}

  async create(dto: CreateUserDto): Promise<User> {
    return this.userRepository.save(await this._toEntity(dto));
  }

  async verifyCredentials(email: string, password: string): Promise<User> {
    const user = await this.userRepository.findOne({ where: { email } });
    const valid = await this.userPasswordHasherService.verify(user?.passwordHash ?? null, password);
    if (!user || !valid) {
      throw new UnauthorizedException('Invalid email or password');
    }
    // Only after the password: whoever does not know it learns nothing about the account
    if (user.deactivatedAt) {
      throw new ForbiddenException('This account is deactivated', { errorCode: 'USER_DEACTIVATED' });
    }
    if (this.userPasswordHasherService.needsRehash(user.passwordHash)) {
      const passwordHash = await this.userPasswordHasherService.hash(password);
      const { affected } = await this.userRepository.update(
        // Over the hash just checked, never over a newer one
        { id: user.id, passwordHash: user.passwordHash },
        { passwordHash },
      );
      if (affected) {
        // The login's own re-check compares the hash the row holds now
        user.passwordHash = passwordHash;
      }
    }
    return user;
  }

  // The lock holds a password change back until the login's session exists, so the change revokes it
  async isCurrentPasswordHash(manager: EntityManager, id: string, passwordHash: string): Promise<boolean> {
    const user = await manager.findOne(User, { where: { id }, lock: { mode: 'pessimistic_write' } });
    return user?.passwordHash === passwordHash;
  }

  private async _toEntity(dto: CreateUserDto): Promise<User> {
    const { password, ...fields } = dto;
    return this.userRepository.create({ ...fields, passwordHash: await this.userPasswordHasherService.hash(password) });
  }
}

// user/user.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UserController],
  providers: [UserService, UserPasswordHasherService],
  exports: [UserService],
})
export class UserModule {}
```

`UserService.create()` leaves out its duplicate email check, shown in `error-throw-http-exceptions`. `AuthSessionService` in the auth module calls `verifyCredentials()` for `POST /auth/login`, with the email already normalized by `CreateAuthSessionDto`, and `isCurrentPasswordHash()` in the transaction that saves the session (see `security-auth-jwt`). The shop disables accounts through `deactivatedAt`.

The shop's credential flows, left out of the example, are one way to meet the rules above: an email change mails the old address that it changed; a reset link lives 30 minutes and carries its token in the URL's fragment (`#token=`); the reset sets the password, uses up the user's other links and revokes every session in one transaction, which takes the per-user lock first, as asking for a link does, so two links used at once wait for each other instead of deadlocking.

**A project that stays on bcrypt** keeps the same class and the same `UserService`; only the internals and the DTO limit change:

```typescript
// user/services/user-password-hasher.service.ts
const BCRYPT_COST = 12;

hash(password: string): Promise<string> {
  return bcrypt.hash(password, BCRYPT_COST);
}

async verify(hash: string | null, password: string): Promise<boolean> {
  const matches = await bcrypt.compare(password, hash ?? this.dummyHash);
  return hash !== null && matches;
}

needsRehash(hash: string): boolean {
  return bcrypt.getRounds(hash) < BCRYPT_COST;
}

// user/dtos/create-user.dto.ts
@ApiProperty({ description: 'Initial password, 12 to 128 characters and at most 72 bytes', example: 'correct-horse-battery' })
@IsString()
@MinLength(12)
@MaxLength(128)
// bcrypt reads only the first 72 bytes
@IsByteLength(1, 72)
password: string;
```

In this variant, `onModuleInit()` makes `dummyHash` with bcrypt.

### With Prisma

`passwordHash` is in `PRISMA_OMIT` (see `db-prisma-schema-conventions`), so no query returns it unless it asks. The credentials lookup is the one query that does, and the object it returns never leaves `UserService`:

```typescript
const user = await this.prisma.user.findFirst({ where: { email, deletedAt: null }, omit: { passwordHash: false } });
const valid = await this.userPasswordHasherService.verify(user?.passwordHash ?? null, password);
```

Reference: [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) · [Node.js crypto.argon2()](https://nodejs.org/api/crypto.html) · [PHC string format](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md) · [node-argon2](https://github.com/ranisalt/node-argon2)
