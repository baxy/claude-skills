---
title: Honor Liskov Substitution Principle
impact: HIGH
impactDescription: Ensures implementations are truly interchangeable without breaking callers
tags: dependency-injection, inheritance, solid, lsp
---

## Honor Liskov Substitution Principle

Subtypes must be substitutable for their base types without altering program correctness. In NestJS with dependency injection, this means any implementation of an interface or abstract class must honor the contract completely. A mock payment service used in tests must behave like a real payment service (return similar shapes, handle errors the same way). Violating LSP causes subtle bugs when swapping implementations.

A contract for a call to an outside system names every outcome its caller acts on. For a call that changes something in an outside system (a charge, a transfer, a message sent), those are: done; rejected for this input (the caller may try again with other input, as a customer whose card is declined tries another card); refused (nothing happened, and sending it again fails the same way); and unknown (no answer the code can read: the system may have acted). The caller records each differently (see `db-use-transactions`), so every implementation, fakes included, reports each one the same way: one that throws a 500 for a refusal makes the caller treat it as unknown and send it again forever. What the caller may do with unknown depends on the provider: sending again with the same idempotency key is safe only with a provider that takes one, as payment providers do. A mail provider that takes none would send a second mail, so an unknown mail is recorded for a check, not sent again.

**When one implementation cannot honor a property, the contract says which ones do.** "Sent twice with one key, delivered once" holds, for example, for an SMS provider that takes a key, and not for a mail provider that takes none. The contract does not promise it for all: each implementation declares the properties it has, and the contract suite runs with that declaration and tests the declared behavior, deduplication where it is declared and a second delivery where it is not. A fake stands for one implementation and behaves like it, declaration included: a fake mail channel that deduplicates, where the real one cannot, passes tests that production then fails. Every fake a spec uses runs the contract suite, so none drifts from what it stands for.

**Incorrect (implementation violates the contract):**

```typescript
// Base interface with clear contract
interface PaymentAdapter {
  /**
   * Charges the specified amount.
   * @returns PaymentResult on success
   * @throws PaymentDeclinedException if the card is declined
   */
  charge(amountCents: number, currency: string): Promise<PaymentResult>;
}

// Production implementation - follows the contract, as shown below
@Injectable()
export class StripeAdapter implements PaymentAdapter {
  async charge(amountCents: number, currency: string): Promise<PaymentResult> { /* ... */ }
}

// Mock that violates LSP - different behavior!
@Injectable()
export class MockPaymentService implements PaymentAdapter {
  async charge(amountCents: number, currency: string): Promise<PaymentResult> {
    // VIOLATION 1: Throws for valid input (contract says return PaymentResult)
    if (amountCents > 100_000) {
      throw new Error('Mock does not support large amounts');
    }

    // VIOLATION 2: Returns null instead of PaymentResult
    if (currency !== 'USD') {
      return null as any; // Real service would convert or reject properly
    }

    // VIOLATION 3: Missing required field
    return { success: true } as PaymentResult; // Missing transactionId!
  }
}

// payment/services/payment.service.ts - the payment module's service trusts the contract
@Injectable()
export class PaymentService {
  constructor(@Inject(PAYMENT_ADAPTER) private readonly paymentAdapter: PaymentAdapter) {}

  async charge(order: Order): Promise<string> {
    const result = await this.paymentAdapter.charge(order.totalCents, order.currency);
    // Fails with MockPaymentService: result might be null, and transactionId undefined
    return result.transactionId;
  }
}
```

**Correct (implementations honor the contract):**

```typescript
export const PAYMENT_SUPPORTED_CURRENCIES = ['USD', 'EUR', 'GBP'];

interface PaymentAdapter {
  /**
   * @param currency - ISO 4217 currency code
   * @throws PaymentInvalidCurrencyException if the currency is not supported; nothing is sent
   * @throws PaymentDeclinedException if the card is declined; the customer may pay with another card
   * @throws PaymentRefusedException if the provider refused the request; nothing moved, and sending it again fails the same way
   * @throws ServiceUnavailableException (503) if no answer could be read; the provider may have charged
   */
  charge(amountCents: number, currency: string): Promise<PaymentResult>;

  /**
   * @throws PaymentTransactionNotFoundException if transactionId is invalid
   */
  refund(transactionId: string, amountCents?: number): Promise<PaymentRefundResult>;
}

// payment/adapters/stripe.adapter.ts
@Injectable()
export class StripeAdapter implements PaymentAdapter {
  constructor(private readonly stripeClient: StripeClient) {}

  async charge(amountCents: number, currency: string): Promise<PaymentResult> {
    if (!PAYMENT_SUPPORTED_CURRENCIES.includes(currency)) {
      throw new PaymentInvalidCurrencyException(`Unsupported currency: ${currency}`);
    }
    const answer = await this.stripeClient.createCharge(amountCents, currency);
    if ('refusal' in answer) {
      // Stripe answers 402 for a declined card, another 4xx for a request it refuses
      if (answer.refusal.statusCode === 402) {
        throw new PaymentDeclinedException(answer.refusal.message);
      }
      throw new PaymentRefusedException(answer.refusal.message);
    }
    return {
      success: true,
      transactionId: answer.charge.id,
      // Stripe counts in the smallest unit too
      amountCents: answer.charge.amount,
    };
  }

  async refund(transactionId: string, amountCents?: number): Promise<PaymentRefundResult> {
    // Implementation...
  }
}

// payment/clients/stripe.client.ts
type StripeChargeAnswer = { charge: Stripe.Charge } | { refusal: { statusCode: number; message: string } };

@Injectable()
export class StripeClient {
  constructor(private readonly stripe: Stripe) {}

  async createCharge(amountCents: number, currency: string): Promise<StripeChargeAnswer> {
    try {
      return { charge: await this.stripe.charges.create({ amount: amountCents, currency }) };
    } catch (error) {
      const status = error instanceof Stripe.errors.StripeError ? error.statusCode : undefined;
      if (status !== undefined && status < 500 && status !== 429) {
        // Stripe answered: a 4xx but a 429 charged nothing
        return { refusal: { statusCode: status, message: (error as Stripe.errors.StripeError).message } };
      }
      throw this._toHttpException(error);
    }
  }

  private _toHttpException(error: unknown): unknown { /* ... */ }
}

// test/helpers/fake-payment-adapter.ts
export class FakePaymentAdapter implements PaymentAdapter {
  private readonly transactions = new Map<string, PaymentResult>();
  private sequence = 0;

  async charge(amountCents: number, currency: string): Promise<PaymentResult> {
    if (!PAYMENT_SUPPORTED_CURRENCIES.includes(currency)) {
      throw new PaymentInvalidCurrencyException(`Unsupported currency: ${currency}`);
    }

    if (amountCents === 99_999) {
      throw new PaymentDeclinedException('Card declined (test scenario)');
    }
    if (amountCents === 99_998) {
      throw new PaymentRefusedException('Request refused (test scenario)');
    }
    if (amountCents === 99_997) {
      throw new ServiceUnavailableException('No answer from the provider (test scenario)');
    }

    const result: PaymentResult = {
      success: true,
      transactionId: `mock_${++this.sequence}`,
      amountCents,
    };

    this.transactions.set(result.transactionId, result);
    return result;
  }

  async refund(transactionId: string, amountCents?: number): Promise<PaymentRefundResult> {
    if (!this.transactions.has(transactionId)) {
      throw new PaymentTransactionNotFoundException(transactionId);
    }

    return {
      success: true,
      refundId: `refund_${transactionId}`,
      amountCents: amountCents ?? this.transactions.get(transactionId)!.amountCents,
    };
  }
}

// payment/services/payment.service.ts
@Injectable()
export class PaymentService {
  constructor(@Inject(PAYMENT_ADAPTER) private readonly paymentAdapter: PaymentAdapter) {}

  async charge(order: Order): Promise<PaymentResult> {
    try {
      return await this.paymentAdapter.charge(order.totalCents, order.currency);
    } catch (error) {
      if (error instanceof PaymentDeclinedException) {
        throw new BadRequestException('The payment was declined', { cause: error, errorCode: 'PAYMENT_DECLINED' });
      }
      throw error;
    }
  }
}
```

The interface documents what its types cannot say: the currency's format and the outcome behind each exception it throws, all four of them. `StripeAdapter` reads the answer `StripeClient` returns, never the SDK's error: the client alone calls the SDK, returns a 4xx but a 429 as Stripe's answer, and maps every other failure as `micro-use-http-clients` shows (`_toHttpException()`, left out here), so an unreadable answer is a 503, the unknown outcome. The SDK is a provider of the payment module by its class, `Stripe`, built in a factory, as `S3Client` is. `FakePaymentAdapter` is a double that honors the contract: it validates the currency as the real adapter does, returns the same shape, and throws for an unknown transaction. It declines one amount (`99_999` here), refuses another (`99_998`) and gets no answer for a third (`99_997`), so that specs can test each outcome. `PaymentService` behaves the same with `StripeAdapter` and `FakePaymentAdapter`, so the implementations swap safely; the order module calls this service, never the adapter (see `arch-module-sharing`). A declined payment leaves the order pending, so the customer can pay again; a refusal and an unknown outcome reach the caller as they are, and it records them apart (see `db-use-transactions`).

**Testing LSP compliance:**

```typescript
// test/helpers/payment-adapter.contract.ts
export function testPaymentAdapterContract(
  createAdapter: () => PaymentAdapter,
) {
  describe('PaymentAdapter contract', () => {
    let adapter: PaymentAdapter;

    beforeEach(() => {
      adapter = createAdapter();
    });

    it('returns PaymentResult with all required fields', async () => {
      const result = await adapter.charge(1000, 'USD');
      expect(result).toHaveProperty('success');
      expect(result).toHaveProperty('transactionId');
      expect(result).toHaveProperty('amountCents');
      expect(typeof result.transactionId).toBe('string');
    });

    it('throws PaymentInvalidCurrencyException for unsupported currency', async () => {
      await expect(adapter.charge(1000, 'INVALID'))
        .rejects.toThrow(PaymentInvalidCurrencyException);
    });

    it('throws PaymentTransactionNotFoundException for invalid refund', async () => {
      await expect(adapter.refund('nonexistent'))
        .rejects.toThrow(PaymentTransactionNotFoundException);
    });
  });
}

// test/unit/payment/fake-payment-adapter.spec.ts
describe('FakePaymentAdapter', () => {
  testPaymentAdapterContract(() => new FakePaymentAdapter());
});
```

Every implementation must pass the shared suite. The fake has no injected dependencies, so its spec constructs it directly in the unit tier (see `test-organize-by-tier`). `StripeAdapter` has an injected client, so it is never constructed by hand (see `di-prefer-constructor-injection`): the payment module's e2e suite covers it with the SDK mocked at its class, `Stripe` (see `test-mock-external-services`).

**A property only some implementations have:**

```typescript
// test/helpers/notification-channel-adapter.contract.ts
export type NotificationChannelAdapterTraits = { deduplicatesByKey: boolean };

export function testNotificationChannelAdapterContract(
  createAdapter: () => NotificationChannelAdapter,
  traits: NotificationChannelAdapterTraits,
) {
  describe('NotificationChannelAdapter contract', () => {
    it('reports a lost answer as unknown', async () => { /* ... */ });

    if (traits.deduplicatesByKey) {
      it('delivers once when sent twice with one key', async () => { /* ... */ });
    } else {
      it('delivers again when sent twice with one key', async () => { /* ... */ });
    }
  });
}

// test/unit/notification/fake-notification-channel-adapter.spec.ts
describe('FakeNotificationChannelAdapter', () => {
  testNotificationChannelAdapterContract(() => new FakeNotificationChannelAdapter(), { deduplicatesByKey: false });
});
```

The suite runs against the channels of `di-avoid-service-locator`. Each implementation declares what its provider guarantees (`NotificationEmailChannelAdapter` sends through SES, which takes no key), and a fake declares what the implementation it stands for declares, here the email channel's. No fake may be kinder than its provider: where the provider does not deduplicate, the suite expects a second delivery.

Reference: [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)
