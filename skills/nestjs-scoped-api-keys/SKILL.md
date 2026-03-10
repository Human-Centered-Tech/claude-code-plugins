# NestJS Scoped API Keys

Pattern for adding scoped API key authentication to a NestJS application that already uses session-based auth (e.g., SuperTokens, Passport). Supports per-key roles, project scoping, and coexistence with existing auth guards.

## Problem

You need to support API access (for agents, integrations, CI/CD) alongside your existing session-based auth, without duplicating permission logic. API keys should have scoped permissions (role, allowed projects) and go through the same guards as session users.

## When to apply

- Adding API/webhook access to a NestJS app that currently only supports session auth
- Building agent-to-platform integrations
- Any NestJS project that needs both interactive (browser) and programmatic (API key) access

## Architecture

### 1. Database schema

Store hashed keys, never raw keys:
```typescript
// schema.ts (Drizzle example)
export const apiKeys = pgTable("api_keys", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: text("name").notNull(),
  keyHash: text("key_hash").notNull(),       // SHA-256 of the raw key
  keyPrefix: text("key_prefix").notNull(),   // First 8 chars for identification
  role: apiKeyRoleEnum("role").notNull(),     // Same roles as your user system
  projectIds: jsonb("project_ids").$type<string[]>(), // null = all projects
  createdBy: text("created_by").notNull(),
  lastUsedAt: timestamp("last_used_at", { withTimezone: true }),
  revokedAt: timestamp("revoked_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

### 2. API key service

Generate keys with a recognizable prefix for easy identification:
```typescript
@Injectable()
export class ApiKeyService {
  private hash(raw: string): string {
    return createHash("sha256").update(raw).digest("hex");
  }

  async create(params: { name: string; role: UserRole; projectIds?: string[]; createdBy: string }) {
    const rawKey = `yourprefix_${randomBytes(32).toString("hex")}`;  // 64 hex chars
    const keyHash = this.hash(rawKey);
    const keyPrefix = rawKey.slice(0, 12);

    const rows = await this.db.insert(apiKeys).values({
      name: params.name,
      keyHash,
      keyPrefix,
      role: params.role,
      projectIds: params.projectIds ?? null,
      createdBy: params.createdBy,
    }).returning();

    return { ...rows[0]!, rawKey };  // Only time the raw key is returned
  }

  async findByRawKey(rawKey: string) {
    const keyHash = this.hash(rawKey);
    const [row] = await this.db.select().from(apiKeys)
      .where(and(eq(apiKeys.keyHash, keyHash), isNull(apiKeys.revokedAt)));
    if (!row) return null;

    // Fire-and-forget: update last_used_at
    this.db.update(apiKeys).set({ lastUsedAt: new Date() })
      .where(eq(apiKeys.id, row.id)).then(() => {}, () => {});

    return row;
  }
}
```

### 3. Extend the auth guard

Add API key lookup as a new path in your existing session auth guard:
```typescript
@Injectable()
export class SessionAuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly apiKeyService: ApiKeyService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Check @Public() decorator first
    const isPublic = this.reflector.getAllAndOverride<boolean>(
      IS_PUBLIC_KEY, [context.getHandler(), context.getClass()],
    );
    if (isPublic) return true;

    const req = context.switchToHttp().getRequest();
    const apiKeyHeader = req.headers["x-api-key"];

    // Path 1: Session auth (existing logic)
    // ...

    // Path 2: API key auth
    if (apiKeyHeader) {
      const keyRecord = await this.apiKeyService.findByRawKey(apiKeyHeader);
      if (!keyRecord) return false;

      req.user = { id: keyRecord.id, role: keyRecord.role, email: "apikey" };
      req.authType = "apikey";
      req.apiKeyProjectIds = keyRecord.projectIds;  // For project access guard
      return true;
    }

    return false;
  }
}
```

### 4. The `@Public()` decorator

For endpoints that need no auth at all (registration, health checks):
```typescript
export const IS_PUBLIC_KEY = "isPublic";
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

Use with `Reflector` in the guard (shown above). The guard checks for this metadata before doing any auth work.

### 5. Project access guard integration

If you have a project access guard, extend it to check API key scoping:
```typescript
if (req.authType === "apikey") {
  const allowedProjects = req.apiKeyProjectIds;
  if (!allowedProjects) {
    // Null = unrestricted (but still subject to role checks)
    return req.user.role === "developer";
  }
  return allowedProjects.includes(projectId);
}
```

### 6. Optional service injection

When adding webhook/notification services that fire from existing services, use `@Optional()` so tests don't need to provide the dependency:
```typescript
constructor(
  @Optional() @Inject(WebhookService)
  private readonly webhookService?: WebhookService,
) {}

// Usage: safe even if webhookService is undefined
this.webhookService?.fire("event", projectId, payload)
  .catch((e) => this.logger.warn(`Webhook failed: ${e}`));
```

## Key patterns

- **Hash, don't encrypt** — API keys are looked up by hash (like passwords), never decrypted
- **Prefix for identification** — `yourprefix_abc123...` lets users identify which key they're using without exposing the secret
- **Revoke, don't delete** — Set `revokedAt` instead of deleting rows; `findByRawKey` filters out revoked keys
- **Same role system** — API keys use the same roles as users, so existing `@RequireRole()` decorators work unchanged
- **Project scoping** — `projectIds: null` means unrestricted; `projectIds: ["uuid1", "uuid2"]` restricts to specific projects
- **Fire-and-forget `lastUsedAt`** — Don't block the request to update usage timestamp

## Checklist

- [ ] API keys stored as SHA-256 hashes in the database
- [ ] Raw key only returned once at creation time
- [ ] Auth guard checks `@Public()` before doing auth work
- [ ] API key path sets same `req.user` shape as session auth
- [ ] Project access guard handles `apiKeyProjectIds` scoping
- [ ] Existing `@RequireRole()` decorators work for both session and API key users
- [ ] `lastUsedAt` updated on each use (fire-and-forget)
- [ ] Revocation sets `revokedAt` rather than deleting the row
