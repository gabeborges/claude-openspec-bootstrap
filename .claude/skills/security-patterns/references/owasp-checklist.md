# OWASP Top 10 Checklist for Next.js + Supabase

Detailed security checks tailored to this stack: **Next.js (App Router), Supabase, Stripe, Google OAuth**.

Use this checklist when:
- Reviewing PRs (code-reviewer, security-auditor)
- Designing new features (security-engineer)
- Auditing existing code

---

## 1. Broken Access Control

**Risk:** Users access resources they shouldn't (other users' data, admin functions).

### Checks

- [ ] **RLS enabled** on all tables with user data
- [ ] **RLS policies tested** with multiple user accounts
- [ ] **User ID from session only** (`session.user.id`), NEVER from request body/query/headers
- [ ] **Authorization checks** on EVERY endpoint (API routes, Server Actions, Server Components)
- [ ] **No client-side auth logic** — auth checks happen on server
- [ ] **Admin routes protected** — verify role from session, not request
- [ ] **File uploads scoped** to authenticated user (no arbitrary paths)
- [ ] **Direct object references** validated (user can't access `/api/orders/123` if order belongs to someone else)

### Next.js Specific

```typescript
// BAD: User ID from request
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const userId = searchParams.get('userId') // ATTACKER CONTROLS THIS
}

// GOOD: User ID from session
export async function GET(request: Request) {
  const supabase = createRouteHandlerClient({ cookies })
  const { data: { session } } = await supabase.auth.getSession()

  if (!session) {
    return new Response('Unauthorized', { status: 401 })
  }

  const userId = session.user.id // TRUSTED
}
```

### Supabase RLS Example

```sql
-- Users can only read their own orders
CREATE POLICY "Users can view own orders"
  ON orders
  FOR SELECT
  USING (auth.uid() = user_id);
```

---

## 2. Cryptographic Failures

**Risk:** Sensitive data exposed due to weak crypto, missing encryption, or hardcoded secrets.

### Checks

- [ ] **HTTPS enforced** (Vercel does this by default; verify no mixed content)
- [ ] **No secrets in code** (API keys, passwords, tokens)
- [ ] **Secrets in environment variables** (`.env.local`, Vercel env vars)
- [ ] **`.env` files in `.gitignore`**
- [ ] **No secrets in logs** (no `console.log(process.env)`)
- [ ] **No secrets in client code** (NEXT_PUBLIC_* only for non-sensitive config)
- [ ] **Supabase service role key NEVER exposed** to client
- [ ] **Passwords never stored in plaintext** (Supabase Auth handles this)
- [ ] **Sensitive data encrypted at rest** (Supabase encrypts by default; verify for custom storage)

### Verification

```bash
# Check staged files for secrets before commit
git diff --staged | grep -i 'password\|api_key\|secret\|token'

# Should return nothing
```

### Client vs Server Secrets

```typescript
// SAFE: Public config
const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL

// SAFE: Anon key (client-safe, respects RLS)
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY

// DANGER: Service role key (server-only, bypasses RLS!)
const supabaseServiceKey = process.env.SUPABASE_SERVICE_ROLE_KEY // NEVER in client code
```

---

## 3. Injection

**Risk:** SQL injection, command injection, XSS, etc.

### SQL Injection

- [ ] **No raw SQL** with user input
- [ ] **Supabase client** for all queries (parameterized)
- [ ] **No string concatenation** in queries

```typescript
// GOOD: Parameterized query
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('email', userEmail)

// BAD: String concatenation
const query = `SELECT * FROM users WHERE email = '${userEmail}'` // VULNERABLE
```

### XSS (Cross-Site Scripting)

- [ ] **User-generated content sanitized** before rendering
- [ ] **React escapes by default** (don't use `dangerouslySetInnerHTML` without sanitization)
- [ ] **DOMPurify** for sanitizing HTML/Markdown

```typescript
import DOMPurify from 'isomorphic-dompurify'

// Sanitize before rendering
const safeHTML = DOMPurify.sanitize(userInput)

return <div dangerouslySetInnerHTML={{ __html: safeHTML }} />
```

### Command Injection

- [ ] **No shell commands** with user input
- [ ] **If shell commands required**, use parameterized APIs (not `exec`)

---

## 4. Insecure Design

**Risk:** Missing threat modeling, insufficient security requirements.

### Checks

- [ ] **Threat model exists** for new features (STRIDE framework)
- [ ] **Security requirements** defined in `security.md`
- [ ] **Attack scenarios considered** (what could go wrong?)
- [ ] **Defense in depth** (multiple layers of security, not relying on one control)

### STRIDE Quick Check

For each endpoint:

- **Spoofing:** Can someone impersonate another user? → Session verification
- **Tampering:** Can data be modified? → RLS, validation
- **Repudiation:** Can actions be denied? → Audit logs
- **Information Disclosure:** Can unauthorized users access data? → Auth checks, RLS
- **Denial of Service:** Can the endpoint be overwhelmed? → Rate limiting
- **Elevation of Privilege:** Can users gain unauthorized access? → Role checks

---

## 5. Security Misconfiguration

**Risk:** Default settings, unnecessary features, missing security headers.

### Checks

- [ ] **No default credentials** (Supabase generates secure defaults)
- [ ] **Security headers configured** (CSP, X-Frame-Options, etc.)
- [ ] **Error messages don't leak info** (no stack traces in production)
- [ ] **Debug mode disabled** in production
- [ ] **CORS configured** (not `*` in production)

### Next.js Security Headers

```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'X-Frame-Options',
    value: 'DENY',
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff',
  },
  {
    key: 'Referrer-Policy',
    value: 'origin-when-cross-origin',
  },
  {
    key: 'Content-Security-Policy',
    value: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline';",
  },
]

module.exports = {
  async headers() {
    return [
      {
        source: '/:path*',
        headers: securityHeaders,
      },
    ]
  },
}
```

### Error Handling

```typescript
// BAD: Leaks internal details
catch (error) {
  return new Response(error.stack, { status: 500 })
}

// GOOD: Generic error
catch (error) {
  console.error('Internal error:', error) // Server logs only
  return new Response('Internal Server Error', { status: 500 })
}
```

---

## 6. Vulnerable and Outdated Components

**Risk:** Using dependencies with known vulnerabilities.

### Checks

- [ ] **`npm audit` passes** (no high/critical vulnerabilities)
- [ ] **Dependencies up to date** (especially security-critical ones: auth, crypto, parsing)
- [ ] **Lockfile committed** (`package-lock.json` ensures reproducible builds)
- [ ] **Renovate/Dependabot enabled** (automated dependency updates)

### Audit Commands

```bash
# Check for vulnerabilities
npm audit

# Fix automatically (if possible)
npm audit fix

# Manual review required for breaking changes
npm audit fix --force
```

### Critical Dependencies to Monitor

- `next`
- `react`, `react-dom`
- `@supabase/auth-helpers-nextjs`
- `stripe`
- `zod` (input validation)

---

## 7. Identification and Authentication Failures

**Risk:** Weak auth, session fixation, credential stuffing.

### Checks

- [ ] **Supabase Auth used** (don't roll your own)
- [ ] **Session expiration configured** (reasonable TTL)
- [ ] **Multi-factor auth supported** (if applicable)
- [ ] **Password reset flow secure** (Supabase handles this)
- [ ] **No session data in localStorage** (use httpOnly cookies)
- [ ] **Session verified on EVERY request** (don't cache session indefinitely)

### Session Verification Pattern

```typescript
// Verify session on every request
const { data: { session }, error } = await supabase.auth.getSession()

if (!session) {
  return new Response('Unauthorized', { status: 401 })
}

// Session is valid, proceed
```

### Google OAuth Security

- [ ] **OAuth scopes minimal** (only request what you need)
- [ ] **Callback URL whitelisted** (in Google Cloud Console)
- [ ] **State parameter used** (CSRF protection, Supabase handles this)

---

## 8. Software and Data Integrity Failures

**Risk:** Unsigned code, unverified webhooks, CI/CD compromise.

### Checks

- [ ] **Stripe webhook signatures verified**
- [ ] **No code from CDNs** without SRI (Subresource Integrity)
- [ ] **CI/CD secrets secured** (use GitHub Secrets, not hardcoded)
- [ ] **Dependency integrity** (package-lock.json committed)

### Stripe Webhook Verification

```typescript
import { stripe } from '@/lib/stripe'
import { headers } from 'next/headers'

export async function POST(request: Request) {
  const body = await request.text()
  const signature = headers().get('stripe-signature')!

  let event

  try {
    // CRITICAL: Verify signature
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    console.error('Webhook signature verification failed:', err.message)
    return new Response('Webhook Error', { status: 400 })
  }

  // Webhook is verified, safe to process
  switch (event.type) {
    case 'checkout.session.completed':
      // ...
  }
}
```

---

## 9. Security Logging and Monitoring Failures

**Risk:** Attacks go undetected, incidents can't be investigated.

### Checks

- [ ] **Audit logs** for sensitive actions (user creation, role changes, data exports)
- [ ] **Log security events** (failed auth, unauthorized access attempts)
- [ ] **No sensitive data in logs** (passwords, tokens, PII)
- [ ] **Logs include** user ID, timestamp, action, IP (if applicable)
- [ ] **Logs centralized** (Vercel logs, or external service like Datadog)

### Safe Logging Pattern

```typescript
// GOOD: Log metadata
console.log({
  event: 'user.login.success',
  userId: session.user.id,
  timestamp: new Date().toISOString(),
  ip: request.headers.get('x-forwarded-for'),
})

// BAD: Log sensitive data
console.log({
  event: 'user.login.success',
  email: 'user@example.com', // PII
  password: 'abc123', // NEVER LOG PASSWORDS
})
```

### What to Log

**Do log:**
- User ID (from session)
- Action type (`user.update`, `order.create`)
- Timestamp
- Request ID (for tracing)
- Non-sensitive error codes

**Do NOT log:**
- Passwords (plaintext or hashed)
- API keys, tokens, secrets
- PII (emails, names, addresses)
- PHI (health data)
- Full request/response bodies

---

## 10. Server-Side Request Forgery (SSRF)

**Risk:** Attacker tricks server into making requests to internal/external resources.

### Checks

- [ ] **User-provided URLs validated** (whitelist, not blacklist)
- [ ] **No arbitrary fetch** from user input
- [ ] **Internal IPs blocked** (127.0.0.1, 169.254.x.x, 10.x.x.x, 192.168.x.x)
- [ ] **Timeout on external requests** (prevent resource exhaustion)

### SSRF Prevention Pattern

```typescript
// BAD: Arbitrary URL from user
export async function POST(request: Request) {
  const { url } = await request.json()
  const response = await fetch(url) // ATTACKER CONTROLS THIS
}

// GOOD: Whitelist allowed domains
const ALLOWED_DOMAINS = ['api.stripe.com', 'api.github.com']

export async function POST(request: Request) {
  const { url } = await request.json()

  const parsed = new URL(url)

  if (!ALLOWED_DOMAINS.includes(parsed.hostname)) {
    return new Response('Invalid URL', { status: 400 })
  }

  // Safe to fetch
  const response = await fetch(url)
}
```

---

## Summary: Pre-Merge Security Gate

Before approving any PR, verify:

- [ ] **A1: Broken Access Control** — Auth checks present, RLS enabled
- [ ] **A2: Cryptographic Failures** — No secrets in code/logs
- [ ] **A3: Injection** — Input validated, parameterized queries
- [ ] **A4: Insecure Design** — Threat model exists
- [ ] **A5: Security Misconfiguration** — Security headers, no debug mode
- [ ] **A6: Vulnerable Components** — `npm audit` passes
- [ ] **A7: Auth Failures** — Session verified on every request
- [ ] **A8: Integrity Failures** — Webhook signatures verified
- [ ] **A9: Logging Failures** — Actions logged, no sensitive data
- [ ] **A10: SSRF** — URLs validated/whitelisted

If ANY check fails: **REQUEST_CHANGES**, create remediation ticket in `tasks.yaml`.

---

## Remediation Severity Levels

**SEV-1 (Critical):**
- Secrets in code/logs
- Missing auth checks
- RLS disabled on user data tables
- SQL injection vulnerabilities

**SEV-2 (High):**
- Missing input validation
- Unverified webhook signatures
- Missing rate limiting on auth endpoints
- PII in logs

**SEV-3 (Medium):**
- Missing security headers
- Outdated dependencies with known CVEs
- Missing audit logs
- Weak error messages (info leakage)

**SEV-1/2 block merge. SEV-3 can be deferred** (create ticket, don't block).
