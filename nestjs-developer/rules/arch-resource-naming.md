---
title: Name Everything in a Module After the Module
impact: HIGH
impactDescription: Any name shows which module owns it, names never collide, and a module's code is found with one search
tags: architecture, naming, modules, entities, services, conventions
---

## Name Everything in a Module After the Module

A resource module is named for its resource, in the singular, and **every class and file it declares starts with that name**, apart from the few names listed at the end of this rule, and so does a property injecting one of its own classes (`orderPricingService`, `orderItemRepository`); other dependencies are named for their class (`paymentService`, `dataSource`). `OrderItemService` read in another module says where it comes from without opening it; `ItemService` does not, and collides with the next module that has items. `grep -r Order src` finds the whole module, and its files sort together in every folder.

**One singular name per resource.** The module folder, the module, controller, service and entity classes, their files and the injected properties all use the singular resource name. It is plural in exactly two places: the URL (`@Controller('orders')`, see `api-controller-conventions`) and the database table (`@Entity('orders')`, see `db-schema-conventions`).

| Element | Name |
|---|---|
| Module folder | `src/order/` |
| Module | `OrderModule` in `order.module.ts` |
| Entity | `Order` in `entities/order.entity.ts`; table `orders` |
| Service | `OrderService` in `services/order.service.ts`; injected as `orderService` |
| Controller | `OrderController` in `controllers/order.controller.ts` |
| Test folders | `test/e2e/order/order.spec.ts` (mirror `src/`) |

Uncountable nouns keep their form (`analytics/`, `metadata/`).

**Everything else in the module takes the prefix too**, whatever its type folder:

| Kind | Example in `src/order/` |
|---|---|
| Sub-resource entity | `OrderItem` in `entities/order-item.entity.ts`; table `order_items` |
| Its service and controller | `OrderItemService`, `OrderItemController` |
| Helper services | `OrderPricingService`, `OrderNumberGeneratorService`, in `services/order-number-generator.service.ts` |
| Other providers | named for their kind, in that kind's folder: `OrderOwnerGuard` in `guards/order-owner.guard.ts` |
| DTOs (verb first, see `api-dto-naming`) | `CreateOrderItemDto`, `OrderItemIdParamDto`, `FindAllOrderItemsQueryDto`, `OrderItemResponseDto` |
| Enums and types | `OrderStatus`, `OrderItemKind`, `OrderSummary` |
| Error, event, listener and processor classes | `OrderNotCancelableError`, `OrderCreatedEvent`, `OrderPaymentEventsListener`, `OrderExportProcessor` |
| Exported constants | `ORDER_REASON_MAX_LENGTH`, `ORDER_CANCELABLE_STATUSES` |
| Injected properties | `orderItemRepository`, `orderPricingService` |

- **Sub-resources keep the prefix in code and in the table**, and drop it in the URL when the parent route already gives the context: `OrderItem`, `order_items`, `GET /orders/:id/items`. A table has no parent to read it from; a route does. When the parent route belongs to another module, the segment keeps the prefix: payment refunds listed under orders are `GET /orders/:id/payment-refunds`, served by the payment module.
- **Listeners: one class per source module, one method per event.** The class takes the prefix of the module it lives in, then the module whose events it handles: `OrderPaymentEventsListener` in `order/listeners/order-payment-events.listener.ts`, with `onPaymentCaptured()` for `payment.captured` and `onPaymentRefunded()` for `payment.refunded`. Two modules that listen to the same event never declare the same class, and everything a module does in reaction to another module is in one file. A module listening to its own events uses `{Module}EventsListener`.
- **Internal routes put `Internal` right after the module name**: `OrderInternalController` for `/internal/orders`, `OrderInternalIdParamDto`, `OrderInternalFile`; DTOs keep their verb first, `CreateOrderInternalDto` (see `api-dto-naming`). The module name still leads every name, and `grep Internal` lists the internal surface.
- **One name per resource, in every form.** The module folder carries the resource's full name, and nothing in it uses a shorter or longer variant: `GiftCard` lives in `gift-card/`, never `card/` or `voucher/`; a `gift-card` module does not hold a `Card` entity next to a `GiftCardIdParamDto`.
- **A query or param piece used only inside the module takes the prefix** (`OrderSortQueryDto`, `OrderLangQueryDto`). One reused by several modules stays in the module whose data it describes, with that module's prefix, and the others import it from there. Only a piece about no module's data, such as pagination, is the api module's, with the `Api` prefix (`ApiPaginationQueryDto` in `api/dtos/api-pagination.query.dto.ts`).
- **A route that nests this module's resource under another module's resource** gets a param DTO named for the route, parent first, the same way `OrderItemIdParamDto` names `/orders/:id/items/:itemId`: `OrderShipmentIdParamDto` in `shipment/` for `/orders/:id/shipments/:shipmentId`, next to `ShipmentIdParamDto` for `/shipments/:id`.
- **The prefix appears once.** A name that already starts with the module name does not repeat it: `OrderItem`, never `OrderOrderItem`. Param DTOs follow `api-dto-naming` and are named for the deepest resource (`OrderItemIdParamDto`), not for every resource in the route.
- **Everything in `services/` is a service**: its class ends in `Service` and its file in `.service.ts`, whatever it does (`UserPasswordHasherService` in `user/services/user-password-hasher.service.ts`, never `UserPasswordHasher` in `user-password-hasher.ts`). Other injectables are named for their kind and live in that kind's folder (`guards/`, `strategies/`, `listeners/`).
- **Plain functions keep descriptive names**, which usually contain the noun already: `calculateOrderTotal()` in `utils/calculate-order-total.ts`. A prefix glued onto a verb (`orderCalculateTotal`) reads worse.
- **Constants that stay in their file** (not exported) are the implementation details of that file, named for what they hold, like local variables: `HASH_OPTIONS`, `PG_UNIQUE_VIOLATION`, `REFRESH_TOKEN_BYTES`. The prefix is for names other files see; an exported constant always takes it. A value that sets behavior clients or operators see is never one of these: it is exported from `constants/` with the prefix even while one file uses it (`AUTH_LOGIN_ACCOUNT_LIMIT` in `auth/constants/auth-rate-limits.constant.ts`, see `arch-feature-modules`).
- **A sub-resource that outgrows its parent gets its own module.** When other modules use it on its own, when it has its own lifecycle, or when its names pass three words (`OrderItemReturnLabelService`), it is a resource of its own: `return/` with `ReturnLabel`, `ReturnLabelService`.

**In a module whose existing names break this rule**, new names follow the rule (code added to a `card/` module that holds `GiftCard` is named `GiftCard…`). Do not rename existing names as part of an unrelated change: a rename touches every importer and buries the real change in the diff. List them in your summary as a separate refactor.

**Every module follows this rule, resource or capability.** A module built around a capability is named for the capability (`auth/`, `mail/`, `health/`, `logger/`, `scheduler/`), and everything it declares starts with that name, as in a resource module: `AuthController`, `AuthSessionService`, the `AuthSession` entity in table `auth_sessions`, `CreateAuthSessionDto`, `AuthJwtStrategy`, `AuthJwtGuard`, `AuthRolesGuard`, `HealthShutdownService`, `HealthUnavailableFilter`, `LoggerTypeOrmQueryLogger`. So does the api module (`src/api/`, the HTTP layer, see `test-use-testing-module`): `ApiExceptionFilter`, `ApiResponseInterceptor`, `ApiPaginationQueryDto`, `ApiRequest`, `API_VALIDATION_PIPE_OPTIONS`, in files that start with `api-`. A name that already starts with the module name keeps it once (`ApiEnvelopedResponse`, `AuthSessionService`). A route starts with its module's name too: in the plural for a resource module (`/users`, `/orders`), as it is for a capability module (`/auth/login`, `/health/live`). The one exception is a provider's callback, `POST /{provider}/events` (`/stripe/events`), named for the provider that calls it whatever module serves it (see `api-controller-conventions`).

The only names without the prefix:

- **Plain functions and decorators** are named for what they do, as Nest names its own: `calculateOrderTotal()`, `resolveRequestId()`, `@Public()`, `@Roles()`, `@AuthUser()`, `@SkipRateLimits()`.
- **Clients and adapters of an outside system** are named for that system, the way the third-party code they stand for is named: `StripeClient` in `payment/clients/stripe.client.ts`, `StripeAdapter` in `payment/adapters/stripe.adapter.ts`, `SendGridAdapter` (see `arch-feature-modules`). An AWS service also names the vendor (`AwsS3Client`, `AwsS3Adapter`), since the SDK's own clients have the plain names. The name cannot clash with another module's: an app has one client per outside system (see `arch-module-sharing`), and the folder says which module holds it. The interface an adapter implements is the module's own contract and keeps the prefix, named for its role with the `Adapter` suffix, never `Interface`: `PaymentAdapter`, `NotificationEmailAdapter`, behind the tokens `PAYMENT_ADAPTER` and `NOTIFICATION_EMAIL_ADAPTER` (see `di-use-interfaces-tokens`).
- **A test double of a module's interface** is named for what it is and the interface: `FakePaymentAdapter`.
- **A param DTO for a route that nests this module's resource under another module's** is named for the route, parent first: `OrderShipmentIdParamDto` in `shipment/` (above).
- `src/shared/` and `src/config/` hold no module, so their files are named for what they hold (`paginated-result.type.ts`, `db.config.ts`).

**Incorrect (generic names that hide the owner and collide across modules):**

```typescript
src/order/
├── entities/
│   ├── order.entity.ts
│   └── item.entity.ts               // Item: whose item? the catalog module has items too
├── services/
│   ├── orders.service.ts            // plural
│   ├── item.service.ts
│   └── pricing.service.ts           // PricingService: also the name of the pricing module's service
├── dtos/
│   └── create-item.dto.ts
├── types/
│   └── status.enum.ts               // Status: imported elsewhere, says nothing
└── order.module.ts

@Injectable()
export class ItemService {
  constructor(@InjectRepository(Item) private readonly repo: Repository<Item>) {}
}
```

**Correct (the module name starts every name it owns):**

```typescript
src/order/
├── controllers/
│   ├── order.controller.ts          // /orders
│   └── order-item.controller.ts     // /orders/:id/items
├── entities/
│   ├── order.entity.ts              // table orders
│   └── order-item.entity.ts         // table order_items
├── services/
│   ├── order.service.ts
│   ├── order-item.service.ts
│   └── order-pricing.service.ts
├── dtos/
│   ├── create-order-item.dto.ts
│   └── order-item-id.param.dto.ts
├── types/
│   └── order-status.enum.ts
├── utils/
│   └── calculate-order-total.ts
└── order.module.ts

@Injectable()
export class OrderItemService {
  constructor(
    @InjectRepository(OrderItem) private readonly orderItemRepository: Repository<OrderItem>,
    private readonly orderPricingService: OrderPricingService,
  ) {}
}
```
