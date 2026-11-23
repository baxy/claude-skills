---
title: Bind Params and Queries to DTO Classes
impact: MEDIUM
impactDescription: Pipes and DTOs ensure clean, validated data reaches your handlers
tags: api, pipes, validation, transformation, params
---

## Bind Params and Queries to DTO Classes

Bind `@Param()`, `@Query()` and `@Body()` to DTO classes and let the global `ValidationPipe` validate and transform them. Avoid inline parameter extraction (`@Param('id', ParseUUIDPipe) id: string`, `@Query('page') page: string`): every route then declares its parameters once, in a class that is validated, documented in Swagger, and reusable by longer routes through inheritance (see `api-dto-naming`).

Value conversions belong in the DTO too: `@Type(() => Number)` and `@Type(() => Date)` for numbers and dates, `@Transform` for anything else (splitting a comma-separated list, trimming). A custom `PipeTransform` is only for an argument that is not a DTO, which these conventions rarely have.

**Incorrect (manual parsing, or inline extraction per parameter):**

```typescript
@Controller('users')
export class UserController {
  // Manual validation in every handler
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param('id') id: string): Promise<User> {
    if (!isUUID(id)) {
      throw new BadRequestException('Invalid UUID');
    }
    return this.userService.findById(authUser, id);
  }
}

@Controller('orders/:id/items')
export class OrderItemController {
  // Inline pipes: validation scattered over handlers, invisible to Swagger
  // descriptions, and repeated on every route that takes the same params
  @Get()
  findAll(
    @AuthUser() authUser: JwtUser,
    @Param('id', ParseUUIDPipe) id: string,
    @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
    @Query('offset', new DefaultValuePipe(0), ParseIntPipe) offset: number,
  ): Promise<OrderItem[]> {
    return this.orderItemService.findAll(authUser, id, limit, offset);
  }
}
```

**Correct (param and query DTOs, conversions in the DTO):**

```typescript
// user/dtos/user-id.param.dto.ts
export class UserIdParamDto {
  @ApiProperty({ description: 'UUID of the user', example: '00000000-0000-0000-0000-000000000000' })
  @IsUUID()
  id: string;
}

// order/dtos/find-all-order-items.query.dto.ts
export class FindAllOrderItemsQueryDto extends ApiPaginationQueryDto {
  @ApiPropertyOptional({ description: 'Only items of this stock keeping unit', example: 'SKU-001' })
  @IsOptional()
  @IsString()
  @MaxLength(64)
  sku?: string;
}

@Controller('users')
export class UserController {
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto): Promise<User> {
    return this.userService.findById(authUser, params.id);
  }
}

@Controller('orders/:id/items')
export class OrderItemController {
  @Get()
  findAll(
    @AuthUser() authUser: JwtUser,
    @Param() params: OrderIdParamDto,
    @Query() query: FindAllOrderItemsQueryDto,
  ): Promise<PaginatedResult<OrderItem>> {
    return this.orderItemService.findAll(authUser, params.id, query);
  }
}

// report/dtos/find-all-reports.query.dto.ts
export class FindAllReportsQueryDto extends ApiPaginationQueryDto {
  @ApiProperty({ description: 'Start of the period (ISO 8601)', example: '2026-01-15T10:30:00.000Z' })
  @Type(() => Date)
  @IsDate()
  @IsStorableDate()
  from: Date;

  @ApiProperty({ description: 'End of the period (ISO 8601)', example: '2026-02-15T10:30:00.000Z' })
  @Type(() => Date)
  @IsDate()
  @IsStorableDate()
  to: Date;

  @ApiPropertyOptional({ description: 'Comma-separated category keys', example: 'shoes,bags' })
  @IsOptional()
  @Transform(({ value }) => (typeof value === 'string' ? value.split(',').map((v) => v.trim()) : value))
  @IsArray()
  @ArrayMaxSize(20)
  @IsString({ each: true })
  @MaxLength(50, { each: true })
  categories?: string[];
}
```

The global `ValidationPipe` is registered in `ApiModule.setup()` (see `security-validate-all-input`). `OrderItemController` takes the order module's `OrderIdParamDto` (see `api-dto-naming`) and a query that extends the pagination piece. `FindAllReportsQueryDto` declares its conversions on the DTO: the dates, and the comma-separated list turned into an array. `@IsStorableDate()`, a validator in `api/validators/`, accepts dates from 1970 to 9999: PostgreSQL stores them, and a query can add to them.

Reference: [NestJS Pipes](https://docs.nestjs.com/pipes) · [Validation](https://docs.nestjs.com/techniques/validation)
