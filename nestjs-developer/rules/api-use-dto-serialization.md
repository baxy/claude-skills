---
title: Control Response Serialization
impact: MEDIUM
impactDescription: Prevents accidental data exposure while keeping simple endpoints simple
tags: api, dto, serialization, class-transformer, entities
---

## Control Response Serialization

Register `ClassSerializerInterceptor` globally, in `ApiModule.setup()` and innermost (see `api-use-interceptors`), and decide, per column, what may leave the service. Two shapes are acceptable:

- **Returning the entity.** Fine when every column that must not leave the service carries `@Exclude()` together with `@ApiHideProperty()` (so it is hidden from both the response body and the OpenAPI schema). This is opt-out: a newly added column is public until someone hides it, so adding a column always includes deciding whether to hide it.
- **Returning a response DTO** (`{resource}-response.dto.ts`). Use one when the response shape differs from the entity (computed fields, a subset, nested projections, role-dependent fields) or for a public contract. A DTO with `@Expose()` and `excludeExtraneousValues: true` is opt-in: only listed fields leave the service. **Always pass `excludeExtraneousValues: true` when an entity is turned into a response DTO**, with `plainToInstance()` or `@SerializeOptions({ type })`: the conversion copies every entity property into the DTO and the entity's `@Exclude()` does not survive it, so without the option `passwordHash` is returned.

Entities are persistence classes (see `db-typeorm-schema-conventions`): never bind a request body to an entity, never put `@Expose` or `@ApiProperty` on one, and never manually spread entity fields into ad-hoc objects.

**Incorrect (unhidden sensitive columns, entities as inputs, manual spreading):**

```typescript
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  email: string;

  @Column()
  passwordHash: string; // Returned to every client

  @Column({ default: false })
  @Exclude({ toPlainOnly: true }) // Entity used as request input - clients can set isAdmin
  isAdmin: boolean;
}

@Controller('users')
export class UserController {
  @Post()
  create(@Body() user: User): Promise<User> { // Entity bound to the request body
    return this.userService.create(user);
  }

  @Get(':id')
  async findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto) {
    const user = await this.userService.findById(authUser, params.id);
    // Manual spreading - easy to forget a field, repeated per endpoint
    return { id: user.id, email: user.email, name: user.name };
  }
}
```

**Correct (hidden columns on entities, response DTOs where the shape differs):**

```typescript
// api/api.module.ts
app.useGlobalInterceptors(
  app.get(ApiLoggerInterceptor),
  app.get(ApiResponseInterceptor),
  new ClassSerializerInterceptor(app.get(Reflector)),
);

// user/entities/user.entity.ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  email: string;

  @Column()
  name: string;

  @ApiHideProperty()
  @Exclude()
  @Column()
  passwordHash: string;

  @ApiHideProperty()
  @Exclude()
  @DeleteDateColumn({ type: 'timestamptz', nullable: true })
  deletedAt: Date | null;
}

@Controller('users')
export class UserController {
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id);
  }

  @Get(':id/summary')
  @ApiEnvelopedResponse(UserSummaryResponseDto, { description: 'The user with their order count' })
  @SerializeOptions({ type: UserSummaryResponseDto, excludeExtraneousValues: true })
  findSummary(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findByIdWithOrders(authUser, params.id);
  }
}

// user/dtos/user-summary-response.dto.ts
export class UserSummaryResponseDto {
  @ApiProperty({ description: 'UUID of the user', example: '00000000-0000-0000-0000-000000000000' })
  @Expose()
  id: string;

  @ApiProperty({ description: 'Display name', example: 'Jane Doe' })
  @Expose()
  name: string;

  @ApiProperty({ description: 'Number of orders the user has placed', example: 3 })
  @Expose()
  @Transform(({ obj }) => obj.orders?.length ?? 0)
  orderCount: number;
}

export class UserProfileResponseDto {
  @Expose()
  id: string;

  @Expose()
  name: string;

  @Expose({ groups: ['admin', 'owner'] })
  email: string;
}

// auth/controllers/auth.controller.ts
@Get('me')
@SerializeOptions({ type: UserProfileResponseDto, groups: ['owner'], excludeExtraneousValues: true })
findMe(@AuthUser() authUser: JwtUser): Promise<User> {
  return this.userService.findById(authUser, authUser.id);
}
```

The serializer is the innermost interceptor in `ApiModule.setup()`, so the envelope wraps the serialized body. The entity carries its persistence decorators, plus `@Exclude()` and `@ApiHideProperty()` together. `findById()` answers the entity's own shape, so it returns the entity, and the hidden columns are stripped. `findSummary()` answers a different shape, so `@SerializeOptions()` converts the entity it returns to a response DTO, which is opt-in: only its `@Expose()` fields are returned. `@ApiEnvelopedResponse()` is in `api-document-with-openapi`. Role-dependent fields are serialization groups on a response DTO, as in `UserProfileResponseDto`, and `findMe()` passes `excludeExtraneousValues: true` so `passwordHash` is not copied in.

**Return entity instances, not spread copies.** `ClassSerializerInterceptor` applies `@Exclude()` only to class instances, and `repository.save()` returns the object it was given. `save({ ...user, ...dto })` or `manager.save(User, { ... })` therefore returns a plain object, and every hidden column on it (`passwordHash`, `deletedAt`) reaches the client. Create new rows with `repository.create()` (or `manager.create()`). Update with one `update()` and return the row read afterwards, which is an instance (see `db-typeorm-use-transactions`).

**Incorrect (a plain object comes back, so `@Exclude()` is skipped and `passwordHash` is returned):**

```typescript
return this.userRepository.save({ ...user, ...dto });
```

**Correct (`create()` returns an instance, and so does the read after an update):**

```typescript
return this.userRepository.save(this.userRepository.create(fields));

await this.userRepository.update({ id, deletedAt: IsNull() }, dto);
return this.findById(authUser, id);
```

**`@SerializeOptions({ type })` is for a handler that returns one object or an array.** The serializer converts what the handler returns, and a paginated handler returns the `PaginatedResult` wrapper (`{ result, totalCount, limit, offset }`, see `perf-optimize-database`), not the items. With `excludeExtraneousValues: true` every page then arrives as `data: {}`; without it the items are never converted. A page of entities needs nothing, because the serializer applies `@Exclude()` to the entity instances inside the wrapper. A page of response DTOs is converted item by item in the service (once the route has a second version with another DTO, in each version's controller instead, see `api-versioning`):

**Incorrect (converts the wrapper, so the client gets `data: {}`):**

```typescript
@SerializeOptions({ type: UserListItemResponseDto, excludeExtraneousValues: true })
@Get()
findAll(@Query() query: FindAllUsersQueryDto): Promise<PaginatedResult<User>> {
  return this.userService.findAll(query);
}
```

**Correct (the service converts each item; the serializer leaves the wrapper alone):**

```typescript
return {
  result: users.map((user) => plainToInstance(UserListItemResponseDto, user, { excludeExtraneousValues: true })),
  totalCount,
  limit: query.limit,
  offset: query.offset,
};
```

**A public response never carries another user's internal id.** An internal user id on content anyone can read (in the shop, the author of a review) lets anyone link everything one account wrote. Expose only the fields the product chose to show, and compute caller-relative facts (such as an `isOwn` flag, from the caller of a route that takes an optional token) on the server.

### With Prisma

Prisma returns **plain objects**, not class instances: nothing on them carries `@Exclude()`, and Swagger cannot read a generated type. Two layers replace returning the entity, and both are required:

- **Sensitive fields are never fetched.** Password hashes, token hashes and secrets are in `PRISMA_OMIT`, the global `omit` of `PrismaService` (see `db-prisma-schema-conventions`), so Prisma leaves them out of every result. The types follow only on `PrismaService` itself: inside a transaction (the `$transaction` callback's `tx`, a `Prisma.TransactionClient` parameter) the result types still list the omitted fields, which are absent at runtime. Code that reads one there compiles and gets `undefined`; passed as the expected value of a compare-and-swap, it turns the update into an unconditional write. Read an omitted field only from a query that opts back in, `omit: { passwordHash: false }`, in the owning module's service, which never returns that object; `strictUndefinedChecks` then also refuses the `undefined` a missing opt-in would put in a `where`.
- **Every route that returns data has a response DTO**, converted with `excludeExtraneousValues: true`: `@SerializeOptions({ type, excludeExtraneousValues: true })` on a handler that returns one object or an array, and `plainToInstance()` per item in the service for a page, as above. A column added later stays private until someone adds it to the DTO.
- Never bind a request body to a Prisma input type (`@Body() data: Prisma.UserCreateInput`): validation needs a request DTO (see `security-validate-all-input`), and the input type lets a client set any column, relations included.
- A service returns the type the omit leaves (below), not the generated `User`, which still has `passwordHash`. A method that takes `tx` declares that return type too, since its inferred one lists the omitted fields.

```typescript
// user/types/user-record.type.ts
export type UserRecord = Prisma.UserGetPayload<{ omit: typeof PRISMA_OMIT.user }>;

// user/controllers/user.controller.ts
@Get(':id')
@ApiEnvelopedResponse(UserResponseDto, { description: 'The user' })
@SerializeOptions({ type: UserResponseDto, excludeExtraneousValues: true })
findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<UserRecord> {
  return this.userService.findById(authUser, params.id);
}

// user/services/user.service.ts
const user = await this.prisma.user.findFirst({ where: { email, deletedAt: null }, omit: { passwordHash: false } });
```

`UserRecord` is a user as `PrismaService` returns it, without the `PRISMA_OMIT` fields. The query in the service is the one that selects `passwordHash` (see `security-hash-passwords`).

Reference: [NestJS Serialization](https://docs.nestjs.com/techniques/serialization)
