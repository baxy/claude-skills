---
title: Keep Data Access in Services Through Injected Repositories
impact: HIGH
impactDescription: One layer owns each query, and route tests exercise the filters that scope it
tags: architecture, repository, data-access, typeorm
---

## Keep Data Access in Services Through Injected Repositories

A service reads and writes its module's entities through TypeORM's repository, injected with `@InjectRepository(Entity)`. Queries, including query builders, live in the service method that needs them. Do not add custom repository classes (`OrderRepository` wrapping `Repository<Order>`) between the service and TypeORM.

Why:

- **The filters that matter stay testable.** In the e2e tier the repository is mocked at TypeORM's token (`getRepositoryToken(Order)`, see `test-use-testing-module`), so the service's own code still runs: a dropped ownership condition shows up in what reached the repository. Behind a custom repository the whole query is replaced by a canned return value, and a missing `WHERE` passes every test.
- **One layer, one place to look.** A custom repository splits each feature across two classes whose boundary nobody agrees on, and the service ends up as a pass-through.
- **Real SQL is proven in the database tier** (see `test-database-tier`), which neither layout gets for free.

Rules:

- Controllers never inject repositories; they call services (see `api-controller-conventions`).
- A service injects repositories only for its own module's entities (`TypeOrmModule.forFeature([...])` in its module). Data owned by another module is read through that module's exported service.
- Name the property for the entity: `@InjectRepository(Order) private readonly orderRepository: Repository<Order>`.
- A query that grows large moves into a `private _build...Query()` method of the same service. When a service grows past one concern, split the service (see `arch-single-responsibility`), not the data access.
- Writes across several entities use `dataSource.transaction()` in the service (see `db-use-transactions`). When the transaction must include another module's rows, that module's service exposes a method taking the `EntityManager`, so the write joins the caller's transaction without the caller touching the other module's entities.

**Incorrect (custom repository class, or another module's repository):**

```typescript
// order/repositories/order.repository.ts - an extra layer the service only forwards to
@Injectable()
export class OrderRepository {
  constructor(@InjectRepository(Order) private readonly repository: Repository<Order>) {}

  findPageForUser(userId: string, query: FindAllOrdersQueryDto): Promise<[Order[], number]> {
    return this.repository.findAndCount({ where: { userId }, skip: query.offset, take: query.limit });
  }
}

@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    @InjectRepository(User) private readonly userRepository: Repository<User>, // Another module's entity
  ) {}
}

// e2e test: the mock replaces the whole query, so the userId filter is never checked
{ provide: OrderRepository, useValue: { findPageForUser: jest.fn().mockResolvedValue([[order], 1]) } }
```

**Correct (repository injected into the service, query in the service):**

```typescript
// order/order.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Order]), UserModule],
  controllers: [OrderController],
  providers: [OrderService],
})
export class OrderModule {}

// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
    private readonly userService: UserService,
  ) {}

  async findAll(authUser: JwtUser, query: FindAllOrdersQueryDto): Promise<PaginatedResult<Order>> {
    const qb = this.orderRepository
      .createQueryBuilder('order')
      .where('order.userId = :userId', { userId: authUser.id });

    if (query.status) {
      qb.andWhere('order.status = :status', { status: query.status });
    }

    const [result, totalCount] = await qb
      .orderBy('order.createdAt', 'DESC')
      .addOrderBy('order.id', 'DESC')
      .skip(query.offset)
      .take(query.limit)
      .getManyAndCount();

    return { result, totalCount, limit: query.limit, offset: query.offset };
  }
}
```

`UserService` is how the order module reads users. The e2e test mocks TypeORM's token, so the service's filters still run:

```typescript
{ provide: getRepositoryToken(Order), useValue: orderRepository }
```

### With Prisma

A service queries through `PrismaService` (see `db-prisma-schema-conventions`), injected as `private readonly prisma: PrismaService`, using the delegate of each model its module owns: `this.prisma.order.findMany(...)`. The rules above hold, with these changes:

- The module imports `PrismaModule` instead of `TypeOrmModule.forFeature([...])`. Every delegate is reachable from every service, so nothing stops `this.prisma.user` in the order module: a service uses only its own module's delegates, and another module's data goes through that module's service.
- Never create a `PrismaClient` yourself, in a service or per request: each one opens its own connection pool.
- No repository classes wrapping `this.prisma.order` either. The e2e tier mocks `PrismaService` (see `test-use-testing-module`), so the `where` and `data` the service builds are what the test checks.
- A filter that grows large moves into a `private _build...Where()` method returning `Prisma.OrderWhereInput`. An optional condition is `Prisma.skip` when its value is missing, never `undefined`, which `strictUndefinedChecks` refuses (see `db-prisma-schema-conventions`).
- Writes across several models use `this.prisma.$transaction()`; another module joins through a method taking `tx: Prisma.TransactionClient` (see `db-prisma-use-transactions`).
- Raw SQL uses the tagged templates `$queryRaw` and `$executeRaw`, which send values as parameters. Never `$queryRawUnsafe` or `$executeRawUnsafe` with interpolated input.

```typescript
// order/order.module.ts
@Module({
  imports: [PrismaModule, UserModule],
  controllers: [OrderController],
  providers: [OrderService],
})
export class OrderModule {}

// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly userService: UserService,
  ) {}

  async findAll(authUser: JwtUser, query: FindAllOrdersQueryDto): Promise<PaginatedResult<OrderResponseDto>> {
    const where: Prisma.OrderWhereInput = { userId: authUser.id, deletedAt: null, status: query.status ?? Prisma.skip };

    const [orders, totalCount] = await Promise.all([
      this.prisma.order.findMany({ where, orderBy: [{ createdAt: 'desc' }, { id: 'desc' }], skip: query.offset, take: query.limit }),
      this.prisma.order.count({ where }),
    ]);

    return {
      result: orders.map((order) => plainToInstance(OrderResponseDto, order, { excludeExtraneousValues: true })),
      totalCount,
      limit: query.limit,
      offset: query.offset,
    };
  }
}
```

The page and its total are two plain queries, since a sequential `$transaction` gives them no common snapshot (see `perf-optimize-database`). The items are converted to the response DTO in the service (see `api-use-dto-serialization`). The e2e test mocks `PrismaService`, so the service's filters still run:

```typescript
{ provide: PrismaService, useValue: prisma }
```

Reference: [NestJS Database](https://docs.nestjs.com/techniques/database#repository-pattern)
