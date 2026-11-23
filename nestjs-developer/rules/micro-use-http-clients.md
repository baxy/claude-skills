---
title: Call Outside Systems Through Clients with Timeouts and Mapped Errors
impact: HIGH
impactDescription: A slow or failing provider costs a request a clear 503, not the service its connections, and its logs never get the provider's key
tags: microservices, http, clients, axios, timeouts, ssrf
---

## Call Outside Systems Through Clients with Timeouts and Mapped Errors

Every HTTP call to another system goes through a client in the module's `clients/` folder (see `arch-feature-modules`). A REST API without an official SDK, a provider's or another of our services, is called with `HttpService` from `@nestjs/axios`. The module registers its own `HttpModule` with `registerAsync`, from the provider's config namespace (see `devops-use-config-module`): the base URL, the key and the timeout live there, never in the call. Tests replace `HttpService` (see `test-mock-external-services`).

**A provider with an official SDK (AWS, Stripe) is called through its SDK**, never through hand-built requests. Signing and building requests by hand is code to maintain and to get wrong, and it gives up what the SDK handles: AWS's credential chain, for one, which finds IAM roles and temporary credentials, where hand-signed requests only work with long-lived keys. The SDK's client is a provider of the module, built in a factory from the provider's config namespace and injected by its class (`S3Client`, `Stripe`); a token only when the SDK exports no class to inject. The module's client in `clients/` wraps it with the same habits as below. The request id is not sent: only our own services read it.

- **Every client has a timeout.** The AWS SDK's is `requestHandler: { connectionTimeout, requestTimeout, throwOnRequestTimeout: true }`: without the last, a request past its timeout only logs a warning and keeps waiting. axios has none (`timeout: 0`), so a provider that stops answering holds the request, and whoever waits for it, until the socket closes. `timeout` aborts a call whose socket stays silent that long.
- **The request id goes along.** Every call sends `x-request-id` with the request's id (`cls.getId()`). Another of our services takes it as its own request id (see `devops-use-logging`), so one search finds the call on both sides.
- **The client maps failures to HTTP exceptions.** No answer (a timeout, a refused or reset connection), a 5xx or a 429 means the provider is unavailable: `ServiceUnavailableException` (503) with the error as its `cause`, and a 429's `Retry-After` as `retryAfterSeconds` (see `error-throw-http-exceptions`). Any other failure means our request was wrong: `InternalServerErrorException`, with the cause. A status the provider documents as an answer (a 404 for an unknown tracking number) comes back with its body through `validateStatus`, for the adapter or the service to interpret (see `arch-feature-modules`). For a call with a side effect that must not happen twice (a charge, a transfer, a message sent, a label created), every 4xx but a 429 is such an answer: the provider refused the request and did nothing, which the caller records and never sends again (see `db-use-transactions`), while a 500 would leave it unable to tell a refusal from a call that may have gone through. No `AxiosError` leaves the client: it carries the request's headers and body, and the logs keep only the message and stack of a cause.
- **No retries inside a request.** The caller is waiting, and a retried POST can charge twice. SDKs retry on their own (the AWS SDK makes 3 attempts by default, Stripe's 3), so turn it off (`maxAttempts: 1`, `maxNetworkRetries: 0`). Some still send a call again once when its connection closed under it (Stripe's Node SDK does, whatever the setting), so a call with a side effect that must not happen twice always carries an idempotency key. Work that must eventually succeed goes through a queue job, whose retries back off (see `micro-use-queues`).
- **An idempotency key lasts as long as the provider keeps it.** Stripe removes a key once it is at least 24 hours old, and a request sent again with it after that is a new request. A call sent again with its key stops before then: the adapter declares the lifetime, and the caller marks what is still unknown past it for a person (see `db-use-transactions`).
- **What the client is told is ours to choose.** A provider's reason can help an attacker: a decline code that says a card was reported stolen tells a card thief which cards are flagged. The adapter maps a provider's failure reason to a short set of codes the client can act on, and keeps the provider's own code out of every response, stored in a column none of them shows (for card payments, Stripe's advised set: `insufficient_funds`, `expired_card`, `incorrect_cvc`, everything else `card_declined`).
- **Input never chooses the host.** It goes in `params` and in path segments encoded with `encodeURIComponent`, so a `/`, `..` or `?` in it cannot change the call. When the feature needs a URL a customer gives (a webhook target, a file to import), validate it in the DTO (see `security-validate-all-input`) and check, at every connection, the address the call actually reaches: a hostname can resolve to an internal address, or resolve somewhere else after validation.

**Incorrect (the shared axios instance, input in the path, the key in the URL, raw errors):**

```typescript
@Injectable()
export class ShipmentService {
  constructor(private readonly httpService: HttpService) {} // No base URL, no timeout

  async findTracking(trackingNumber: string): Promise<ShipmentTracking> {
    const { data } = await firstValueFrom(
      // "1Z9/../accounts" changes the path; the key ends up in every log of the URL
      this.httpService.get(`https://api.carrier.example/v1/trackings/${trackingNumber}?key=${process.env.CARRIER_API_KEY}`),
    );
    return data; // A hung provider never times out; an AxiosError reaches the filter as a 500
  }
}
```

**Correct (a configured client, mapped failures, the request id passed on):**

```typescript
// src/config/carrier-api.config.ts
export const carrierApiConfig = registerAs('carrierApi', () => ({
  baseUrl: process.env.CARRIER_API_BASE_URL || '',
  apiKey: process.env.CARRIER_API_KEY || '',
  timeoutMs: parseInt(process.env.CARRIER_API_TIMEOUT_MS || '5000', 10),
}));

// shipment/shipment.module.ts
@Module({
  imports: [
    HttpModule.registerAsync({
      inject: [carrierApiConfig.KEY],
      useFactory: (carrierApiSettings: ConfigType<typeof carrierApiConfig>) => ({
        baseURL: carrierApiSettings.baseUrl,
        timeout: carrierApiSettings.timeoutMs,
        headers: { 'x-api-key': carrierApiSettings.apiKey },
      }),
    }),
  ],
  controllers: [ShipmentController],
  providers: [CarrierClient, ShipmentService],
})
export class ShipmentModule {}

// shipment/clients/carrier.client.ts
type CarrierTrackingAnswer = { status: 200 | 404; data: unknown };

@Injectable()
export class CarrierClient {
  constructor(
    private readonly httpService: HttpService,
    private readonly cls: ClsService,
  ) {}

  async getTracking(trackingNumber: string): Promise<CarrierTrackingAnswer> {
    try {
      const { status, data } = await firstValueFrom(
        this.httpService.get(`/trackings/${encodeURIComponent(trackingNumber)}`, {
          headers: { 'x-request-id': this.cls.getId() },
          // The carrier answers 404 for an unknown tracking number
          validateStatus: (status) => status === 200 || status === 404,
        }),
      );
      return { status: status as CarrierTrackingAnswer['status'], data };
    } catch (error) {
      throw this._toHttpException(error);
    }
  }

  private _toHttpException(error: unknown): unknown {
    if (!isAxiosError(error)) {
      return error;
    }
    const status = error.response?.status;
    if (status === undefined || status === 429 || status >= 500) {
      // NaN when the header is an HTTP date
      const retryAfterSeconds = Number(error.response?.headers['retry-after']);
      return new ServiceUnavailableException({ message: 'The carrier is unavailable', retryAfterSeconds }, { cause: error });
    }
    return new InternalServerErrorException('The carrier rejected the request', { cause: error });
  }
}
```

`HttpModule.registerAsync()` gives the shipment module an axios instance of its own. `CarrierClient` is the transport only: what a tracking answer means is `ShipmentService`'s business. A `retryAfterSeconds` of `NaN` is not a positive integer, so the exception filter sends no `Retry-After` for it (see `error-use-exception-filters`).

**A URL from a customer** may reach only a public address, checked at every connection:

- **Its own agents.** The client has its own `http` and `https` agents whose `lookup` checks every address a name resolves to. Every socket they open is checked, so one kept alive for the next delivery was checked when it opened. The process's global agent is shared with every other client, and may hold a socket another client opened to an internal host of the same name and port.
- **No proxy, no redirect.** `proxy: false`, or a proxy variable in the environment resolves the name on the proxy, past the check; `maxRedirects: 0`, since a redirect can point anywhere.
- **A deadline and a size cap.** `timeout` only bounds a silent socket: a receiver that sends a byte every few seconds holds the call for hours, and the worker with it. The call also gets a deadline for the whole call (`signal: AbortSignal.timeout(...)`) and a cap on the answer (`maxContentLength`).
- **Blocked ranges** include 6to4 (`2002::/16`), NAT64 local-use (`64:ff9b:1::/48`) and site-local IPv6 (`fec0::/10`), besides the private, loopback and link-local ones.
- **One code for the customer.** A refused address and a network error are both `unreachable` in what the customer sees: telling them apart would let a customer probe which internal names exist. The precise reason goes to the log.

```typescript
// webhook/clients/webhook.client.ts
const NON_PUBLIC_ADDRESSES = new BlockList();
for (const [network, prefix] of [['0.0.0.0', 8], ['10.0.0.0', 8], ['100.64.0.0', 10], ['127.0.0.0', 8], ['169.254.0.0', 16], ['172.16.0.0', 12], ['192.168.0.0', 16]] as const) {
  NON_PUBLIC_ADDRESSES.addSubnet(network, prefix, 'ipv4');
}
// BlockList checks IPv4-mapped addresses (::ffff:10.0.0.1) against the IPv4 ranges
for (const [network, prefix] of [['::', 128], ['::1', 128], ['fc00::', 7], ['fe80::', 10], ['fec0::', 10], ['2002::', 16], ['64:ff9b:1::', 48]] as const) {
  NON_PUBLIC_ADDRESSES.addSubnet(network, prefix, 'ipv6');
}

const isNonPublic = (address: string): boolean => NON_PUBLIC_ADDRESSES.check(address, isIP(address) === 6 ? 'ipv6' : 'ipv4');

// Node calls it for every socket the agents open: a name that resolves elsewhere after validation is still refused
function publicOnlyLookup(hostname: string, options: LookupOptions, callback: LookupCallback): void {
  lookup(hostname, { ...options, all: true }, (error, addresses) => {
    if (error) {
      return callback(error, '');
    }
    if (addresses.length === 0 || addresses.some(({ address }) => isNonPublic(address))) {
      return callback(new WebhookNonPublicAddressError(`${hostname} resolves to a non-public address`), '');
    }
    return options.all ? callback(null, addresses) : callback(null, addresses[0].address, addresses[0].family);
  });
}

@Injectable()
export class WebhookClient implements OnApplicationShutdown {
  private readonly httpAgent = new HttpAgent({ keepAlive: true, lookup: publicOnlyLookup });
  private readonly httpsAgent = new HttpsAgent({ keepAlive: true, lookup: publicOnlyLookup });

  constructor(
    private readonly httpService: HttpService,
    @Inject(webhookConfig.KEY) private readonly webhookSettings: ConfigType<typeof webhookConfig>,
  ) {}

  async deliver(url: string, body: Buffer, headers: Record<string, string>): Promise<WebhookAttempt> {
    const host = new URL(url).hostname.replace(/^\[|\]$/g, '');
    // Node skips lookup for an address
    if (isIP(host) && isNonPublic(host)) {
      return { statusCode: null, error: 'unreachable', detail: 'non_public_address' };
    }
    try {
      const { status } = await firstValueFrom(
        this.httpService.post(url, body, {
          headers,
          httpAgent: this.httpAgent,
          httpsAgent: this.httpsAgent,
          proxy: false,
          maxRedirects: 0,
          maxContentLength: 64 * 1024,
          // timeout only bounds a silent socket
          signal: AbortSignal.timeout(this.webhookSettings.deadlineMs),
          validateStatus: () => true,
        }),
      );
      return { statusCode: status, error: null };
    } catch (error) {
      return this._failure(error);
    }
  }

  onApplicationShutdown(): void {
    this.httpAgent.destroy();
    this.httpsAgent.destroy();
  }
}
```

`validateStatus: () => true` makes every status the receiver's answer, recorded as such. `_failure()` gives the customer `timeout` or `unreachable` and keeps the cause for the log.

**Signing what the app sends.** When the app delivers to a customer's system (a webhook), each delivery is signed so the receiver can check it came from the app and is recent: an HMAC-SHA256, keyed with the webhook's secret, over a timestamp and the exact body bytes sent. Sign the bytes that go out (a `Buffer`), not an object serialized again, and give each attempt a fresh timestamp and the same delivery id, so the receiver can tell a retry from a new delivery. The receiver's documentation tells it to compare in constant time and to refuse an old timestamp. The header names and the registration flow are the app's own. In the shop, a delivery carries `X-Shop-Signature: t=<unix seconds>,v1=<hex HMAC-SHA256>`, computed over `<t>.<body>`, and `X-Shop-Delivery` with the delivery id, and the secret is shown once, when the webhook is registered.

**Correct (an SDK client, private to the module, behind the module's service):**

```typescript
// storage/utils/build-s3-client-config.ts
export function buildS3ClientConfig(awsSettings: ConfigType<typeof awsConfig>): S3ClientConfig {
  const { endpoint, timeoutMs } = awsSettings.s3;
  return {
    // No credentials: the SDK's credential chain finds them
    region: awsSettings.region,
    ...(endpoint ? { endpoint, forcePathStyle: true } : {}),
    maxAttempts: 1,
    requestHandler: { connectionTimeout: timeoutMs, requestTimeout: timeoutMs, throwOnRequestTimeout: true },
  };
}

// storage/storage.module.ts
@Module({
  providers: [
    { provide: S3Client, inject: [awsConfig.KEY], useFactory: (awsSettings) => new S3Client(buildS3ClientConfig(awsSettings)) },
    AwsS3Client,
    AwsS3Adapter,
    { provide: STORAGE_ADAPTER, useExisting: AwsS3Adapter },
    StorageService,
  ],
  exports: [StorageService],
})
export class StorageModule {}

// storage/types/storage-folder.enum.ts
export enum StorageFolder {
  ProductImages = 'product-images',
  UserAvatars = 'user-avatars',
  Reports = 'reports',
}

// storage/interfaces/storage-adapter.interface.ts
export interface StorageAdapter {
  put(folder: StorageFolder, key: string, file: StorageFile): Promise<void>;
  read(folder: StorageFolder, key: string): Promise<Readable>;
  delete(folder: StorageFolder, key: string): Promise<void>;
  // Throws for a private folder
  publicUrl(folder: StorageFolder, key: string): string;
}

// storage/adapters/aws-s3.adapter.ts
@Injectable()
export class AwsS3Adapter implements StorageAdapter {
  constructor(
    private readonly awsS3Client: AwsS3Client,
    @Inject(awsConfig.KEY) private readonly awsSettings: ConfigType<typeof awsConfig>,
  ) {}

  put(folder: StorageFolder, key: string, file: StorageFile): Promise<void> {
    return this.awsS3Client.putObject({ bucket: this.awsSettings.s3.bucket, key: this._objectKey(folder, key), ...file });
  }

  // read() and delete() alike

  private _objectKey(folder: StorageFolder, key: string): string {
    return `${folder}/${key}`;
  }
}

// storage/clients/aws-s3.client.ts
const NO_ANSWER_CODES = new Set(['ECONNREFUSED', 'ECONNRESET', 'ETIMEDOUT', 'EHOSTUNREACH', 'ENOTFOUND', 'EAI_AGAIN']);

@Injectable()
export class AwsS3Client implements OnApplicationShutdown {
  constructor(private readonly s3: S3Client) {}

  async putObject({ bucket, key, body, contentType, cacheControl }: StorageS3Object): Promise<void> {
    try {
      await this.s3.send(new PutObjectCommand({ Bucket: bucket, Key: key, Body: body, ContentType: contentType, CacheControl: cacheControl }));
    } catch (error) {
      throw this._toHttpException(error);
    }
  }

  onApplicationShutdown(): void {
    this.s3.destroy();
  }

  private _toHttpException(error: unknown): HttpException {
    const status = (error as { $metadata?: { httpStatusCode?: number } }).$metadata?.httpStatusCode;
    if (status === undefined ? this._isNoAnswer(error) : status === 429 || status >= 500) {
      const retryAfter = error instanceof S3ServiceException ? error.$response?.headers['retry-after'] : undefined;
      return new ServiceUnavailableException(
        { message: 'The file storage is unavailable', retryAfterSeconds: Number(retryAfter) },
        { cause: error },
      );
    }
    return new InternalServerErrorException('The file storage rejected the request', { cause: error });
  }

  private _isNoAnswer(error: unknown): boolean {
    const { name, code } = error as { name?: string; code?: string };
    return name === 'TimeoutError' || NO_ANSWER_CODES.has(code ?? '');
  }
}

// product/services/product-image.service.ts
await this.storageService.put(StorageFolder.ProductImages, image.storageKey, {
  body: dto.file.buffer,
  contentType,
  cacheControl: PRODUCT_IMAGE_CACHE_CONTROL,
});
```

`buildS3ClientConfig()` takes the SDK's settings from the `aws` namespace and passes no credentials (see `devops-use-config-module`); its `endpoint` points at a local S3-compatible server, such as MinIO. Every module that keeps files uses `StorageService`, which injects `STORAGE_ADAPTER` and passes each call on; the token, the adapter and the client stay inside the storage module (see `arch-module-sharing`). `StorageAdapter` keeps files by folder and key, whatever stores them, and `AwsS3Adapter` puts every folder in the app's one bucket, named in the settings, under the folder's prefix (`product-images/{key}`), so neither the bucket nor the provider reaches the modules that store files. `AwsS3Client` is the transport: the requests, the timeouts and what a failure answers, as listed above, and `S3Client.destroy()` closes its open connections at shutdown. The product module names its folder and its images' caching.

**One bucket per environment, one folder per kind of file.** The app keeps its files in one bucket, and each environment has its own (`dev`, `stage` and `prod` never share a bucket, so test data never mixes with real data). Each kind of file is a folder, a key prefix in that bucket: product images, user avatars and reports each under their own. The bucket is private: files anyone may see (product images) are served through a CDN in front of their folder, and private files (reports) through a short-lived presigned URL, never through a public bucket or object ACL. What differs by kind of file is set by prefix: the lifecycle rules (reports expire), the CDN's path, the IAM policy. A kind of file gets a bucket of its own only for a setting the provider applies to a whole bucket, such as Object Lock or another region.

A missing credential (`CredentialsProviderError`) has no status and is not a missing answer: it is our setup that is wrong, so it is a 500.

Reference: [AWS SDK for JavaScript v3: S3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/) · [NestJS HTTP module](https://docs.nestjs.com/techniques/http-module) · [axios request config](https://axios-http.com/docs/req_config) · [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
