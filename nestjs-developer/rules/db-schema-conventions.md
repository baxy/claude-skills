---
title: Keep the Schema Persistence-Only and Consistent
impact: MEDIUM-HIGH
impactDescription: The schema model describes the database and nothing else, with uniform names and columns across tables
tags: database, schema, conventions, naming, postgres
---

## Keep the Schema Persistence-Only and Consistent

The schema model (the entities in TypeORM, the `.prisma` files in Prisma) represents the database schema and carries persistence details only. Anything that describes a response or opts fields into an API schema (`@ApiProperty`, `@Expose`) belongs in a DTO (see `api-use-dto-serialization`). This rule says what every ORM must do; the forms are in `db-typeorm-schema-conventions` for TypeORM and `db-prisma-schema-conventions` for Prisma.

Conventions:

- The model is **singular** and the table **plural**: `Order` maps to `orders`, `OrderItem` to `order_items`. Always give the table name explicitly (see `arch-resource-naming`).
- Every database object (tables, columns, join columns, junction tables, foreign keys) is named in lowercase `snake_case`, and properties are camelCase in code.
- Timestamps are `timestamptz`, never `timestamp` without a time zone.
- Soft deletes: a nullable `deleted_at` `timestamptz`, kept out of responses. Deleting a row sets it, and every query skips the rows where it is set; raw SQL filters `deleted_at IS NULL` itself.
- Complex structured data is `jsonb`, nullable or with a default when the value is optional.
- A set of values is a native enum type or a `varchar`, chosen by how stable the set is. A closed set that code and data depend on, such as an order's status, is a native enum (`order_status`): the database refuses any other value, and adding one is a generated migration. Removing or renaming a value means rebuilding the type, so only a stable set gets one. A set that grows with features (roles, report kinds, notification types) is a `varchar`, or a `varchar[]` for several: adding a value changes only code, and the TypeScript enum and `@IsEnum()` in the DTOs are the check. Never a `varchar` with a CHECK listing the values: it needs a migration for every new value, like an enum, and TypeORM does not even notice the changed CHECK (see `db-typeorm-schema-conventions`). Either way, one TypeScript enum is the list in code, with snake_case values (`Pending = 'pending'`). Prisma generates one for a native enum, its members named as the database's values (`pending`); code that writes `OrderStatus.Pending` adds a const object over it (see `db-prisma-schema-conventions`).
- The owning side of a relation keeps its foreign key as a plain property next to the relation (`userId` beside `user`), on the `user_id` column.
- Sensitive columns (password hashes, tokens) never reach a response.
- Indexes and constraints are named explicitly, the same way whatever the ORM: `IDX_`, `UQ_` or `CHK_`, then the table, then the columns or the purpose (`IDX_orders_user_id_status`, `UQ_users_email_active`, `CHK_orders_total_cents_non_negative`). Database-tier tests match on these names (see `test-database-tier`). Primary and foreign keys keep the names the ORM gives them.
- Every structural change ships with a generated migration (see `db-use-migrations`, which also says what to hand over when the migration CLI cannot run).

### Why `snake_case`

- **PostgreSQL folds unquoted identifiers to lower case, and quoting makes them case-sensitive.** `FOO`, `foo` and `"foo"` are the same name, but `"Foo"` is a different one (PostgreSQL docs, *Identifiers and Key Words*).
- **Without a naming rule, an ORM names a column after its property.** `createdAt` becomes a column literally named `"createdAt"`. Every raw query, migration, `psql` session, BI tool and report must then double-quote it. An unquoted `created_at` or `createdAt` fails with `column "createdat" does not exist`.
- **The PostgreSQL community's own advice is "Don't use NamesLikeThis, use names_like_this"** (PostgreSQL wiki, *Don't Do This*). Mixed-case names force you to always or never quote them, and tools disagree on which they do. With only `a-z`, `0-9` and `_` you never have to think about quoting.
- **The database is shared beyond the ORM.** Analysts, other services, data pipelines and DBAs read it without knowing your models. `snake_case` is the convention they expect.

Reference: [PostgreSQL: Identifiers and Key Words](https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS) · [PostgreSQL wiki: Don't Do This](https://wiki.postgresql.org/wiki/Don't_Do_This#Don.27t_use_upper_case_table_or_column_names)
