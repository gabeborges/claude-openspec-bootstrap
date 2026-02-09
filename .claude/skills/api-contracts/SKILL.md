---
name: api-contracts
description: API endpoint design conventions for Next.js Route Handlers + Supabase — request/response schemas, error format, pagination, and contract testing.
---

# API Contracts

Standardized API endpoint design for Next.js Route Handlers with Zod validation, consistent error envelopes, and cursor-based pagination over Supabase.

## Scope

**Use for:**
- Designing new API endpoints (Route Handlers in `app/api/`)
- Defining request/response schemas with Zod
- Standardizing error responses
- Implementing cursor-based pagination for Supabase queries
- Contract testing between frontend and backend

**Not for:**
- External third-party API integrations (Stripe webhooks, OAuth callbacks)
- GraphQL or tRPC (this project uses REST Route Handlers)
- Database schema design (see `db-migration` skill)

---

# Response Envelope

```typescript
// lib/api/types.ts
type ApiResponse<T> = {
  data: T
  meta?: { cursor?: string | null; hasMore?: boolean; total?: number }
}

type ApiError = {
  error: { code: string; message: string; details?: unknown }
}
```

## Standard Error Codes

| HTTP | Code | When |
|------|------|------|
| 400 | `VALIDATION_ERROR` | Request body/params invalid |
| 401 | `UNAUTHORIZED` | No valid session |
| 403 | `FORBIDDEN` | Valid session, insufficient perms |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Duplicate or state conflict |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

---

# Response Helpers

```typescript
// lib/api/response.ts
import { NextResponse } from 'next/server'
import { ZodError } from 'zod'

export function ok<T>(data: T, meta?: Record<string, unknown>) {
  return NextResponse.json({ data, ...(meta && { meta }) })
}
export function created<T>(data: T) {
  return NextResponse.json({ data }, { status: 201 })
}
export function badRequest(details?: unknown) {
  return NextResponse.json({ error: { code: 'VALIDATION_ERROR', message: 'Invalid request', details } }, { status: 400 })
}
export function unauthorized() {
  return NextResponse.json({ error: { code: 'UNAUTHORIZED', message: 'Authentication required' } }, { status: 401 })
}
export function forbidden() {
  return NextResponse.json({ error: { code: 'FORBIDDEN', message: 'Insufficient permissions' } }, { status: 403 })
}
export function notFound(resource = 'Resource') {
  return NextResponse.json({ error: { code: 'NOT_FOUND', message: `${resource} not found` } }, { status: 404 })
}
export function zodError(err: ZodError) {
  return badRequest(err.issues.map(i => ({ path: i.path, message: i.message })))
}
```

---

# Zod Schema Pattern

Define schemas once, share between frontend and backend.

```typescript
// lib/schemas/items.ts
import { z } from 'zod'

export const CreateItemSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().max(2000).optional(),
  category: z.enum(['A', 'B', 'C']),
})
export const UpdateItemSchema = CreateItemSchema.partial()
export const ItemParamsSchema = z.object({ id: z.string().uuid() })

export type CreateItemInput = z.infer<typeof CreateItemSchema>
```

## Validation Helper

```typescript
// lib/api/validate.ts
import { ZodSchema, ZodError } from 'zod'

export function validate<T>(schema: ZodSchema<T>, data: unknown):
  { success: true; data: T } | { success: false; error: ZodError } {
  const result = schema.safeParse(data)
  if (result.success) return { success: true, data: result.data }
  return { success: false, error: result.error }
}
```

---

# Route Handler Pattern

```typescript
// app/api/items/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'
import { CreateItemSchema } from '@/lib/schemas/items'
import { ok, created, unauthorized, zodError } from '@/lib/api/response'
import { validate } from '@/lib/api/validate'
import { paginatedQuery } from '@/lib/api/pagination'

export async function GET(request: Request) {
  const supabase = createRouteHandlerClient({ cookies })
  const { data: { session } } = await supabase.auth.getSession()
  if (!session) return unauthorized()

  const { searchParams } = new URL(request.url)
  return paginatedQuery(supabase, 'items', {
    cursor: searchParams.get('cursor'),
    limit: Number(searchParams.get('limit') || 20),
    orderBy: 'created_at',
    filter: (q) => q.eq('user_id', session.user.id),
  })
}

export async function POST(request: Request) {
  const supabase = createRouteHandlerClient({ cookies })
  const { data: { session } } = await supabase.auth.getSession()
  if (!session) return unauthorized()

  const body = await request.json()
  const result = validate(CreateItemSchema, body)
  if (!result.success) return zodError(result.error)

  const { data, error } = await supabase
    .from('items')
    .insert({ ...result.data, user_id: session.user.id })
    .select()
    .single()

  if (error) throw error
  return created(data)
}
```

For dynamic routes (`app/api/items/[id]/route.ts`), validate params with `ItemParamsSchema` and use `await params` to access the `id` (Next.js 15 async params).

---

# Cursor-Based Pagination

```typescript
// lib/api/pagination.ts
import { SupabaseClient } from '@supabase/supabase-js'
import { ok } from './response'

type PaginationOptions = {
  cursor?: string | null; limit?: number; orderBy?: string
  ascending?: boolean; filter?: (query: any) => any
}

export async function paginatedQuery(
  supabase: SupabaseClient, table: string, options: PaginationOptions
) {
  const { cursor, limit = 20, orderBy = 'created_at', ascending = false, filter } = options
  const safeLimit = Math.min(Math.max(limit, 1), 100)

  let query = supabase.from(table).select('*', { count: 'exact' })
    .order(orderBy, { ascending }).limit(safeLimit + 1)

  if (filter) query = filter(query)
  if (cursor) query = ascending ? query.gt(orderBy, cursor) : query.lt(orderBy, cursor)

  const { data, error, count } = await query
  if (error) throw error

  const hasMore = (data?.length ?? 0) > safeLimit
  const items = hasMore ? data!.slice(0, safeLimit) : (data ?? [])
  const nextCursor = hasMore ? items[items.length - 1]?.[orderBy] : null

  return ok(items, { cursor: nextCursor, hasMore, total: count })
}
```

---

# Rate Limiting Tiers

Cross-reference `security-patterns` skill for implementation details.

| Tier | Limit | Apply to |
|------|-------|----------|
| Standard | 60 req/min | Most authenticated endpoints |
| Strict | 10 req/min | Auth, password reset, invites |
| Generous | 200 req/min | Read-only list/search endpoints |
| Webhook | No limit | Verified webhook endpoints |

---

# Contract Testing

```typescript
// app/api/items/__tests__/contract.test.ts
import { z } from 'zod'

const ItemResponseSchema = z.object({
  data: z.object({ id: z.string().uuid(), name: z.string(), category: z.enum(['A', 'B', 'C']) }),
})
const ErrorResponseSchema = z.object({
  error: z.object({ code: z.string(), message: z.string(), details: z.unknown().optional() }),
})

it('success response matches schema', async () => {
  const body = await (await POST(validRequest())).json()
  expect(() => ItemResponseSchema.parse(body)).not.toThrow()
})

it('error response matches schema', async () => {
  const res = await POST(invalidRequest())
  expect(() => ErrorResponseSchema.parse(await res.json())).not.toThrow()
  expect(res.status).toBe(400)
})
```

---

# Artifact: api-contracts.md

Produce `openspec/changes/<change-name>/api-contracts.md` using this template:

```markdown
# API Contracts: <Change Name>

## Endpoints

### `POST /api/<resource>`
- **Auth**: Required | **Rate limit**: Standard
- **Request**: `z.object({ ... })`
- **Success (201)**: `{ "data": { ... } }`
- **Errors**: 400, 401, 409

### `GET /api/<resource>`
- **Auth**: Required | **Rate limit**: Generous
- **Query**: `cursor`, `limit` (max 100)
- **Success (200)**: `{ "data": [...], "meta": { "cursor": "...", "hasMore": true } }`

## Shared Schemas
- `lib/schemas/<resource>.ts`
```

---

# SDD Phase Integration

## Specs Phase
- Include endpoint inventory in `openspec/changes/<change-name>/specs/<capability>/spec.md`
- Each endpoint: method, path, auth requirement, request/response shapes

## Design Phase
- Produce `api-contracts.md` artifact in `openspec/changes/<change-name>/api-contracts.md`
- Document Zod schemas, pagination strategy, and error handling in `design.md`

## Tasks Phase
- Add endpoint implementation tasks in `openspec/changes/<change-name>/tasks.md`
- Include contract test tasks; cross-reference `testing-strategy` skill

## Apply Phase
- Run contract tests to verify response shapes
- Verify all endpoints return standard error envelope
- Confirm rate limiting tier is applied per endpoint

---
