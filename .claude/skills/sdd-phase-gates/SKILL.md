---
name: sdd-phase-gates
description: Phase transition checklists for OpenSpec SDD workflow — validates artifact completeness, cross-cutting concern coverage, and quality gates before advancing phases.
---

# SDD Phase Gates

Validation checklists for transitioning between OpenSpec phases. Ensures artifact completeness and cross-cutting concern coverage before advancing.

## Scope

**Use for:**
- Validating readiness to advance from one SDD phase to the next
- Checking cross-cutting concerns (security, testing, performance, API contracts, deployment)
- Ensuring artifact completeness before phase transition
- Identifying missing coverage across domain skills

**Not for:**
- Defining how to write artifacts (see individual skill docs)
- Replacing domain skill guidance (this is the meta-validation layer)

---

# Phase Transition Gates

## Gate 1: Proposal -> Specs

**Artifact check:** `openspec/changes/<change-name>/proposal.md` exists and contains:
- [ ] Problem statement clearly defined
- [ ] Proposed solution summarized
- [ ] Scope boundaries (in/out) documented
- [ ] Affected areas identified

**Cross-cutting check:**
- [ ] Security implications noted if touching auth, data access, or external APIs (`security-patterns`)
- [ ] Performance impact flagged if adding new pages, heavy components, or large data queries (`performance-patterns`)
- [ ] API surface changes identified if adding/modifying endpoints (`api-contracts`)

**Advance when:** Proposal is approved and scope is clear.

---

## Gate 2: Specs -> Design

**Artifact check:** `openspec/changes/<change-name>/specs/<capability>/spec.md` exists for each capability and contains:
- [ ] Acceptance criteria in testable AC-N format (`testing-strategy`)
- [ ] Security acceptance criteria for auth/authz/validation (`security-patterns`)
- [ ] Performance acceptance criteria (PC-N) for user-facing pages (`performance-patterns`)
- [ ] Endpoint inventory with method, path, auth requirement (`api-contracts`)
- [ ] Data model requirements (tables, columns, relationships)

**Cross-cutting matrix:**

| Concern       | Skill                | Required in spec when...                      |
|---------------|----------------------|-----------------------------------------------|
| Auth/Authz    | `security-patterns`  | Feature accesses user data or protected routes |
| Input validation | `security-patterns` | Feature accepts user input                  |
| Test mapping  | `testing-strategy`   | Always — every AC needs a test type tag        |
| API shape     | `api-contracts`      | Feature exposes or modifies API endpoints      |
| Performance   | `performance-patterns` | Feature adds pages or heavy data queries     |
| Data model    | `db-migration`       | Feature requires schema changes                |

**Advance when:** All capabilities have complete specs with tagged acceptance criteria.

---

## Gate 3: Design -> Tasks

**Artifact check:** `openspec/changes/<change-name>/design.md` exists and contains:
- [ ] Architecture overview (component tree, data flow)
- [ ] Security Design section with STRIDE analysis (`security-patterns`)
- [ ] Performance section: caching strategy, bundle impact, required indexes (`performance-patterns`)
- [ ] API contract details or reference to `api-contracts.md` artifact (`api-contracts`)
- [ ] Test architecture: mocks needed, test infrastructure changes (`testing-strategy`)
- [ ] Deployment strategy: feature flags, migration order (`deployment-lifecycle`)
- [ ] Database migration plan if schema changes needed (`db-migration`)

**Optional artifacts in change directory:**
- [ ] `api-contracts.md` if feature has 3+ endpoints (`api-contracts`)
- [ ] `test-plan.md` if feature has complex test requirements (`testing-strategy`)

**Advance when:** Design covers all cross-cutting concerns relevant to the feature.

---

## Gate 4: Tasks -> Apply

**Artifact check:** `openspec/changes/<change-name>/tasks.md` exists and contains:
- [ ] Implementation tasks for all capabilities in specs
- [ ] Test tasks mapped to acceptance criteria (`testing-strategy`)
- [ ] Security verification tasks: RLS policies, input validation (`security-patterns`)
- [ ] Migration tasks if schema changes needed (`db-migration`)
- [ ] Deployment tasks: env vars, feature flags, smoke tests (`deployment-lifecycle`)
- [ ] Tasks ordered with dependencies (migration before app code, etc.)

**Task completeness check:**
- [ ] Every AC-N in specs has a corresponding implementation task
- [ ] Every AC-N has a corresponding test task
- [ ] No orphan tasks (every task traces to a spec requirement)

**Advance when:** Task list is complete, ordered, and all dependencies are documented.

---

## Gate 5: Apply -> Archive

**Execution check:** All tasks in `tasks.md` marked complete, and:
- [ ] All unit/integration tests pass: `vitest run --coverage` (`testing-strategy`)
- [ ] Coverage thresholds met (statements 80%, branches 75%, functions 80%, lines 80%)
- [ ] All e2e tests pass: `playwright test` (`testing-strategy`)
- [ ] Contract tests pass for all API endpoints (`api-contracts`)
- [ ] Security checklist completed (`security-patterns`)
- [ ] RLS policies tested with authorized and unauthorized access
- [ ] No secrets in code, logs, or diffs
- [ ] Bundle budgets verified: `ANALYZE=true npm run build` (`performance-patterns`)
- [ ] Lighthouse audit passes CWV "Good" thresholds
- [ ] Database migrations applied successfully (`db-migration`)
- [ ] Feature flag disabled in production before merge (`deployment-lifecycle`)
- [ ] Smoke tests pass after deploy (`deployment-lifecycle`)
- [ ] Error rate monitored for 15 minutes post-deploy

**Advance when:** All checks pass. Any failure blocks archival.

---

# Quick Reference: Skill-to-Phase Matrix

| Skill                  | Proposal | Specs | Design | Tasks | Apply | Archive |
|------------------------|----------|-------|--------|-------|-------|---------|
| `security-patterns`    |    *     |  **   |  **    |  **   |  **   |         |
| `testing-strategy`     |          |  **   |  **    |  **   |  **   |         |
| `api-contracts`        |    *     |  **   |  **    |  **   |  **   |         |
| `performance-patterns` |    *     |  **   |  **    |  *    |  **   |         |
| `deployment-lifecycle` |          |       |  **    |  **   |  **   |  **     |
| `db-migration`         |    *     |  *    |  **    |  **   |  **   |         |
| `interface-design`     |          |  *    |  **    |  *    |       |         |
| `ux-states`            |          |  *    |  **    |  *    |       |         |
| `compliance-patterns`  |    *     |  *    |  **    |  *    |  **   |         |

`**` = required at this phase | `*` = check if relevant

---

# Escalation

If a gate check fails:

1. **Missing artifact**: Create the artifact before advancing. Do not skip.
2. **Missing cross-cutting coverage**: Add the missing section to the relevant artifact. Reference the appropriate skill.
3. **Conflicting requirements**: Document the conflict in `design.md` and flag for human review.
4. **Blocked by external dependency**: Add a blocking note in `tasks.md` and do not advance past Tasks phase.

---
