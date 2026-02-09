---
name: ux-states
description: Ensure complete UI state coverage (loading, empty, error, populated) and state-to-component mapping for interactive features
---

# UX States

Ensure every interactive element and screen has complete state coverage and meets accessibility standards.

---

## State Enumeration Framework

Every interactive element and screen MUST define these states:

### Core States
- **Loading** — Data is being fetched. Show skeleton, spinner, or placeholder.
- **Empty** — No data exists. Show empty state message with action.
- **Populated** — Data exists and is displayed normally.
- **Error** — Operation failed. Show error message with retry option.

### Edge States
- **Disabled** — Element is visible but not interactive.
- **Offline** — Network unavailable. Show offline indicator.
- **Permission Denied** — User lacks access. Show permission message.
- **Rate Limited** — Too many requests. Show throttle message with time.

### Interactive States
- **Default** — Element's resting state.
- **Hover** — Pointer over element (desktop only).
- **Active** — Element is being pressed/clicked.
- **Focus** — Element has keyboard focus.

### Missing states = broken experience. Define all before implementation.

---

## Accessibility (State-Specific)

For the full accessibility checklist (ARIA roles, keyboard navigation, contrast, forms), see the **web-design-guidelines** skill. This section covers only state-specific accessibility concerns.

### State Change Announcements
- Use `aria-live="polite"` regions to announce state transitions (loading to populated, error appearance)
- Loading states: announce "Loading..." to screen readers via `aria-busy="true"` on the container
- Error states: use `role="alert"` to immediately announce errors
- Empty states: ensure the empty message is in the document flow (not only visual)

### Focus Management During State Transitions
- When transitioning from loading to error, move focus to the error message or retry button
- When transitioning from loading to populated, do NOT steal focus (let user continue from current position)
- When a retry action triggers loading, return focus to retry button after resolution

### Disabled State Accessibility
- Use `aria-disabled="true"` instead of removing elements from tab order
- Provide tooltip or `aria-describedby` explaining why the element is disabled
- Maintain visual indication of disabled state with sufficient contrast (3:1 minimum)

---

## Screen State Documentation Format

Use this template from ui-designer to document flows and screens:

```markdown
## Flow: {Flow Name}

### Entry Point
{How the user reaches this flow}

### Screens
#### {Screen Name}
- **States**: loading | empty | populated | error
- **Key elements**: {interactive elements and their behavior}
- **Accessibility**: {ARIA roles, keyboard nav, focus management}

### Transitions
{Screen A} → {action} → {Screen B}

### Error Handling
{What happens on failure at each step}
```

### Example

```markdown
## Flow: User List

### Entry Point
Navigation: sidebar → "Users"

### Screens
#### User List Screen
- **States**: loading (skeleton rows) | empty ("No users found") | populated (table) | error (retry banner)
- **Key elements**: Search input (debounced 300ms), sortable column headers, pagination controls
- **Accessibility**: Table uses `role="grid"`, search has `aria-label="Search users"`, sort announces via `aria-live`

### Transitions
User List → click row → User Detail
User List → click "Add User" → Create User Form

### Error Handling
- Network error: Show retry banner at top
- Permission error: Show "Access Denied" message
- Rate limit: Show throttle message with countdown
```

---

## Component State Patterns

Map screen states to component props (from frontend-designer):

### State Prop Pattern

```markdown
## Component: {ComponentName}

### States
- **Loading**: {what renders}
- **Empty**: {what renders}
- **Populated**: {what renders}
- **Error**: {what renders}

### Props
| Prop | Type | Required | Description |
|------|------|----------|-------------|
| isLoading | boolean | no | Shows loading state |
| error | Error | null | no | Shows error state |
| data | T[] | no | Data to render |
| isEmpty | boolean | no | Explicitly shows empty state |
```

### Example

```markdown
## Component: UserListPage

### States
- **Loading**: PageLayout + DataTable skeleton
- **Empty**: PageLayout + EmptyState("No users found")
- **Populated**: PageLayout + DataTable with rows
- **Error**: PageLayout + ErrorBanner with retry

### Props
| Prop | Type | Required | Description |
|------|------|----------|-------------|
| — | — | — | Page-level component, no external props |

### Children
- SearchInput — debounced search filter
- DataTable — sortable, paginated user rows
- Pagination — cursor-based page controls

### Data Requirements
- `GET /users?search={q}&sort={field}&cursor={c}` via useUsers hook
```

---

## State Implementation Guidelines

### State Priority
1. **Error** takes priority over all other states
2. **Loading** takes priority over empty/populated
3. **Empty** only shows when no loading and no error
4. **Populated** is the default happy path

### State Transitions
- All state transitions should be smooth (no flash of content)
- Loading → Populated should feel instant when data is cached
- Error → Retry should clear error before showing loading
- Debounce rapid state changes (e.g., search input)

### State Persistence
- Error messages should persist until user action
- Loading states should have minimum duration (avoid flash)
- Empty states should suggest next action
- Populated states should indicate refresh/update availability

---

## SDD Phase Integration

UX state coverage must be addressed at every phase of the OpenSpec pipeline:

### Specs Phase
- Every user-facing scenario in `openspec/changes/<change-name>/specs/<capability>/spec.md` MUST enumerate all states: **loading**, **empty**, **populated**, **error**
- Acceptance criteria must explicitly cover edge states (offline, permission denied, rate limited) where applicable
- Missing state definitions in a spec = incomplete spec; do not proceed to design

### Design Phase
- Map each state to component props and document in `openspec/changes/<change-name>/design.md`
- Include a "UI States" section in the design showing state-to-component mapping
- Document state transition behavior (animation, focus management, minimum loading duration)

### Tasks Phase
- Include state coverage verification tasks in `openspec/changes/<change-name>/tasks.md`
- Each UI task must include: "Verify all states render correctly (loading, empty, populated, error)"
- Add specific tasks for edge state handling if the feature involves network requests or permissions

---

## Stack-Specific Patterns (React / Next.js App Router)

### Loading States with Suspense

Use React Suspense boundaries and Next.js `loading.tsx` for loading states:

```tsx
// app/users/loading.tsx — Automatic loading UI for the route segment
export default function Loading() {
  return <UserListSkeleton />
}

// app/users/page.tsx — Server Component with async data
export default async function UsersPage() {
  const users = await getUsers()
  // This renders only after data loads; loading.tsx shows during fetch
  return <UserList users={users} />
}
```

For component-level loading within a page:

```tsx
// Wrap async components in Suspense for granular loading states
import { Suspense } from 'react'

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />
      </Suspense>
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />
      </Suspense>
    </div>
  )
}
```

### Error States with error.tsx

Use Next.js error boundaries for route-level error handling:

```tsx
// app/users/error.tsx — Automatic error UI for the route segment
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

### Empty States

Handle empty states explicitly in Server Components:

```tsx
export default async function UsersPage() {
  const users = await getUsers()

  if (users.length === 0) {
    return (
      <EmptyState
        title="No users found"
        description="Get started by inviting your first team member."
        action={{ label: 'Invite User', href: '/users/invite' }}
      />
    )
  }

  return <UserList users={users} />
}
```

### Client Component State Management

For interactive components requiring client-side state transitions:

```tsx
'use client'

import { useTransition } from 'react'

export function UserActions({ userId }: { userId: string }) {
  const [isPending, startTransition] = useTransition()

  const handleDelete = () => {
    startTransition(async () => {
      await deleteUser(userId)
    })
  }

  return (
    <button
      onClick={handleDelete}
      disabled={isPending}
      aria-busy={isPending}
    >
      {isPending ? 'Deleting...' : 'Delete User'}
    </button>
  )
}
```

---

## Common Patterns

### Data Table States
- **Loading**: Skeleton rows matching expected layout
- **Empty**: "No items found" with "Add Item" CTA
- **Populated**: Table with data, sort, filter, pagination
- **Error**: Error banner above table, retain previous data if available

### Form States
- **Default**: Empty or pre-filled inputs
- **Validating**: Show inline validation on blur
- **Submitting**: Disable form, show loading on submit button
- **Success**: Show success message, clear form or redirect
- **Error**: Show error message, keep form enabled for retry

### Modal/Dialog States
- **Opening**: Animate in, trap focus
- **Open**: Focus first focusable element
- **Closing**: Animate out, return focus to trigger element
- **Closed**: Remove from DOM or hide with display:none

---

## Quick Reference

When defining any UI element, ask:

1. **What are ALL possible states?** (Not just happy path)
2. **How does keyboard navigation work?** (Tab order, shortcuts)
3. **What do screen readers announce?** (Labels, state changes)
4. **Does contrast meet standards?** (4.5:1 minimum)
5. **Where does focus go?** (After actions, on errors)

Missing any of these = incomplete spec. Address before implementation.
