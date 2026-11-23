---
title: Name and Structure DTOs Consistently
impact: MEDIUM
impactDescription: Any DTO's file and class name can be predicted from the route it serves
tags: api, dto, naming, params, queries, conventions
---

## Name and Structure DTOs Consistently

DTOs live in the module's `dtos/` folder (see `arch-feature-modules`). File names are kebab-case; class names are the PascalCase file name with the suffix shown.

| Kind | File | Class |
|---|---|---|
| Create body | `create-{resource}.dto.ts` | `CreateOrderDto` |
| Update body | `update-{resource}.dto.ts` | `UpdateOrderDto` |
| Action body | `{verb}-{resource}.dto.ts` (`cancel-order`, `ship-order`) | `CancelOrderDto` |
| Collection action body (`POST /{resources}/{verb}`) | `{verb}-{resource}.dto.ts` (`check-product-import`); a check of a create extends the create DTO (see `api-use-mapped-types`) | `CheckProductImportDto` |
| Search body (`POST /{resources}/search`) | `search-{resources}.dto.ts` | `SearchOrdersDto` |
| Internal route body | `{verb}-{resource}-internal.dto.ts` | `CreateOrderInternalDto` |
| Route params | `{resource}-id.param.dto.ts`, named for the deepest resource (`order-item-id` for `/orders/:id/items/:itemId`) | `OrderIdParamDto`, `OrderItemIdParamDto` |
| Endpoint query | `{verb}-{resources}.query.dto.ts`; list endpoints use `find-all-{resources}` | `FindAllOrdersQueryDto` |
| Reusable query piece | `{concern}.query.dto.ts` (`pagination`, `sort`) | `ApiPaginationQueryDto` |
| Response | `{resource}-response.dto.ts` | `OrderResponseDto` |
| Nested input shape | in the DTO's file; `{module}-{shape}.dto.ts` when several DTOs use it | `OrderItemDto`, `OrderAddressDto` |

Rules behind the table:

- **Params use `.param.dto.ts`** (dot, like `.query.dto.ts`), never `-param.dto.ts`. Longer routes extend the shorter route's param DTO, so each parameter is declared once.
- **Queries start with a verb** when they belong to one endpoint. A piece reused by several queries is named for its concern, and endpoint queries extend it. List queries extend `ApiPaginationQueryDto`.
- **Request bodies start with a verb** (`create-`, `update-`, `cancel-`, `search-`), never a `-request` suffix.
- **Responses end in `-response`**, never a bare noun (`order.dto.ts` reads like an input).
- **Nested input shapes** are named for the shape, with the module's prefix. A shape used by only one DTO lives in that DTO's file. A shape several DTOs use gets its own file in the `dtos/` of the module whose data it describes (`OrderAddressDto` in `order/dtos/order-address.dto.ts`, the order's shipping address), and a DTO of another module that nests it imports it from there (see `arch-import-boundaries`). It never moves to the api module, which holds only what works the same for every module (`ApiPaginationQueryDto`), or to `shared/`, which holds no DTOs.
- The Create DTO is the source of truth for editable fields; Update DTOs derive from it (see `api-use-mapped-types`).

**Incorrect (mixed separators, missing verbs, ambiguous nouns):**

```typescript
dtos/
├── order-id-param.dto.ts          // dash instead of .param
├── report.query.dto.ts            // one endpoint's query without a verb
├── checkout-request.dto.ts        // -request suffix instead of a verb
├── order.dto.ts                   // response named like an input
└── order-item-id-param.dto.ts     // dash again, and repeats the order id field instead of extending

// Each route re-declares the same params
export class OrderItemIdParamDto {
  @IsUUID() id: string;            // Duplicated from OrderIdParamDto
  @IsUUID() itemId: string;
}
```

**Correct (predictable names, params declared once):**

```typescript
order/dtos/
├── order-id.param.dto.ts
├── order-item-id.param.dto.ts
├── create-order.dto.ts
├── update-order.dto.ts
├── cancel-order.dto.ts
├── search-orders.dto.ts
├── find-all-orders.query.dto.ts
├── export-orders.query.dto.ts
├── order-address.dto.ts
└── order-response.dto.ts

api/dtos/
└── api-pagination.query.dto.ts

// order-id.param.dto.ts
export class OrderIdParamDto {
  @ApiProperty({ description: 'UUID of the order', example: '00000000-0000-0000-0000-000000000000' })
  @IsUUID()
  id: string;
}

// order-item-id.param.dto.ts
export class OrderItemIdParamDto extends OrderIdParamDto {
  @ApiProperty({ description: 'UUID of the order item', example: '00000000-0000-0000-0000-000000000000' })
  @IsUUID()
  itemId: string;
}

// api/dtos/api-pagination.query.dto.ts
export class ApiPaginationQueryDto {
  @ApiPropertyOptional({ description: 'Maximum number of items to return', example: 20 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit: number = 20;

  @ApiPropertyOptional({ description: 'Number of items to skip from the start', example: 0 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  // Larger values lose precision, and PostgreSQL rejects an OFFSET past bigint with an error
  @Max(Number.MAX_SAFE_INTEGER)
  offset: number = 0;
}

// find-all-orders.query.dto.ts
export class FindAllOrdersQueryDto extends ApiPaginationQueryDto {
  @ApiPropertyOptional({ description: 'Only orders in this status', example: 'paid' })
  @IsOptional()
  @IsEnum(OrderStatus)
  status?: OrderStatus;
}

// create-order.dto.ts
export class OrderItemDto {
  @ApiProperty({ description: 'Stock keeping unit of the product', example: 'SKU-001' })
  @IsString()
  @MaxLength(64)
  sku: string;

  @ApiProperty({ description: 'Number of units, 1 to 99', example: 2 })
  @IsInt()
  @Min(1)
  @Max(99)
  quantity: number;
}

export class CreateOrderDto {
  @ApiProperty({ description: 'Items in the order, 1 to 50', type: [OrderItemDto] })
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(50)
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items: OrderItemDto[];

  @ApiProperty({ description: 'Where the order is shipped', type: OrderAddressDto })
  // @ValidateNested() alone accepts a missing address, and an array
  @IsObject()
  @ValidateNested()
  @Type(() => OrderAddressDto)
  shippingAddress: OrderAddressDto;
}

// order-response.dto.ts
export class OrderResponseDto {
  @ApiProperty({ description: 'UUID of the order', example: '00000000-0000-0000-0000-000000000000' })
  @Expose()
  id: string;

  @ApiProperty({ description: 'Current order status', example: 'paid' })
  @Expose()
  status: OrderStatus;
}
```

`OrderIdParamDto` serves `GET /orders/:id`, and `OrderItemIdParamDto` serves `GET /orders/:id/items/:itemId` by extending it. `ApiPaginationQueryDto` is the reusable piece, and it always caps the page size; `FindAllOrdersQueryDto`, one endpoint's query, extends it. `create-order.dto.ts` keeps `OrderItemDto`, a nested shape only it uses, and imports `OrderAddressDto`, which `PUT /orders/:id/shipping-address` also takes as its body. `search-orders.dto.ts`, the body of `POST /orders/search`, holds its nested filter shapes too.

**Search body (`POST /{resources}/search`, see `api-controller-conventions`):** used when the criteria do not fit a query string. It follows the same rules as any other body, plus:

- Pagination works exactly as for `GET` lists: the DTO extends `ApiPaginationQueryDto`, so `limit` is capped the same way.
- Every criterion is optional, and every array is capped (`@ArrayMaxSize`) so one request cannot fan out into an unbounded query.
- The exception is a lookup by a secret value, such as a gift card's code (in the body, so it stays out of URLs and their logs): the secret is required, since an optional one would list every record to anyone, and the route answers the one match or none, without paging. It is rate limited like a login (see `security-rate-limiting`).
- Ranges are nested objects (`{ min, max }`, `{ from, to }`); nested shapes used only by this DTO stay in its file.
- Sort fields are whitelisted with `@IsIn`. The service maps them to columns and never interpolates a client string into SQL.

```typescript
// order/constants/order-sort-fields.constant.ts
export const ORDER_SORT_FIELDS = ['createdAt', 'totalCents', 'status'] as const;

// order/types/order-sort-field.type.ts
export type OrderSortField = (typeof ORDER_SORT_FIELDS)[number];

// order/dtos/search-orders.dto.ts
export class OrderTotalRangeDto {
  @ApiPropertyOptional({ description: 'Minimum order total in cents, inclusive', example: 1000 })
  @IsOptional()
  @IsInt()
  @Min(0)
  @Max(100_000_000)
  min?: number;

  @ApiPropertyOptional({ description: 'Maximum order total in cents, inclusive', example: 50000 })
  @IsOptional()
  @IsInt()
  @Min(0)
  @Max(100_000_000)
  max?: number;
}

export class OrderDateRangeDto {
  @ApiPropertyOptional({ description: 'Start of the period, inclusive (ISO 8601)', example: '2026-01-15T10:30:00.000Z' })
  @IsOptional()
  @Type(() => Date)
  @IsDate()
  @IsStorableDate()
  from?: Date;

  @ApiPropertyOptional({ description: 'End of the period, exclusive (ISO 8601)', example: '2026-02-15T10:30:00.000Z' })
  @IsOptional()
  @Type(() => Date)
  @IsDate()
  @IsStorableDate()
  to?: Date;
}

export class OrderSortDto {
  @ApiProperty({ description: 'Field to sort by', example: 'createdAt' })
  @IsIn(ORDER_SORT_FIELDS)
  field: OrderSortField;

  @ApiProperty({ description: 'Sort direction', example: 'desc' })
  @IsIn(['asc', 'desc'])
  direction: 'asc' | 'desc';
}

export class SearchOrdersDto extends ApiPaginationQueryDto {
  @ApiPropertyOptional({ description: 'Match orders in any of these statuses', example: ['paid', 'shipped'] })
  @IsOptional()
  @IsArray()
  @ArrayMaxSize(10)
  @IsEnum(OrderStatus, { each: true })
  statuses?: OrderStatus[];

  @ApiPropertyOptional({ description: 'Match orders containing any of these SKUs', example: ['SKU-001', 'SKU-002'] })
  @IsOptional()
  @IsArray()
  @ArrayMaxSize(50)
  @IsString({ each: true })
  @MaxLength(64, { each: true })
  skus?: string[];

  @ApiPropertyOptional({ description: 'Order total range', type: OrderTotalRangeDto })
  @IsOptional()
  @IsObject()
  @ValidateNested()
  @Type(() => OrderTotalRangeDto)
  totalCents?: OrderTotalRangeDto;

  @ApiPropertyOptional({ description: 'Creation date range', type: OrderDateRangeDto })
  @IsOptional()
  @IsObject()
  @ValidateNested()
  @Type(() => OrderDateRangeDto)
  createdAt?: OrderDateRangeDto;

  @ApiPropertyOptional({ description: 'Sort order, applied in sequence', type: [OrderSortDto] })
  @IsOptional()
  @IsArray()
  @ArrayMaxSize(3)
  @ValidateNested({ each: true })
  @Type(() => OrderSortDto)
  sort?: OrderSortDto[];
}

// order/services/order.service.ts
const ORDER_SORT_COLUMNS: Record<OrderSortField, string> = {
  createdAt: 'o.createdAt',
  totalCents: 'o.totalCents',
  status: 'o.status',
};

async search(authUser: JwtUser, dto: SearchOrdersDto): Promise<PaginatedResult<Order>> {
  const query = this.orderRepository
    .createQueryBuilder('o')
    .where('o.userId = :userId', { userId: authUser.id });

  if (dto.statuses?.length) query.andWhere('o.status IN (:...statuses)', { statuses: dto.statuses });
  if (dto.skus?.length) query.innerJoin('o.items', 'item', 'item.sku IN (:...skus)', { skus: dto.skus });
  if (dto.totalCents?.min !== undefined) query.andWhere('o.totalCents >= :min', { min: dto.totalCents.min });
  if (dto.totalCents?.max !== undefined) query.andWhere('o.totalCents <= :max', { max: dto.totalCents.max });
  if (dto.createdAt?.from) query.andWhere('o.createdAt >= :from', { from: dto.createdAt.from });
  if (dto.createdAt?.to) query.andWhere('o.createdAt < :to', { to: dto.createdAt.to });

  for (const { field, direction } of dto.sort ?? [{ field: 'createdAt', direction: 'desc' }]) {
    query.addOrderBy(ORDER_SORT_COLUMNS[field], direction === 'asc' ? 'ASC' : 'DESC');
  }

  const [result, totalCount] = await query.skip(dto.offset).take(dto.limit).getManyAndCount();
  return { result, totalCount, limit: dto.limit, offset: dto.offset };
}
```

`@IsStorableDate()`, a validator in `api/validators/`, accepts dates from 1970 to 9999: PostgreSQL stores them, and a query can add to them. `ORDER_SORT_FIELDS` is in the module's `constants/` and `OrderSortField` in its `types/` (see `arch-feature-modules`), since the DTO and the service both use them. In the service, the criteria become parameterized conditions, and each sort field, already whitelisted by the DTO, is looked up in `ORDER_SORT_COLUMNS`, so the SQL holds only the service's own column names.

Reference: [NestJS Validation](https://docs.nestjs.com/techniques/validation)
