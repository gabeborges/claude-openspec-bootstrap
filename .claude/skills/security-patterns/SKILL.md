---
name: security-patterns
description: Security knowledge for Next.js + Supabase stack — auth patterns, OWASP checks, threat modeling, and secure implementation guidance.
---

# Security Patterns

Consolidated security knowledge for building secure features on Next.js (App Router) + Supabase + Stripe + Google OAuth.

## Scope

**Use for:**
- Defining security patterns for new features
- Auditing implementations for vulnerabilities
- Reviewing code for security compliance
- Threat modeling API endpoints and data flows

**Not for:**
- Infrastructure security (hosting, network, DNS)
- Third-party service security (that's their job)

---

# Authentication & Authorization

## Supabase Auth Patterns

This project uses **Supabase Auth** with **Google OAuth**.

### Server-Side Auth Pattern

```typescript
// app/api/some-route/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'

export async function GET(request: Request) {
  const supabase = createRouteHandlerClient({ cookies })

  // Always verify session first
  const { data: { session }, error } = await supabase.auth.getSession()

  if (!session) {
    return new Response('Unauthorized', { status: 401 })
  }

  // User ID from session, NEVER from request
  const userId = session.user.id

  // ... proceed with authenticated logic
}
```

### Server Component Auth Pattern

```typescript
// app/dashboard/page.tsx
import { createServerComponentClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const supabase = createServerComponentClient({ cookies })
  const { data: { session } } = await supabase.auth.getSession()

  if (!session) {
    redirect('/login')
  }

  // ... render authenticated UI
}
```

### Client Component Auth Pattern

```typescript
// components/UserProfile.tsx
'use client'

import { createClientComponentClient } from '@supabase/auth-helpers-nextjs'
import { useEffect, useState } from 'react'

export function UserProfile() {
  const supabase = createClientComponentClient()
  const [user, setUser] = useState(null)

  useEffect(() => {
    const getUser = async () => {
      const { data: { session } } = await supabase.auth.getSession()
      setUser(session?.user ?? null)
    }

    getUser()

    // Listen for auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      setUser(session?.user ?? null)
    })

    return () => subscription.unsubscribe()
  }, [supabase])

  // ... render
}
```

### Authorization Rules

**NEVER bypass Supabase RLS (Row Level Security).**

- User ID comes from `session.user.id`, NEVER from request body/query/headers
- RLS policies enforce data access control at the database level
- Every table with user data MUST have RLS enabled
- Client code CANNOT bypass RLS (it's enforced server-side by Postgres)

### OAuth Scopes

Current scopes (Google OAuth):
- `email`
- `profile`

**Do not add scopes** without spec reference and explicit justification.

---

# Input Validation

## Server-Side Validation (Required)

**NEVER trust client input.** Validate everything on the server.

### Pattern: Zod Schemas

```typescript
import { z } from 'zod'

const UserCreateSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['user', 'admin']),
})

export async function POST(request: Request) {
  const body = await request.json()

  // Validate input
  const result = UserCreateSchema.safeParse(body)

  if (!result.success) {
    return new Response(
      JSON.stringify({ error: 'Invalid input', details: result.error.issues }),
      { status: 400, headers: { 'Content-Type': 'application/json' } }
    )
  }

  const validatedData = result.data

  // ... proceed with validated data
}
```

### Validation Checklist

- [ ] **Length limits** on all string inputs (prevent DoS)
- [ ] **Type validation** (string, number, boolean, enum)
- [ ] **Format validation** (email, URL, UUID)
- [ ] **Range validation** for numbers
- [ ] **Enum whitelisting** for fixed sets of values
- [ ] **Sanitization** of HTML/Markdown if displayed

### SQL Injection Prevention

Use **Supabase client** or **parameterized queries** only. NEVER build SQL strings from user input.

```typescript
// GOOD: Parameterized query
const { data, error } = await supabase
  .from('users')
  .select('*')
  .eq('id', userId)

// BAD: String concatenation (DO NOT DO THIS)
const query = `SELECT * FROM users WHERE id = '${userId}'`
```

---

# Secrets Management

## What Goes Where

### Environment Variables (`.env.local`)

**Local development only.** Never commit these files.

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOi...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOi...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-...
```

### Public vs Private

**NEXT_PUBLIC_*** variables are exposed to the browser. Only use for non-sensitive config.

- `NEXT_PUBLIC_SUPABASE_URL` — OK (public)
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` — OK (anon key is safe for client)
- `SUPABASE_SERVICE_ROLE_KEY` — NEVER expose to client (bypasses RLS!)
- `STRIPE_SECRET_KEY` — Server-only
- `STRIPE_WEBHOOK_SECRET` — Server-only

### .gitignore

Verify these are ignored:

```
.env
.env.*
.env.local
.env.development.local
.env.test.local
.env.production.local
```

### Secret Stores (Production)

For production, use:
- Vercel Environment Variables (encrypted)
- AWS Secrets Manager
- Doppler / Vault (if applicable)

NEVER hardcode secrets in code, even for "temporary" testing.

---

# Threat Modeling

Use this framework when designing new features.

## STRIDE Framework

For each new endpoint or data flow, ask:

**S — Spoofing:** Can someone impersonate another user?
- Mitigation: Verify session, never trust client-provided user IDs

**T — Tampering:** Can data be modified in transit or at rest?
- Mitigation: HTTPS (enforced), database constraints, audit logs

**R — Repudiation:** Can a user deny performing an action?
- Mitigation: Audit logs with user ID + timestamp

**I — Information Disclosure:** Can unauthorized users access data?
- Mitigation: RLS policies, auth checks, no sensitive data in logs

**D — Denial of Service:** Can the endpoint be overwhelmed?
- Mitigation: Rate limiting, input size limits, pagination

**E — Elevation of Privilege:** Can a user gain unauthorized permissions?
- Mitigation: RLS policies, role checks, server-side validation

## Threat Modeling Checklist

For each new feature:

- [ ] Identify all endpoints and data flows
- [ ] Apply STRIDE to each endpoint
- [ ] Document required auth/authz checks
- [ ] Define input validation requirements
- [ ] Identify sensitive data (PII, PHI, credentials)
- [ ] Specify logging requirements (what to log, what NOT to log)
- [ ] Consider rate limiting needs
- [ ] Review for OWASP Top 10 vulnerabilities (see `references/owasp-checklist.md`)

---

# Supabase Row Level Security (RLS)

## Non-Negotiable Rule

**Every table with user data MUST have RLS enabled.**

## RLS Pattern: User-Owned Data

```sql
-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only read their own data
CREATE POLICY "Users can view own profile"
  ON users
  FOR SELECT
  USING (auth.uid() = id);

-- Policy: Users can update their own data
CREATE POLICY "Users can update own profile"
  ON users
  FOR UPDATE
  USING (auth.uid() = id);
```

## RLS Pattern: Shared Data

```sql
-- Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Policy: Users can view projects they belong to
CREATE POLICY "Users can view own projects"
  ON projects
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM project_members
      WHERE project_members.project_id = projects.id
      AND project_members.user_id = auth.uid()
    )
  );
```

## RLS Pattern: Admin-Only Data

```sql
-- Enable RLS
ALTER TABLE admin_logs ENABLE ROW LEVEL SECURITY;

-- Policy: Only admins can view logs
CREATE POLICY "Admins can view logs"
  ON admin_logs
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM users
      WHERE users.id = auth.uid()
      AND users.role = 'admin'
    )
  );
```

## Bypassing RLS (Service Role)

**Only use `SUPABASE_SERVICE_ROLE_KEY` on the server** for admin operations.

```typescript
// Server-side only!
import { createClient } from '@supabase/supabase-js'

const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!, // Bypasses RLS!
)

// Use sparingly, only for admin/system operations
const { data, error } = await supabaseAdmin
  .from('users')
  .select('*')
```

NEVER expose service role key to client.

---

# Logging & Monitoring

## What to Log

- Request ID
- User ID (from session)
- Timestamp
- Action/event type
- Non-sensitive error codes
- Performance metrics (latency, size)

## What NOT to Log

- Passwords (plaintext or hashed)
- API keys / tokens
- PII/PHI payloads (names, emails, addresses, health data)
- Full request/response bodies (may contain secrets)

## Safe Logging Pattern

```typescript
// GOOD: Log metadata only
console.log({
  requestId: generateId(),
  userId: session.user.id,
  action: 'user.update',
  timestamp: new Date().toISOString(),
  success: true,
})

// BAD: Log sensitive data
console.log({
  user: { email: 'user@example.com', password: 'abc123' }, // DO NOT DO THIS
})
```

---

# Stripe Security

## Webhook Signature Verification

**ALWAYS verify webhook signatures.** Unsigned webhooks can be forged.

```typescript
// app/api/webhooks/stripe/route.ts
import { stripe } from '@/lib/stripe'
import { headers } from 'next/headers'

export async function POST(request: Request) {
  const body = await request.text()
  const signature = headers().get('stripe-signature')!

  let event

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    console.error('Webhook signature verification failed:', err.message)
    return new Response('Webhook signature verification failed', { status: 400 })
  }

  // ... process verified event
}
```

## Price ID Security

**Do not accept price IDs from client.**

```typescript
// BAD: Client chooses price
const { priceId } = await request.json() // Client can modify this!

// GOOD: Server determines price based on plan
const planToPriceId = {
  starter: process.env.STRIPE_PRICE_STARTER!,
  pro: process.env.STRIPE_PRICE_PRO!,
}
const priceId = planToPriceId[plan]
```

---

# Rate Limiting

## Pattern: Vercel Edge Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'), // 10 requests per 10 seconds
})

export async function middleware(request: NextRequest) {
  const ip = request.ip ?? '127.0.0.1'
  const { success } = await ratelimit.limit(ip)

  if (!success) {
    return new Response('Too Many Requests', { status: 429 })
  }

  return NextResponse.next()
}

export const config = {
  matcher: '/api/:path*',
}
```

## When to Rate Limit

- All public API endpoints
- Login/signup endpoints (prevent brute force)
- Password reset endpoints
- High-cost operations (image processing, reports)

---

# OWASP Top 10 Quick Reference

For detailed checks, see `references/owasp-checklist.md`.

1. **Broken Access Control** → RLS policies, auth checks
2. **Cryptographic Failures** → HTTPS, env vars, no hardcoded secrets
3. **Injection** → Parameterized queries, input validation
4. **Insecure Design** → Threat modeling (STRIDE)
5. **Security Misconfiguration** → No default passwords, secure headers
6. **Vulnerable Components** → Dependency scanning (npm audit)
7. **Identification/Authentication Failures** → Supabase Auth, session checks
8. **Software/Data Integrity Failures** → Webhook signature verification
9. **Security Logging Failures** → Log actions, not sensitive data
10. **SSRF** → Validate/whitelist external URLs

---

# Security Checklist (New Features)

Before marking a feature "done":

- [ ] All endpoints require authentication (unless explicitly public)
- [ ] User ID extracted from session, NEVER from request
- [ ] RLS policies defined and tested
- [ ] Input validation on all server endpoints (Zod schemas)
- [ ] No secrets in code, logs, or diffs
- [ ] Stripe webhook signatures verified
- [ ] Rate limiting applied to public/high-cost endpoints
- [ ] Sensitive data NOT logged (PII, PHI, passwords, tokens)
- [ ] OWASP Top 10 reviewed (see `references/owasp-checklist.md`)
- [ ] Threat model documented (STRIDE)

---

# When to Escalate

If you encounter:
- Security requirement conflicting with spec → Create `spec-change-requests.yaml` entry
- Vulnerability in existing code → Create remediation ticket in `tasks.yaml`
- Dependency with known CVE → Update dependency, document in `decisions-log.md`

---

# References

- `references/owasp-checklist.md` — Detailed OWASP Top 10 checks for Next.js + Supabase
