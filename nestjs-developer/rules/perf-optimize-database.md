---
title: Optimize Database Queries
impact: HIGH
impactDescription: Database queries are typically the largest source of latency
tags: performance, database, queries, optimization
---

## Optimize Database Queries

Select only needed columns, use proper indexes, avoid over-fetching relations, and consider query performance when designing your data access. Most API slowness traces back to inefficient database queries.

PostgreSQL creates no index for a foreign key: index each one that a query filters or joins on (`order_payments.order_id`, read with every order). A partial index (`WHERE status = 'processing'`) serves only queries with that condition, so it does not replace the plain one.

**Incorrect (over-fetching data and missing indexes):**

```typescript
// Select everything when you need few fields
@Injectable()
export class UserService {
  async findAllEmails(): Promise<string[]> {
    const users = await this.userRepository.find();
    // Fetches ALL columns for ALL users
    return users.map((u) => u.email);
  }
}

@Injectable()
export class OrderService {
  async findWithItemCount(authUser: JwtUser, id: string): Promise<OrderWithItemCount> {
    const order = await this.orderRepository.findOne({
      where: { id, userId: authUser.id },
      relations: { items: true },
    });
    // Every item loaded to count the items
    return { id: order.id, status: order.status, itemCount: order.items.length };
  }
}

// No indexes on frequently queried columns
@Entity()
export class Order {
  @Column()
  userId: string; // No index - full table scan on every lookup

  @Column()
  status: string; // No index - slow status filtering
}
```

**Correct (select only needed data with proper indexes):**

Select only the columns needed, use the query builder for complex selections, and fetch only the relations needed, of the module's own entities (another module's rows come from its service, see `db-avoid-n-plus-one`):

```typescript
@Injectable()
export class UserService {
  async findAllEmails(): Promise<string[]> {
    const users = await this.userRepository.find({
      select: { email: true },
    });
    return users.map((u) => u.email);
  }
}

@Injectable()
export class OrderService {
  async findWithItemCount(authUser: JwtUser, id: string): Promise<OrderWithItemCount> {
    return this.orderRepository
      .createQueryBuilder('order')
      .select('order.id', 'id')
      .addSelect('order.status', 'status')
      .addSelect('COUNT(item.id)', 'itemCount')
      .leftJoin('order.items', 'item')
      .where('order.id = :id AND order.userId = :userId', { id, userId: authUser.id })
      .groupBy('order.id')
      .getRawOne();
  }

  async findWithItems(authUser: JwtUser, id: string): Promise<Order | null> {
    return this.orderRepository.findOne({
      where: { id, userId: authUser.id },
      relations: { items: true },
      select: {
        id: true,
        status: true,
        totalCents: true,
        items: {
          id: true,
          sku: true,
          quantity: true,
        },
      },
    });
  }
}
```

Index the columns queries filter and sort on, the deletes of a scheduled cleanup included (by expiry, by retention date): unindexed, each run reads the whole table (see `db-typeorm-schema-conventions` for the entity rules):

```typescript
@Entity('orders')
// Also serves queries on userId alone
@Index('IDX_orders_user_id_status', ['userId', 'status'])
@Index('IDX_orders_created_at', ['createdAt'])
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'uuid' })
  userId: string;

  @Column({ type: 'enum', enum: OrderStatus, enumName: 'order_status', default: OrderStatus.Pending })
  status: OrderStatus;

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date;
}
```

Always paginate list endpoints: offset-based, limit 20 and offset 0 by default (query DTOs extend `ApiPaginationQueryDto`, see `api-dto-naming`). `totalCount` is every matching row, not the page length, and `totalCountCapped` says a public search's total stopped at its cap (below). The list holds only the caller's own orders, newest first. Its order is stable because the sort ends on a unique column (`id`), so rows with the same `createdAt` keep their place from one page to the next:

```typescript
// shared/types/paginated-result.type.ts
export type PaginatedResult<T> = {
  result: T[];
  totalCount: number;
  limit: number;
  offset: number;
  totalCountCapped?: true;
};

@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
  ) {}

  async findAll(authUser: JwtUser, query: FindAllOrdersQueryDto): Promise<PaginatedResult<Order>> {
    const [result, totalCount] = await this.orderRepository.findAndCount({
      where: { userId: authUser.id },
      skip: query.offset,
      take: query.limit,
      order: { createdAt: 'DESC', id: 'DESC' },
    });

    return { result, totalCount, limit: query.limit, offset: query.offset };
  }
}
```

The response envelope interceptor (see `api-use-interceptors`) turns this into `{ success: true, statusCode: 200, data: result, meta: { totalCount, limit, offset } }`.

**A text search needs an index made for its operator.** A plain btree index on a text column, under any collation but `C` (databases are usually created with one like `en_US.utf8`), serves equality and sorting, and no `LIKE` or `ILIKE` at all, not even a prefix: such a search reads the whole table, once for the page and once for its total. Checked on PostgreSQL 18 under `en_US.utf8` with sequential scans turned off: `LIKE 'name a%'` and `ILIKE 'name a%'` still scanned the table with only the plain index; an index with `text_pattern_ops` served the prefix `LIKE`, never `ILIKE`; a trigram GIN index (`USING gin (name gin_trgm_ops)`, from the `pg_trgm` extension) served `ILIKE` with a `%` on both sides. So a prefix search on an exact value takes a `text_pattern_ops` index, and a case-insensitive or substring search a trigram index, created with its extension in a migration (see `db-prisma-use-migrations`). The client's text inside the pattern has its `\`, `%` and `_` escaped, or `%` matches every row (`escapeLike()`, see `db-prisma-schema-conventions`; TypeORM's `Like()` and `ILike()` take a pattern and escape nothing either).

**A public search bounds its own cost.** A route anyone may call, whose query matches by text, does work in proportion to what it matches: a common word matches most of the table, and ranking every match and counting them all took 1.8 seconds per request on a table of 500,000 rows in a review. The search ranks only a capped set of candidates, and its total stops at the cap: the page is ranked within the first matches the index returns, not every match, which is the price of a bounded request, and the total counts at most one row past the cap. A total past the cap reaches the client as the cap with `totalCountCapped: true` in `meta` (see `api-use-interceptors`), which the API docs explain as "more than". The cap (for example 1000) lives in the module's `constants/`, since clients see it.

A public product search, where `$1` is the pattern (`'%' || escaped text || '%'`), `$2` the cap, `$3` the text, and `$4` and `$5` the page. The first statement, the page, ranks only the first `$2` matches; the second, the total, counts at most one row past the cap:

```sql
SELECT id, name FROM (
  SELECT id, name FROM products WHERE name ILIKE $1 AND deleted_at IS NULL LIMIT $2
) AS candidates
ORDER BY similarity(name, $3) DESC, id
LIMIT $4 OFFSET $5;
SELECT count(*) AS total FROM (SELECT 1 FROM products WHERE name ILIKE $1 AND deleted_at IS NULL LIMIT $2 + 1) AS matches;
```

**Overlap checks use the range operator.** `starts_at < :end AND ends_at > :start` bounds an index on one side only and reads every past row; `tstzrange(starts_at, ends_at) && tstzrange(:start, :end)` uses the GiST index of the exclusion constraint that forbids the overlap.

**Connection pool.** TypeORM pools connections per DataSource (the `pg` driver defaults to 10). Size it with `poolSize` (or `extra: { max }`) from a setting (`DB_POOL_SIZE`, see `devops-use-config-module`) so that pool size times the number of instances, plus migration and admin connections, stays under the database's `max_connections`, and state that sum in the README, since every instance's queue workers and scheduled jobs share its pool. Behind a pooler such as PgBouncer, size against the pooler's limit instead.

### With Prisma

- Fetch only the fields a response needs with `select`, and count in the database with `_count` in a `select`: never load a relation to count it (see `db-prisma-avoid-n-plus-one`).
- Indexes are `@@index([...], map: "IDX_...")` in the schema (see `db-prisma-schema-conventions`).
- A page and its total are two plain queries, as TypeORM's `findAndCount()` runs them (the full `findAll` is in `arch-data-access-in-services`). They can disagree when a row is written between them, with either ORM. A sequential `$transaction([findMany, count])` does not prevent that under `READ COMMITTED`, where each statement reads its own snapshot, and it holds a connection for its whole length, failing with `P2028` after `maxWait` while the pool is busy, where plain queries wait their turn. When the total must match the page, run the pair with `isolationLevel: RepeatableRead` (see `db-prisma-use-transactions`).
- The pool belongs to the `@prisma/adapter-pg` adapter in `PrismaService`: size it with `max` in the adapter's pool options (`pg` defaults to 10), by the same sum as above.

Reference: [TypeORM Query Builder](https://typeorm.io/select-query-builder)
