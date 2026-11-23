---
name: nestjs-developer
description: NestJS coding conventions and best practices covering modules and folder layout, naming, dependency injection, REST controllers, DTOs and validation, TypeORM entities or Prisma schemas and migrations on PostgreSQL, Swagger/OpenAPI, error handling, security, testing tiers, logging and code style. Use when writing, generating or refactoring NestJS code (modules, controllers, services, DTOs, guards, interceptors, entities, Prisma models, migrations, tests, main.ts), or when working in a project that depends on @nestjs packages. For reviewing existing code or pull requests, use nestjs-reviewer.
metadata:
  version: "1.0.0"
---

# NestJS Developer

Conventions for building NestJS applications. Each rule lives in `rules/<rule-id>.md` with the reasoning, an incorrect example and a correct example. Where a rule differs from generic NestJS advice or from the NestJS docs' defaults, follow the rule.

## Assumed stack

The rules are written for NestJS 12 on Fastify (new projects, unless asked for another adapter), or Express or the Node adapter `@nestjs/platform-node` (a project already on one), TypeORM or Prisma 7 on PostgreSQL, `class-validator`/`class-transformer`, `@nestjs/swagger`, Vitest or Jest with `supertest`, and `nestjs-pino`. When the project uses a different library for one of these (Mongoose, Drizzle, Zod...), keep the project's library and apply the intent of the rule. Outside the `db-` rules the examples use TypeORM; a rule where Prisma behaves differently ends with a "With Prisma" section, and the `db-prisma-` rules give the client and query forms.

The examples are CommonJS-style imports without file extensions and use Jest's mock API (`jest.fn()`); `vi.fn()` and most of Vitest's API keep Jest's names, and `references/test-runner-vitest.md` lists the forms that differ. `references/` holds what depends on the project's setup:

- `references/project-setup.md`: how to read an existing project's setup, how to start a new one, and what changes in an ESM project (`.js` import extensions, `Relation<>` on entities, the TypeORM CLI), and how to set up Prisma 7.
- `references/test-runner-vitest.md` and `references/test-runner-jest.md`: tier configs, scripts and the Jest-to-Vitest mapping.

## How to use

1. Check the project's `package.json` first: `"type": "module"` means ESM, the test runner is whichever of `vitest` or `jest` it lists, the ORM is whichever of `typeorm` or `prisma` it lists (a new project uses TypeORM unless asked for Prisma), and the HTTP adapter is whichever of `@nestjs/platform-fastify`, `@nestjs/platform-express` or `@nestjs/platform-node` it lists (a new project uses Fastify unless asked for another). Keep all four as they are; for an ESM project, read `references/project-setup.md` before writing code. When no `package.json` is available, follow what the existing files show (`.js` import extensions, `vi.` or `jest.`); with no files at all it is a new project (ESM with Vitest). State the assumption in your answer.
2. Find the task in the table below and **read every rule file listed for it before writing code**. The one-line summaries in the index are reminders, not the rule.
3. The `style-` rules apply to every file, comment, message and doc you write. Read `style-code-comments` before writing code: a correct example keeps only comments that say why, so copy one only where the same reason holds in your code, and a line that only names a file shows where the code goes.
4. Before reporting a change done, check it against the update list in `docs-keep-readmes-current`, and update the README in the same change when it applies. Re-read every comment you added against `style-code-comments`, and every integration spec against `test-integration-suites`: these are the rules most often missed.
5. A business choice the task leaves open (what counts as revenue, a limit, what happens at a boundary) is not yours to make in code: ask before building it. When no one can answer in time, take the choice cheapest to change later, write it into the spec as an assumption and name it first in your report (`docs-specs-plans-and-notes`).
6. Each example focuses on one rule and may leave out unrelated decorators (Swagger, guards) for brevity. Apply all relevant rules together.

| Task | Rules to read |
|---|---|
| New project | `references/project-setup.md` (`nest new`, ESM with Vitest), `references/test-runner-vitest.md`, `test-organize-by-tier`, `test-use-testing-module` (`ApiModule.setup()`), `devops-use-config-module`, `arch-import-boundaries`, `error-report-to-sentry` (`src/sentry/preload.ts`, the start scripts), `docs-keep-readmes-current` (its README) |
| New feature module | `arch-feature-modules`, `arch-resource-naming`, `arch-shared-folder`, `arch-import-boundaries`, `arch-module-sharing`, `arch-avoid-circular-deps`, `arch-service-conventions`, `arch-data-access-in-services` |
| New or changed service or provider | `arch-service-conventions`, `arch-single-responsibility`, `arch-data-access-in-services`, `di-prefer-constructor-injection`, `di-scope-awareness`, `error-throw-http-exceptions` |
| Swappable implementation (interface, token, test double) or a call to an outside system | `arch-feature-modules` (`clients/`, `adapters/`), `micro-use-http-clients`, `di-use-interfaces-tokens`, `di-interface-segregation`, `di-liskov-substitution`, `di-avoid-service-locator` |
| New or changed endpoint | `api-controller-conventions`, `api-dto-naming`, `api-use-pipes`, `security-validate-all-input`, `api-document-with-openapi`, `api-use-dto-serialization`, `security-use-guards` (`@Public()`/`@Roles()`), `api-custom-param-decorators`, `error-throw-http-exceptions`, `test-use-testing-module` (its e2e cases) |
| Update or derived DTO | `api-use-mapped-types`, `api-dto-naming` |
| List endpoint or search | `perf-optimize-database` (pagination), `api-dto-naming` (`ApiPaginationQueryDto`, search body), `db-avoid-n-plus-one` with `db-typeorm-avoid-n-plus-one` or `db-prisma-avoid-n-plus-one` |
| File upload | `api-file-uploads` |
| Entity or schema change | `arch-resource-naming`, `db-schema-conventions` with `db-typeorm-schema-conventions` or `db-prisma-schema-conventions`, `db-use-migrations` with `db-typeorm-use-migrations` or `db-prisma-use-migrations`, `test-database-tier` |
| Multi-step write, or a change that depends on the current state (a status, a balance, a limit) | `db-use-transactions` with `db-typeorm-use-transactions` or `db-prisma-use-transactions` |
| Authentication or access control | `security-auth-jwt`, `security-hash-passwords`, `security-use-guards`, `api-custom-param-decorators`, `security-rate-limiting` |
| Errors | `error-throw-http-exceptions`, `error-use-exception-filters`, `error-handle-async-errors`, `error-report-to-sentry` |
| Response shape | `api-use-interceptors` (envelope), `api-use-dto-serialization` |
| Rich text input or HTML output | `security-sanitize-output` |
| Breaking API change | `api-versioning`, `api-use-dto-serialization` |
| Caching | `perf-use-caching` |
| Real-time updates (SSE, WebSockets) | `api-real-time-updates`, `arch-use-events`, `devops-graceful-shutdown` |
| Writing a spec or a plan, or keeping notes | `docs-specs-plans-and-notes` |
| Tests | `test-organize-by-tier`, `test-use-testing-module`, `test-guards-and-interceptors`, `test-mock-external-services`; `test-database-tier` and `test-integration-suites` for those tiers; the runner's file in `references/` for config and scripts; `devops-use-config-module` (a helper that connects to the database, Redis or S3) |
| Command-line command (an operator's task with no route) | `arch-feature-modules` (the `cli` module), `devops-use-config-module`, `error-report-to-sentry`, `test-use-testing-module`, `test-integration-suites` |
| Events, queues, background work | `arch-use-events`, `micro-use-queues`, `micro-use-patterns`, `error-handle-async-errors`, `arch-feature-modules` (the `scheduler/` module), `error-report-to-sentry` (job errors) |
| `main.ts` and bootstrap | `api-platform-agnostic`, `devops-use-config-module`, `devops-use-logging`, `devops-graceful-shutdown`, `test-use-testing-module` (shared `ApiModule.setup()`), `error-report-to-sentry` (`node --import ./dist/sentry/preload.js`, `--require` in CommonJS) |
| Lifecycle hooks, startup or shutdown work | `perf-async-hooks`, `devops-graceful-shutdown`, `perf-lazy-loading` |
| Health checks and deployment | `micro-use-health-checks`, `devops-graceful-shutdown` |
| New setting or environment variable | `devops-use-config-module` |
| Configurable infrastructure module | `arch-use-dynamic-modules`, `devops-use-config-module` |

## Rule index

The sections are ordered by impact; the prefix is the start of each rule ID.

### 1. Architecture (CRITICAL) - `arch-`

- `arch-feature-modules` - Feature modules and type folders (`types/` for type aliases, `interfaces/` for contracts, `constants/`, `clients/`, `adapters/`, `workers/`, `queues/`); pure `utils/`; one `scheduler/` module with `@Interval` jobs; a `cli` module for a command line, operator errors as an exit code
- `arch-resource-naming` - Singular resource names; every class, type, constant, file and table in a module, resource or capability, and each property injecting one of its classes, starts with the module name; functions, decorators, constants local to one file, clients and adapters of an outside system (named for the system) and test doubles keep their own names, and a param DTO of a route nested under another module's resource is named for the route, parent first
- `arch-shared-folder` - `src/shared/` holds domain-neutral files used by two or more modules, in type folders; no services; imports no module and no ORM
- `arch-import-boundaries` - Relative imports inside a module, `@/` alias across modules
- `arch-module-sharing` - Proper module exports/imports, avoid duplicate providers
- `arch-avoid-circular-deps` - Avoid circular module dependencies
- `arch-single-responsibility` - Focused services over "god services"
- `arch-service-conventions` - Singular services, injection and parameter order, domain validation
- `arch-data-access-in-services` - `@InjectRepository` in services, queries in the service, no custom repository classes
- `arch-use-events` - Event-driven decoupling; `{object}.{action}` event names
- `arch-use-dynamic-modules` - Configurable modules with `ConfigurableModuleBuilder`, the options token in `constants/`, no module-definition file

### 2. Dependency Injection (CRITICAL) - `di-`

- `di-prefer-constructor-injection` - Constructor over property injection
- `di-scope-awareness` - Singleton, request and transient scopes
- `di-use-interfaces-tokens` - Injection tokens for interfaces
- `di-avoid-service-locator` - No `ModuleRef.get()` service locator
- `di-interface-segregation` - Small, role-based interfaces
- `di-liskov-substitution` - Implementations honor their interface contract

### 3. Error Handling (HIGH) - `error-`

- `error-throw-http-exceptions` - Services throw typed HTTP exceptions; `errorCode` where a client branches, and on every domain 400; `retryAfterSeconds` for a known retry delay
- `error-use-exception-filters` - One platform-agnostic global filter with the error envelope; an `errorCode` on every error; `Retry-After`; errors logged as pino's `err` field
- `error-handle-async-errors` - No unhandled rejections from background work
- `error-report-to-sentry` - Sentry through `src/sentry/`, started with `node --import ./dist/sentry/preload.js` (`--require` in CommonJS); errors only, each event rebuilt from an allowlist, secret values redacted in events and log lines

### 4. Security (HIGH) - `security-`

- `security-validate-all-input` - Global `ValidationPipe`, `class-validator` DTOs, nested objects
- `security-auth-jwt` - Passport `AuthJwtStrategy` building a complete `JwtUser`, the algorithm pinned; with refresh tokens of its own, one session row that alone decides, and an exchange that rechecks the user
- `security-hash-passwords` - Argon2id through Node's built-in `crypto.argon2()` (the `argon2` package or bcrypt only where already used) in one `UserPasswordHasherService`, uniform login failures, one normalized email form, a malformed hash as a 401
- `security-use-guards` - Global `AuthJwtGuard` + `AuthRolesGuard`, `@Public()`, `@Roles()`; both skip a context that is not HTTP
- `security-sanitize-output` - Sanitize user HTML, never reflect unvalidated input
- `security-rate-limiting` - Rate limiting with `@nestjs/throttler`: the library's tracker (IPv6 by /64), `Retry-After` from the guard's 429, counters in memory, or in Redis with a fallback to memory when the spec requires limits across instances

### 5. Performance (HIGH) - `perf-`

- `perf-optimize-database` - Lean queries, indexes, a text search's index for its operator; offset pagination with `{ result, totalCount, limit, offset }`; a public search's cost and total capped
- `perf-use-caching` - Caching with invalidation
- `perf-async-hooks` - Async lifecycle hooks
- `perf-lazy-loading` - Lazy-load rarely used modules

### 6. Testing (MEDIUM-HIGH) - `test-`

- `test-organize-by-tier` - Unit, e2e, database and integration tiers under `test/`, mirroring `src/`; helpers in `test/helpers/` or the integration tier's own, never in `src/`; coverage excludes only `main.ts` and generated code
- `test-use-testing-module` - Test services through their routes: real service, mocked repositories
- `test-guards-and-interceptors` - Drive guards and interceptors through probe routes
- `test-database-tier` - Constraints and migrations against a real database of the tier's own, rolled back per case
- `test-integration-suites` - Order-independent integration specs against a live API
- `test-mock-external-services` - Mock external dependencies

### 7. Database & ORM (MEDIUM-HIGH) - `db-`

Each topic is an ORM-neutral rule, which says what to do and why, plus one form per ORM, `db-typeorm-{topic}` and `db-prisma-{topic}`, with the APIs and the examples. Read the neutral rule and the form for the project's ORM.

- `db-schema-conventions` - A persistence-only schema model; singular model and plural table; snake_case names; `timestamptz`; soft deletes; explicitly named indexes and constraints
- `db-typeorm-schema-conventions` - Persistence decorators only, plus `@Exclude()` + `@ApiHideProperty()`; required `SnakeNamingStrategy` on every DataSource; `IDX_`, `UQ_` and `CHK_` names; a migration DataSource that checks the certificate
- `db-use-migrations` - No automatic schema sync; structural migrations generated from the model; verify no drift; never edit an applied migration; expand and contract for destructive changes; run as a deploy step
- `db-typeorm-use-migrations` - `synchronize` hardcoded to `false`; `migration:generate`, and `migration:create` for data only; a batched backfill with `transaction = false`; filenames; `search_path`; never `migrationsRun`
- `db-use-transactions` - Transactions for multi-step writes, joined through the other module's service; state changes as conditional updates; one lock order across tables, conflicts retried and then 409; a row lock or an advisory lock for read-then-write limits, `SERIALIZABLE` for checks no key names; no external calls inside
- `db-typeorm-use-transactions` - `dataSource.transaction()`, `manager: EntityManager`, conditional updates checked through `affected`, the `database` module's `DatabaseTransactionService` (`retried()`, `serializable()`) and `isTransactionConflict()`
- `db-avoid-n-plus-one` - Related rows in one query or one batch, never a query per row; joins inside one module, another module's rows in one call to its service; DataLoader for GraphQL fields
- `db-typeorm-avoid-n-plus-one` - `relations`, query builder joins and `select`; no lazy relations in lists
- `db-prisma-schema-conventions` - One `.prisma` file per owning module; `@@map`/`@map` to snake_case; `@db.Timestamptz(3)`, the database's clock and `Timestamptz(6)` for times SQL compares with `now()`; LIKE wildcards escaped in `contains` and `startsWith`; enum values as stored; soft deletes filtered in every query; `IDX_`/`UQ_` names through `map:`; `PRISMA_OMIT`; `PrismaService` on the pg adapter with a UTC session, a pool-wide `lock_timeout` and transaction limits from settings; `partialIndexes` and `strictUndefinedChecks`; a CLI that checks the certificate
- `db-prisma-use-migrations` - `migrate dev --create-only`, review, `migrate dev`, `generate`; `migrate diff --exit-code` as the drift check; never `db push`; CHECK constraints and an index's extension by hand; fix forward, a local mistake too; a batched backfill alone in its file; a checked baseline for an existing database; `migrate deploy` as the deploy step, from its own job
- `db-prisma-use-transactions` - Interactive `$transaction` with `tx: Prisma.TransactionClient`; `updateMany` checked through `count`; every lock wait bounded by the pool's `lock_timeout`; `PrismaTransactionService` (`retried()`, `serializable()`) and `isTransactionConflict()` by PostgreSQL's code; a page and its total as two plain queries, or a `RepeatableRead` batch when they must agree
- `db-prisma-avoid-n-plus-one` - `include` and nested `select`, one `in` query for many parents; no `relationJoins` preview

### 8. DevOps & Deployment (MEDIUM-HIGH) - `devops-`

- `devops-use-config-module` - `src/config/` folder (not a module): one `registerAs` file per concern, namespaced keys, `validate` hook checking optional numbers too; `APP_ENV` required (`dev`, `stage`, `prod`); TLS to the database and Redis outside `dev`; every variable in `.env.example`, kept in sync by a unit test
- `devops-use-logging` - Structured pino logging with correlation fields; request lines with the caller and the path, no query string; errors in the `err` field; secrets and query values redacted
- `devops-graceful-shutdown` - Zero-downtime deployments; the drain is a setting, `0` for a local run; queues and workers disconnect past their close limit

### 9. API Design (MEDIUM) - `api-`

- `api-controller-conventions` - Singular controllers, REST routes and methods, actions on an item or the collection, status codes, handlers
- `api-dto-naming` - DTO file and class naming, param inheritance, pagination query
- `api-use-pipes` - Param and query DTO classes instead of inline pipes
- `api-use-mapped-types` - Derive DTOs with `PartialType`/`PickType`/`OmitType`
- `api-use-dto-serialization` - Hidden columns on entities, response DTOs where the shape differs
- `api-use-interceptors` - Cross-cutting concerns, the response envelope and the request lifecycle order
- `api-document-with-openapi` - Described DTO fields, decorated controllers, success responses inside the envelope, example conventions
- `api-custom-param-decorators` - `@AuthUser() authUser: JwtUser` and composed decorators
- `api-platform-agnostic` - Fastify for new projects; same code on every adapter (Express, Fastify, the Node adapter): one `ApiModule.createFastifyAdapter()`, neutral request type, CORS
- `api-file-uploads` - Validated multipart uploads with `nestjs-form-data`; a bounded file name
- `api-real-time-updates` - SSE when only the server sends, an authenticated Socket.IO gateway otherwise; Redis across instances; streams and sockets ended at shutdown; global guards and interceptors skip what is not HTTP
- `api-versioning` - URI versioning only for breaking changes; deprecation headers (RFC 9745) set by a middleware, so refusals carry them

### 10. Microservices (MEDIUM) - `micro-`

- `micro-use-patterns` - Message and event patterns
- `micro-use-queues` - Background jobs with BullMQ; the record saved first, a failed enqueue marking it failed; a final failure read from `finishedOn`; job schedulers read before an upsert, compared with the queue's default options; a dashboard with its own error handling, logging, limits and command timeout
- `micro-use-http-clients` - Outgoing calls through a configured client: a timeout, the request id, failures mapped to 503 or 500, no retries in a request, customer URLs checked at every connection
- `micro-use-health-checks` - Liveness and readiness probes; the database pinged on a connection of its own; a filter on the health controller answers their 503s

### 11. Code Style (MEDIUM) - `style-`

- `style-code-comments` - No comment by default; names carry the meaning; comments only say why; stale comments are deleted
- `style-private-underscore-prefix` - `_` prefix on private methods only
- `style-no-em-dash` - Never use the em dash character
- `style-us-english` - US-English spelling everywhere

### 12. Documentation (MEDIUM) - `docs-`

- `docs-specs-plans-and-notes` - Specs and plans only in `docs/specs/` and `docs/plans/` as `{yyyy-mm-dd}-{ticket}-{name}.md` (ticket only when there is one; required on a ticket branch), paired by name; private notes in gitignored `docs/notes/{ticket}/`
- `docs-keep-readmes-current` - A README with fixed sections, updated in the same change as what it describes
