---
title: Sanitize User HTML and Never Reflect Unvalidated Input
impact: HIGH
impactDescription: Stored or reflected markup cannot run in a browser that renders API data
tags: security, xss, sanitization, html, helmet
---

## Sanitize User HTML and Never Reflect Unvalidated Input

A JSON API is not executed by the browser, but its data is rendered by clients that may trust it. Three habits keep stored and reflected markup harmless:

- **Rich-text fields are sanitized on the way in**, in the DTO, with an allow-list (`sanitize-html`). Sanitizing once, at the boundary, means every consumer of the stored value gets the same safe content.
- **Only text within its length limit is sanitized.** `@Transform` runs before validation, so a transform that sanitizes every value hands the sanitizer a body of any size. `sanitize-html` takes time that grows with the square of nested tags (a second for 256 KiB of `<b>`, half a minute for the 1 MiB a request body may hold), and every other request waits meanwhile, health probes included. The transform passes text over the limit on unchanged, and `@MaxLength` refuses it.
- **Plain-text fields are stored as sent** (a name, a review title): validated for type and length, never run through `sanitize-html`. They are text, not markup, and the client escapes them when it renders them, as it must for any JSON string. `sanitize-html` with no allowed tags would also encode the text itself: `Tom & Jerry` would be stored as `Tom &amp; Jerry`, and `a < b` as `a &lt; b`.
- **Error messages only echo validated values.** A UUID validated by a param DTO can appear in a message (see `error-throw-http-exceptions`); free text from the client never does. The `ValidationPipe`'s own messages are the one exception: `forbidNonWhitelisted` names the unknown property (`property x should not exist`) so the client can fix its request (see `security-validate-all-input`), and the error is JSON, which no client renders as markup.
- **A mail to an address nobody has verified carries no text its sender chose.** Anyone can sign up with someone else's email, so a welcome mail that repeats the name given at sign-up lets them send their own words from the app's address. Such a mail is the app's fixed text; a mail to a verified address may quote what its owner entered.
- **Responses are JSON**, which Nest sets by default. An endpoint that must return HTML builds it from escaped values, and `helmet` sets a Content Security Policy on every response (see `api-platform-agnostic` for the per-platform package).
- **A response header protects only a page the browser loads itself.** HTML that a front end fetches (a route behind a bearer token cannot be navigated to) and then shows in a frame through `srcdoc` or a `blob:` URL is a document of the front end: it takes the front end's policy, and this response's Content Security Policy never applies. Then escaping is the only protection, and the API docs state the rendering contract: an `<iframe sandbox>` without `allow-scripts` or `allow-same-origin`. Or the route returns the data and the front end renders it.

**Incorrect (raw HTML stored, input reflected, HTML built by concatenation):**

```typescript
export class CreateProductReviewDto {
  @IsString()
  body: string; // <script>...</script> stored as-is and served to every client
}

@Controller('products')
export class ProductController {
  @Get(':id/page')
  @Header('Content-Type', 'text/html')
  async findPage(@Param() params: ProductIdParamDto): Promise<string> {
    const product = await this.productService.findById(params.id);
    return `<html><body>${product.description}</body></html>`; // Stored content concatenated into HTML
  }
}

// Unvalidated input echoed back
throw new BadRequestException(`Unknown sort field ${query.sort}`);
```

**Correct (sanitized at the boundary, validated echoes, CSP):**

```typescript
// product/utils/sanitize-review-body.ts
const REVIEW_BODY_OPTIONS: sanitizeHtml.IOptions = {
  allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
  allowedAttributes: { a: ['href', 'title'] },
  allowedSchemes: ['http', 'https', 'mailto'],
};

export function sanitizeReviewBody(body: string): string {
  return sanitizeHtml(body, REVIEW_BODY_OPTIONS);
}

// product/constants/product-limits.constant.ts
export const PRODUCT_REVIEW_BODY_MAX_LENGTH = 5_000;

// product/dtos/create-product-review.dto.ts
export class CreateProductReviewDto {
  @ApiProperty({ description: 'Review body; basic formatting tags are kept', example: '<p>Fits <b>great</b></p>' })
  @IsString()
  @MaxLength(PRODUCT_REVIEW_BODY_MAX_LENGTH)
  // Only text within the limit: sanitize-html's time grows with the square of nested tags
  @Transform(({ value }) =>
    typeof value === 'string' && value.length <= PRODUCT_REVIEW_BODY_MAX_LENGTH ? sanitizeReviewBody(value) : value,
  )
  body: string;

  @ApiProperty({ description: 'Review title, plain text', example: 'Comfortable all day' })
  @IsString()
  @MaxLength(200)
  title: string;
}

throw new BadRequestException('Unknown sort field');

// api/api.module.ts, in setup()
const helmetOptions = {
  contentSecurityPolicy: {
    directives: { defaultSrc: ["'self'"], scriptSrc: ["'self'"], imgSrc: ["'self'", 'data:', 'https:'] },
  },
};
await app.register(helmet, helmetOptions);
```

`sanitizeReviewBody()` stays in the module that uses it until a second module does (see `arch-shared-folder`). The review title is plain text: stored as sent, escaped by the client that renders it. The sort field is validated with `@IsIn` in its DTO, so the message needs no echo at all. On Fastify, `helmet` is `@fastify/helmet`; on Express, register it with `app.use(helmet(helmetOptions))`.

Reference: [OWASP XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) · [sanitize-html](https://github.com/apostrophecms/sanitize-html)
