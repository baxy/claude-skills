---
name: nestjs-reviewer
description: Reviews NestJS code changes, branches and pull requests against the nestjs-developer conventions (architecture, dependency injection, REST controllers, DTOs, security, TypeORM or Prisma, testing, code style). Use when asked to review NestJS code or a NestJS pull request, to check a controller, service, module, guard, interceptor, DTO, entity, Prisma schema, migration or test in a NestJS project, or to validate a NestJS feature after implementing it.
allowed-tools: Read, Grep, Glob, Bash
metadata:
  version: "1.0.0"
  requires: nestjs-developer
---

# NestJS Reviewer

Reviews NestJS code against the rules of the `nestjs-developer` skill. Every Critical and Warning finding is backed by `file:line` evidence, a concrete fix, and the ID of the rule it breaks, or, for a defect no rule covers, the input or state that makes it fail.

## Sources

- `../nestjs-developer/rules/<rule-id>.md` (relative to this skill folder) - the rule, with incorrect and correct examples. Read the rule before reporting a finding against it. If the `nestjs-developer` folder is missing, review with `references/checklist.md` and `references/anti-patterns.md` only and say so in the Summary.
- `../nestjs-developer/references/` - setup that depends on the project: `project-setup.md` (ESM projects, NestJS 12, TypeORM 1.x and Prisma 7) and the Vitest and Jest test-runner files. Checklist items tagged `references/...` point here. Read `package.json` first: the module system and the test runner it declares decide which of these apply.
- `references/checklist.md` - what to check per area, each item tagged with its rule ID.
- `references/anti-patterns.md` - common problems, with the fix.

## Workflow

Copy this checklist into your response and tick it off:

```
- [ ] 1. Scope identified
- [ ] 2. Each affected area checked
- [ ] 3. Findings verified
- [ ] 4. Report written
```

1. **Scope.** For a PR or branch, start from the diff against the base branch (`git diff <base>...HEAD`). Review the changed files plus what they directly depend on (their module, DTOs, entities, tests). Flag pre-existing code only when the change relies on it.
2. **Check each affected area** with the matching sections of `references/checklist.md` (a new endpoint touches Controllers, Swagger, DTOs, Guards, Testing and Documentation). Read every new or changed comment against the Code Style items, whatever the area. For every candidate finding, open the rule file and confirm the rule says what you are about to claim.
3. **Verify findings** (required). Every Critical and Warning has the path, line, exact snippet and a concrete fix. A candidate no rule covers that names an input or state that breaks the code is tagged `no rule` and is a Warning, or Critical when it is one of the table's Critical cases (a crash, a broken endpoint, data loss); never a Suggestion. Drop any candidate that no rule covers and that names neither a concrete defect nor a concrete gain.
4. **Report** in the format below.

## Severity

| Severity | When |
|---|---|
| Critical | Security hole (unguarded protected route; password stored plain, fast-hashed or encrypted; a sensitive field that can reach a response: with TypeORM a column without `@Exclude()`, with Prisma a field missing from `PRISMA_OMIT`; a result converted to a response DTO without `excludeExtraneousValues: true`; a request body bound to an entity or a Prisma input type; a price, total, balance or caller id taken from the request; injection, raw SQL built from input included (`$queryRawUnsafe`); query values in the logs; secret in code; every proxy trusted (`trustProxy: true`, `trust proxy` set to `true`), with which a client chooses its own address and passes every per-address rate limit; a Sentry setup that sends more than its allowlist; a migration CLI's connection that skips TLS or the certificate check while `DB_SSL` is on (`prisma.config.ts` without `sslaccept=strict`, the TypeORM migration DataSource without `ssl`)), data loss or corruption (multi-entity write without a transaction, or a query inside one that commits on its own, such as `this.prisma` inside `$transaction()`; a read-check-write on a value concurrent requests also change (a balance, a stock level, a quota, a seat count); with Prisma, a write whose `where` can receive `undefined` while `strictUndefinedChecks` is off; `synchronize: true` or `synchronize` read from config; an edited applied migration; a destructive migration without expand and contract; a call that changes something in an outside system whose record does not keep a refusal and an unknown outcome apart), or code that crashes or breaks an endpoint |
| Warning | Any other violation of a `nestjs-developer` rule, including the `style-` rules, or a concrete defect no rule covers (a logic error, an unhandled case) shown with the input or state that triggers it; tag it `no rule` instead of a rule ID |
| Suggestion | An improvement no rule requires, that names what it gains: readability, performance, developer experience. A preference, a test for a case the specs already cover, or another way to write code a rule accepts is not a finding, whatever it gains: a response DTO in place of a returned entity is one |

The rule's impact level (CRITICAL to LOW) orders findings within a severity; it does not set the severity.

## Constraints

- The `db-` rules are an ORM-neutral rule per topic plus a TypeORM form (`db-typeorm-*`) and a Prisma form (`db-prisma-*`), on PostgreSQL. Read the ORM from `package.json`: apply the TypeORM items of "Database and Migrations" to a TypeORM project and "Database with Prisma" to a Prisma project. In a project on another ORM (Drizzle, MikroORM), check the intent (transactions, generated migrations, no N+1, hidden sensitive fields) and do not flag the absence of TypeORM or Prisma APIs.
- When the project follows a documented convention of its own that conflicts with a rule, report the conflict once as a Suggestion instead of a Warning on every occurrence.
- Returning an entity is not a finding by itself, at any severity: a Suggestion to give the resource one response shape included (`api-use-dto-serialization`).
- bcrypt in a project that already stores bcrypt hashes is not a finding. bcrypt cost below 12, or no 72-byte limit on the password, is a Warning; a new project choosing bcrypt over Argon2id is a Warning (`security-hash-passwords`).
- When the PR says it makes a change whose required companion is missing from the changed files (an entity change without a generated migration, a new endpoint without its e2e spec), report the gap at the severity it would have and state the assumption. Downgrade it only when the PR shows the companion exists or the rule does not apply.
- Do not suggest architectural rewrites unless a Critical issue requires one.
- This skill reviews. Change code only when the user asks for fixes.

## Report format

1. **Summary** - scope reviewed and the key observations in 2-4 sentences.
2. **Critical issues (must fix)**
3. **Warnings (should fix)**
4. **Suggestions (consider)**
5. **Positive observations** - patterns worth keeping.
6. **Recommendations** - prioritized next steps.

Write "None" under an empty section. Each finding in sections 2-4 follows this shape:

````markdown
**Unguarded delete route** - `security-use-guards`
`src/order/controllers/order.controller.ts:58`
```typescript
@Public()
@Delete(':id')
delete(@Param() params: OrderIdParamDto): Promise<void> {
```
Why: `@Public()` skips the global JWT guard, so anyone can delete any order.
Fix: remove `@Public()`; the global guard then requires a token, and the service scopes the delete to the caller's orders.
````
