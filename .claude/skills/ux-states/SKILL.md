---
name: ux-states
description: State enumeration and accessibility framework for interactive elements
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

## Accessibility Checklist

All UI work MUST meet these accessibility requirements:

### ARIA Roles
- Use semantic HTML first (`<button>`, `<nav>`, `<main>`, etc.)
- Add ARIA roles only when semantic HTML is insufficient
- Common roles: `role="grid"`, `role="dialog"`, `role="alert"`, `role="status"`
- Always validate role usage against ARIA spec

### Keyboard Navigation
- All interactive elements must be keyboard accessible
- Logical tab order (verify with Tab key)
- Escape closes modals/overlays
- Enter/Space activates buttons and controls
- Arrow keys navigate lists, grids, tabs

### Focus Management
- Visible focus indicators (not `outline: none` without replacement)
- Focus moves to appropriate element after actions (e.g., modal open/close)
- Focus trap in modals and overlays
- Skip links for main content

### Contrast Requirements
- Text contrast minimum 4.5:1 for normal text
- 3:1 for large text (18pt+ or 14pt+ bold)
- 3:1 for UI components and graphical objects
- Test with browser dev tools or contrast checkers

### Screen Reader Considerations
- Use `aria-label` for icons without text
- Use `aria-live` for dynamic content updates
- Use `aria-describedby` for additional context
- Announce state changes (e.g., sort direction, filter applied)
- Test with VoiceOver (macOS) or NVDA (Windows)

### Form Accessibility
- Associate labels with inputs (`<label for="id">`)
- Use `aria-required` for required fields
- Provide clear error messages linked with `aria-describedby`
- Group related inputs with `<fieldset>` and `<legend>`

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
