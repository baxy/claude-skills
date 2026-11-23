---
title: Keep Integration Suites Independent and Honest
impact: MEDIUM
impactDescription: Integration tests that pass or fail for the right reason, in any order
tags: testing, integration, isolation, authorization
---

## Keep Integration Suites Independent and Honest

The integration tier (`test/integration/`, see `test-organize-by-tier`) calls the running API over real HTTP against its real stack (database, storage, events). It shares that stack across specs and across runs, so it needs stricter habits than the in-process tiers.

- **Specs are order-independent.** Each spec creates the state it asserts on, so any spec runs alone. Never read rows another spec created, and never add a sequencer to pin the order: a fixed order hides the dependency instead of catching it.
- **The shared database accumulates.** Specs do not clean up, so rows pile up across runs, except rows that count toward a limit every caller shares (the shop's 200 categories): those specs delete what they created, through the API, or the suite fails after a few runs. Never assert on the position of pre-existing or seeded rows in a page; assert on data the test just created. Assert a total only for a set the test created itself.
- **Count with the total, not the page.** `data.length` is bounded by `limit`; use the response's total count when the claim is about the whole set.
- **Assert the status, then read the body.** Use a client that returns `{ status, headers, body }` without throwing on HTTP errors, and narrow success vs error before reading fields.
- **Keep authorization out of functional tests.** Functional specs run as one default caller with enough permissions to arrange state. Authorization gets its own suite that varies roles, and data isolation between users gets its own suite; both are order-independent like every other spec. A functional failure then reads as "the feature broke", not "a policy changed".
- Structure: the top-level `describe` is the resource (`Orders`); inside it, `VERB /path` blocks in CRUD or lifecycle order, positives beside negatives; a specific scenario of one endpoint gets a parenthetical suffix (`POST /orders/:id/cancel (already shipped)`).
- **State no route can create goes through one helper per change.** When the API has no route for a state a spec needs (an admin, since no route grants a role; a session that has reached its end), the helper does what an operator would do: it writes the row through the migration DataSource and checks that exactly one row changed, so a `DB_*` that points at another database than the API's fails there instead of as a puzzling 403 later. Specs never write SQL themselves; the integration tier then needs the database's credentials as well as the API's URL.
- **Separate clients send their own address.** A spec that needs several callers, each with its own rate limit, sends `X-Forwarded-For` per client, and the API under test trusts the test runner as its proxy (`APP_TRUSTED_PROXIES=127.0.0.1`). That is how a load balancer reaches a deployed API, so the rate limits are tested the way they run.
- **An outside system with no local server gets a stand-in.** The stack this tier runs against, like a developer's local run, must not call the real provider (SES would deliver real mail). A stand-in in `test/helpers/` (`ses-stand-in.ts`) is a small HTTP server that answers the calls the SDK makes as the provider does, records what it received for the specs to read, and lets a spec choose a failure (for example by the recipient's address: a refusal, throttling, no answer). The app reaches it through the SDK's endpoint setting (`AWS_SES_ENDPOINT`), which `validate-env.ts` allows as `http` only under `APP_ENV` `dev` (see `devops-use-config-module`), and a `package.json` script starts it with `node`. Run by Node directly, the file uses only erasable TypeScript syntax (no enums, no parameter properties), which Node strips without a flag from 22.18 (23.6 on the 23 line); the README names that version and the stand-in's port setting.
- **A command-line command is tested here, as an operator runs it.** The spec starts the built command (`node --import ./dist/sentry/preload.js dist/cli/main.js create-admin ...`) against the stack's database, with its secret on standard input, and checks the exit code and the output: 0 and a result the API shows (the new account signs in), and for an operator's mistake (a taken email) 1, the message on stderr and no crash marker (see `arch-feature-modules`).
- **`main.ts` needs no spec of its own.** It creates the app, calls `ApiModule.setup()`, whose security headers and CORS the e2e tier checks (see `api-platform-agnostic`), and listens, which every spec here goes through.
- Run one file at a time (`--no-file-parallelism` in Vitest, `--runInBand` in Jest): the suites share database state.

**Incorrect (order-dependent, position-based, policy mixed into features):**

```typescript
describe('Orders', () => {
  let orderId: string;

  it('creates an order', async () => {
    const res = await caller.post('/orders', { items: [{ sku: 'SKU-001', quantity: 1 }] });
    orderId = res.body.data.id; // Next spec depends on this one running first
  });

  it('lists orders', async () => {
    const res = await caller.get('/orders?limit=10');
    expect(res.body.data[0].status).toBe('shipped'); // Position of a pre-existing row
    expect(res.body.data.length).toBe(1);             // Page length used as a total
  });

  it('forbids another user from canceling', async () => { // Authorization inside a functional suite
    const res = await as({ userId: OTHER_USER_ID }).post(`/orders/${orderId}/cancel`, {});
    expect(res.status).toBe(404);
  });
});
```

**Correct (self-contained specs, fresh data, separate policy suites):**

```typescript
// test/integration/order/order.spec.ts
describe('Orders', () => {
  describe('POST /orders', () => {
    it('creates an order', async () => {
      const res = await caller.post<Order>('/orders', { items: [{ sku: 'SKU-001', quantity: 1 }] });
      expect(res.status).toBe(201);
    });

    it('rejects an order without items', async () => {
      const res = await caller.post('/orders', { items: [] });
      expect(res.status).toBe(400);
    });
  });

  describe('GET /orders', () => {
    it('includes a just-created order on the first page', async () => {
      const created = await createOrder();
      const res = await caller.get<Order[]>('/orders?limit=100');

      expect(res.status).toBe(200);
      expect(ok(res).data.some((order) => order.id === created.id)).toBe(true);
    });
  });

  describe('POST /orders/:id/cancel (already shipped)', () => {
    it('rejects canceling a shipped order', async () => {
      const order = await createShippedOrder();
      const res = await caller.post(`/orders/${order.id}/cancel`, { reason: 'Changed my mind' });
      expect(res.status).toBe(400);
    });
  });
});

// test/integration/order/order-isolation.spec.ts
describe('Orders isolation', () => {
  describe('POST /orders/:id/cancel', () => {
    it('hides another user\'s order', async () => {
      const order = await createOrder();
      const res = await as({ userId: OTHER_USER_ID }).post(`/orders/${order.id}/cancel`, {});
      expect(res.status).toBe(404);
    });
  });
});
```

### With Prisma

There is no migration DataSource: the helper for a state no route can create writes through a plain `pg` client built from `dbConfig()`, with its search path, as the database tier's constraint specs do (see `test-database-tier`), and checks the result's `rowCount`. In the example, `createDatabaseClient()` returns a `pg` `Client` built from `dbConfig()`, and `grantAdminRole()` does what an operator would do, since no route grants a role.

```typescript
// test/integration/helpers/grant-admin-role.ts
export async function grantAdminRole(email: string): Promise<void> {
  const client = createDatabaseClient();
  await client.connect();
  try {
    const { rowCount } = await client.query(`UPDATE users SET roles = array_append(roles, 'admin') WHERE email = $1 AND deleted_at IS NULL`, [email]);
    if (rowCount !== 1) {
      throw new Error(`Expected to grant the role to one user, granted it to ${rowCount}: do DB_* and the API use one database?`);
    }
  } finally {
    await client.end();
  }
}
```

Reference: [Vitest CLI](https://vitest.dev/guide/cli) · [Jest CLI](https://jestjs.io/docs/cli#--runinband)
