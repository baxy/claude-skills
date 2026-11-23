---
title: Follow Controller and REST Conventions
impact: MEDIUM-HIGH
impactDescription: Every controller reads the same way, and every endpoint follows predictable REST semantics
tags: api, controllers, rest, http, routes, naming, conventions
---

## Follow Controller and REST Conventions

Controllers are thin: they bind and validate input, delegate to the service, and return its result. All business logic and domain validation live in services (see `arch-service-conventions`). In NestJS every REST decision is made in the controller (the path in `@Controller`, the method decorator, `@HttpCode`), so the REST rules live here too.

### Naming

- The controller class and file are **singular**: `OrderController` in `order/controllers/order.controller.ts` (see `arch-resource-naming`). More controllers in a module carry the module name too: `OrderItemController`.
- The route path is **plural**, lowercase kebab-case: `@Controller('orders')`, `@Controller('orders/:id/items')`, `@Controller('payment-methods')`.

### URLs

- Collections are plural nouns; an item is the collection plus its id: `/orders`, `/orders/:id`.
- **Every route addresses an item by `:id`, and `/me` exists only in the auth module.** `GET /auth/me` returns the authenticated caller. Every other route, the user module's included, takes the item's id (`GET /users/:id`), and the service scopes it to what the caller may see (see `security-use-guards`): another customer's account answers 404, as another customer's order does. Any other way to address an item exists only when the spec asks for it, and two routes of one method never take different keys in the same place, since they match the same URLs.
- Sub-resources nest under their parent: `/orders/:id/items/:itemId`. The resource's own id is `:id`; ids below it are `:{child}Id`. A nested route must verify the child belongs to the parent and return 404 if it does not.
- **At most 4 collections deep**, counting the root: `/orders/:id/items/:itemId/adjustments/:adjustmentId/notes` is the limit. Anything deeper becomes a top-level collection filtered by its parent (`GET /notes?adjustmentId=...`).
- Filtering, sorting and pagination are query parameters, never path segments: `GET /orders?status=paid&sort=-createdAt&limit=20`, not `/orders/paid`.
- No trailing slashes, no file extensions, no verbs, except action segments on `POST` (below), `download` on `GET`, and the authentication flows (next).
- **Authentication flows are the one place for verbs.** They act on the caller's own credentials, not on a resource the client addresses, and their short names are what clients expect. One controller of the auth module serves the flows the app has, each a `POST` named for its flow, and nothing else uses verbs. The same controller serves `GET /auth/me`. The shop's flows, served by its `AuthController` (`@Controller('auth')`), as an example:

  | Route | Does | Answers |
  |---|---|---|
  | `POST /auth/login` | Checks the credentials and starts a session | 200, the access and refresh tokens |
  | `GET /auth/me` | Returns the authenticated caller | 200, the caller's account |
  | `POST /auth/refresh` | Exchanges a refresh token for a new pair | 200, the new pair |
  | `POST /auth/logout` | Ends the caller's session | 204 |
  | `POST /auth/forgot-password` | Sends a reset link, whether or not the email exists | 202 |
  | `POST /auth/reset-password` | Sets a new password with the link's token | 204 |

  Handlers are named for the flow (`login`, `refresh`, `logout`, `forgotPassword`, `resetPassword`), and `GET /auth/me`'s is `findMe`. `GET /auth/me` reads the caller's account through the user module's service (`userService.findById(authUser, authUser.id)`): the auth module answers who is calling, and the user's data stays the user module's.

### HTTP methods

| Method | Use for | Success status |
|---|---|---|
| `GET` | Read a collection or an item. Always safe: no side effects, whatever the last segment says. | 200 |
| `POST /{resources}` | Create with a **server-assigned id**; repeating the request creates another resource. | 201 with the created resource; 202 when the work behind it is queued (see `micro-use-queues`) |
| `POST /{resources}/:id/{verb}` | Run an **action**: a command with business rules or side effects that is not a plain field change (`cancel`, `ship`, `refund`). Body only when the action has inputs (`{ reason }`); no body otherwise. | 200 with the updated resource, 202 if processed asynchronously, 204 if nothing to return |
| `POST /{resources}/{verb}` | Run an **action on the collection that stores nothing**: a check or dry run of what a create would do, with the create's body (`POST /product-imports/check` answers a file's row errors without importing it), or what a browser needs to reach the API, such as a session cookie (see `micro-use-queues`). No resource is created, so it is not a `POST /{resources}`. | 200 with the result; 204 when the answer is only a header or a cookie |
| `POST /{provider}/events` | A provider's callback (`/stripe/events`): see below. | 204 for every event it accepted or already handled; 400 for a bad signature |
| `POST /{resources}/search` | Search whose criteria do not fit a query string (too long, nested, or sensitive). No side effects; document it. Prefer `GET` with query parameters otherwise. | 200 |
| `PUT` | **Idempotent** create-or-replace at a known target: full replacement, upsert, or "ensure exists". | 201 when created; 200 when replaced or already present, or a documented error (typically 409) when the business rule rejects a repeat |
| `PATCH /{resources}/:id` | Partial update of plain fields. State transitions go through actions instead. | 200 with the updated resource |
| `DELETE /{resources}/:id` | Delete (or soft-delete). Idempotent. | 204 |

**Choosing between `POST` and `PUT` for creation** - ask: if the client sends the same request twice, does the server end up with one thing or two?

- **Two** (a new id each time): `POST`. `POST /orders` creates a second order.
- **One** (create or replace at a known target, or make sure something exists): `PUT` is allowed. `PUT /users/:id/avatar`, `PUT /orders/:id/shipping-address`, `PUT /products/:id/reviews/mine` when the spec asks for one review per caller and product and a repeat does not create another (below).
- **Idempotent means no second resource, not an identical response.** HTTP defines idempotency by the effect on the server (RFC 9110), so a repeated creating `PUT` must never create another resource. What the repeat *returns* is a business decision; document it on the endpoint:
  - **Return the existing resource (200)** when a repeat is harmless: "make sure the avatar exists", or a client retrying after a timeout. The status differs per call, so the handler sets it through the adapter (see `api-platform-agnostic`) and returns the resource in both cases.
  - **Return an error (typically 409 Conflict)** when the caller needs to know the resource already existed: creating a draft version while one is already open, so the client decides whether to continue the existing draft.
- A `PUT` that sets a value the client already holds and creates nothing a client reads back (a new password) answers 204, as a body-less change does; a `PUT` that may create something (an item on a list, an avatar) answers 201 or 200 with it.
- A `PUT` that creates goes to a URL that names the item, never to the collection's URL, which reads like "replace the whole collection" to generic HTTP clients. A named segment such as `mine` (`PUT /products/:id/reviews/mine`) exists only when the spec asks for it (see above).

**Downloads**: `GET /{resources}/:id` returns metadata and `GET /{resources}/:id/download` returns the content, with `Content-Disposition: attachment` or a redirect to a presigned URL. It is still a safe `GET`: recording an audit entry is fine, changing state is not.

**A provider's callback** (Stripe's events) sits outside the resource paths, at `POST /{provider}/events`, never in an item's slot (`/orders/payment-webhook` would take `payment-webhook` for an order id). It is `@Public()`, `VERSION_NEUTRAL` (one stable path the provider is configured with, see `api-versioning`) and out of Swagger (`@ApiExcludeController()`), with a rate limit of its own (see `security-rate-limiting`). It checks the provider's signature over the raw body (`rawBody: true` in `NestFactory.create()`): a parsed and serialized body no longer matches. It is idempotent by the event's id, and answers 2xx for an event it already handled or does not act on, so the provider stops sending it.

**Error statuses**: 400 invalid input or a broken domain rule (such as an illegal state transition), 401 unauthenticated, 403 not allowed, 404 missing (or not visible to the caller), 409 the request clashes with existing data (a duplicate, a concurrent change).

### Handlers

- **Handler names**: `findAll`, `findById`, `findAll{SubResources}` (`findAllItems`), `create`, `update` (PATCH), `replace` (PUT replacement), `upsert` or `create{Thing}` (idempotent PUT creation), `delete`, `search`, `download`, and the action's verb (`cancel`, `ship`). A handler for a singular sub-resource of the item adds its name: `update{Thing}` (PATCH `/:id/profile`), `replace{Thing}` (PUT `/:id/shipping-address`).
- **Parameter order**: the authenticated user (`@AuthUser() authUser: JwtUser`, see `api-custom-param-decorators`), then `@Param()`, then `@Query()` or `@Body()`, then optional extras.
- **DTO classes for `@Param()`, `@Query()` and `@Body()`** - no inline extraction (see `api-use-pipes`, `api-dto-naming`).
- **Status codes are explicit** where Nest's default differs from the table: `@HttpCode(HttpStatus.NO_CONTENT)` on `DELETE`, `@HttpCode(HttpStatus.OK)` on actions (on an item or on the collection) and `POST .../search`, `@HttpCode(HttpStatus.ACCEPTED)` when the work is queued (Nest defaults every `POST` to 201).
- **`@ApiFormDataRequest()`** on endpoints that accept file uploads: the api module's decorator, which passes its options to `@FormDataRequest()` (see `api-file-uploads`).
- **Return service results directly** - no wrappers in the controller; the response envelope comes from a global interceptor (see `api-use-interceptors`).
- Swagger decorators per `api-document-with-openapi`.

**Incorrect (plural class, verbs in paths, wrong methods, logic in the controller):**

```typescript
@Controller('order')                                       // Singular path
export class OrdersController {                            // Plural class
  @Get('getOrder/:id')                                     // Verb in a GET path, camelCase
  async getOrder(@Param('id') id: string, @Req() req: Request) {
    const order = await this.orderRepository.findOne({ where: { id } }); // Data access in the controller
    return { data: order };                                // Ad-hoc wrapper
  }

  @Get(':id/cancel')                                       // State change on GET
  cancel(@Param('id') id: string) { /* ... */ }

  @Patch(':id')
  update(@Param() params: OrderIdParamDto, @Body() dto: UpdateOrderDto) {
    return this.orderService.update(params.id, dto);       // dto.status: transitions hidden in PATCH
  }

  @Put()                                                   // PUT that creates a new order on every call
  create(@Body() dto: CreateOrderDto) { /* ... */ }

  @Get('status/:status')                                   // Filter as a path segment
  findByStatus(@Param('status') status: string) { /* ... */ }

  @Delete(':id')                                           // 200 instead of 204
  async remove(@Body() dto: DeleteOptionsDto, @Param('id') id: string) {
    await this.orderService.delete(id, dto);
    return { deleted: true };
  }
}
```

**Correct (singular class, plural kebab-case routes, REST semantics):**

```typescript
@ApiTags('Orders')
@ApiBearerAuth('bearer')
@Controller('orders')
export class OrderController {
  constructor(
    private readonly orderService: OrderService,
    private readonly httpAdapterHost: HttpAdapterHost,
  ) {}

  @ApiOperation({ summary: 'List orders', description: 'Returns a page of the caller\'s orders, newest first.' })
  @Get()
  findAll(@AuthUser() authUser: JwtUser, @Query() query: FindAllOrdersQueryDto) {
    return this.orderService.findAll(authUser, query);
  }

  @ApiOperation({ summary: 'Search orders', description: 'Searches orders with criteria too complex for a query string.' })
  @HttpCode(HttpStatus.OK)
  @Post('search')
  search(@AuthUser() authUser: JwtUser, @Body() dto: SearchOrdersDto) {
    return this.orderService.search(authUser, dto);
  }

  @ApiOperation({ summary: 'Get an order', description: 'Returns one of the caller\'s orders.' })
  @Get(':id')
  findById(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto) {
    return this.orderService.findById(authUser, params.id);
  }

  @ApiOperation({ summary: 'Download the invoice', description: 'Returns the order invoice as a PDF attachment.' })
  @Get(':id/download')
  async download(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto): Promise<StreamableFile> {
    const pdf = await this.orderService.renderInvoice(authUser, params.id);
    return new StreamableFile(pdf, {
      type: 'application/pdf',
      disposition: `attachment; filename="invoice-${params.id}.pdf"`,
    });
  }

  @ApiOperation({ summary: 'Create an order', description: 'Creates a pending order for the caller.' })
  @Post()
  create(@AuthUser() authUser: JwtUser, @Body() dto: CreateOrderDto) {
    return this.orderService.create(authUser, dto);
  }

  @ApiOperation({ summary: 'Update an order', description: 'Updates the note and delivery instructions.' })
  @Patch(':id')
  update(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto, @Body() dto: UpdateOrderDto) {
    return this.orderService.update(authUser, params.id, dto);
  }

  @ApiOperation({
    summary: 'Set the shipping address',
    description: 'Creates the shipping address (201) or replaces it (200), and returns it.',
  })
  @Put(':id/shipping-address')
  async replaceShippingAddress(
    @AuthUser() authUser: JwtUser,
    @Param() params: OrderIdParamDto,
    @Body() dto: OrderAddressDto,
    @Res({ passthrough: true }) response: unknown,
  ): Promise<OrderShippingAddress> {
    const { address, created } = await this.orderService.replaceShippingAddress(authUser, params.id, dto);
    this.httpAdapterHost.httpAdapter.status(response, created ? HttpStatus.CREATED : HttpStatus.OK);
    return address;
  }

  @ApiOperation({ summary: 'Cancel an order', description: 'Cancels a pending or paid order.' })
  @HttpCode(HttpStatus.OK)
  @Post(':id/cancel')
  cancel(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto, @Body() dto: CancelOrderDto) {
    return this.orderService.cancel(authUser, params.id, dto);
  }

  @ApiOperation({ summary: 'Delete an order', description: 'Soft-deletes the order and its items.' })
  @HttpCode(HttpStatus.NO_CONTENT)
  @Delete(':id')
  delete(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto): Promise<void> {
    return this.orderService.delete(authUser, params.id);
  }
}
```

`SearchOrdersDto` is the search body `api-dto-naming` describes. `UpdateOrderDto` has no `status` field, so a status changes only through an action; the cancel action's `CancelOrderDto` holds an optional `reason`. The shipping address `PUT` answers 201 when it created the address and 200 when it replaced it, so it sets the status per call through the adapter (see `api-platform-agnostic`) and returns the address either way.

A second controller in the module is named for its resource and serves the nested route:

```typescript
@ApiTags('Order items')
@ApiBearerAuth('bearer')
@Controller('orders/:id/items')
export class OrderItemController {
  constructor(private readonly orderItemService: OrderItemService) {}

  @ApiOperation({ summary: 'List order items', description: 'Returns the items of one of the caller\'s orders.' })
  @Get()
  findAll(@AuthUser() authUser: JwtUser, @Param() params: OrderIdParamDto) {
    return this.orderItemService.findAll(authUser, params.id);
  }

  @ApiOperation({ summary: 'Get an order item', description: 'Returns 404 if the item is not part of the order.' })
  @Get(':itemId')
  findById(@AuthUser() authUser: JwtUser, @Param() params: OrderItemIdParamDto) {
    return this.orderItemService.findById(authUser, params.id, params.itemId);
  }
}
```

Reference: [NestJS Controllers](https://docs.nestjs.com/controllers) · [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/) · [RFC 9110 method semantics](https://www.rfc-editor.org/rfc/rfc9110#name-method-definitions)
