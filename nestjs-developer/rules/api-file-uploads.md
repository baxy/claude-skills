---
title: Handle File Uploads with Validated Form-Data DTOs
impact: MEDIUM
impactDescription: Uploads are size- and type-checked at the boundary, alongside the other fields
tags: api, uploads, multipart, nestjs-form-data, validation
---

## Handle File Uploads with Validated Form-Data DTOs

Accept multipart uploads with `nestjs-form-data` so files are validated in the same DTO as the other fields, by the same global `ValidationPipe`. It reads the request itself: on Express it needs nothing more. Nest 12 also has upload interceptors of its own on both platforms (`FileInterceptor` from `@nestjs/platform-express` or `@nestjs/platform-fastify/multipart`, with `ParseFilePipe`), which validate the file in a pipe on the handler parameter, apart from the DTO. Keep `nestjs-form-data`, so a file and its fields are one DTO, validated and documented like every other input.

**On Fastify, the adapter registers `@fastify/multipart`.** Fastify refuses a multipart body with a 415 until a parser for it is registered. Set `multipart: true` on the `FastifyAdapter`, which registers `@fastify/multipart` (an optional peer of `@nestjs/platform-fastify`) with its defaults. Set it where the adapter is built (`ApiModule.createFastifyAdapter()`), so the test apps get it too. Those defaults leave the body unread, for `nestjs-form-data` to read within the route's limits. Never set the plugin's `attachFieldsToBody`, a common setup in examples: the plugin would then read every multipart body first, into memory and under its own 1 MiB file limit, on every route, and the upload would reach the DTO empty.

- Register `NestjsFormDataModule` once (global) with `MemoryStoredFile` storage.
- Put the api module's `@ApiFormDataRequest(options)` on every endpoint that accepts uploads (`POST`, `PUT` or `PATCH`, following the method rules in `api-controller-conventions`). It applies the boundary interceptor (below) and passes its options, the `limits`, to `@FormDataRequest()`.
- Type file fields with a type of the module's own, a `Pick` of `MemoryStoredFile` in `types/` (`UserAvatarFile`), document them with `@ApiProperty({ type: 'string', format: 'binary' })`, and validate them with `@IsFile()`, `@MaxFileSize()` and, for a format with a signature (an image, a PDF), `@HasMimeType()`. Typed as the class itself, a field breaks the build under pnpm: the Swagger CLI plugin imports the class from its path inside `node_modules/.pnpm`, which Node cannot resolve, and the app fails to start (a variant of [nestjs/swagger#4071](https://github.com/nestjs/swagger/issues/4071) still open in 12.1.2). The type carries a comment with that link and the condition for removing it (see `style-code-comments`).
- **A file's name is the client's text, bounded like any other.** `originalName` is whatever the client wrote in the part's header, and only busboy's header limit bounds it: a 15,000-character name was stored and returned in every listing. A name that is stored, logged or returned gets a length limit, checked with the other fields: `@MaxLength` cannot reach a property inside the file object, so the api module's `@MaxFileNameLength(n)` (in `api/validators/`, counting code points as `@MaxCodePoints` does, see `security-validate-all-input`) refuses a longer one with a 400. The limit is the length of the column that stores the name (255 in the example).
- **Read up to twice the allowed size.** `limits.fileSize` in `@ApiFormDataRequest()`'s options is where busboy stops reading, and it refuses a file once it reaches that size, at exactly that size too, with a message naming the limit in bytes. Set it to twice the allowed size, and keep the real limit, with a message that names it, in `@MaxFileSize(max, { message })`. The usual oversized file then gets the real limit in its message; only an upload past twice the size gets busboy's.
- Multipart sends every non-file field as a string. JSON fields (stored as `jsonb`) need a `@Transform` that parses strings, and must then still be validated. Keep invalid JSON as the raw string so the validator rejects it with a 400 instead of `JSON.parse` throwing a 500.
- **An image other users see is re-encoded on the server, never stored as sent** (with `sharp`, see `references/project-setup.md`). The upload carries what the sender's device wrote into it: a phone photo's GPS location and camera serial in its EXIF data, bytes appended after the image, a declared size of 30000 by 30000 pixels that costs every viewer gigabytes to decode. Refuse a declared size over a pixel limit before decoding (`limitInputPixels`), apply any crop, resize to the largest size the clients show, and store only the re-encoded file, which keeps no metadata. A declared width or height also stays out of an `integer` column unchecked: one past its range is a 500.
- **A text file has no signature.** `@HasMimeType()` reads a file's type from its first bytes, and for a format without a signature (CSV, JSON, plain text) it falls back on the `Content-Type` the client sent, which proves nothing. Such a file is checked by reading it: a strict decode (`new TextDecoder('utf-8', { fatal: true })`) and a parser that checks the header row refuse what is not the expected text.
- **Parsing an uploaded text file is bounded by its work, not only by its bytes.** Within the byte limit, a file can hold millions of rows (25 MB of `a\n` is 13 million records) or one field of 25 MB. A reader that parses the whole file before it counts the rows, or builds each field one character at a time (`field += char`, a string of one-character pieces, about 32 bytes of heap per byte read), takes gigabytes of heap and holds the event loop for seconds. The reader stops at the row limit plus one, refuses a record wider than the header at its first field too many, and cuts each field out of the text with `slice()`, undoing doubled quotes once per field. A streaming parser given the same limits (`csv-parse` with `to_line` and `max_record_size`) works too. A unit test reads a file at the byte limit made of the shortest rows, and one made of a single field, with the test workers under a heap limit (for example Vitest's `execArgv: ['--max-old-space-size=512']`), and expects the refusal.
- Upload routes get a rate limit of their own, well under the general one: each request buffers up to the read limit and writes to the object storage (see `security-rate-limiting`).
- Checks that depend on business rules (which file types a user's plan allows) stay in the service.
- **The file before the row, both ways.** An upload stores the file first and adds its row second, and deletes the file again when the row cannot be added (a limit reached meanwhile, the record gone): a row then always has its file, and a failure leaves at most a file nobody points to, which is logged. A removal deletes the file first and the row second: a storage that does not answer leaves the record whole for the request to be repeated, and deleting a file twice is harmless on S3.
- **Files outlive their caches and their records.** A file served with a long cache lifetime (each upload under a key of its own, `immutable`) stays in browser and CDN caches after it is removed: say so in the API docs. The files of a soft-deleted record are deleted after a retention, by a method of the module's service that the `scheduler/` module calls on an interval (see `arch-feature-modules`), or they stay public in the bucket for good.
- **A multipart body without a boundary is a 400.** `nestjs-form-data` 11.0.1 lets busboy's error for it reach the filter as a 500. The api module's `@ApiFormDataRequest()` applies an interceptor that answers 400 first, then `@FormDataRequest()` with the route's options, and every upload route uses it.

**Incorrect (unvalidated file, JSON parsing that can crash):**

```typescript
@Patch(':id/profile')
@UseInterceptors(FileInterceptor('avatar'))         // Express-only (multer); file outside the DTO
updateProfile(@UploadedFile() avatar: Express.Multer.File, @Body() dto: UpdateUserProfileDto) {
  // No size or type check: any file of any size reaches the service
  return this.userService.updateProfile(dto, avatar);
}

export class UpdateUserProfileDto {
  @IsOptional()
  @Transform(({ value }) => JSON.parse(value))      // Throws on invalid JSON - a 500, not a 400
  preferences?: Record<string, unknown>;
}
```

**Correct (one validated DTO for fields and files):**

```typescript
// app.module.ts
@Module({
  imports: [NestjsFormDataModule.config({ storage: MemoryStoredFile, isGlobal: true })],
})
export class AppModule {}

// user/dtos/update-user-profile.dto.ts
export class UpdateUserProfileDto {
  @ApiPropertyOptional({ description: 'Display name', example: 'Jane Doe' })
  @IsOptional()
  @IsString()
  @MaxLength(100)
  name?: string;

  @ApiPropertyOptional({
    description: 'UI preferences, sent as a JSON string',
    example: { theme: 'dark', language: 'en' },
  })
  @IsOptional()
  @Transform(({ value }) => {
    if (typeof value !== 'string') return value;
    try {
      return JSON.parse(value);
    } catch {
      // Left for @IsObject() to refuse with a 400, where JSON.parse would throw a 500
      return value;
    }
  })
  @IsObject()
  preferences?: Record<string, unknown>;

  @ApiPropertyOptional({ description: 'Profile picture (max 5 MB)', type: 'string', format: 'binary' })
  @IsOptional()
  @IsFile()
  @MaxFileSize(USER_AVATAR_MAX_SIZE_BYTES, { message: 'avatar must be at most 5 MB' })
  @HasMimeType(['image/jpeg', 'image/png', 'image/webp'], { message: 'avatar must be a JPEG, PNG or WebP image' })
  @MaxFileNameLength(255)
  avatar?: UserAvatarFile;
}

// api/validators/max-file-name-length.validator.ts
export function MaxFileNameLength(max: number): PropertyDecorator {
  return ValidateBy({
    name: 'maxFileNameLength',
    validator: {
      validate: (file: unknown) => {
        const name = (file as { originalName?: unknown } | undefined)?.originalName;
        // Code points, as PostgreSQL counts them
        return typeof name === 'string' && [...name].length <= max;
      },
      defaultMessage: (args) => `${args?.property} must have a file name of at most ${max} characters`,
    },
  });
}

// user/constants/user-limits.constant.ts
export const USER_AVATAR_MAX_SIZE_BYTES = 5 * 1024 * 1024;

// user/types/user-avatar-file.type.ts
// Workaround for a variant of https://github.com/nestjs/swagger/issues/4071 (12.1.2): a field typed MemoryStoredFile
// makes the Swagger plugin import it from node_modules/.pnpm, and the app fails to start. Remove once it imports from
// nestjs-form-data
export type UserAvatarFile = Pick<MemoryStoredFile, 'buffer' | 'size' | 'mimeType' | 'originalName'>;

// api/api.module.ts
return new FastifyAdapter({ trustProxy: trustedProxies.length > 0 ? trustedProxies : false, multipart: true });

// user/controllers/user.controller.ts
@ApiConsumes('multipart/form-data')
// busboy refuses a file on reaching this size, with a message of its own: twice the size leaves the usual refusal
// to @MaxFileSize
@ApiFormDataRequest({ limits: { fileSize: 2 * USER_AVATAR_MAX_SIZE_BYTES, files: 1 } })
@Patch(':id/profile')
updateProfile(@AuthUser() authUser: JwtUser, @Param() params: UserIdParamDto, @Body() dto: UpdateUserProfileDto) {
  return this.userService.updateProfile(authUser, params.id, dto);
}

// user/services/user.service.ts
if (dto.avatar) {
  const avatarKey = randomUUID();
  await this.storageService.put(StorageFolder.UserAvatars, avatarKey, {
    body: await reencodeUserAvatar(dto.avatar.buffer),
    contentType: 'image/webp',
  });
}
```

The `PATCH` is a partial update, so every field of the DTO is optional, and the service reads the file's content from the DTO. It stores the avatar through the storage module's `StorageService` (see `micro-use-http-clients`): the re-encoded image, never the bytes as sent, under a key generated for each upload, never the client's `originalName`, before the row points to it. `reencodeUserAvatar()`, in `user/utils/`, applies the pixel limit, the resize and the re-encoding above with `sharp`. `createFastifyAdapter()` is where `multipart: true` goes on Fastify.

**A text upload read within its limits:**

```typescript
// product/utils/read-product-import.ts
export function readProductImport(file: Buffer): ProductImportRow[] {
  let text: string;
  try {
    // A CSV has no signature: decoding it is the type check
    text = new TextDecoder('utf-8', { fatal: true }).decode(file);
  } catch {
    throw new BadRequestException('The file is not UTF-8 text');
  }
  const rows: ProductImportRow[] = [];
  let header: string[] | undefined;
  for (const fields of csvRecords(text, { maxFields: PRODUCT_IMPORT_COLUMNS.length })) {
    if (!header) {
      header = checkProductImportHeader(fields);
      continue;
    }
    if (rows.length === PRODUCT_IMPORT_MAX_ROWS) {
      throw new BadRequestException(`A file holds at most ${PRODUCT_IMPORT_MAX_ROWS} rows`);
    }
    rows.push(toProductImportRow(header, fields));
  }
  return rows;
}
```

The work ends with the limits, not after the whole file: the reader stops at the first row past the limit, one row read too many and no more. `csvRecords()` is a generator that yields one record at a time, cuts each field with `text.slice(start, end)`, and throws at the first field past `maxFields`, before the rest of the record is read. `checkProductImportHeader()` answers 400 unless the columns are the expected ones.

Reference: [nestjs-form-data](https://github.com/dmitriy-nz/nestjs-form-data)
