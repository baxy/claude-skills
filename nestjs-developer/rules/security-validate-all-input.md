---
title: Validate All Input with DTOs and the Global ValidationPipe
impact: HIGH
impactDescription: First line of defense: nothing unvalidated reaches a service
tags: security, validation, dto, pipes
---

## Validate All Input with DTOs and the Global ValidationPipe

Every request body, query string and route parameter is bound to a DTO class with `class-validator` decorators and checked by one global `ValidationPipe`, registered in `ApiModule.setup()` with the shared `API_VALIDATION_PIPE_OPTIONS` (see `test-use-testing-module`):

- `whitelist: true` strips properties without decorators; `forbidNonWhitelisted: true` rejects them with a 400 instead, so a client sending `role` or `status` learns it is not accepted.
- `forbidUnknownValues: true` rejects an object that has no validation metadata. Nest's `ValidationPipe` defaults it to `false`, and then a nested object whose `@Type` was forgotten passes without being validated. A request DTO therefore needs at least one validator.
- `transform: true` returns DTO instances. Convert query and form values explicitly with `@Type(() => Number)` / `@Type(() => Date)`; do not turn on `enableImplicitConversion`, which converts `'false'` to `true`.
- Nested objects and arrays need `@ValidateNested()` **and** `@Type(() => Child)`. Without both, the nested payload is not validated at all. `@ValidateNested()` also passes a property that is missing, and an array where an object belongs: every nested object, optional or required, adds `@IsObject()`, and a nested array `@IsArray()`. A nested class has no field named `push`: class-transformer then builds an array sent for the object as an instance of that class, so `[1]` fails with a 500 before any validator runs, and `[]` passes `@IsObject()`.
- **`@IsOptional()` lets `null` through as well as a missing field**, on every optional field, not only on those `PartialType` makes optional (see `api-use-mapped-types`). `{ "currency": null }` then reaches the service, and a NOT NULL column turns it into a 500. An optional body field over a NOT NULL column takes `@ValidateIf((dto, value) => value !== undefined)` instead, so a `null` fails its validators with a 400; a field the client may clear is nullable in the entity. Query DTOs need nothing: a query string has no `null`.
- Every string has a `@MaxLength`, an uploaded file's name included (`@MaxFileNameLength()`, see `api-file-uploads`), every array an `@ArrayMaxSize`, every number a `@Max`, page sizes and offsets included (see `api-dto-naming`), and every date the api module's `@IsStorableDate()` (in `api/validators/`: 1970 to 9999). An unbounded number reaches the database, which rejects one past its column or `OFFSET` range with a 500; so does a date past `timestamptz`'s range, and one JavaScript cannot add a second to (year 275760) becomes `Invalid Date` in the query.
- **A string stored in a `varchar(n)` column is bounded by what the column counts.** `@MaxLength` counts the way validator.js does: a surrogate pair as one character, and a variation selector (U+FE0E, U+FE0F) after a character not at all. PostgreSQL counts code points, variation selectors included, so 150 red hearts written as U+2764 U+FE0F pass `@MaxLength(200)` and are 300 characters to a `varchar(200)`: a 500. Such a field takes the api module's `@MaxCodePoints(n)` (in `api/validators/`, counting code points and stopping past `n`) in place of `@MaxLength(n)`.
- **A value the server can know never comes from the request.** A DTO carries only what the client chooses (what it buys and how many, an address); prices, totals, discounts, balances and the caller's identity come from the server's own data (the caller from `@AuthUser()`, see `api-custom-param-decorators`). A field the server would only check against its own value is not in the DTO at all: with it there, one forgotten comparison lets the client set the price.
- Ids use `@IsUUID()` (any version), so the nil UUID used in documentation examples (see `api-document-with-openapi`) passes "Try it out".
- A URL the server will call (a webhook target, a file to import) is `@IsUrl({ protocols: ['https'], require_protocol: true })`. That does not stop it reaching an internal address: the client checks the address at every connection (see `micro-use-http-clients`).
- Passwords: `@MinLength(12)` and `@MaxLength(128)`; length beats composition rules. Never `@Transform` a password, and hash it before it reaches an entity (see `security-hash-passwords`).

**Incorrect (raw input, undecorated DTOs):**

```typescript
@Controller('users')
export class UserController {
  @Post()
  create(@Body() body: any) {
    return this.userService.create(body); // Anything reaches the service
  }

  @Get()
  findAll(@Query() query: any) {
    return this.userService.findAll(query.limit); // A string, possibly "-1" or "abc"
  }
}

export class CreateUserDto {
  name: string;  // No decorators: whitelist strips it, or without whitelist anything passes
  email: string;
}
```

**Correct (validated DTOs, one global pipe):**

```typescript
// user/dtos/create-user.dto.ts
export class CreateUserDto {
  @ApiProperty({ description: 'Display name', example: 'Jane Doe' })
  @IsString()
  @IsNotEmpty()
  @MaxLength(100)
  @Transform(({ value }) => (typeof value === 'string' ? value.trim() : value))
  name: string;

  @ApiProperty({ description: 'Login email address', example: 'jane@example.com' })
  @IsEmail()
  @MaxLength(254)
  @Transform(({ value }) => (typeof value === 'string' ? value.trim().toLowerCase() : value))
  email: string;

  @ApiProperty({ description: 'Initial password, 12 to 128 characters', example: 'correct-horse-battery' })
  @IsString()
  @MinLength(12)
  @MaxLength(128)
  password: string;
}

// user/dtos/find-all-users.query.dto.ts
export class FindAllUsersQueryDto extends ApiPaginationQueryDto {
  @ApiPropertyOptional({ description: 'Matches the start of the name or email', example: 'jane' })
  @IsOptional()
  @IsString()
  @MaxLength(100)
  search?: string;
}

// user/dtos/user-id.param.dto.ts
export class UserIdParamDto {
  @ApiProperty({ description: 'UUID of the user', example: '00000000-0000-0000-0000-000000000000' })
  @IsUUID()
  id: string;
}

// order/dtos/create-order.dto.ts
export class CreateOrderDto {
  @ApiProperty({ description: 'Where the order is shipped', type: OrderAddressDto })
  @IsObject()
  @ValidateNested()
  @Type(() => OrderAddressDto)
  shippingAddress: OrderAddressDto;

  @ApiProperty({ description: 'Items in the order, 1 to 50', type: [OrderItemDto] })
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(50)
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items: OrderItemDto[];
}

// user/controllers/user.controller.ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Post()
  create(@Body() dto: CreateUserDto): Promise<User> {
    return this.userService.create(dto);
  }

  @Roles(Role.Admin)
  @Get()
  findAll(@Query() query: FindAllUsersQueryDto): Promise<PaginatedResult<User>> {
    return this.userService.findAll(query);
  }

  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id);
  }
}
```

`FindAllUsersQueryDto` takes its pagination from the shared `ApiPaginationQueryDto`, and `CreateOrderDto` shows a nested object and a nested array.

Queries built from validated values still use parameters (`where('user.email = :email', { email })`), never string concatenation.

Reference: [NestJS Validation](https://docs.nestjs.com/techniques/validation)
