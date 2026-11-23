---
title: Keep Import Paths Aligned with Module Boundaries
impact: MEDIUM
impactDescription: Cross-module dependencies are visible at a glance and survive file moves
tags: architecture, imports, modules, path-alias
---

## Keep Import Paths Aligned with Module Boundaries

The import path tells the reader whether a dependency stays inside the module or crosses into another one.

- **Inside a module, import relatively**: `./dtos/create-order.dto` from the module root, `../services/order.service` from inside a type folder.
- **Across modules, use the `@/` alias with the module name first**: `@/payment/services/payment.service`.
- **What describes another module's data or rules is imported from that module**, its DTOs and types included: a DTO that nests another module's shape imports it with the alias. It is not copied, and a second consumer does not move it to the api module or `shared/`; only a domain-neutral helper moves to `shared/` once a second module needs it (see `arch-shared-folder`).
- **Never reach into a sibling module with a relative path** (`../../payment/...`).
- **Never use the alias for the importing module's own files.**
- **The root files import relatively**: `app.module.ts` and `main.ts` sit at the root of `src/`, so `./user/user.module` and `./api/api.module` are inside the same tree. `sentry/preload.ts` imports the Sentry module's own files relatively, like any file of a module.

With this, `grep -r "from '@/payment/" src` lists every consumer of the payment module, and moving a module only breaks the imports that genuinely cross into it.

**Incorrect (relative paths across modules, alias inside a module):**

```typescript
// src/order/services/order.service.ts
import { PaymentService } from '../../payment/services/payment.service'; // Crosses a module boundary relatively
import { CreateOrderDto } from '@/order/dtos/create-order.dto';             // Alias for the module's own file
import { Order } from '../../order/entities/order.entity';                  // Detour out of and back into the module
```

**Correct (relative within, alias across):**

```typescript
// src/order/services/order.service.ts
import { PaymentService } from '@/payment/services/payment.service';
import { JwtUser } from '@/shared/types/jwt-user.type';
import { CreateOrderDto } from '../dtos/create-order.dto';
import { Order } from '../entities/order.entity';

// src/order/order.module.ts
import { OrderController } from './controllers/order.controller';
import { OrderService } from './services/order.service';
```

The alias is declared in `tsconfig.json` without `baseUrl`: TypeScript 6 rejects it, and `paths` resolve from the tsconfig.

```jsonc
// tsconfig.json
{ "compilerOptions": { "paths": { "@/*": ["./src/*"] } } }
```

Tests reach their subject through the same alias; the runner reads it as shown in `references/test-runner-vitest.md` or `references/test-runner-jest.md`. In an ESM project every import, the alias included, ends in `.js` (`@/payment/services/payment.service.js`); see `references/project-setup.md`.

Reference: [TypeScript paths](https://www.typescriptlang.org/tsconfig/#paths)
