---
title: Push Updates with SSE, or with an Authenticated WebSocket Gateway
impact: MEDIUM
impactDescription: Live updates reach the right user on any instance, authenticated, without breaking the HTTP setup or the shutdown
tags: api, real-time, sse, websockets, socket.io, redis
---

## Push Updates with SSE, or with an Authenticated WebSocket Gateway

**Use SSE when only the server sends** (notifications, progress, a live feed): an `@Sse()` route is plain HTTP, so the global guards, CORS, rate limiting and the load balancer treat it like any route. **Use a WebSocket gateway only when the client also sends messages over the connection** (chat, collaborative editing). Do not poll for updates.

Both need three things the HTTP setup does not give them:

- **Updates raised on any instance.** An update (a domain event, see `arch-use-events`) happens on one instance, while the user's connection may be on another. SSE streams read from a Redis channel every instance subscribes to; a gateway uses Socket.IO's Redis adapter.
- **A Redis outage they survive.** The clients that subscribe (the stream's subscriber, the adapter's two) keep their commands waiting for the reconnect (`maxRetriesPerRequest: null`): with ioredis's default of 20 retries, a subscribe or publish they send fails about 10 seconds into an outage, and the Socket.IO adapter never catches it, so the crash handler ends the process (see `devops-use-logging`). ioredis subscribes again by itself after a reconnect. The Socket.IO adapter also never handles the promises of its commands, so the ones still waiting when its clients disconnect at shutdown fail with "Connection is closed." and nothing to catch them, and the crash handler ends the shutdown with exit 1: its clients are a `Redis` subclass that handles every command's promise, and logs a failure other than that one. The redis module builds every one of these connections, as it builds the client the rate limits use, and closes them once the gateways are closed (see `arch-module-sharing`): a stream publishes and subscribes through `RedisService`, and `main.ts` gets the adapter's pair from `RedisService.createAdapterClients()`.
- **An end and a limit per user.** A connection lasts as long as its access token: it ends when the token expires, and the client reconnects with a fresh one. Without that, a stolen token keeps a stream open, through a logout and an account deletion. An SSE route reads the expiry with `@AuthTokenExpiresAt()` in `api/decorators/`, from the token the global guard has already verified. Each user holds at most a few open streams and sockets together (5 in the example, a constant in the module's `constants/`), counted on each instance when one opens and released when it ends: what a user holds open costs that instance's memory and sockets, and a count shared through Redis would be left too high by an instance that crashed. A stream past the limit gets 429, and a socket past it is disconnected.
- **A shutdown that ends them.** `app.close()` waits for every open HTTP connection, and a stream never ends by itself. Once the drain is over (see `devops-graceful-shutdown`), the stream service completes every stream and refuses a new one with 503, and the gateway disconnects its clients, both in `beforeApplicationShutdown`; the clients reconnect to another instance. Not `forceCloseConnections: true`, which also cuts the requests still running.
- **Global interceptors that step aside.** They also run on SSE and gateway handlers. `ApiResponseInterceptor` and `ApiLoggerInterceptor` (see `api-use-interceptors`) return `next.handle()` unchanged for a context that is not HTTP, and the envelope also for an SSE handler: wrapped in the envelope, an SSE event reaches the client as `{"data": ...}`, without its type or id.

```typescript
// api/interceptors/api-response.interceptor.ts
intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
  if (context.getType() !== 'http' || this.reflector.get<boolean>(SSE_METADATA, context.getHandler())) {
    return next.handle();
  }
  // ... the envelope
}
```

The interceptor is an api module provider, so it injects `Reflector`, and `ApiModule.setup()` registers it with `app.get(ApiResponseInterceptor)` (see `test-use-testing-module`); `SSE_METADATA` comes from `@nestjs/common/constants`, and the envelope is the one in `api-use-interceptors`.

### SSE

- The handler returns `Observable<MessageEvent>`; Nest unsubscribes when the client disconnects.
- The browser's `EventSource` cannot send an `Authorization` header, so clients read the stream with a fetch-based SSE client that sends the bearer token. Never put the token in the query string: it ends up in access logs.

```typescript
// notification/controllers/notification.controller.ts
@Controller('notifications')
export class NotificationController {
  constructor(private readonly notificationStreamService: NotificationStreamService) {}

  @Sse('stream')
  stream(@AuthUser() authUser: JwtUser, @AuthTokenExpiresAt() tokenExpiresAt: Date): Observable<MessageEvent> {
    return this.notificationStreamService.streamFor(authUser.id, tokenExpiresAt);
  }
}

// notification/services/notification-stream.service.ts
type NotificationUpdate = { userId: string; type: string; data: object };

const UPDATES_CHANNEL = 'notification-updates';

@Injectable()
export class NotificationStreamService implements OnModuleInit, BeforeApplicationShutdown {
  private readonly updates$ = new Subject<NotificationUpdate>();
  private isShuttingDown = false;

  constructor(private readonly redisService: RedisService) {}

  onModuleInit(): void {
    this.redisService.subscribe(UPDATES_CHANNEL, (message) => this.updates$.next(JSON.parse(message) as NotificationUpdate));
  }

  async publish(update: NotificationUpdate): Promise<void> {
    await this.redisService.publish(UPDATES_CHANNEL, JSON.stringify(update));
  }

  streamFor(userId: string, tokenExpiresAt: Date): Observable<MessageEvent> {
    if (this.isShuttingDown) {
      throw new ServiceUnavailableException('The server is shutting down; reconnect to receive notifications');
    }
    return this.updates$.pipe(
      filter((update) => update.userId === userId),
      map((update) => ({ type: update.type, data: update.data })),
      takeUntil(timer(tokenExpiresAt)),
    );
  }

  beforeApplicationShutdown(): void {
    this.isShuttingDown = true;
    this.updates$.complete();
  }
}

// redis/clients/redis-pub-sub.client.ts
const logger = new Logger('Redis');
// What ioredis rejects waiting commands with on disconnect
const CONNECTION_CLOSED_MESSAGE = 'Connection is closed.';

// The Socket.IO adapter never handles its commands' promises: one rejected at the shutdown's disconnect would end
// the process
export class RedisPubSubClient extends RedisClient {
  override sendCommand(...args: Parameters<Redis['sendCommand']>): unknown {
    args[0].promise.catch((error: unknown) => {
      if (!(error instanceof Error && error.message === CONNECTION_CLOSED_MESSAGE)) {
        logger.warn({ err: error }, 'A Redis command failed');
      }
    });
    return super.sendCommand(...args);
  }
}

// redis/services/redis.service.ts
@Injectable()
export class RedisService implements OnApplicationShutdown {
  private readonly logger = new Logger(RedisService.name);
  private readonly handlers = new Map<string, RedisMessageHandler[]>();
  private readonly adapterClients: Redis[] = [];

  constructor(
    @Inject(REDIS_CLIENT) private readonly redisClient: Redis,
    // A subscribed connection can do nothing else
    @Inject(REDIS_SUBSCRIBER) private readonly redisSubscriber: Redis,
    @Inject(redisConfig.KEY) private readonly redisSettings: ConfigType<typeof redisConfig>,
  ) {
    // Here, not in onModuleInit: a subscription made in another module's onModuleInit is not missed
    this.redisSubscriber.on('message', (channel: string, message: string) => {
      for (const handler of this.handlers.get(channel) ?? []) {
        handler(message);
      }
    });
    this.redisSubscriber.on('ready', () => {
      for (const channel of this.handlers.keys()) {
        this._subscribe(channel);
      }
    });
  }

  async publish(channel: string, message: string): Promise<void> {
    await this.redisClient.publish(channel, message);
  }

  // Not awaited: the API serves while Redis is down, and the channel is subscribed again at each reconnection
  subscribe(channel: string, handler: RedisMessageHandler): void {
    const handlers = this.handlers.get(channel);
    if (handlers) {
      handlers.push(handler);
      return;
    }
    this.handlers.set(channel, [handler]);
    if (this.redisSubscriber.status === 'ready') {
      this._subscribe(channel);
    }
  }

  // The Socket.IO Redis adapter publishes and subscribes on connections of its own
  createAdapterClients(): RedisAdapterClients {
    const options = buildRedisPubSubOptions(this.redisSettings);
    const clients = {
      publisher: new RedisPubSubClient(options, 'the gateways (publisher)'),
      subscriber: new RedisPubSubClient(options, 'the gateways (subscriber)'),
    };
    this.adapterClients.push(clients.publisher, clients.subscriber);
    return clients;
  }

  // disconnect(), not quit(): quit() waits for a Redis that is down, and the shutdown with it
  onApplicationShutdown(): void {
    for (const client of [...this.adapterClients, this.redisSubscriber, this.redisClient]) {
      client.disconnect();
    }
  }

  private _subscribe(channel: string): void {
    this.redisSubscriber.subscribe(channel).catch((error: unknown) => {
      this.logger.warn({ err: error }, `Subscribing to ${channel} failed; it is tried again when Redis is back`);
    });
  }
}
```

Every instance subscribes to one channel, so a stream receives the updates raised on any instance, and it ends when the caller's access token expires. `RedisPubSubClient` serves every connection that publishes or subscribes; `RedisClient`, which it extends, logs when Redis becomes unavailable and when it is back, not every reconnection attempt. `RedisService` is the redis module's public face: `REDIS_SUBSCRIBER` is a `RedisPubSubClient` the module provides and does not export, and `buildRedisPubSubOptions()` sets `maxRetriesPerRequest: null`. A publish fails at once while Redis is down, and the message is lost. The stream service completes every stream in `beforeApplicationShutdown()`, before the server waits for its open connections, and answers 503 to a stream opened after that. `RedisService.onApplicationShutdown()` runs once the HTTP server and the gateways are closed.

### WebSocket gateways

- Socket.IO through `@nestjs/platform-socket.io`. The gateway lives in the module's `gateways/` folder (`notification/gateways/notification.gateway.ts`, `NotificationGateway`) and only routes: what a message does belongs to a service (see `arch-single-responsibility`). Domain events reach it through a listener (see `arch-use-events`).
- **The connection is authenticated, not each message.** `handleConnection` checks the token from `client.handshake.auth.token` (never the query string) with the auth module's `AuthTokenService.verify()`, which verifies it the way `AuthJwtStrategy` does: the same `algorithms`, `issuer` and `audience`, and the same claims check, through `toJwtUser()` in `auth/utils/` (see `security-auth-jwt`). It returns the `JwtUser` and the token's expiry; the gateway never sees `JwtService`, which stays inside the auth module (see `arch-module-sharing`). The gateway keeps the `JwtUser` in `client.data.authUser` and disconnects a client whose token is missing or invalid. The connection ends when the token expires; the client reconnects with a fresh one. A role check for a message reads `client.data.authUser`.
- **The global guards skip every context that is not HTTP.** `APP_GUARD`s run on every `@SubscribeMessage` handler and read an HTTP request: `AuthJwtGuard` and `AuthRolesGuard` return `true` first thing when `context.getType() !== 'http'`, and the throttler guard skips such a context in `shouldSkip()`. A gateway that accepts messages from clients limits their rate itself, in a guard on the gateway that counts every message and checks the sender's role: guards run before pipes, so a count after validation lets any number of invalid messages through, each one validated and answered. The global `ValidationPipe` reaches gateways (see below); the gateway declares none of its own.
- **Errors go out as WebSocket errors.** Global filters do not run on gateways, but the global `ValidationPipe` does, and its `BadRequestException` would reach the client as "Internal server error". The gateway's `ApiWsExceptionFilter` sends an HTTP exception's response object instead, and reports any other error to Sentry (see `error-report-to-sentry`).
- **Every instance shares one Redis** through `ApiRedisIoAdapter`, set in `main.ts`. With `transports: ['websocket']` on the server and the client, no sticky sessions are needed. CORS comes from the `app` config in the adapter: the gateway decorator cannot read config.
- **Only the gateways' namespaces take connections.** Socket.IO's default namespace (`/`) accepts any client and keeps it, whatever the gateways check: the adapter refuses connections to it.

```typescript
// notification/gateways/notification.gateway.ts
@UseFilters(ApiWsExceptionFilter)
@WebSocketGateway({ namespace: 'notifications', transports: ['websocket'] })
export class NotificationGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  private readonly server: Namespace;

  constructor(
    private readonly notificationService: NotificationService,
    private readonly authTokenService: AuthTokenService,
  ) {}

  async handleConnection(client: Socket): Promise<void> {
    try {
      const { authUser, expiresAt } = await this.authTokenService.verify(String(client.handshake.auth.token));
      client.data.authUser = authUser;
      await client.join(`user:${authUser.id}`);
      client.data.expiryTimer = setTimeout(() => client.disconnect(true), expiresAt.getTime() - Date.now());
    } catch {
      client.disconnect(true);
    }
  }

  handleDisconnect(client: Socket): void {
    clearTimeout(client.data.expiryTimer);
  }

  @SubscribeMessage('notification.read')
  markAsRead(@ConnectedSocket() client: Socket, @MessageBody() dto: MarkNotificationReadDto): Promise<NotificationResponseDto> {
    return this.notificationService.markAsRead(client.data.authUser, dto.id);
  }

  sendToUser(userId: string, event: string, payload: object): void {
    this.server.to(`user:${userId}`).emit(event, payload);
  }
}

// notification/listeners/notification-order-events.listener.ts
@Injectable()
export class NotificationOrderEventsListener {
  constructor(private readonly notificationGateway: NotificationGateway) {}

  @OnEvent('order.paid')
  onOrderPaid(event: OrderPaidEvent): void {
    this.notificationGateway.sendToUser(event.userId, 'order.paid', { orderId: event.orderId });
  }
}

// api/filters/api-ws-exception.filter.ts
@Catch()
export class ApiWsExceptionFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    if (exception instanceof HttpException) {
      // BaseWsExceptionFilter answers "Internal server error" for anything but a WsException
      super.catch(new WsException(exception.getResponse()), host);
      return;
    }
    if (!(exception instanceof WsException)) {
      SentryService.reportServerError(exception);
    }
    super.catch(exception, host);
  }
}

// auth/guards/auth-jwt.guard.ts
if (context.getType() !== 'http') {
  // Only an HTTP request carries the bearer token: a WebSocket client is authenticated when it connects
  return true;
}

// auth/guards/auth-throttler.guard.ts
protected async shouldSkip(context: ExecutionContext): Promise<boolean> {
  return context.getType() !== 'http' || super.shouldSkip(context);
}

// api/adapters/api-redis-io.adapter.ts
export class ApiRedisIoAdapter extends IoAdapter {
  constructor(app: INestApplicationContext, private readonly redisClients: RedisAdapterClients, private readonly corsOrigins: string[]) {
    super(app);
  }

  createIOServer(port: number, options?: ServerOptions): Server {
    const server: Server = super.createIOServer(port, { ...options, cors: { origin: this.corsOrigins } } as ServerOptions);
    server.adapter(createAdapter(this.redisClients.publisher, this.redisClients.subscriber));
    // Socket.IO's default namespace accepts any client, whatever the gateways check
    server.of('/').use((socket, next) => next(new Error('Unknown namespace')));
    return server;
  }
}

// main.ts
const app = await NestFactory.create<NestFastifyApplication>(AppModule, ApiModule.createFastifyAdapter(appConfig().trustedProxies), {
  bufferLogs: true,
});
app.useWebSocketAdapter(new ApiRedisIoAdapter(app, app.get(RedisService).createAdapterClients(), app.get(appConfig.KEY).corsOrigins));
```

`sendToUser()` reaches the user's sockets on every instance, through the Redis adapter. The context check opens `canActivate()` in `AuthJwtGuard`, and `AuthRolesGuard` starts the same way. The redis module builds the adapter's connections and closes them. `main.ts` sets the adapter, not `ApiModule.setup()`, so e2e tests need no Redis, and it sets no `forceCloseConnections`: the streams and sockets end in `beforeApplicationShutdown`.

Reference: [NestJS Server-Sent Events](https://docs.nestjs.com/techniques/server-sent-events) · [NestJS Gateways](https://docs.nestjs.com/websockets/gateways) · [NestJS WebSocket adapters](https://docs.nestjs.com/websockets/adapter) · [Socket.IO Redis adapter](https://socket.io/docs/v4/redis-adapter/)
