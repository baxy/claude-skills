---
title: Use Injection Tokens for Interfaces
impact: HIGH
impactDescription: Implementations can be swapped per environment or in tests without touching consumers
tags: dependency-injection, tokens, interfaces
---

## Use Injection Tokens for Interfaces

TypeScript interfaces do not exist at runtime, so they cannot be injection tokens. Inject an abstraction through a `Symbol` token declared next to the interface, or through an abstract class (which exists at runtime and needs no `@Inject`). Do not use string tokens: two modules can pick the same string by accident, and a typo fails only at startup.

- The token, the interface and the production binding live in the module that owns the abstraction: the token in `payment/constants/payment-adapter.constant.ts`, the interface in `payment/interfaces/payment-adapter.interface.ts`, the implementation in `payment/adapters/stripe.adapter.ts`. The data the contract passes around is a `type`, not an interface, in `types/` (`payment/types/payment-result.type.ts`, see `arch-feature-modules`).
- The token stays inside its module. The module's own service injects it (`PaymentService` injects `PAYMENT_ADAPTER`), and other modules inject that service, never the token (see `arch-module-sharing`).
- Choose the implementation from config (see `devops-use-config-module`), never from `process.env.NODE_ENV`.
- Test doubles live in `test/`, and tests swap them in with `overrideProvider()` or a provider in the testing module. Production modules never import a mock.

**Incorrect (interface as a type-only token, mock selected by NODE_ENV):**

```typescript
@Injectable()
export class OrderService {
  constructor(private readonly paymentAdapter: PaymentAdapter) {} // Interface: nothing to inject at runtime
}

@Module({
  providers: [
    {
      provide: 'PAYMENT_ADAPTER', // String token
      useClass: process.env.NODE_ENV === 'test' ? MockPaymentAdapter : StripeAdapter, // Mock shipped in production code
    },
  ],
})
export class PaymentModule {}
```

**Correct (Symbol token, binding in the owning module, doubles in tests):**

```typescript
// payment/constants/payment-adapter.constant.ts
export const PAYMENT_ADAPTER = Symbol('PAYMENT_ADAPTER');

// payment/types/payment-result.type.ts
export type PaymentResult = {
  success: boolean;
  transactionId: string;
  amountCents: number;
};

// payment/interfaces/payment-adapter.interface.ts
export interface PaymentAdapter {
  charge(amountCents: number, currency: string): Promise<PaymentResult>;
}

// payment/payment.module.ts
@Module({
  providers: [
    StripeAdapter,
    { provide: PAYMENT_ADAPTER, useExisting: StripeAdapter },
    PaymentService,
  ],
  exports: [PaymentService],
})
export class PaymentModule {}

// payment/services/payment.service.ts
@Injectable()
export class PaymentService {
  constructor(@Inject(PAYMENT_ADAPTER) private readonly paymentAdapter: PaymentAdapter) {}

  charge(amountCents: number, currency: string): Promise<PaymentResult> {
    return this.paymentAdapter.charge(amountCents, currency);
  }
}

// order/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(private readonly paymentService: PaymentService) {}
}

// test/e2e/payment/payment.service.spec.ts
const moduleRef = await Test.createTestingModule({ imports: [PaymentModule] })
  .overrideProvider(PAYMENT_ADAPTER)
  .useValue(new FakePaymentAdapter())
  .compile();
```

`PaymentAdapter` is a contract classes implement, so it is an interface, while `PaymentResult` is data, so a `type`. The test replaces the binding with the double from `test/helpers/fake-payment-adapter.ts`.

Alternatively, an abstract class is its own token, and its consumers need no `@Inject`:

```typescript
export abstract class PaymentAdapter {
  abstract charge(amountCents: number, currency: string): Promise<PaymentResult>;
}

@Module({ providers: [{ provide: PaymentAdapter, useClass: StripeAdapter }, PaymentService], exports: [PaymentService] })
export class PaymentModule {}

@Injectable()
export class PaymentService {
  constructor(private readonly paymentAdapter: PaymentAdapter) {}
}
```

Reference: [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
