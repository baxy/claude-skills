# Jest Setup

Applies when `jest` is in the project's `devDependencies`. It is the runner `nest new` generates for CommonJS projects. The tiers, layout and naming come from `rules/test-organize-by-tier`; this file only covers how Jest implements them.

## One config per tier

Each tier has a config file under `test/` that selects its directory with `testRegex`. A coverage config combines the stack-free tiers as `projects`.

```jsonc
// test/jest-unit.json
{
  "rootDir": "..",
  "moduleFileExtensions": ["js", "json", "ts"],
  "testRegex": "test/unit/.+\\.spec\\.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" },
  "moduleNameMapper": { "^@/(.*)$": "<rootDir>/src/$1" },
  "testEnvironment": "node"
}

// test/jest-coverage.json
{
  "rootDir": "..",
  "projects": ["<rootDir>/test/jest-unit.json", "<rootDir>/test/jest-e2e.json"],
  "collectCoverageFrom": ["src/**/*.ts", "!src/main.ts"]
}
```

`jest-e2e.json`, `jest-db.json` and `jest-integration.json` differ from `jest-unit.json` in their `testRegex`, and `jest-db.json` adds a setup file (below). `moduleNameMapper` resolves the `@/` alias (see `arch-import-boundaries`). With Prisma, the coverage config also excludes the generated client: `"!src/prisma/generated/**"`.

## The database tier's environment

The database tier runs on a database of its own and connects with the `.env` settings (see `test-database-tier`). The migration DataSource loads `.env` with `process.loadEnvFile()`, which works for the TypeORM CLI and under Vitest, but not under Jest. Jest gives each spec file its own copy of `process.env`, made before the file loads, and `process.loadEnvFile()` writes to the real one: the spec's copy never sees the values, so the DataSource connects with its defaults. When several files run in one process (`--runInBand`, as this tier runs), the next file's copy gets the values instead, so a run can pass or fail by the order of its files. The same call in a `setupFiles` entry does not reach the copy either.

So `test/jest-db.json` adds `"setupFiles": ["<rootDir>/test/db/env.ts"]`, a file that parses `.env` with `util.parseEnv()` and writes each value into the copy, then names the tier's database (the name is an example):

```typescript
// test/db/env.ts
import { existsSync, readFileSync } from 'node:fs';
import { parseEnv } from 'node:util';

// Jest gives each spec file a copy of process.env, which process.loadEnvFile() does not reach
if (existsSync('.env')) {
  for (const [name, value] of Object.entries(parseEnv(readFileSync('.env', 'utf8')))) {
    process.env[name] ??= value;
  }
}
process.env.DB_NAME = 'shop_test';
```

It runs before any spec loads. A variable set in the shell wins over `.env`, as with `process.loadEnvFile()`, and neither the DataSource nor `ConfigModule` overrides a variable that is already set, so `DB_NAME` stays the tier's. The DataSource keeps its `process.loadEnvFile()` call for the CLI. The Prisma tier's `pg` client reads the same variables and gets them from the same file. Checked with Jest 30.3 and ts-jest 29.4 on Node 26: without the file, the DataSource's options held `localhost` and an empty user; with it, the `.env` host and user and the tier's database.

## Scripts

```jsonc
// package.json
"scripts": {
  "test": "pnpm run test:unit && pnpm run test:e2e",
  "test:unit": "node --experimental-vm-modules ./node_modules/jest/bin/jest.js --config ./test/jest-unit.json",
  "test:e2e": "node --experimental-vm-modules ./node_modules/jest/bin/jest.js --config ./test/jest-e2e.json",
  "test:db": "node --experimental-vm-modules ./node_modules/jest/bin/jest.js --config ./test/jest-db.json --runInBand",
  "test:integration": "node --experimental-vm-modules ./node_modules/jest/bin/jest.js --config ./test/jest-integration.json --runInBand",
  "test:coverage": "node --experimental-vm-modules ./node_modules/jest/bin/jest.js --config ./test/jest-coverage.json --coverage"
}
```

On NestJS 11 and earlier, the scripts call `jest` directly instead of the `node --experimental-vm-modules` prefix.

- `--runInBand` runs one file at a time. The database and integration tiers share state, so they need it.
- Coverage is measured in one run over the stack-free tiers, as `test-organize-by-tier` requires.

## NestJS 12

- NestJS 12 packages are ESM-only. Jest loads them only on **Node 24.9 or later**; older Node versions fail with `ERR_REQUIRE_ASYNC_MODULE`. `nest upgrade` moves Jest to v30, and the generated scripts start Jest through `node --experimental-vm-modules`: keep that prefix.
- `@nestjs/testing` works the same with either runner, so the rule examples apply unchanged.
- Jest stays the runner for a CommonJS project. A new ESM project uses Vitest (see `test-runner-vitest.md`) rather than Jest's experimental ESM mode; an ESM project already on Jest keeps it (see `project-setup.md`).

Reference: [NestJS Testing](https://docs.nestjs.com/fundamentals/testing) · [Jest configuration](https://jestjs.io/docs/configuration) · [NestJS migration guide](https://docs.nestjs.com/migration-guide)
