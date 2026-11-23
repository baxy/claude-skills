---
title: Use Lazy Loading for Large Modules
impact: MEDIUM
impactDescription: Improves startup time for large applications
tags: performance, lazy-loading, modules, optimization
---

## Use Lazy Loading for Large Modules

NestJS can lazy-load a module, deferring its initialization until first use. This is valuable where cold start time matters (serverless, CLI) and a module is heavy and rarely needed.

- **Only modules without controllers, gateways or resolvers.** A lazy-loaded module's routes are never registered, so HTTP-facing modules always load eagerly.
- Import it through the `@/` alias like any cross-module import (see `arch-import-boundaries`).
- Resolving providers from the lazy module with `moduleRef.get()` is one of the two allowed uses of `ModuleRef` (see `di-avoid-service-locator`).
- **Nothing in the eagerly loaded code imports the lazy module's files as values.** A static import of its provider class, even only to pass it to `moduleRef.get()`, loads that file at startup, and with it the heavy library the module exists to defer. Import the provider class dynamically together with the module, or resolve it by a `Symbol` token from a `constants/` file that imports nothing heavy; the type comes in with `import type`, which leaves no import in the compiled code.
- **One load for concurrent first calls, and another try after a failure.** Keep the promise of the load, not its result, so the calls that arrive while it runs wait for the same load; forget it when it fails, so the next call tries again instead of getting the same rejection forever.
- **Test that the library stays unloaded until first use.** Counting calls to `LazyModuleLoader.load()` passes even when a static import loaded the library at startup. Mock the library with a factory that records its loading and returns the real module, and assert it has not loaded after startup and has after the first call.

**Incorrect (loading everything eagerly):**

```typescript
// Load everything eagerly in a large app
@Module({
  imports: [
    UserModule,
    OrderModule,
    PaymentModule,
    PdfRenderingModule, // Heavy (headless browser), used by one job
    BulkImportModule,   // Used once a month
  ],
})
export class AppModule {}

// All modules initialize at startup, even if never used
// Slow cold starts in serverless
// Memory wasted on unused modules
```

**Correct (lazy load rarely-used modules):**

```typescript
// order/services/order-invoice-pdf.service.ts
// A value import here would load the PDF library at startup
import type { PdfRenderingService } from '@/pdf-rendering/services/pdf-rendering.service';

@Injectable()
export class OrderInvoicePdfService {
  private pdfRenderingService: Promise<PdfRenderingService> | undefined;

  constructor(private readonly lazyModuleLoader: LazyModuleLoader) {}

  async render(order: Order): Promise<Buffer> {
    const pdfRenderingService = await this._pdfRenderingService();
    return pdfRenderingService.render('invoice', order);
  }

  private _pdfRenderingService(): Promise<PdfRenderingService> {
    this.pdfRenderingService ??= this._loadPdfRenderingService().catch((error: unknown) => {
      this.pdfRenderingService = undefined;
      throw error;
    });
    return this.pdfRenderingService;
  }

  private async _loadPdfRenderingService(): Promise<PdfRenderingService> {
    const [{ PdfRenderingModule }, { PdfRenderingService }] = await Promise.all([
      import('@/pdf-rendering/pdf-rendering.module'),
      import('@/pdf-rendering/services/pdf-rendering.service'),
    ]);
    const moduleRef = await this.lazyModuleLoader.load(() => PdfRenderingModule);
    return moduleRef.get(PdfRenderingService);
  }
}

// test/e2e/order/order.spec.ts
// A jest.mock factory may only reach variables whose names start with mock
const mockLoaded = { pdfkit: false };
jest.mock('pdfkit', () => {
  mockLoaded.pdfkit = true;
  return jest.requireActual('pdfkit');
});

it('loads the PDF library on the first invoice, not at startup', async () => {
  expect(mockLoaded.pdfkit).toBe(false);
  await request(app.getHttpServer()).get(`/orders/${ORDER_ID}/download`).expect(200);
  expect(mockLoaded.pdfkit).toBe(true);
});
```

`OrderInvoicePdfService` loads the PDF module, and the library it imports, on the first PDF: the module and the class come in through `import()`, never at the top of a file loaded at startup, and a failed load is forgotten so the next call tries again. The spec watches the library itself, not the loader. Jest runs the `jest.mock` factory when `pdfkit` is first imported, so the flag is still `false` once the app has started in `beforeAll`, and the test guard sets the caller of the download. With Vitest, the flag comes from `vi.hoisted()`, and the mock is `vi.mock('pdfkit', async (importOriginal) => { ...; return importOriginal(); })`.

Reference: [NestJS Lazy Loading Modules](https://docs.nestjs.com/fundamentals/lazy-loading-modules)
