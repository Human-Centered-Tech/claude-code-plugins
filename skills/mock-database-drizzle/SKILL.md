# Mock Database for Drizzle ORM

Pattern for building an in-memory mock of a Drizzle database for unit and integration tests in NestJS projects. Eliminates the need for a real database in tests while supporting Drizzle's chainable query API.

## Problem

Drizzle ORM uses a deeply chainable API (`db.select().from(table).where(cond).orderBy(...).limit(n)`). Mocking individual methods with `jest.fn()` is fragile and doesn't test the actual query patterns your services use. You need a mock that stores data in memory and correctly handles the chain.

## When to apply

- Writing unit tests for NestJS services that use Drizzle ORM
- Integration tests where you want to avoid spinning up a real Postgres instance
- Any test that needs to verify data was inserted/updated/deleted correctly

## Architecture

Create a `createMockDatabase()` factory that returns a mock implementing Drizzle's chainable API backed by an in-memory `Map<string, Row[]>`.

### Key components

1. **Store** — `Map<string, Row[]>` keyed by table name (extracted via `Symbol.for("drizzle:Name")`)
2. **Condition parser** — Walks Drizzle's SQL AST (`queryChunks`) to extract column/value/operator triples from `eq()`, `ne()`, `and()` calls
3. **Chainable methods** — Each method returns an object with the next valid methods in the chain

### Table name extraction

Drizzle stores table names using a well-known symbol:
```typescript
function tableName(table: unknown): string {
  const nameSymbol = Symbol.for("drizzle:Name");
  const t = table as Record<string | symbol, unknown>;
  if (nameSymbol in t) return t[nameSymbol] as string;
  return "__unknown__";
}
```

### Condition parsing

Drizzle's `eq()`, `ne()`, `and()` produce SQL objects with `queryChunks` arrays containing:
- **StringChunk** — `{ value: [" = "] }` or `{ value: [" <> "] }`
- **Column** — `{ name: "column_name", table: ... }`
- **Param** — `{ value: actualValue, encoder: ... }`

Walk the chunks to extract filter predicates, then apply them to the in-memory rows.

### Column name mapping

Drizzle uses snake_case SQL column names but your TypeScript rows use camelCase. The mock needs to check both:
```typescript
function snakeToCamel(s: string): string {
  return s.replace(/_([a-z])/g, (_, c) => c.toUpperCase());
}

function matchesRow(row: Row, conditions: Condition[]): boolean {
  for (const cond of conditions) {
    const camelKey = snakeToCamel(cond.column);
    const rowVal = camelKey in row ? row[camelKey] : row[cond.column];
    if (cond.op === "eq" && rowVal !== cond.value) return false;
    if (cond.op === "ne" && rowVal === cond.value) return false;
  }
  return true;
}
```

### Supported operations

The mock should support these chains at minimum:
```
db.insert(table).values(data).returning()
db.insert(table).values(data).onConflictDoUpdate({...}).returning()
db.select().from(table).where(cond).orderBy(...).limit(n)
db.select({fields}).from(table).where(cond)
db.update(table).set(data).where(cond).returning()
db.delete(table).where(cond).returning()
db.execute(sql)
```

### Making chains awaitable

Drizzle queries resolve when awaited. For `select().from()`, add a `then` method so `await db.select().from(table)` works:
```typescript
const fromResult = {
  where: jest.fn(/* ... */),
  orderBy: jest.fn(/* ... */),
  then: (resolve: (v: Row[]) => void) => resolve(allRows()),
};
```

For `where()` results that chain to `orderBy()` and `limit()`, return the array with extra methods attached:
```typescript
const withChaining: Row[] & { limit: jest.Mock; orderBy: jest.Mock } =
  Object.assign([...projected], {
    limit: jest.fn((n: number) => projected.slice(0, n)),
    orderBy: jest.fn(function(this: typeof withChaining) { return withChaining; }),
  });
```

### Auto-generated defaults

Real databases provide defaults (UUIDs, timestamps). The mock should do the same:
```typescript
if (row.id === undefined) row.id = randomUUID();
for (const key of ["createdAt", "addedAt", "updatedAt"]) {
  if (row[key] === undefined) row[key] = new Date();
}
```

## Usage in tests

```typescript
// Provide as a NestJS module override
const mockDb = createMockDatabase();

const module = await Test.createTestingModule({
  imports: [YourModule],
}).overrideProvider(DATABASE_TOKEN)
  .useValue(mockDb)
  .compile();

// After test actions, verify directly:
const rows = mockDb._getRows("your_table");
expect(rows).toHaveLength(1);
expect(rows[0].status).toBe("active");

// Clear between tests:
mockDb._clear();
```

## Gotchas

- **Field projection** with `db.select({ alias: table.column })` requires mapping the Drizzle column's `.name` property to the alias
- **Aggregate functions** like `count()` produce SQL expressions without `.name`/`.table` — detect with `isSqlExpression()` and return computed values
- **`and()` with a single argument** still wraps in a SQL object — your parser must handle this
- **Draft/status filtering** — if your service filters by status (e.g., `ne(items.status, "draft")`), integration tests should verify data directly from `mockDb._getRows()` rather than through HTTP endpoints that apply additional filtering
