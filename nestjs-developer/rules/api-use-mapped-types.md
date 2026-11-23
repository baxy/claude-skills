---
title: Derive DTOs with Mapped Types
impact: MEDIUM
impactDescription: Keeps Create/Update/Response DTOs in sync and their validation rules consistent
tags: api, dto, mapped-types, validation
---

## Derive DTOs with Mapped Types

Use separate DTOs per operation (`CreateXDto`, `UpdateXDto`, `XResponseDto`), but derive the input DTOs from each other with `PartialType`, `PickType`, `OmitType` and `IntersectionType` instead of copying fields; a response DTO is the exception (below). Copied DTOs drift: a field gets a new validator in one place and not the other, and update endpoints silently accept data that create rejects. Mapped types copy both the properties and their `class-validator`/`class-transformer` decorators.

Import mapped types from `@nestjs/swagger` when the project uses Swagger (so the derived DTOs keep their OpenAPI metadata), otherwise from `@nestjs/mapped-types`.

**Incorrect (hand-copied DTOs that drift apart):**

```typescript
export class CreateUserDto {
  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(12)
  password: string;
}

// Copied by hand - MaxLength was forgotten, password can be changed here
export class UpdateUserDto {
  @IsOptional()
  @IsString()
  @MinLength(2)
  name?: string;

  @IsOptional()
  @IsEmail()
  email?: string;

  @IsOptional()
  @IsString()
  password?: string; // Password changes should go through a dedicated flow
}

// One DTO reused for everything - client can set server-owned fields
@Patch(':id')
update(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto, @Body() dto: User) {
  return this.userService.update(authUser, params.id, dto); // dto.role, dto.id accepted!
}
```

**Correct (derive DTOs from a single source of truth):**

```typescript
import { IntersectionType, OmitType, PartialType, PickType } from '@nestjs/swagger';

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

  @ApiPropertyOptional({ description: 'UUID of the team the user joins', example: '00000000-0000-0000-0000-000000000000' })
  @IsOptional()
  @IsUUID()
  teamId?: string;
}

// user/dtos/update-user.dto.ts
export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['password', 'teamId'] as const),
  { skipNullProperties: false },
) {}

// user/dtos/change-user-password.dto.ts
export class ChangeUserPasswordDto extends PickType(CreateUserDto, ['password'] as const) {
  @ApiProperty({ description: 'The current password', example: 'old-horse-battery' })
  @IsString()
  @MaxLength(128)
  currentPassword: string;
}

// user/dtos/find-all-users.query.dto.ts
export class FindAllUsersQueryDto extends IntersectionType(
  ApiPaginationQueryDto,
  PickType(PartialType(CreateUserDto), ['name', 'email'] as const),
) {}

@Controller('users')
export class UserController {
  @Patch(':id')
  update(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto, @Body() dto: UpdateUserDto): Promise<User> {
    return this.userService.update(authUser, params.id, dto);
  }

  @Roles(Role.Admin)
  @Get()
  findAll(@Query() query: FindAllUsersQueryDto): Promise<PaginatedResult<User>> {
    return this.userService.findAll(query);
  }
}
```

`CreateUserDto` is the source of truth for editable fields, and its email is lowercased because login looks emails up lowercased. `UpdateUserDto` makes every field optional and keeps the validators and docs; the fields that must not change after creation are omitted first, and the `as const` is required. Its `skipNullProperties: false` still validates a `null`, which `@IsOptional()` would let through to a NOT NULL column (below). `ChangeUserPasswordDto` is a dedicated DTO for a dedicated flow. `FindAllUsersQueryDto` combines the pagination piece (`api/dtos/api-pagination.query.dto.ts`, see `api-dto-naming`) with optional filters taken from the Create DTO.

Never re-declare a Create DTO's fields in its Update DTO: if `name` or `email` appear in both files, use `PartialType` instead.

**A body DTO made with `PartialType` passes `{ skipNullProperties: false }`.** By default `PartialType` marks every field `@IsOptional()`, which skips validation for `null` as well as for a missing field. `PATCH /users/:id` with `{ "name": null }` then reaches the database, and the NOT NULL column turns it into a 500. With the option, a missing field is still optional and a `null` fails the field's validators: a 400. A field the client may really clear is nullable in the entity and carries its own `@ValidateIf((dto) => dto.field !== null)`. Query DTOs need nothing: a query string has no `null`. The option covers only the fields `PartialType` makes optional: a field written with `@IsOptional()` by hand lets `null` through the same way (see `security-validate-all-input`).

**A response DTO declares its fields itself.** Its fields carry `@Expose()` (see `api-use-dto-serialization`), which a Create DTO does not have, and a mapped type of the Create DTO would also copy its validators and `@Transform`s onto the response. `PickType(CreateUserDto, ['email', 'name'])` with `excludeExtraneousValues: true` therefore serializes to `{}`.

Reference: [NestJS Mapped Types](https://docs.nestjs.com/techniques/validation#mapped-types)
