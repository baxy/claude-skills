---
title: Throw HTTP Exceptions from Services
impact: HIGH
impactDescription: Keeps controllers thin and error statuses decided where the rule lives
tags: error-handling, exceptions, services
---

## Throw HTTP Exceptions from Services

In an HTTP application, services throw `HttpException` subclasses directly: `NotFoundException` for a missing (or not visible) resource, `BadRequestException` for invalid input or a domain rule the request breaks (an illegal state transition, insufficient stock), `ConflictException` when the request clashes with existing data (a duplicate, a concurrent change), `ForbiddenException` when a rule forbids the caller an action on a resource it can see. Never return error objects for the controller to inspect.

A service whose callers outside HTTP must tell its errors apart by type (a queue processor deciding whether to retry) throws a domain error class instead, such as `EntityNotFoundException`. The global filter maps it to a status (see `error-use-exception-filters`); do not add a second filter for it. A command line reads an `HttpException`'s status (see `arch-feature-modules`), so a service it shares with the API keeps throwing `HttpException`s.

When a client needs to tell two errors with the same status apart (an email already taken versus a username already taken, insufficient stock versus a closed store), give the exception an `errorCode`: `throw new ConflictException('Email already registered', { errorCode: 'EMAIL_TAKEN' })`. A domain refusal answered with 400, the status of the route's validation errors, always gets an `errorCode` of its own (`ORDER_NOT_CANCELABLE`, `INSUFFICIENT_STOCK`), or it reads as `BAD_REQUEST`, like every invalid field. Codes are `UPPER_SNAKE_CASE` and never change once clients use them; the message can still be reworded. The global filter copies the code into the error response (see `error-use-exception-filters`). Any other error no client needs to tell apart from others of its status needs no code of its own: the filter gives it the name of its status (`NOT_FOUND`). The option exists from NestJS 12; on NestJS 11, pass the code in the response object instead: `new ConflictException({ message: 'Email already registered', errorCode: 'EMAIL_TAKEN' })`.

When the client should retry after a known delay (a 503 while a dependency is down), pass `retryAfterSeconds` in the response object: `throw new ServiceUnavailableException({ message: 'Payment provider unavailable', retryAfterSeconds: 30 })`. NestJS 12's exception options have no field for it, so it travels the way `errorCode` does on NestJS 11. The global filter writes it as the `Retry-After` header and leaves it out of the body.

Messages may include identifiers that were validated by a DTO (a UUID from `UserIdParamDto`); never echo unvalidated input back (see `security-sanitize-output`).

**Incorrect (return error objects instead of throwing):**

```typescript
@Injectable()
export class UserService {
  async findById(authUser: JwtUser, id: string): Promise<{ user?: User; error?: string }> {
    this._assertVisible(authUser, id);
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) {
      return { error: 'User not found' }; // Every caller must remember to check
    }
    return { user };
  }
}

@Controller('users')
export class UserController {
  @Get(':id')
  async findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto) {
    const result = await this.userService.findById(authUser, params.id);
    if (result.error) {
      throw new NotFoundException(result.error);
    }
    return result.user;
  }
}
```

**Correct (the service throws, the controller returns):**

```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly userPasswordHasherService: UserPasswordHasherService,
  ) {}

  async findById(authUser: JwtUser, id: string): Promise<User> {
    this._assertVisible(authUser, id);
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }
    return user;
  }

  async create(dto: CreateUserDto): Promise<User> {
    try {
      return await this.userRepository.save(await this._toEntity(dto));
    } catch (error) {
      // The unique index decides: a lookup first would let two sign-ups with one email both pass it
      if (isUniqueViolation(error, 'UQ_users_email_active')) {
        throw new ConflictException('Email already registered', { cause: error, errorCode: 'EMAIL_TAKEN' });
      }
      throw error;
    }
  }

  async update(authUser: JwtUser, id: string, dto: UpdateUserDto): Promise<User> {
    this._assertVisible(authUser, id);
    const { affected } = await this.userRepository.update({ id, deletedAt: IsNull() }, dto);
    if (!affected) {
      throw new NotFoundException(`User #${id} not found`);
    }
    return this.findById(authUser, id);
  }

  private _assertVisible(authUser: JwtUser, id: string): void {
    if (id !== authUser.id && !authUser.roles.includes(Role.Admin)) {
      throw new NotFoundException(`User #${id} not found`);
    }
  }
}

@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id);
  }
}

// shared/errors/entity-not-found.exception.ts
export class EntityNotFoundException extends Error {
  constructor(
    public readonly entity: string,
    public readonly id: string,
  ) {
    super(`${entity} #${id} not found`);
  }
}
```

`_toEntity()` hashes the password (see `security-hash-passwords`). `_assertVisible()` gives an account that is not the caller's the 404 of a missing one, unless the caller is an admin (see `security-use-guards`). The update is one statement checked through `affected` (see `db-typeorm-use-transactions`). `EntityNotFoundException` is for services whose callers outside HTTP tell errors apart by type, and `ApiExceptionFilter` maps it to 404.

Reference: [NestJS Built-in HTTP exceptions](https://docs.nestjs.com/exception-filters#built-in-http-exceptions)
