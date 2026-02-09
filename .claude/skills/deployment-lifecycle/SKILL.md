---
name: deployment-lifecycle
description: Deployment workflow patterns for Vercel + Supabase — environment promotion, feature flags, rollback procedures, and CI/CD pipeline stages.
---

# Deployment Lifecycle

Deployment workflow patterns for Vercel + Supabase, covering environment promotion, feature flags, rollback, and CI/CD pipeline integration.

## Scope

**Use for:**
- Designing deployment strategies for new features
- Environment promotion workflows (preview -> staging -> production)
- Feature flag patterns using environment variables
- Rollback procedures
- CI/CD pipeline stages and smoke tests
- Coordinating database migrations with deployments

**Not for:**
- Infrastructure provisioning (Vercel/Supabase account setup)
- DNS or domain configuration
- Database migration planning (see `db-migration` skill)

---

# Environment Strategy

## Three Environments

| Environment | Vercel         | Supabase Project | Branch   | URL                      |
|-------------|----------------|------------------|----------|--------------------------|
| Preview     | Preview deploy | Dev project      | Feature  | `<branch>.vercel.app`    |
| Staging     | Preview deploy | Staging project  | `staging`| `staging.example.com`    |
| Production  | Production     | Prod project     | `main`   | `example.com`            |

## Environment Variable Naming

```bash
# Pattern: <SERVICE>_<CONTEXT>
NEXT_PUBLIC_SUPABASE_URL=           # per-environment
NEXT_PUBLIC_SUPABASE_ANON_KEY=      # per-environment
SUPABASE_SERVICE_ROLE_KEY=          # per-environment, server-only
STRIPE_SECRET_KEY=                  # per-environment (test vs live)
STRIPE_WEBHOOK_SECRET=              # per-environment

# Feature flags (see Feature Flags section)
NEXT_PUBLIC_FF_NEW_CHECKOUT=true
FF_ENABLE_BATCH_EXPORT=true
```

Set variables in Vercel dashboard per environment scope (Production, Preview, Development).

---

# CI/CD Pipeline Stages

```
Push to feature branch
  |
  v
[1. Lint & Type Check] -----> fail? -> block merge
  |
  v
[2. Unit & Integration Tests] -> fail? -> block merge
  |
  v
[3. Build] -----------------> fail? -> block merge
  |
  v
[4. Vercel Preview Deploy] --> URL posted to PR
  |
  v
[5. E2E Tests on Preview] --> fail? -> block merge
  |
  v
PR approved + merged to main
  |
  v
[6. Production Deploy] -----> Vercel auto-deploys main
  |
  v
[7. Smoke Tests] -----------> fail? -> instant rollback
```

## GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main, staging]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npm run test:coverage
      - run: npm run build

  e2e:
    needs: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run test:e2e
        env:
          BASE_URL: ${{ env.VERCEL_PREVIEW_URL }}
```

---

# Feature Flags

Use environment variables for simple feature flags. No external service needed for small teams.

## Pattern

```typescript
// lib/flags.ts
export const flags = {
  newCheckout: process.env.NEXT_PUBLIC_FF_NEW_CHECKOUT === 'true',
  batchExport: process.env.FF_ENABLE_BATCH_EXPORT === 'true',
} as const

export type FeatureFlag = keyof typeof flags
```

## Usage in Server Components

```typescript
import { flags } from '@/lib/flags'

export default function PricingPage() {
  return flags.newCheckout ? <NewCheckout /> : <LegacyCheckout />
}
```

## Usage in Route Handlers

```typescript
import { flags } from '@/lib/flags'

export async function POST(request: Request) {
  if (!flags.batchExport) {
    return Response.json(
      { error: { code: 'NOT_FOUND', message: 'Not found' } },
      { status: 404 }
    )
  }
  // ... feature logic
}
```

## Flag Lifecycle

1. **Add**: Set `NEXT_PUBLIC_FF_<NAME>=false` in all environments
2. **Enable preview**: Set `true` in Preview environment only
3. **Enable staging**: Set `true` in staging
4. **Enable production**: Set `true` in Production
5. **Remove**: After stable in production, remove flag and dead code path

---

# Database Migration Coordination

Cross-reference: `db-migration` skill for migration planning.

## Deploy Order for Schema Changes

**Expand phase (safe to deploy):**
1. Deploy migration adding new columns/tables (backward-compatible)
2. Deploy application code that writes to both old and new schema
3. Backfill data if needed

**Contract phase (after verification):**
1. Deploy application code that reads only from new schema
2. Deploy migration removing old columns/tables
3. Verify no errors in production logs

**Rule:** Migrations run BEFORE application code deploys. Configure this in CI:

```yaml
# In deployment pipeline
- name: Run migrations
  run: npx supabase db push
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}

- name: Deploy application
  run: vercel deploy --prod
```

---

# Rollback Procedures

## Instant Rollback (Vercel)

Vercel keeps all previous deployments. Rollback via:
- **Dashboard**: Deployments -> select previous -> "Promote to Production"
- **CLI**: `vercel rollback`

Rollback does NOT revert database migrations. Plan migrations to be backward-compatible (expand/contract pattern).

## Rollback Decision Matrix

| Signal                        | Action                        |
|-------------------------------|-------------------------------|
| Smoke tests fail              | Immediate rollback            |
| Error rate > 1% (5 min)      | Immediate rollback            |
| CWV regression on key pages  | Investigate, rollback if P0   |
| Single user report            | Investigate, no auto-rollback |

## Smoke Tests

Run after every production deploy:

```typescript
// scripts/smoke-test.ts
const PROD_URL = process.env.PROD_URL || 'https://example.com'

const checks = [
  { name: 'Homepage loads', url: '/', status: 200 },
  { name: 'API health', url: '/api/health', status: 200 },
  { name: 'Auth redirect', url: '/dashboard', status: 307 },
]

for (const check of checks) {
  const res = await fetch(`${PROD_URL}${check.url}`, { redirect: 'manual' })
  if (res.status !== check.status) {
    console.error(`FAIL: ${check.name} - expected ${check.status}, got ${res.status}`)
    process.exit(1)
  }
  console.log(`PASS: ${check.name}`)
}
```

---

# Health Check Endpoint

```typescript
// app/api/health/route.ts
export async function GET() {
  return Response.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: process.env.VERCEL_GIT_COMMIT_SHA?.slice(0, 7) ?? 'dev',
  })
}
```

---

# SDD Phase Integration

## Design Phase
- Include "Deployment Strategy" section in `openspec/changes/<change-name>/design.md`:
  - Feature flag requirements
  - Migration deploy order (coordinate with `db-migration` skill)
  - Rollback considerations

## Tasks Phase
- Add deployment tasks in `openspec/changes/<change-name>/tasks.md`:
  - Environment variable setup per environment
  - Feature flag creation
  - Smoke test additions
  - Migration coordination steps

## Apply Phase
- Verify feature flag is disabled in production before merge
- Run smoke tests after deploy
- Monitor error rate for 15 minutes post-deploy

## Archive Phase
- Document post-deploy verification results
- Clean up feature flags after stable rollout
- Update environment variable documentation if changed

---
