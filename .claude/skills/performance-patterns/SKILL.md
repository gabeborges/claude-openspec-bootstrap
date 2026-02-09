---
name: performance-patterns
description: Performance budgets and optimization patterns for Next.js + Supabase — Core Web Vitals targets, bundle analysis, caching strategies, and database query performance.
---

# Performance Patterns

Performance budgets, optimization patterns, and monitoring for Next.js App Router + Supabase applications.

## Scope

**Use for:**
- Setting Core Web Vitals targets for new pages
- Bundle size analysis and optimization
- Caching strategies (ISR, SWR, CDN)
- Supabase query optimization and connection pooling
- Image and font optimization
- Adding performance criteria to specs and design docs

**Not for:**
- Load testing or stress testing execution
- CDN/infrastructure configuration
- Third-party service performance (Stripe checkout speed, etc.)

---

# Core Web Vitals Targets

| Metric | Good     | Needs Work | Poor    |
|--------|----------|------------|---------|
| LCP    | < 2.5s   | 2.5-4.0s   | > 4.0s  |
| INP    | < 200ms  | 200-500ms  | > 500ms |
| CLS    | < 0.1    | 0.1-0.25   | > 0.25  |

**Project targets:** All pages MUST hit "Good" thresholds on mobile 4G.

---

# Next.js Bundle Optimization

## Bundle Analysis

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

/** @type {import('next').NextConfig} */
const nextConfig = {
  // ... other config
}

module.exports = withBundleAnalyzer(nextConfig)
```

Run: `ANALYZE=true npm run build`

## Bundle Budgets

| Bundle          | Budget  | Action if exceeded      |
|-----------------|---------|-------------------------|
| First Load JS   | < 100kB | Split or lazy-load      |
| Per-route JS    | < 50kB  | Code-split components   |
| Total page size | < 300kB | Audit dependencies      |

## Dynamic Imports

Lazy-load heavy components that are below the fold or conditionally rendered:

```typescript
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(() => import('@/components/Chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // only if truly client-only
})
```

## Tree Shaking

Import only what you need:

```typescript
// GOOD: Named import (tree-shakeable)
import { format } from 'date-fns'

// BAD: Full library import
import _ from 'lodash'

// GOOD: Cherry-pick lodash
import debounce from 'lodash/debounce'
```

---

# Image Optimization

Always use `next/image` for images:

```tsx
import Image from 'next/image'

<Image
  src="/hero.webp"
  alt="Hero image"
  width={1200}
  height={630}
  priority          // only for above-the-fold LCP images
  sizes="(max-width: 768px) 100vw, 50vw"
  placeholder="blur" // use for large images
  blurDataURL="..."  // base64 placeholder
/>
```

**Rules:**
- Set `priority` only on the LCP image (usually hero/banner)
- Always provide `sizes` for responsive images
- Use WebP/AVIF formats (Next.js handles conversion)
- Set explicit `width` and `height` to prevent CLS

---

# Caching Strategies

## Static Generation with ISR

```typescript
// app/blog/[slug]/page.tsx
export const revalidate = 3600 // revalidate every hour

export async function generateStaticParams() {
  const posts = await getAllPostSlugs()
  return posts.map(slug => ({ slug }))
}
```

## Client-Side Caching with SWR

```typescript
'use client'
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

export function useItems() {
  const { data, error, isLoading, mutate } = useSWR('/api/items', fetcher, {
    revalidateOnFocus: false,
    dedupingInterval: 10000, // 10s dedup
  })

  return { items: data?.data ?? [], error, isLoading, mutate }
}
```

## Route Handler Caching

```typescript
// app/api/public-data/route.ts
export async function GET() {
  const data = await fetchData()

  return Response.json({ data }, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
    },
  })
}
```

## Cache Strategy Matrix

| Data Type        | Strategy              | TTL     |
|------------------|-----------------------|---------|
| Static content   | ISR                   | 1 hour  |
| User-specific    | No cache / SWR client | -       |
| Public lists     | CDN + s-maxage        | 60s     |
| Search results   | SWR client            | 10s     |
| Auth state       | Never cache           | -       |

---

# Supabase Query Optimization

## Select Only What You Need

```typescript
// BAD: Select all columns
const { data } = await supabase.from('items').select('*')

// GOOD: Select specific columns
const { data } = await supabase.from('items').select('id, name, created_at')
```

## Use Indexes

Ensure columns used in `.eq()`, `.order()`, and cursor pagination have database indexes:

```sql
CREATE INDEX idx_items_user_id ON items(user_id);
CREATE INDEX idx_items_created_at ON items(created_at DESC);
CREATE INDEX idx_items_user_created ON items(user_id, created_at DESC);
```

## Connection Pooling

Use Supabase connection pooling for serverless environments:

```bash
# .env.local
# Direct connection (migrations only)
DATABASE_URL=postgresql://postgres:[password]@db.[ref].supabase.co:5432/postgres

# Pooled connection (application queries)
DATABASE_URL_POOLED=postgresql://postgres.[ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres?pgbouncer=true
```

In application code, use the pooled connection string. Direct connections are only for migrations.

## Avoid N+1 Queries

```typescript
// BAD: N+1 query pattern
const { data: items } = await supabase.from('items').select('*')
for (const item of items) {
  const { data: author } = await supabase.from('users').select('name').eq('id', item.user_id)
}

// GOOD: Join in single query
const { data } = await supabase
  .from('items')
  .select('*, user:users(name)')
```

---

# Font Optimization

```typescript
// app/layout.tsx
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',      // prevent FOIT, reduce CLS
  variable: '--font-inter',
})
```

Use `display: 'swap'` to prevent invisible text during font load.

---

# Performance Acceptance Criteria Format

When adding performance criteria to specs:

```markdown
## Performance Criteria
- **PC-1**: Page LCP < 2.5s on mobile 4G
- **PC-2**: First Load JS < 100kB
- **PC-3**: CLS < 0.1 (no layout shifts after load)
- **PC-4**: API response time < 200ms (p95) for list endpoints
- **PC-5**: Database query time < 50ms (p95) for indexed queries
```

---

# SDD Phase Integration

## Specs Phase
- Add performance acceptance criteria (PC-N) to `openspec/changes/<change-name>/specs/<capability>/spec.md`
- Tag pages that are likely LCP-critical

## Design Phase
- Include "Performance" section in `openspec/changes/<change-name>/design.md`:
  - Caching strategy for each data type
  - Bundle impact estimate for new dependencies
  - Database indexes required
  - Image optimization approach

## Tasks Phase
- Add performance verification tasks in `openspec/changes/<change-name>/tasks.md`
- Include: bundle analysis, Lighthouse audit, query plan review

## Apply Phase
- Run `ANALYZE=true npm run build` to verify bundle budgets
- Run Lighthouse on affected pages (target all "Good" CWV)
- Verify database indexes are created for new queries

---
