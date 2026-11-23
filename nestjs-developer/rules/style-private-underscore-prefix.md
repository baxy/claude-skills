---
title: Prefix Private Methods with an Underscore
impact: MEDIUM
impactDescription: Private methods are recognizable at every call site
tags: style, naming, methods, private
---

## Prefix Private Methods with an Underscore

**This rule applies only to private methods.** Every method declared `private` gets a `_` prefix, and keeps the `private` keyword: `private _assertCancelable()`, `private _calculateTotalCents()`. At a call site, `this._calculateTotalCents()` then reads as internal without jumping to the declaration.

It does **not** apply to:

- public or protected methods;
- properties and fields, including `private readonly` injected dependencies (`private readonly orderRepository`, never `_orderRepository`);
- functions, exported or not, constants, parameters or local variables. An unused parameter keeps a plain name too (`(label, body) => ...`): `no-unused-vars` already ignores a parameter that comes before a used one, and one after the last used parameter is left out.

**Incorrect (missing prefix, or prefix in the wrong places):**

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly _orderRepository: Repository<Order>, // Prefix on an injected dependency
  ) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const totalCents = this.calculateTotalCents(dto.items); // Private method without prefix
    return this._orderRepository.save(this._orderRepository.create({ userId: authUser.id, totalCents }));
  }

  _buildReference(order: Order): string {                   // Public method with a prefix
    return `ORD-${order.id.slice(0, 8)}`;
  }

  private calculateTotalCents(items: OrderItemDto[]): number { /* ... */ }
}
```

**Correct (prefix on private methods only):**

```typescript
@Injectable()
export class OrderService {
  constructor(private readonly orderRepository: Repository<Order>) {}

  async create(authUser: JwtUser, dto: CreateOrderDto): Promise<Order> {
    const totalCents = this._calculateTotalCents(dto.items);
    return this.orderRepository.save(this.orderRepository.create({ userId: authUser.id, totalCents }));
  }

  buildReference(order: Order): string {
    return `ORD-${order.id.slice(0, 8)}`;
  }

  private _calculateTotalCents(items: OrderItemDto[]): number { /* ... */ }
}
```
