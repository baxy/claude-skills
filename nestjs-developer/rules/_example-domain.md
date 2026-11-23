# Example domain

Every rule example uses one domain: the API of an online shop. Keep new examples inside it, so a reader moving between rules meets the same resources, fields and names. The examples illustrate a rule; they are not a schema for a project to copy.

| Module | Resources | Examples about |
|---|---|---|
| `user` | `User`: `id`, `email`, `name`, `phone`, `passwordHash`, `roles`, `deactivatedAt`, `createdAt`, `updatedAt`, `deletedAt` | serialization, passwords, mapped types, avatar uploads, account deactivation |
| `auth` | sessions (`AuthSession`, one per login) and refresh tokens, through `POST /auth/login`, `/auth/refresh` and `/auth/logout`; `JwtUser`: `id`, `email`, `roles`, `sessionId` | authentication, guards |
| `order` | `Order`: `id`, `userId`, `status`, `cancelReason`, `totalCents`, `currency`, `shippingAddress` (`jsonb`), `items`, `shippedAt`, `createdAt`, `updatedAt`, `deletedAt`. `OrderItem`: `id`, `orderId`, `productId`, `sku`, `quantity`, `unitPriceCents` (copied from the product when the order is placed) | most rules, invoice PDFs |
| `product` | `Product`: `id`, `name`, `sku`, `priceCents`, `categoryId`, `deletedAt`. `ProductReview`, whose body is rich text, one per user and product | relations, N+1, sanitization |
| `category` | `Category`: `id`, `key`, `name` | caching, list filters |
| `inventory` | `InventoryItem`: `sku`, `quantity` (table `inventory_items`) | conditional updates in a transaction |
| `gift-card` | `GiftCard`: `id`, `balanceCents`. `GiftCardRedemption`: `giftCardId`, `orderId`, `amountCents` | balances, retried transactions |
| `payment` | `PaymentAdapter`, implemented by `StripeAdapter` and `FakePaymentAdapter`; `StripeClient` | interfaces, tokens, test doubles |
| `storage` | `StorageAdapter` over the environment's one bucket, a folder per kind of file (`StorageFolder.ProductImages`, `UserAvatars`, `Reports`), implemented by `AwsS3Adapter`; `AwsS3Client`, private to the module | one client per outside system, SDK clients, file storage |
| `shipment` | `Shipment`: `id`, `orderId`, `trackingNumber`; tracking it through `CarrierClient` | outgoing HTTP calls |
| `notification` | notifications pushed over SSE or a gateway | real-time updates |
| `report` | reports a queue worker renders | background jobs |
| `recommendation`, `mail`, `webhook` | event consumers, mail, customer webhooks | events, config, SSRF |

- `OrderStatus`: `Pending`, `Paid`, `Shipped`, `Canceled` (US spelling, see `style-us-english`). An order can be canceled while `Pending` or `Paid`. A declined payment leaves it `Pending`.
- Money is an integer in the currency's smallest unit, and its name says so: `totalCents`, `unitPriceCents`, `priceCents`, `balanceCents`, `amountCents` (see `style-code-comments`).
- Ids are UUIDs. Tables are plural snake_case: `orders`, `order_items`, `gift_card_redemptions`.
- Routes: `/orders`, `/orders/:id`, `/orders/:id/cancel`, `/orders/:id/items/:itemId`, `/orders/:id/shipments/:shipmentId`, `/products/:id/reviews`, `/users`, `/users/:id`, `/auth/login`, `/auth/me`, `/notifications/stream`, `/shipments/by-tracking-number/:trackingNumber` (the shop's spec asks for a lookup by tracking number, among the caller's own orders). In a nested route the parent's id is `:id` and the child's is named for it (`:itemId`).
- Events: `order.created`, `order.paid`.
- Test data: the user `jane@example.com`; UUIDs that pass `@IsUUID()`, `00000000-0000-4000-8000-000000000001` and up (see `test-use-testing-module`).
