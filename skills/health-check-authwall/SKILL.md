# Health Check Auth Wall

Audit and fix deploy health checks for applications behind an authentication wall.

## Problem

When a web application uses auth middleware (SuperTokens, Auth0, Clerk, NextAuth, etc.), unauthenticated requests to the root URL typically return a redirect (302, 307, or 308) to the login page — not a 200. Naive health checks that only accept `200` will report the deployment as failed even though the app is running perfectly.

This is a common CI/CD false-failure that wastes time debugging "deploy failures" that aren't actually failures.

## When to apply

- Setting up deploy health checks for any app with an auth wall
- Debugging health check failures where the app is visibly working
- Writing GitHub Actions, Railway, Vercel, or any CI/CD deploy pipelines

## What to check

### 1. Identify expected HTTP status for unauthenticated root requests

```bash
# Check what your app actually returns without auth
curl -s -o /dev/null -w "%{http_code}" https://your-app.example.com/
```

Common responses from auth-protected apps:
- **307** — SuperTokens (Temporary Redirect to `/auth`)
- **302** — NextAuth, Auth0 (Found, redirect to login)
- **308** — Some frameworks (Permanent Redirect)
- **401** — API-only apps without a redirect

### 2. Fix the health check (pick one approach)

**Option A: Accept redirect status codes** (simplest)
```yaml
# GitHub Actions example
- name: Health check
  run: |
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://your-app.example.com/)
    if [ "$STATUS" = "200" ] || [ "$STATUS" = "302" ] || [ "$STATUS" = "307" ]; then
      echo "Healthy (HTTP $STATUS)"
      exit 0
    fi
    echo "Unhealthy (HTTP $STATUS)"
    exit 1
```

**Option B: Add a dedicated health endpoint** (more robust, recommended for production)
```typescript
// Next.js: app/api/health/route.ts
export async function GET() {
  return Response.json({ status: "ok" });
}

// NestJS: health.controller.ts
@Public()  // Bypass auth guard
@Controller("health")
export class HealthController {
  @Get()
  check() { return { status: "ok" }; }
}
```

Then point your health check at `/api/health` or `/health` instead of `/`.

**Option C: Follow redirects in the health check**
```bash
# -L follows redirects; check that the final page loads
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -L https://your-app.example.com/)
```

### 3. Apply to all environments

If you fix staging, check production too. They likely have the same health check logic.

## Checklist

- [ ] Verified what HTTP status code your app returns for unauthenticated root requests
- [ ] Health check accepts that status code (or uses a dedicated health endpoint)
- [ ] Both staging and production health checks are consistent
- [ ] Health check has a retry loop with backoff (services need time to start)
