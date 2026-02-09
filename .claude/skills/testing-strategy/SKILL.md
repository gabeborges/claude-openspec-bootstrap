---
name: testing-strategy
description: Test architecture and coverage strategy for Next.js + Supabase — mapping spec acceptance criteria to automated tests, coverage gates, and CI integration.
---

# Testing Strategy

Map spec acceptance criteria to automated tests across the test pyramid, with coverage gates and CI integration.

## Scope

**Use for:**
- Defining test architecture for new features
- Mapping acceptance criteria to test types (unit, integration, e2e)
- Configuring Vitest, Playwright, and React Testing Library
- Creating Supabase and Stripe mock/fixture patterns
- Setting coverage thresholds and CI gates

**Not for:**
- Manual QA procedures
- Performance/load testing (see `performance-patterns` skill)
- Security testing (see `security-patterns` skill)

---

# Test Pyramid

```
         /  E2E  \          Playwright — critical user flows only
        /----------\
       / Integration \      Vitest + RTL — component interactions, API routes
      /----------------\
     /      Unit        \   Vitest — pure functions, utils, schemas
    /--------------------\
```

**Target ratios:** ~60% unit, ~30% integration, ~10% e2e.

---

# Vitest Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
    include: ['**/*.test.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'json-summary'],
      thresholds: { statements: 80, branches: 75, functions: 80, lines: 80 },
      exclude: ['node_modules/', 'tests/', '**/*.d.ts', '**/*.config.{ts,js}', '**/types/'],
    },
  },
  resolve: { alias: { '@': path.resolve(__dirname, './') } },
})
```

```typescript
// tests/setup.ts
import '@testing-library/jest-dom/vitest'
import { cleanup } from '@testing-library/react'
import { afterEach, vi } from 'vitest'

afterEach(() => { cleanup() })

vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), replace: vi.fn(), back: vi.fn() }),
  usePathname: () => '/',
  useSearchParams: () => new URLSearchParams(),
  redirect: vi.fn(),
}))
```

---

# Mocking Patterns

## Supabase Client Mock

```typescript
// tests/mocks/supabase.ts
import { vi } from 'vitest'

export const mockSupabaseClient = {
  auth: {
    getSession: vi.fn().mockResolvedValue({
      data: { session: { user: { id: 'user-123', email: 'test@test.com' } } },
      error: null,
    }),
    onAuthStateChange: vi.fn(() => ({
      data: { subscription: { unsubscribe: vi.fn() } },
    })),
  },
  from: vi.fn(() => ({
    select: vi.fn().mockReturnThis(),
    insert: vi.fn().mockReturnThis(),
    update: vi.fn().mockReturnThis(),
    delete: vi.fn().mockReturnThis(),
    eq: vi.fn().mockReturnThis(),
    single: vi.fn().mockResolvedValue({ data: null, error: null }),
    order: vi.fn().mockReturnThis(),
    limit: vi.fn().mockReturnThis(),
    range: vi.fn().mockResolvedValue({ data: [], error: null, count: 0 }),
  })),
}

vi.mock('@supabase/auth-helpers-nextjs', () => ({
  createClientComponentClient: () => mockSupabaseClient,
  createServerComponentClient: () => mockSupabaseClient,
  createRouteHandlerClient: () => mockSupabaseClient,
}))
```

## Stripe Mock

```typescript
// tests/mocks/stripe.ts
import { vi } from 'vitest'

export const mockStripe = {
  checkout: { sessions: { create: vi.fn().mockResolvedValue({ id: 'cs_test_123', url: 'https://checkout.stripe.com/test' }) } },
  webhooks: { constructEvent: vi.fn().mockReturnValue({ type: 'checkout.session.completed', data: { object: {} } }) },
  customers: { create: vi.fn().mockResolvedValue({ id: 'cus_test_123' }) },
}

vi.mock('@/lib/stripe', () => ({ stripe: mockStripe }))
```

---

# Test Examples

## Component Test (React Testing Library)

```typescript
import { render, screen, waitFor } from '@testing-library/react'
import { UserProfile } from '../UserProfile'
import { mockSupabaseClient } from '@/tests/mocks/supabase'

it('renders user email when authenticated', async () => {
  mockSupabaseClient.auth.getSession.mockResolvedValueOnce({
    data: { session: { user: { id: '1', email: 'user@test.com' } } }, error: null,
  })
  render(<UserProfile />)
  await waitFor(() => { expect(screen.getByText('user@test.com')).toBeInTheDocument() })
})
```

## Route Handler Test

```typescript
import { GET } from '../route'
import { mockSupabaseClient } from '@/tests/mocks/supabase'

it('returns 401 when unauthenticated', async () => {
  mockSupabaseClient.auth.getSession.mockResolvedValueOnce({ data: { session: null }, error: null })
  const response = await GET(new Request('http://localhost/api/items'))
  expect(response.status).toBe(401)
})
```

---

# Playwright E2E

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? 'github' : 'html',
  use: { baseURL: 'http://localhost:3000', trace: 'on-first-retry' },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
  webServer: { command: 'npm run dev', port: 3000, reuseExistingServer: !process.env.CI },
})
```

## Auth Fixture

```typescript
// e2e/fixtures/auth.ts
import { test as base, Page } from '@playwright/test'

export const test = base.extend<{ authedPage: Page }>({
  authedPage: async ({ page }, use) => {
    await page.goto('/login')
    await page.fill('[name="email"]', process.env.E2E_TEST_EMAIL!)
    await page.click('[data-testid="google-login"]')
    await page.waitForURL('/dashboard')
    await use(page)
  },
})
```

---

# Acceptance Criteria Format

Format ACs as testable statements in specs:

```markdown
- **AC-1**: Given authenticated user, when visiting /dashboard, then items are listed
  - Test type: Integration (RTL) + E2E (Playwright)
- **AC-2**: Given unauthenticated user, when visiting /dashboard, then redirected to /login
  - Test type: Integration (RTL)
- **AC-3**: Given invalid input, when POST /api/items, then 400 with validation errors
  - Test type: Unit (Vitest)
```

---

# CI Scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:ci": "vitest run --coverage && playwright test"
  }
}
```

---

# Artifact: test-plan.md

Produce `openspec/changes/<change-name>/test-plan.md` using this template:

```markdown
# Test Plan: <Change Name>

## Coverage Summary
| Type | Count | Framework |
|------|-------|-----------|
| Unit | N | Vitest |
| Integration | N | Vitest + RTL |
| E2E | N | Playwright |

## AC -> Test Mapping
| AC | Description | Test Type | File |
|----|-------------|-----------|------|
| AC-1 | ... | Unit | `__tests__/foo.test.ts` |

## Mocks Required
- [ ] Supabase client (auth + data)
- [ ] Stripe (checkout sessions)

## Coverage Gates
- Statements: 80% | Branches: 75% | Functions: 80% | Lines: 80%
```

---

# SDD Phase Integration

## Specs Phase
- Format acceptance criteria as testable AC-N statements in `openspec/changes/<change-name>/specs/<capability>/spec.md`
- Tag each AC with test type (unit, integration, e2e)

## Design Phase
- Include "Test Architecture" section in `openspec/changes/<change-name>/design.md`
- Document which mocks are needed and any test infrastructure changes

## Tasks Phase
- Add test implementation tasks in `openspec/changes/<change-name>/tasks.md`
- Each feature task should have a corresponding test task
- Include `test-plan.md` artifact creation as a task

## Apply Phase
- Run `vitest run --coverage` and verify thresholds pass
- Run `playwright test` for any new e2e tests
- Verify all ACs have corresponding passing tests

---
