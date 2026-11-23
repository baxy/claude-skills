# Project Setup

How to read an existing project's setup, how to start a new project, and what changes in an ESM project. Checked against NestJS 12.2 (`@nestjs/cli` 12.0.10, TypeScript 6, Vitest 4, TypeORM 1.1) in October 2026, and Prisma 7.10.1 and the `nest new` prompts of `@nestjs/cli` 12.0.15 in November 2026.

## Existing project: follow what it has

Read `package.json` before writing code or tests:

- **Module system:** `"type": "module"` means ESM; apply "ESM differences" below. No `type` field means CommonJS, and the rule examples apply as written.
- **Test runner:** `vitest` in `devDependencies` → `test-runner-vitest.md`; `jest` → `test-runner-jest.md`. Keep the runner the project has: switching runners is its own change, made only when asked.
- **NestJS version:** the `@nestjs/core` major. The "NestJS 12" notes below do not apply to 11.
- **Linter:** oxlint or ESLint, whichever the project already runs.
- **HTTP adapter:** `@nestjs/platform-fastify`, `@nestjs/platform-express` or, from NestJS 12.3, `@nestjs/platform-node`. Keep the one the project has (see `api-platform-agnostic`).
- **ORM:** `typeorm` → the `db-typeorm-` rules; `prisma` → the `db-prisma-` rules and "Prisma 7 projects" below.

## New project

Start from `nest new` with the current CLI and keep its defaults, then apply the rules on top.

1. Node: a new project runs on the current Active LTS (Node 26 in November 2026; check https://nodejs.org/en/about/previous-releases), never an older line because it happens to be installed. Pin the major in `.nvmrc` (`26`), in `engines.node` (`">=26"`) and in the image's base (`node:26-alpine`), and state it in the README. The CLI itself needs Node 22.22.3+, 24.15+ or 26+, so a machine on an older Node installs the LTS first.
2. `npx @nestjs/cli@latest new <name> --type esm --no-observe --skip-skills`: ESM, the module-system prompt's default. The project comes with Vitest, oxlint, TypeScript 6 and `module: nodenext`. The CLI (12.0.15, November 2026) asks for the name, the package manager, observability and agent skills, and without a terminal (a coding agent's shell) it answers each with its default instead of asking: yes to both of the last two.
   - `--no-observe`: observability is `@nestjs/observe`, which sends traces, logs and metrics to an outside service. The error reporting here sends errors only, by design (`error-report-to-sentry`).
   - `--skip-skills`: the agent skills are instructions for AI coding tools from `@nestjs/skills`, a second set beside these rules, as the generated `AGENTS.md` is (step 3).
   - `--skip-git`, for a project inside an existing repository, also skips the `.gitignore`: the CLI writes it only after its `git init`. Write it yourself then, with at least `/dist`, `/node_modules`, `/coverage`, `*.tsbuildinfo` and `.env` (the CLI's own list also has logs and editor folders), plus `src/prisma/generated/` with Prisma.
   It also comes on Express: switch it to Fastify, the adapter for new projects. Remove `@nestjs/platform-express` and `@types/express`, add `@nestjs/platform-fastify`, `@fastify/helmet` for the security headers and `@fastify/static`, which the Swagger UI needs on Fastify (without it the app exits right after starting). Create the adapter with `ApiModule.createFastifyAdapter()` in `api/api.module.ts` (see `api-platform-agnostic`). A project that already runs on Express keeps Express.

   **Fastify, not the Node adapter, on purpose.** NestJS 12.3 added `@nestjs/platform-node`, Nest on Node's own `node:http` with neither Express nor Fastify underneath, and its docs recommend it from 12.3 and plan it as the default of NestJS 13. New projects here still use Fastify. The Node adapter's lead is about 5% in NestJS's own benchmark (Nest controllers answering JSON, where Fastify is about 40% ahead of Express), and it disappears once a request waits on the database. Fastify has years in production, while the Node adapter shipped in late 2026 and each release since has fixed it. And some libraries do not run on it yet: `@nestjs/mercurius` needs Fastify, `@graphql-hive/nestjs` accepts only Express and Fastify, and middleware that loads `express` itself needs Express installed. Reconsider at NestJS 13, or once the adapter has gone a few releases without fixes and every library the project needs supports it. A project already on it keeps it (see `api-platform-agnostic`).
3. Lay out the tests by tier (`test-organize-by-tier`, `test-runner-vitest.md`): replace the generated `vitest.config.ts` and `vitest.config.e2e.ts` with the per-tier projects config and remove the sample `AppController`, `AppService` and their specs.
   Delete the generated `AGENTS.md`. It is instructions for AI coding tools, and it recommends what these rules replace: `@nestjs/authentication` instead of Passport and JWT (`security-auth-jwt`), Zod instead of `class-validator`, `@nestjs/resilience` retries inside a request (`micro-use-http-clients`). An agent reading both gets two conflicting sets of instructions; people have the README (`docs-keep-readmes-current`).
4. Keep the generated `tsconfig.json` and only add the `@/` alias to it (`arch-import-boundaries`). When you write the project by hand, start from the file below, which is what `nest new` generates for ESM plus `paths`: TypeScript 6 turns `strict` on by default, and without `strictPropertyInitialization: false` every entity and DTO field fails with TS2564.

   ```jsonc
   // tsconfig.json
   {
     "compilerOptions": {
       "module": "nodenext",
       "moduleResolution": "nodenext",
       "resolvePackageJsonExports": true,
       "esModuleInterop": true,
       "isolatedModules": true,
       "declaration": true,
       "removeComments": true,
       "emitDecoratorMetadata": true,
       "experimentalDecorators": true,
       "allowSyntheticDefaultImports": true,
       "target": "ES2023",
       "sourceMap": true,
       "outDir": "./dist",
       "rootDir": ".",
       "incremental": true,
       "skipLibCheck": true,
       "strict": true,
       "strictPropertyInitialization": false,
       "types": ["vitest/globals", "node"],
       "paths": { "@/*": ["./src/*"] }
     }
   }
   ```

5. Add the stack: `@nestjs/config`, `@nestjs/typeorm` with `typeorm` and `pg`, `typeorm-naming-strategies` (or, when the user asks for Prisma, the packages in "Prisma 7 projects" below), `@nestjs/swagger`, `class-validator` and `class-transformer`, `nestjs-pino` with `pino-http` (and `pino-pretty` for local runs), `nestjs-cls`, `@nestjs/terminus`, `@nestjs/throttler`, plus the auth packages from `security-auth-jwt` and `security-hash-passwords`.
6. Create the api module (`src/api/api.module.ts` with `forRoot()`, `setup()` and `createFastifyAdapter()`) and `test/helpers/create-test-app.ts` before the first feature (`test-use-testing-module`).
7. Add a `typecheck` script, `tsc --noEmit -p tsconfig.json`, and run it with `lint` before every PR. Nothing else checks the types of `test/`: `build` compiles only `src/`, Vitest strips types without checking them, and `oxlint --type-aware` reports lint rules, not compiler errors.
8. With pnpm 12, approve the dependencies' build scripts in `pnpm-workspace.yaml`, or the install fails with `ERR_PNPM_IGNORED_BUILDS`: `allowBuilds: { '@scarf/scarf': false }` for the telemetry `postinstall` that `@nestjs/swagger` pulls in through `swagger-ui-dist`. Passwords need no approval: a new project hashes with Node's built-in `crypto.argon2()` (see `security-hash-passwords`); a project that keeps the `argon2` package adds `argon2: true` for its native binary.

**Current majors** (October 2026). When you can't check the registry, use these ranges instead of guessing: a package major that doesn't exist, or one that predates NestJS 12, stops the install.

| Package | Range |
|---|---|
| `@nestjs/common`, `core`, `platform-fastify` (or `platform-express` in an Express project), `testing`, `config`, `typeorm`, `swagger`, `terminus`, `bullmq`, `axios`, `websockets`, `platform-socket.io`, `cli`, `schematics` | `^12.0.0` |
| `@fastify/helmet`, `@fastify/static` | `^13.0.0`, `^10.0.0` (the range `@nestjs/platform-fastify` 12 declares) |
| `@fastify/multipart` (Fastify projects that take uploads) | `^10.0.0`, an optional peer of `@nestjs/platform-fastify` 12, which registers it with `multipart: true` (see `api-file-uploads`) |
| `@nestjs/throttler` | `^6.7.0` |
| `nestjs-pino` (with `pino` `^10`, `pino-http` `^11`) | `^5.3.0` |
| `nestjs-cls` | `^7.0.0` |
| `typeorm` | `^1.1.0` |
| `typeorm-naming-strategies` | `^4.1.0` |
| `prisma`, `@prisma/client`, `@prisma/adapter-pg` | `7.10.1`, exact (see "Prisma 7 projects") |
| `class-validator`, `class-transformer` | `^0.15.0`, `^0.5.1` |
| `@sentry/nestjs` | one exact version of the current 11.x (see `error-report-to-sentry`) |
| `@nestjs/jwt`, `@nestjs/passport`, `@nestjs/cache-manager`, `@nestjs/event-emitter`, `@nestjs/schedule` | `^12.0.0` |
| `passport`, `passport-jwt`, and `@types/passport-jwt` (dev) | `^0.7.0`, `^4.0.1`, `^4.0.1`: `passport-jwt` 4.0.1 ships no types, so `AuthJwtStrategy` needs `@types/passport-jwt` to compile, rather than a declaration file of the app's own |
| `@fastify/cookie` (a Fastify project that reads or sets a cookie, such as the queue dashboard's session, see `micro-use-queues`) | `^11.0.0`, the major for Fastify 5 (11.1.5 in November 2026), rather than a cookie parser of the app's own |
| `csv-parse` (a project that reads uploaded CSV, see `api-file-uploads`) | `^7.0.0` (7.0.3 in November 2026), with ESM and CommonJS exports; given the upload's limits (`to_line`, `max_record_size`) |
| `argon2` | `^0.45.0`, only in a project that already uses it; a new project hashes with Node's `crypto.argon2()` |
| `bullmq` | `^6.0.0` |
| `ioredis` | `^5.0.0`: BullMQ 6 loads it but lists it only as an optional peer, so install it. Not 6: TypeORM 1.1 declares `ioredis` `^5.0.4` as an optional peer, and npm refuses the install |
| `axios` | `^1.21.0` |
| `sharp` (a project that stores images other users see, see `api-file-uploads`) | `^0.35.0`: it ships prebuilt binaries, so pnpm needs no build approval for it |
| `@aws-sdk/client-s3`, `@aws-sdk/client-sesv2` | `^3.0.0`, the same major for every `@aws-sdk/*` package; minor releases come out almost daily |
| `@nestjs/microservices` (an app that talks to other services over a transport, see `micro-use-patterns`) | `^12.0.0`, on the same version as `@nestjs/core` |
| `stripe` (projects that take card payments) | one exact version of the current major: each release sends the Stripe API version it was built for (`stripe` 23.1.0 sends `2026-10-28.endive`), so an upgrade changes the API the app talks to and is a change of its own. The README says so (see `micro-use-http-clients`) |
| `socket.io`, `@socket.io/redis-adapter` | `^4.8.0`, `^8.3.0` |
| `cache-manager` (the library itself) with `keyv` | `^7.0.0`, `^5.0.0` (the `keyv` major that `cache-manager` 7 uses; the Redis store is the app's own, not `@keyv/redis`, see `perf-use-caching`) |
| `pg`, and `@types/pg` (dev) with Prisma | `^8.0.0` |
| `pino-pretty`, `helmet` (Express projects) | `^13.0.0`, `^8.0.0` |
| `typescript`, `vitest`, `vite` | `^6.0.2`, `^4.1.2`, `^8.0.0` (as `nest new` generates) |

**Peer ranges to expect** (October 2026):

- `typeorm-naming-strategies` 4.1.0 declares TypeORM 0.2 and 0.3 only. It works on TypeORM 1.1 (`SnakeNamingStrategy` produces the same names in CommonJS and ESM), so allow it with a package-manager override (`overrides` in npm's `package.json`; `peerDependencyRules.allowedVersions` in `pnpm-workspace.yaml`, since pnpm 10 no longer reads a `pnpm` field in `package.json`) instead of staying on TypeORM 0.3.
- `@nest-lab/throttler-storage-redis` 1.2.0 (Redis storage for `@nestjs/throttler`, added only when the spec requires limits across instances, see `security-rate-limiting`) declares NestJS up to 11. Allow it with the same override.
- BullMQ 6 pulls in `msgpackr-extract`, an optional native build: deny it in `pnpm-workspace.yaml`'s build approvals (`msgpackr-extract: false`), since its prebuilt binaries install as optional dependencies.
- `nestjs-form-data` 11.0.1 (`api-file-uploads`) declares NestJS up to 11 and `class-validator` 0.14. Add it only when the project takes uploads, and check for a release that supports NestJS 12 first.

## Compiler settings (NestJS 11 and 12)

**Types in decorated signatures are imported as types, in CommonJS and ESM alike.** The `tsconfig.json` that `nest new` generates on NestJS 11 and 12 enables `isolatedModules` and `emitDecoratorMetadata`. With both on, a name that is only a type and appears in a decorated constructor parameter or property fails with TS1272 when imported as a value: `import type { ConfigType } from '@nestjs/config'`, `import { type Relation } from 'typeorm'`, interfaces. A class Nest injects by type (a service, a repository's entity) stays a value import: the metadata needs it at runtime.

## ESM differences

The rule examples are written without file extensions. In an ESM project:

- **Relative imports name the emitted file**: `import { OrderService } from '../services/order.service.js'`. The source is `.ts`; the `.js` extension is what Node resolves after the build.
- **The `@/` alias takes the extension too**: `@/payment/services/payment.service.js`. `nest build` rewrites the alias to a relative path in the output, and Vitest resolves it through `resolve.tsconfigPaths`.
- **`tsconfig.json` has no `baseUrl`**: TypeScript 6 fails the build on it (TS5101). `paths` entries are relative to the tsconfig: `"paths": { "@/*": ["./src/*"] }`.
- `import.meta.dirname` replaces `__dirname`; `createRequire(import.meta.url)` gives a `require` where one is still needed.
- `main.ts` ends with a top-level `await bootstrap();`, as generated.
- **TypeORM relations are typed with `Relation<>`** to avoid circular-import errors between entities: `user: Relation<User>;`. Import it as a type, `import { type Relation } from 'typeorm'`: with `isolatedModules` and `emitDecoratorMetadata` (both generated), a plain import of a type used in a decorated property fails with TS1272. Only relation properties take it, never plain columns.
- **The TypeORM CLI runs on a compiled copy of the migration DataSource and the entities.** Node cannot import a decorated `.ts` file, and `typeorm-ts-node-esm` needs `ts-node`, whose ESM resolver fails on every entity that imports `@nestjs/swagger` (looking for `@nestjs/common/constants.js.js`). A `tsconfig.migrations.json` builds the copy with `nest build`, which also rewrites the `@/` alias. Its build info lives in the output folder, which `nest build` deletes, so an incremental build never skips emitting after a clean. Every `package.json` command builds first, so it runs on the current entities and migrations:

  ```jsonc
  // tsconfig.migrations.json
  {
    "extends": "./tsconfig.json",
    "compilerOptions": { "outDir": "./dist-migrations", "tsBuildInfoFile": "./dist-migrations/tsconfig.migrations.tsbuildinfo" },
    "include": ["migrations/**/*.ts", "src/**/*.entity.ts"]
  }

  // package.json
  "migration:build": "nest build --path tsconfig.migrations.json",
  "typeorm": "typeorm -d dist-migrations/migrations/type-orm/config.js",
  "migration:generate": "pnpm migration:build && pnpm typeorm migration:generate",
  "migration:run": "pnpm migration:build && pnpm typeorm migration:run",
  "migration:revert": "pnpm migration:build && pnpm typeorm migration:revert",
  "migration:create": "typeorm migration:create"
  ```

  The DataSource globs `.js` files next to itself (see `db-typeorm-schema-conventions`), so the compiled copy finds the compiled entities and migrations, and the source file imported by a database-tier spec loads none (see `test-database-tier`). `migration:generate migrations/<name>` still writes the new `.ts` migration into `migrations/`. Add `/dist-migrations` to `.gitignore`. A CommonJS project does the same, with `__dirname` in place of `import.meta.dirname`.

  The root `migrations/` folder must stay out of the app's build. The `tsconfig.build.json` that `nest new` 12 generates, in CommonJS and ESM alike, sets `"rootDir": "./src"` and `"include": ["src"]`, so `nest build` still writes `dist/main.js`. The one from `nest new` 11 and earlier sets neither and only excludes `node_modules`, `test`, `dist` and the specs: `migrations/` joins the build, the output moves to `dist/src/main.js` and `dist/migrations/`, and `start:prod` (`node dist/main`) fails. In such a project, add `"migrations"` to that `exclude` and keep `"test"` there: any `.ts` file outside `src/` that the build includes moves the output the same way. Checked with `@nestjs/cli` 11.0.21 and TypeScript 5.9 on a CommonJS project: `dist/src/main.js` without the exclude, and also with `migrations` excluded but `test` not; `dist/main.js` with both excluded, and with the `nest new` 12 layout.
- **`supertest` is a default import**: `import request from 'supertest'`. `import * as request` fails with "is not a function".

## Prisma 7 projects

- **Pin Prisma exactly, all its packages on one version**: `prisma` (dev), `@prisma/client` and `@prisma/adapter-pg` at `7.10.1`, with `pg`, `dotenv` (dev) for `prisma.config.ts`, and `@types/pg` (dev), since `pg` ships no types and the database-tier helpers use its client. npm's `latest` tag is a Prisma 8 release candidate with a different CLI, and Prisma's own update notice suggests `@latest`, so always install with the version. The v7 documentation is under `prisma.io/docs/orm/v7/`.

  ```bash
  pnpm add @prisma/client@7.10.1 @prisma/adapter-pg@7.10.1 pg
  pnpm add -D prisma@7.10.1 dotenv @types/pg
  ```

- **pnpm 12 build approvals**: both Prisma packages run install scripts, so `pnpm-workspace.yaml` allows them next to the other approvals (see "New project"): `prisma: true` for its `preinstall` check and `'@prisma/engines': true` for its `postinstall`, which downloads the schema engine every `migrate` command runs.
- **The CLI reports usage to Prisma** (`checkpoint.prisma.io`: the command, hashes of the project's path, the schema's providers and preview features) and checks for updates, unless `CHECKPOINT_DISABLE` is set. Set `CHECKPOINT_DISABLE=1` in `.env.example`: `prisma.config.ts` loads `.env` before the check runs (checked on 7.10.1), and the `.env.example` test names it as the CLI's own variable, as it does the AWS SDK's (see `devops-use-config-module`). Wherever the CLI runs without `.env`, the variable comes from the environment.

- Prisma 7 needs Node.js 20.19+, 22.12+ or 24+, and TypeScript 5.4+.
- **Generate the client before anything compiles against it.** `"postinstall": "prisma generate"` in `package.json` gives a fresh checkout and CI the client; after a schema change, the migration workflow runs it (see `db-prisma-use-migrations`).
- **`src/prisma/generated/` is generated code**: add it to `.gitignore`, to the linter's ignore patterns and to the coverage excludes (see the runner's file in this folder).
- **ESM**: the generator follows `"type": "module"` and the `tsconfig.json` module settings by itself, so the generated client imports with `.js` extensions and is imported as `@/prisma/generated/client.js`.
- What changed from Prisma 6, and the rules already follow: the database URL lives in `prisma.config.ts` (a `url` in the `datasource` block fails validation); a driver adapter is required; the CLI no longer loads `.env`; `migrate dev` no longer runs `prisma generate`. The rules use the `prisma-client` generator, which writes the client to its `output` folder; `prisma-client-js` still works in 7.10.1 but writes into `node_modules`.

## NestJS 12 and library changes the rules already follow

- **Terminus 12** removed the legacy `HealthIndicator` class; built-in checks take a timeout through `.withTimeout(ms)` (`micro-use-health-checks`).
- **`@Optional()` is no longer inherited**: a subclass redeclares its constructor with the marker (`di-prefer-constructor-injection`).
- **`ValidationPipe` `errorFormat`** stays at its default `'list'`, which the exception filter expects (`error-use-exception-filters`).
- **Lifecycle hooks run by module hierarchy level.** Do not rely on the order of `onModuleInit` (or the shutdown hooks) between providers of different modules; make the dependency explicit instead (`perf-async-hooks`).
- **TypeORM 1.x** removed the string-array forms of `relations` and `select` (use the object form, as the rules do) and throws on `null` or `undefined` in a `where` object: leave an optional filter's key out when it has no value, and match `NULL` with `IsNull()`. QueryBuilder `.where()` calls are not affected.
- **Express 5 route paths** follow `path-to-regexp` 8: a wildcard has a name (`*path` matches one or more segments, `{*path}` also none), an optional part is in braces (`/files/:name{.:ext}`), and `( ) [ ] + ? !` are rejected unless escaped. Nest rewrites a trailing `*` or `(.*)` to `{*path}` and logs a warning. On Fastify, controller paths go to Fastify's own router unchanged, so they stay with plain segments and `:param`s (see `api-platform-agnostic`); middleware paths (`forRoutes('orders/{*path}')`) are read the same way on both.
- **`@nestjs/config`** with a Joi `validationSchema` needs Joi 18 and moves Joi options under `validationOptions.libraryOptions`. The rules use the `validate` hook instead, which is unaffected.
- **Standard Schema validation** (Zod, Valibot, ArkType through `StandardSchemaValidationPipe`) is supported in NestJS 12. The NestJS docs name no default and present `ValidationPipe` first. The rules keep `class-validator` DTOs, new projects included, because the DTO classes are also the Swagger models and the base of the mapped types; a project already validating with Zod keeps Zod and applies the intent of the validation rules.
- **`errorCode` on `HttpExceptionOptions`** gives clients a stable code to branch on. The global exception filter copies it into the error envelope (`error-use-exception-filters`), and services set it where a client needs to tell errors apart and on every domain 400 (`error-throw-http-exceptions`).

Reference: [NestJS migration guide](https://docs.nestjs.com/migration-guide) · [NestJS Testing](https://docs.nestjs.com/fundamentals/testing) · [nestjs/schematics templates](https://github.com/nestjs/schematics/tree/master/src/lib/application/files) · [TypeORM FAQ: ESM projects](https://typeorm.io/docs/help/faq) · [TypeORM 1.0 upgrade guide](https://typeorm.io/docs/releases/1.0/upgrading-from-0.3) · [Upgrade to Prisma 7](https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7)
