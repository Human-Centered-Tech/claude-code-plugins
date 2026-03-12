---
name: postgres-best-practices
description: Reference guide and audit checklist for PostgreSQL schema design, query performance, operational health, and security. Use when designing new tables, reviewing slow queries, setting up a Postgres deployment, or hardening database security.
---

Audit the current project's PostgreSQL usage against the best practices below. When used as a reference, apply the relevant sections to the task at hand. When running a full audit, check every section and report findings.

## 1. Schema Design

### Naming conventions

- Use `snake_case` for all identifiers — tables, columns, indexes, constraints
- Table names should be **plural** (`users`, `projects`, `api_keys`) — pick one convention and enforce it project-wide
- Boolean columns: prefix with `is_`, `has_`, or `can_` (`is_active`, `has_verified_email`)
- Timestamp columns: suffix with `_at` (`created_at`, `updated_at`, `deleted_at`, `last_login_at`)
- Foreign keys: `<referenced_table_singular>_id` (`user_id`, `project_id`)

### Data types — use the right type

| Need | Use | Avoid |
|------|-----|-------|
| Primary key | `uuid v7` (time-ordered) or `bigint` / `bigserial` | `uuid v4` (random — causes B-tree page splits and write amplification), `serial` (32-bit overflow risk) |
| Variable text | `text` | `varchar(255)` — Postgres `text` has no performance penalty vs `varchar`; arbitrary length limits cause bugs later |
| Fixed-length codes | `char(n)` or `text` with a `CHECK` | `varchar` without a constraint |
| Money / financial | `numeric(precision, scale)` | `float` / `double precision` — floating-point rounding errors |
| Timestamps | `timestamptz` (`timestamp with time zone`) | `timestamp` (without tz) — silently drops timezone info, causes bugs across regions |
| IP addresses | `inet` | `text` — `inet` supports indexing and range queries natively |
| JSON data | `jsonb` | `json` — `jsonb` is binary, indexable, and supports containment operators |
| Enums | `text` with a `CHECK` constraint, or a Postgres `ENUM` type | Free-form `text` with no validation |
| Yes/No flags | `boolean` | `smallint` or `char(1)` |

### Use Postgres native types instead of text

Postgres has purpose-built types that give you validation, indexing, and operators for free. Storing structured data as `text` throws all of that away.

| Data | Native type | What you get over `text` |
|------|------------|--------------------------|
| IP addresses | `inet` | Range queries (`<<`), containment checks, rejects invalid IPs |
| CIDR blocks | `cidr` | Network containment (`>>=`, `<<=`), subnet math |
| MAC addresses | `macaddr` | Vendor prefix lookups, canonical formatting |
| Date ranges | `daterange`, `tstzrange` | Overlap (`&&`), containment (`@>`), exclusion constraints |
| Durations | `interval` | Arithmetic (`now() + interval '30 days'`), comparison |
| Coordinates | `point` | Distance calculations, GiST indexing |
| Case-insensitive text | `citext` (extension) | Equality without `lower()` everywhere, indexable |
| URLs / emails | `text` + `CHECK` | At minimum, validate format at the database level |

**Example — the difference matters:**
```sql
-- Bad: stored as text, no validation, can't do range queries
ALTER TABLE audit_log ADD COLUMN ip_address text;

-- Good: validated on insert, supports subnet queries
ALTER TABLE audit_log ADD COLUMN ip_address inet;

-- Now you can query: "all requests from 10.0.0.0/8"
SELECT * FROM audit_log WHERE ip_address << '10.0.0.0/8';
```

When in doubt, check if Postgres has a type for it before defaulting to `text`.

### Constraints — be explicit

- **Every table** should have a primary key
- Add `NOT NULL` to every column that should never be null — don't rely on application logic
- Use `CHECK` constraints for business rules that live at the data level:
  ```sql
  ALTER TABLE orders ADD CONSTRAINT positive_amount CHECK (amount > 0);
  ALTER TABLE users ADD CONSTRAINT valid_email CHECK (email ~* '^.+@.+\..+$');
  ```
- Use `UNIQUE` constraints (not just unique indexes) for business uniqueness rules — they generate better error messages
- Always name your constraints explicitly — auto-generated names are hard to reference in migrations:
  ```sql
  -- Good
  ALTER TABLE orders ADD CONSTRAINT orders_user_id_fk FOREIGN KEY (user_id) REFERENCES users(id);
  -- Bad: unnamed, Postgres auto-generates something like "orders_user_id_fkey"
  ```

### Default values

- `created_at` → `DEFAULT now()`
- `updated_at` → `DEFAULT now()` (update via trigger or application code)
- UUIDs → prefer **UUIDv7** (time-ordered) over v4 (random). v7 embeds a millisecond timestamp in the first 48 bits, so new rows append to B-tree indexes instead of causing random page splits. Generation options:
  - **Application code** (recommended): most languages have UUIDv7 libraries — generate the ID before insert
  - **`pg_uuidv7` extension**: `DEFAULT uuid_generate_v7()` — if available on your Postgres host
  - **Fallback**: `DEFAULT gen_random_uuid()` generates v4 (built-in since Postgres 13) — acceptable but loses ordering benefits
  - **Note**: UUIDv7 leaks creation time in the ID. Don't use as secret tokens, and be aware IDs reveal when a record was created if exposed publicly.
- Boolean flags → explicit `DEFAULT false` or `DEFAULT true`, never leave nullable

### Normalize repeatable concepts into related tables

When a column is likely to need multiple instances or new "kinds" over time, use a related table instead of adding more columns.

**Bad** — numbered or typed columns that grow with requirements:
```sql
-- Adding a 4th phone type requires a migration and leaves nulls everywhere
ALTER TABLE users ADD COLUMN phone_home text;
ALTER TABLE users ADD COLUMN phone_work text;
ALTER TABLE users ADD COLUMN phone_mobile text;
```

**Good** — a related table where new kinds are just rows:
```sql
CREATE TABLE phone_numbers (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id),
  type text NOT NULL CHECK (type IN ('home', 'work', 'mobile', 'fax')),
  number text NOT NULL,
  is_primary boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_phone_numbers_user_id ON phone_numbers(user_id);
```

**Rule of thumb**: if you're tempted to add a number suffix or type prefix to a column name (`phone_1`, `address_billing`, `email_secondary`), it should probably be a separate table.

This applies to: phone numbers, emails, addresses, notes, tags, attachments, links, and any other concept where "how many" or "what kinds" may grow.

**When columns are fine**: the thing is truly singular and always present (`created_at`, `email` as a login identifier), or there's a small, fixed set that will never grow.

### Foreign key delete behavior — be intentional

Every foreign key should have an explicit `ON DELETE` action. The default (`NO ACTION`) raises an error if you try to delete a referenced parent, which is safe but not always what you want. Think about each relationship:

- **`RESTRICT` / `NO ACTION` (default)** — use as the safe default. Forces the application to explicitly handle deletions, making the scope visible in code.
- **`CASCADE`** — use when child rows are meaningless without the parent. Deleting the parent should clean up its dependents:
  ```sql
  -- project_members have no meaning without the project
  ALTER TABLE project_members
    ADD CONSTRAINT project_members_project_id_fk
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE;
  ```
  Good candidates: join tables, settings, child line items, per-parent config.
- **`SET NULL`** — use when the child should survive but the reference is optional:
  ```sql
  -- Tasks should survive if the assigned user is deleted
  ALTER TABLE tasks
    ADD CONSTRAINT tasks_assigned_to_fk
    FOREIGN KEY (assigned_to) REFERENCES users(id) ON DELETE SET NULL;
  ```
  Good candidates: `assigned_to`, `created_by`, `updated_by` — audit fields where the record outlives the user.

**Anti-patterns**:
- Blindly using `CASCADE` everywhere — a single parent delete can silently wipe thousands of rows across multiple tables
- Never using `CASCADE` — orphaned rows accumulate as junk data with no parent
- Using `CASCADE` on a table that uses soft deletes — contradictory (parent is "deleted" but children are hard-deleted)
- Missing indexes on FK columns — makes cascade deletes slow because Postgres has to seq-scan the child table

### Don't store what you can compute

Never store values that can be derived from existing data — they go stale the moment they're written.

**Bad** — stored values that drift from reality:
```sql
ALTER TABLE users ADD COLUMN age integer;           -- Wrong on their next birthday
ALTER TABLE users ADD COLUMN full_name text;         -- Drifts if first/last name changes
ALTER TABLE orders ADD COLUMN total numeric;         -- Drifts if line items are edited
ALTER TABLE users ADD COLUMN days_since_signup integer; -- Wrong tomorrow
```

**Good** — store the source of truth, compute when needed:
```sql
-- Store date_of_birth, compute age at query time
SELECT date_part('year', age(date_of_birth))::integer AS age FROM users;

-- Store first_name and last_name, concatenate when needed
SELECT first_name || ' ' || last_name AS full_name FROM users;

-- Compute order total from line items
SELECT order_id, sum(quantity * unit_price) AS total FROM order_items GROUP BY order_id;
```

**If you need it indexed or queried often**, use a Postgres **generated column** (Postgres 12+) — it's computed automatically and stays in sync:
```sql
ALTER TABLE users ADD COLUMN full_name text
  GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED;
```

Generated columns can be indexed and used in `WHERE` clauses, but the value is always derived from the source columns — never manually set.

**Exception**: pre-computed aggregates are acceptable for performance (e.g., a `comment_count` on a post) — but only with triggers or application logic that keeps them in sync, and only when the query cost of computing on the fly is proven to be a problem.

### Soft deletes

If the project uses soft deletes:
- Use a `deleted_at timestamptz` column (null = not deleted)
- Add a partial index for common queries: `CREATE INDEX idx_users_active ON users(id) WHERE deleted_at IS NULL;`
- Ensure all application queries filter on `deleted_at IS NULL` unless intentionally querying deleted records

## 2. Indexing Strategy

### When to add indexes

- Every foreign key column — Postgres does **not** auto-index foreign keys (unlike MySQL)
- Columns that appear in `WHERE`, `JOIN`, `ORDER BY`, or `GROUP BY` clauses frequently
- Columns used in uniqueness checks

### Index types — pick the right one

| Scenario | Index type |
|----------|-----------|
| Equality lookups (`=`), range queries (`<`, `>`, `BETWEEN`) | B-tree (default) |
| Full-text search | GIN on `tsvector` column |
| JSONB containment queries (`@>`, `?`, `?&`) | GIN on `jsonb` column |
| Geometric / spatial data | GiST |
| Low-cardinality "does it exist" checks | BRIN (for large, naturally-ordered tables) |

### Partial indexes

Use partial indexes when you only query a subset of rows:
```sql
-- Only index active users — smaller index, faster lookups
CREATE INDEX idx_users_active_email ON users(email) WHERE deleted_at IS NULL;

-- Only index incomplete orders
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending';
```

### Composite indexes

- Column order matters: put the most selective (highest cardinality) column first
- A composite index on `(a, b, c)` serves queries on `(a)`, `(a, b)`, and `(a, b, c)` but NOT `(b, c)` alone
- Don't create single-column indexes that duplicate the leading column of an existing composite index

### Anti-patterns

- **Over-indexing** — every index slows down writes. Audit unused indexes periodically:
  ```sql
  SELECT schemaname, relname, indexrelname, idx_scan
  FROM pg_stat_user_indexes
  WHERE idx_scan = 0
  ORDER BY pg_relation_size(indexrelid) DESC;
  ```
- **Indexing boolean columns alone** — too low cardinality to be useful. Combine with other columns or use a partial index instead
- **Missing foreign key indexes** — causes slow `DELETE` cascades and slow JOINs

## 3. Query Performance

### Use EXPLAIN ANALYZE

Always check query plans for slow queries:
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

Red flags in query plans:
- **Seq Scan** on large tables — usually means a missing index
- **Nested Loop** with high row counts — consider a Hash Join or add an index
- **Sort** with `external merge Disk` — `work_mem` is too low or the sort needs an index
- **Rows estimated vs actual** differ by 10x+ — run `ANALYZE` on the table to update statistics

### N+1 query detection

Look for patterns where the application:
1. Fetches a list of parent records
2. Then loops through each, making a separate query per record

**Fix**: Use `JOIN`, subquery, or `WHERE id = ANY($1::uuid[])` to batch the lookups.

### Query anti-patterns

- **`SELECT *`** — fetches unnecessary columns, breaks if schema changes, prevents index-only scans. Always specify columns explicitly.
- **`LIKE '%term%'`** — leading wildcard prevents index use. Use `pg_trgm` extension with a GIN index for substring search:
  ```sql
  CREATE EXTENSION IF NOT EXISTS pg_trgm;
  CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
  ```
- **`NOT IN (subquery)`** — behaves unexpectedly with NULLs and can be slow. Use `NOT EXISTS` instead:
  ```sql
  -- Bad
  SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM banned_users);
  -- Good
  SELECT * FROM users u WHERE NOT EXISTS (SELECT 1 FROM banned_users b WHERE b.user_id = u.id);
  ```
- **`ORDER BY RANDOM() LIMIT 1`** — scans and sorts the entire table. Use `TABLESAMPLE` or offset-based approaches for large tables.
- **Unbounded queries** — every user-facing query should have a `LIMIT`. Even internal queries should have a sane upper bound.

### Pagination

- **Offset pagination** (`LIMIT 20 OFFSET 100`) degrades as offset grows — Postgres still scans and discards rows
- **Cursor/keyset pagination** is better for large datasets:
  ```sql
  -- First page
  SELECT * FROM items WHERE tenant_id = $1 ORDER BY created_at DESC, id DESC LIMIT 20;
  -- Next page (pass last row's created_at and id)
  SELECT * FROM items
  WHERE tenant_id = $1 AND (created_at, id) < ($2, $3)
  ORDER BY created_at DESC, id DESC LIMIT 20;
  ```

## 4. Operational Best Practices

### Connection pooling

- **Never let your application open unlimited connections** — Postgres forks a process per connection; too many kills performance
- Use a connection pooler: PgBouncer (external) or your ORM's built-in pool
- Recommended pool sizes:
  - Small app (1-2 servers): 10–20 connections
  - Medium app: 20–50 connections
  - Formula starting point: `pool_size = (num_cores * 2) + effective_spindle_count`
- Set `statement_timeout` on the application role to prevent runaway queries:
  ```sql
  ALTER ROLE app_user SET statement_timeout = '30s';
  ```

### Transactions

- Keep transactions short — long transactions hold locks and block autovacuum
- Never hold a transaction open while waiting for user input or external API calls
- Use `READ COMMITTED` isolation (the default) unless you specifically need `REPEATABLE READ` or `SERIALIZABLE`
- Wrap multi-step writes in explicit transactions — don't rely on auto-commit for operations that must be atomic

### Vacuuming and maintenance

- **Autovacuum should always be ON** — never disable it
- Tables with heavy UPDATE/DELETE churn may need more aggressive autovacuum settings:
  ```sql
  ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.05,      -- default 0.2
    autovacuum_analyze_scale_factor = 0.02       -- default 0.1
  );
  ```
- Monitor dead tuples: `SELECT relname, n_dead_tup FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;`
- Run `ANALYZE` after bulk inserts/updates to update query planner statistics

### Migrations

- **Always run migrations in a transaction** so they roll back cleanly on failure
- **`ALTER TABLE ... ADD COLUMN`** with a non-null `DEFAULT` is safe in Postgres 11+ (no table rewrite)
- **Adding an index** — use `CREATE INDEX CONCURRENTLY` to avoid locking the table:
  ```sql
  CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
  ```
  Note: `CONCURRENTLY` cannot run inside a transaction block
- **Renaming columns** — never rename a column in a single deploy. Instead: add new column → backfill → deploy code using new column → drop old column
- **Dropping columns** — code should stop reading the column before the migration drops it

### Backups

- Use `pg_dump` for logical backups (portable, per-database)
- Use continuous archiving (WAL archiving + `pg_basebackup`) for point-in-time recovery
- **Test your restores** — an untested backup is not a backup
- For Railway: Railway provides daily snapshots, but for critical data, also set up an external backup to S3 or similar

### Monitoring

Key metrics to watch:
- **Active connections** vs `max_connections`
- **Cache hit ratio** — should be >99% for production:
  ```sql
  SELECT sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
  FROM pg_statio_user_tables;
  ```
- **Dead tuples** / autovacuum activity
- **Long-running queries**:
  ```sql
  SELECT pid, now() - pg_stat_activity.query_start AS duration, query
  FROM pg_stat_activity
  WHERE state != 'idle' AND now() - pg_stat_activity.query_start > interval '30 seconds'
  ORDER BY duration DESC;
  ```
- **Table bloat** — tables with high dead tuple ratios may need manual `VACUUM FULL` (locks the table)

## 5. Security

### Roles and privileges

- **Never use the `postgres` superuser role** for application connections
- Create a dedicated application role with minimal privileges:
  ```sql
  CREATE ROLE app_user LOGIN PASSWORD 'strong-password-here';
  GRANT CONNECT ON DATABASE mydb TO app_user;
  GRANT USAGE ON SCHEMA public TO app_user;
  GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
  ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;
  ```
- For read-only services (reporting, dashboards): create a separate role with only `SELECT`
- Revoke `CREATE` on `public` schema from `PUBLIC` (Postgres 15+ does this by default):
  ```sql
  REVOKE CREATE ON SCHEMA public FROM PUBLIC;
  ```

### SQL injection prevention

- **Always use parameterized queries** — never concatenate user input into SQL strings
- If you must build dynamic SQL (e.g., dynamic `ORDER BY`), whitelist allowed column names:
  ```typescript
  const ALLOWED_SORT_COLUMNS = ["created_at", "name", "status"];
  if (!ALLOWED_SORT_COLUMNS.includes(sortBy)) throw new BadRequestException("Invalid sort column");
  ```
- Use your ORM's query builder — it parameterizes automatically

### Row-Level Security (RLS)

For multi-tenant applications, consider RLS as a defense-in-depth layer:
```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON projects
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Set per-request in your application:
SET LOCAL app.current_tenant_id = 'tenant-uuid-here';
```

RLS is not a replacement for application-level tenant filtering, but it prevents data leaks if application code has a bug.

### SSL/TLS

- **Always connect with SSL in production** — set `sslmode=require` (or `verify-full` for mutual TLS) in your connection string
- Railway and most managed Postgres providers enforce SSL by default

### Secrets management

- **Never commit database credentials** to version control
- Use environment variables or a secrets manager (Railway variables, AWS Secrets Manager, Vault)
- Rotate credentials periodically and after any team member offboarding

## Audit Checklist

When running a full audit, check each item and report findings:

### Schema
- [ ] All tables have explicit primary keys
- [ ] No `varchar(255)` — using `text` instead (or `varchar(n)` with a meaningful limit)
- [ ] Using `timestamptz` not `timestamp` for all time columns
- [ ] Using `numeric` not `float` for money/financial values
- [ ] Using `jsonb` not `json` for JSON columns
- [ ] All columns that should never be null have `NOT NULL`
- [ ] Foreign keys have explicit constraint names
- [ ] Boolean columns have explicit defaults

### Indexes
- [ ] Every foreign key column is indexed
- [ ] No unused indexes (check `pg_stat_user_indexes`)
- [ ] Partial indexes used where appropriate (soft deletes, status filters)
- [ ] No duplicate indexes (single-column index duplicating composite index prefix)

### Queries
- [ ] No `SELECT *` in application code
- [ ] No N+1 query patterns
- [ ] All user-facing queries have `LIMIT`
- [ ] No `NOT IN (subquery)` — using `NOT EXISTS` instead
- [ ] Large-table pagination uses cursor/keyset, not OFFSET

### Operations
- [ ] Connection pooling is configured with a bounded pool size
- [ ] `statement_timeout` is set on the application role
- [ ] Autovacuum is enabled (never disabled)
- [ ] Migrations use `CREATE INDEX CONCURRENTLY` for new indexes
- [ ] Backup strategy exists and restores have been tested

### Security
- [ ] Application connects with a dedicated role, not `postgres`
- [ ] All queries use parameterized inputs — no string concatenation
- [ ] SSL is enforced on production connections (`sslmode=require`)
- [ ] Database credentials are in environment variables, not in code
- [ ] `CREATE` on public schema is revoked from `PUBLIC`
