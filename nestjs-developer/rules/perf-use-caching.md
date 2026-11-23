---
title: Cache Expensive Reads with Explicit Keys and Invalidation
impact: HIGH
impactDescription: Cuts database load for hot reads without serving stale or another caller's data
tags: performance, caching, redis, cache-manager
---

## Cache Expensive Reads with Explicit Keys and Invalidation

Cache data that is expensive to compute and read far more often than it changes. Every cache entry has a key that identifies exactly what it holds, a TTL, and an invalidation path when the data changes. The key also names the shape of what it holds (`product:catalog:page:v2`): a release that changes the cached shape, a new field or a new API version, changes the key, so it never reads an entry the release before wrote, during a rolling deploy or until the TTL runs out.

- Use `@nestjs/cache-manager` with a shared store (Redis) when more than one instance runs; an in-memory cache differs per instance. An app that may run without Redis (one with no queues, see `micro-use-queues`) caches in a bounded memory store when `REDIS_HOST` is not set, and logs a warning at startup; an app with queues requires `REDIS_HOST` and has no such branch.
- A list each instance keeps in its own memory (the ten bestsellers, refreshed every few minutes) keeps only ids, and the rows are read by primary key per request: a product archived or repriced by an administrator then shows as it is now, as the rule on administrators' writes below asks.
- **The Redis store runs on the app's Redis client**, `REDIS_CLIENT` from the `redis` module (see `security-rate-limiting`): `CacheRedisStore`, a Keyv store of about 30 lines in the cache module. The cache then gets the client's 1-second command timeout, fails at once while disconnected, and adds no second connection. Do not use `@keyv/redis`: it is built on node-redis, whose command timeout does not bound a Redis that accepts the connection and never answers (a paused Redis, a proxy whose backend is gone), so every cached read waits without limit.
- **A failure reaches the code without failing the request.** `cache-manager` wraps a bare store in a Keyv of its own, which turns a failed write into `false`. Build the Keyv yourself, with `throwOnErrors: true` and the `cache` namespace (the queues and, when the spec keeps them there, the rate-limit counters share the Redis, see `security-rate-limiting`). `cache-manager`'s `get()` still reports a failed read as a miss, so the failed write that follows is the signal. The service catches it, reads from the database, leaves the cache alone for a while (30 s) and logs one warning. Test that path with the real store against a closed port and a peer that never answers, not with a mock that rejects.
- TTLs are in **milliseconds** (cache-manager v5 and later).
- **Caller-dependent data needs the caller in the key.** The built-in `CacheInterceptor` keys by URL only, so use it only on routes whose response is the same for every caller (public catalogs, reference data). Everything else is cached in the service, with the user id in the key.
- **A cache hit is a plain object, not an entity.** The shared store keeps JSON, so `ClassSerializerInterceptor` skips `@Exclude()` on what comes back (see `api-use-dto-serialization`) and hidden columns reach the client. In the service, rebuild instances on read with `plainToInstance(Entity, cached)`. On a route cached by `CacheInterceptor`, add `@SerializeOptions({ type: Entity })` so the serializer converts the cached value before applying `@Exclude()`. That works only when the route returns one entity or an array; a paginated route is cached in the service instead, where each item is rebuilt (see `api-use-dto-serialization`).
- Invalidate on write: delete the affected keys in the service method that changes the data, or in a listener for the change event (see `arch-use-events`). A deleted key can come back stale: a read that missed the cache, loaded the rows before the write committed and stores them after its delete keeps the old value for the whole lifetime. Data read often and changed while being read (a public list, a listing's pages) is therefore invalidated by a version instead: each key includes the current version, read before the rows are loaded, and a write stores a new one, so a late store lands under a version nobody reads any more. A version write that failed leaves the old pages reachable, so the service remembers it and writes a new version at its next read.
- **A key built from client input comes from a small, fixed set.** Every query a client can vary (a search text, a price bound, a page size, an offset) is a new entry it can create at will, kept for the whole TTL, in a Redis the queues and, when kept there, the rate-limit counters share. A full Redis then refuses their writes or evicts them. Cache only the queries most requests send (the first pages of each order, at the default page size, without search or filters), say which in the API docs, and let every other query read the database, where indexes serve it. The number of entries is then a figure you can read off the code (3 orders × 5 pages).
- **A write any caller can make does not invalidate a cache every caller shares.** If a review, a like or a vote empties the cached catalog, one customer repeating the write keeps the cache empty for everyone, and every request reaches the database. Data such writes change (the ratings in a product list) may lag by the cache's lifetime there: say so in the API docs, and serve it current where it is read uncached (the product page). Writes by administrators (a price, a publication) still invalidate at once.
- Interceptors only run on controllers and handlers; `@UseInterceptors(CacheInterceptor)` on a service method does nothing.

**Incorrect (caller-blind interceptor, static key, no invalidation):**

```typescript
@Controller('orders')
@UseInterceptors(CacheInterceptor) // Keyed by URL: user B gets user A's cached GET /orders
export class OrderController {
  @Get(':id')
  @CacheKey('order')                // One key for every id: every GET /orders/:id returns the same order
  @CacheTTL(3_600_000)
  findById(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto): Promise<Order> {
    return this.orderService.findById(authUser, params.id);
  }
}
```

**Correct (shared store, keyed entries in the service, invalidation on write):**

```typescript
// cache/stores/cache-redis.store.ts
export class CacheRedisStore extends EventEmitter implements KeyvStoreAdapter {
  readonly opts = {};
  // Keyv sets it
  namespace?: string;

  constructor(private readonly redis: Redis) {
    super();
  }

  async get<Value>(key: string): Promise<StoredData<Value> | undefined> {
    return ((await this.redis.get(key)) ?? undefined) as StoredData<Value> | undefined;
  }

  async set(key: string, value: string, ttlMs?: number): Promise<void> {
    await (ttlMs ? this.redis.set(key, value, 'PX', ttlMs) : this.redis.set(key, value));
  }

  async delete(key: string): Promise<boolean> {
    return (await this.redis.del(key)) > 0;
  }

  async clear(): Promise<void> {
    // The Redis holds more than the cache: without a namespace, clear() would delete the rest too
    if (!this.namespace) {
      throw new Error('A cache without a namespace cannot be cleared');
    }
    for await (const keys of this.redis.scanStream({ match: `${this.namespace}:*`, count: 100 }) as AsyncIterable<string[]>) {
      if (keys.length > 0) {
        await this.redis.unlink(...keys);
      }
    }
  }
}

// cache/cache.module.ts
@Module({
  imports: [
    CacheManagerModule.registerAsync({
      isGlobal: true,
      imports: [RedisModule],
      inject: [REDIS_CLIENT],
      useFactory: (redis: Redis | null): CacheOptions => {
        if (!redis) {
          new Logger('Cache').warn('REDIS_HOST is not set: each instance caches on its own');
          return { stores: new Keyv({ store: new CacheMemoryStore(CACHE_MEMORY_MAX_ENTRIES) }) };
        }
        // Our own Keyv: the one cache-manager would build turns a failed write into `false`
        return { stores: new Keyv({ store: new CacheRedisStore(redis), namespace: 'cache', throwOnErrors: true }) };
      },
    }),
  ],
})
export class CacheModule {}

// product/services/product.service.ts
// v1 names the cached shape: a release that changes it moves to v2 and never reads what v1 wrote
const POPULAR_KEY_PREFIX = 'product:popular:v1';
const POPULAR_VERSION_KEY = `${POPULAR_KEY_PREFIX}:version`;

@Injectable()
export class ProductService {
  private readonly logger = new Logger(ProductService.name);
  private popularVersionStale = false;

  constructor(
    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>,
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
  ) {}

  async findPopular(): Promise<Product[]> {
    // Read before the rows: a store that lands after a write's new version goes under a version nobody reads
    const key = `${POPULAR_KEY_PREFIX}:${await this._popularVersion()}`;
    const cached = await this.cache.get<Product[]>(key);
    if (cached) {
      return plainToInstance(Product, cached);
    }
    const products = await this._queryPopular();
    try {
      await this.cache.set(key, products, 5 * 60_000);
    } catch (error) {
      this.logger.warn({ err: error }, 'The cache failed');
    }
    return products;
  }

  async update(id: string, dto: UpdateProductDto): Promise<Product> {
    const { affected } = await this.productRepository.update({ id, deletedAt: IsNull() }, dto);
    if (!affected) {
      throw new NotFoundException(`Product #${id} not found`);
    }
    await this._writePopularVersion();
    return this.findById(id);
  }

  private async _popularVersion(): Promise<string> {
    if (this.popularVersionStale) {
      await this._writePopularVersion();
    }
    return (await this.cache.get<string>(POPULAR_VERSION_KEY)) ?? 'initial';
  }

  // Never throws: the product is already updated, and a failed write is retried at the next read
  private async _writePopularVersion(): Promise<void> {
    try {
      await this.cache.set(POPULAR_VERSION_KEY, randomUUID());
      this.popularVersionStale = false;
    } catch (error) {
      this.popularVersionStale = true;
      this.logger.warn({ err: error }, 'The cache failed');
    }
  }

  private _queryPopular(): Promise<Product[]> {
    return this.productRepository
      .createQueryBuilder('product')
      .orderBy('product.orderCount', 'DESC')
      .take(20)
      .getMany();
  }
}

// category/controllers/category.controller.ts
@UseInterceptors(CacheInterceptor)
@Controller('categories')
export class CategoryController {
  @Public()
  @CacheTTL(30 * 60_000)
  @SerializeOptions({ type: Category })
  @Get(':id')
  findById(@Param() params: CategoryIdParamDto): Promise<Category> {
    return this.categoryService.findById(params.id);
  }
}
```

`CacheModule` is imported by `AppModule`. `ProductService` caches data that is the same for every caller, under explicit keys. A failed read comes back as a miss, and a hit is plain JSON from Redis, rebuilt into instances so `@Exclude()` applies. Its `catch` blocks are shortened here: they also skip the cache for 30 seconds and warn once, as described above. `update()` is one statement on a live product checked through `affected` (see `db-typeorm-use-transactions`). The popular list is public and read while it changes, so `update()` writes a new version instead of deleting a key: a read that loaded the rows before the write stores them under the old version, which nobody reads any more. A failed version write never fails `update()`, whose change is already stored: it marks the version stale, and the next read writes it first. `CategoryController` returns the same response to every caller, so `CacheInterceptor` is fine there, and `@SerializeOptions()` converts its cache hits, plain objects, so `@Exclude()` still applies. The lifetimes are examples: 5 minutes for the popular products, and 30 minutes for categories, which rarely change.

### With Prisma

- **A cache hit needs no rebuilding into an entity.** Prisma's results are plain objects already, and what keeps a field private is the route's response DTO, converted with `excludeExtraneousValues: true` (see `api-use-dto-serialization`). The conversion applies to a hit as it does to a fresh result: `plainToInstance(ProductResponseDto, item, { excludeExtraneousValues: true })` per item in the service, and `@SerializeOptions({ type: ProductResponseDto, excludeExtraneousValues: true })` on a `CacheInterceptor` route.
- **Cache only results with `PRISMA_OMIT` applied.** A query that opts back into an omitted field (`omit: { passwordHash: false }`) is never cached: the store would keep the field.
- The write is the plain update of `db-prisma-use-transactions`, with `deletedAt: null` in its `where`, followed by the new version as above.

Reference: [NestJS Caching](https://docs.nestjs.com/techniques/caching)
