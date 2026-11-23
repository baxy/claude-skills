---
title: Keep src/shared/ for Domain-Neutral Files Used by Several Modules
impact: HIGH
impactDescription: Shared code has one obvious home, business rules stay with the module that owns them, and no provider is instantiated twice
tags: architecture, shared, modules, utils, dtos, decorators
---

## Keep src/shared/ for Domain-Neutral Files Used by Several Modules

`src/shared/` is a folder of plain files that several modules import directly. It is **not a Nest module**: there is no `shared.module.ts` and no `SharedModule`. A file belongs there only when it passes all three tests:

1. **It is used by at least two modules**, or it is one of the app-wide pieces these rules give every module (below). A helper with one consumer lives in that module's type folder (`order/utils/`). When a second module needs it, move it to `shared/` in that change; when it drops back to one consumer, move it back; when nothing uses it, delete it.
2. **It is domain-neutral.** It knows no module's business rules: pagination, slugs, escaping a `LIKE` pattern. Logic that encodes one module's rules (when a product counts as in stock, how an order total is computed) stays in that module, and other modules import it through the `@/` alias (see `arch-import-boundaries`). A shared file whose name wants a module prefix (`product-label.ts`, `order-total.ts`) belongs in that module.
3. **It is not a provider that needs registering.** Services, controllers and entities never go in `shared/`. An injectable used by several modules gets its own module, named for what it does, that provides and exports it once (`storage/` exporting `StorageService`, see `arch-module-sharing`). Listing the same class in several modules' `providers` creates one instance per module.

**The app-wide pieces** are shared by design, whatever the module count: `types/paginated-result.type.ts`, which services return (see `perf-optimize-database`), and `types/jwt-user.type.ts` with the `Role` enum behind it, which services take (see `security-auth-jwt`). Each stays in `shared/` with a single consumer.

**The HTTP layer is not shared: it is the api module** (`src/api/`, see `test-use-testing-module`). The global filter and interceptors, the decorators controllers use (`@AuthUser()`, `@Public()`, `@Roles()`, the Swagger envelope decorators), the pagination query DTO, the request type and the response shapes live there. It holds only what works the same for every module, and no module's data: a DTO that describes one module's data (an order's address, a product's id) stays in that module, even when other modules nest or extend it (see `api-dto-naming`). A service never imports from it: what a service needs (the page it returns, the caller's `JwtUser`) stays in `shared/`, and the api module imports it from there.

**Dependencies point into `shared/`, never out of it.** A shared file never imports from a module, nor from the ORM. What knows the database, its transactions and its error codes, belongs to the module that owns the connection: `database/` with TypeORM, `prisma/` with Prisma (see `db-typeorm-use-transactions`, `db-prisma-use-transactions`). Services reach it through that module's service, and its `utils/` for the error checks. The root wiring files (`app.module.ts`, `main.ts`) and the api module each count as a consumer: what the app's wiring uses is app-wide. A domain-neutral helper used by the global filter and by one feature module stays in `shared/`: the api module is its second consumer.

**Layout and names:**

- Every file sits in a type folder: `types/`, `utils/`, `constants/`, `errors/`. DTOs, decorators, validators, filters and interceptors are never shared files: those that work the same for every module (`ApiPaginationQueryDto`, `@AuthUser()`, `@IsStorableDate()`) are the api module's, and one about a module's data (`OrderAddressDto`) stays in that module, which other modules import it from. Nothing at the `shared/` root, and no folder named for a feature (`shared/availability/`) or for a kind of provider (`shared/services/`).
- Names describe the concern and carry no module prefix: `slug-generator.ts`, `escape-like.ts`, `paginated-result.type.ts`.
- In a `shared/` folder that already breaks these rules, new files follow them; existing files are moved in a separate refactor, not as part of an unrelated change (the same as `arch-resource-naming`).

**Incorrect (a service, domain rules, loose files and single-consumer helpers in shared):**

```typescript
src/shared/
├── services/
│   └── storage.service.ts            // Listed in the providers of three modules: three instances
├── availability/
│   └── stock-availability.ts         // The inventory module's rules, in a feature folder
├── internal-caller.ts                // Loose at the root
├── product-label.ts                  // Used by one module, and named for one
├── dtos/
│   └── product-id.param.dto.ts       // The product's data: its DTO lives in its module
└── utils/
    ├── normalize-sku.ts              // One consumer
    └── validate-id.ts                // No consumer at all

// product.module.ts, user.module.ts and report.module.ts each repeat:
providers: [ProductService, StorageService],
```

**Correct (domain-neutral files used by several modules; services and rules in their modules):**

```typescript
src/
├── api/                               // What makes the API work, the same for every module
│   ├── decorators/
│   │   └── auth-user.decorator.ts
│   ├── dtos/
│   │   └── api-pagination.query.dto.ts
│   └── api.module.ts
├── inventory/
│   ├── services/
│   │   └── inventory-availability.service.ts  // Exported; product and order inject it
│   └── inventory.module.ts
├── product/
│   ├── dtos/
│   │   └── product-id.param.dto.ts    // The product's data, in its module
│   └── utils/
│       └── normalize-sku.ts           // Its only consumer
├── shared/
│   ├── constants/
│   │   └── system-user.constant.ts    // SYSTEM_USER_ID, used by four modules
│   ├── types/
│   │   ├── jwt-user.type.ts           // App-wide: services take it
│   │   └── paginated-result.type.ts   // App-wide: services return it
│   └── utils/
│       ├── slug-generator.ts          // Two modules
│       └── escape-like.ts             // Three modules
├── storage/                           // Injectable used by several modules: its own module
│   ├── services/
│   │   └── storage.service.ts
│   └── storage.module.ts              // providers and exports: [StorageService]
```

Reference: [NestJS Modules: shared modules](https://docs.nestjs.com/modules#shared-modules)
