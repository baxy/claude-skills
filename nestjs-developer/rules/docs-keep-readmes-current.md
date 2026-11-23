---
title: Keep a README With Fixed Sections and Update It in the Same Change
impact: MEDIUM
impactDescription: Anyone can run, configure and understand the service from its README, and the README never describes a version that no longer exists
tags: documentation, readme, onboarding
---

## Keep a README With Fixed Sections and Update It in the Same Change

The repository root has a `README.md` for the service. It is for the people who run and change the service without reading its code: teammates working without an agent, on-call engineers, the team that consumes its API. It states what the code cannot say briefly, and links to what is generated instead of copying it (endpoints are in Swagger, see `api-document-with-openapi`).

**The README has these sections, in this order:**

1. **Title and purpose**: the service's name, then one paragraph on what it does and who uses it.
2. **Run locally**: prerequisites (Node.js, PostgreSQL, Redis) and the commands (`pnpm start:dev`, see `error-report-to-sentry` for the start scripts).
3. **Configuration**: what whoever runs the service must provide: the required variables, which of them are secrets and how to get each (`openssl rand -hex 32` for `AUTH_JWT_SECRET`), and credentials that come from elsewhere (the role on AWS). Then a link to `.env.example`. Every other variable, with its comment and an example value, lives only in `.env.example` (see `devops-use-config-module`); a second table here would drift from it.
4. **Data**: the database it needs, how to apply the migrations (`pnpm migration:run`, see `db-use-migrations`), and the entities by module (order: `Order`, `OrderItem`).
5. **Interfaces**: its route groups and the Swagger path; the events and queues it publishes and consumes; the other services it calls.
6. **Tests**: the tiers it has and their scripts (see `test-organize-by-tier`).

**Each section stays at the level of its list.** Interfaces gives each route group one line, saying what it is for and who may call it. It does not give a route's fields, statuses or error codes, which Swagger shows. Data names the entities, not how a write is made safe (locks, conditional updates, transactions), which the code shows. Both may add what an operator needs and the code cannot say briefly: an extension the migrations create, the SQL that promotes an administrator, a setting the service needs behind a load balancer. A README that repeats each route and each mechanism goes stale with the next change inside a group, as no one revisits it then.

**Update the README in the same change** whenever the change touches what a section describes:

- a script added, removed or renamed, or a new prerequisite to run the service;
- a **required** environment variable added, removed or renamed (every variable change also updates `.env.example`, see `devops-use-config-module`);
- an entity added or removed;
- a route group (a new controller) added or removed, an event or queue published or consumed, a new call to another service.

Changes inside those boundaries (a bug fix, a new field on an existing DTO, a new route in an existing group, a refactor) do not touch the README, except to correct a sentence the change made false. Before reporting a change done, check it against the list above. A new service is created with its README.

**Incorrect (no README, or one that has drifted):**

```markdown
# orders

TODO

## Env
DATABASE_URL=...      <!-- renamed to DB_HOST and friends two changes ago; MAIL_API_KEY never added; a copy of .env.example that drifted -->
```

**Correct (the six sections, current with the code):**

```markdown
# Orders API

Customer orders: creating an order with line items, listing and reading a customer's own orders,
canceling one. Used by the web shop and the mobile app.

## Run locally
Needs Node.js 26, PostgreSQL 18+ and Redis 8+.

    pnpm install
    pnpm start:dev

## Configuration
    cp .env.example .env

The service refuses to start, naming what is missing, without:

- `APP_ENV`: `dev`, `stage` or `prod`.
- `DB_HOST`, `DB_USERNAME`, `DB_PASSWORD` (a secret), `DB_NAME`: the PostgreSQL database.
- `AUTH_JWT_SECRET` (a secret): the key access tokens are signed with, for example from `openssl rand -hex 32`.
- `AUTH_JWT_ISSUER`, `AUTH_JWT_AUDIENCE`: written into access tokens and required of every token.

Every other variable, with what it does and an example value, is in [`.env.example`](.env.example); its secrets are
empty.

## Data
PostgreSQL, migrated with `pnpm migration:run`. Entities: order (`Order`, `OrderItem`), user (`User`), auth (`AuthSession`, `AuthRefreshToken`).

## Interfaces
- HTTP, with Swagger at `/api/docs`:
  - `/api/v1/orders`: customers place, read and cancel their own orders; administrators read every order.
  - `/api/v1/users`: registration (no token), and the accounts by id: a customer reaches only their own.
  - `/api/v1/auth`: login and token refresh (no token), the caller's own account (`/me`), and logout of the caller's own session.
- Publishes: `order.created`
- Consumes: `payment.captured`
- Calls: the payment provider (Stripe)

## Tests
    pnpm test             # unit + e2e
    pnpm test:db          # on its own migrated database, shop_test: DB_NAME=shop_test pnpm migration:run
    pnpm test:coverage
```

Reference: [Make a README](https://www.makeareadme.com/)
