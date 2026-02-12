# Project Rules (Auto-loaded Every Session)

## Stack
Next.js 15 (App Router), TypeScript, React 19, Tailwind CSS, shadcn/ui, Headless UI, Supabase, Stripe, Google OAuth, Zod
Testing: Vitest + Playwright | Package manager: npm | Workflow: SDD (OpenSpec + Clavix)

## Security (Non-negotiable)

- NEVER publish passwords, API keys, tokens, or credentials in code, docs, commits, PRs, diffs, or logs
- NEVER commit `.env`, `.env.*`, credential files, or config files containing secrets — verify `.gitignore` covers them
- NEVER hardcode credentials — use environment variables or secret stores
- NEVER output secrets even if they exist locally — warn: "Proposal contains sensitive data — remove before proceeding."
- NEVER log PHI/PII payloads — log only request IDs, timestamps, non-sensitive error codes
- NEVER weaken auth checks, permission gates, or move privileged logic to the client
- Supabase RLS is non-negotiable — never bypass it
- Before ANY commit, verify no secrets in staged changes
- If security conflicts with spec: create `spec-change-requests.yaml` entry and stop
- For detailed implementation guidance (auth patterns, STRIDE, OWASP, RLS, Stripe security), see the `security-patterns` skill

## Autonomy & Execution

### Always Proceed Without Asking
- Create, edit, delete files within the project directory
- Install dependencies (`npm install`, `npm add`) required by tasks
- Run build, test, lint, and type-check commands (any command in Testing Commands section)
- Create and switch git branches (`git checkout -b`, `git switch`)
- Run database migrations in development (via `supabase` CLI)
- Create directories and scaffolding
- Stage changes and commit locally (`git add`, `git commit`)
- Run any safe read-only command (`git status`, `git log`, `git diff`, `ls`, `cat`, etc.)

### Always Ask Before
- Pushing to remote repositories (`git push`, especially `--force`)
- Modifying `.env` files or environment variables
- Deleting git branches (`git branch -D` / `-d`)
- Running destructive database operations in production
- Changing CI/CD configuration (GitHub Actions, deployment scripts)
- Any action affecting shared/external systems (webhooks, third-party APIs, production environments)
- Force operations (`--force`, `--hard`, `-f` on destructive commands like `git reset --hard`, `git clean -f`, `rm -rf`)

### When Blocked on a Decision
- Flag the blocker clearly with a brief explanation
- Continue working on all independent/unblocked tasks in parallel
- Return to the blocked item when user provides input
- Never let a single blocker halt all progress
- Use the Task tool to parallelize independent work

### Operating Principles
- Assume permission to act within the stated scope
- If scope is unclear, infer the smallest reasonable scope and proceed
- Do not pause for clarification unless ambiguity affects correctness or security
- Prefer action over asking — fix forward, don't wait
- Use parallel task execution (`Task` tool) for independent work items
- When implementing from `tasks.md`, execute tasks in dependency order but parallelize independent tasks

## Next.js (App Router)
- App Router (`app/`) is authoritative
- Prefer Server Components by default; use `"use client"` only when required
- Server Actions: only for mutations
- Keep components modular and colocated with routes or features

## Styling / UI
- Tailwind for utility styles, shadcn/ui for components, Headless UI for accessible primitives
- No new UI libs without explicit approval or task reference
- If `openspec/ui-design-system.md` exists, UI work MUST comply with it
- If UI work needed and system file missing, run `/interface-design:init` first
- For simple one-off pages/components, prefer the `frontend-design` skill

## Architecture Boundaries

### Feature Isolation
- No cross-feature imports — shared code goes in `lib/`, `components/`, `utils/`

### Data Access (Supabase)
- UI must not call Supabase REST API directly via raw `fetch()` — use official Supabase client helpers (`createClientComponentClient`, `createServerComponentClient`, `createRouteHandlerClient`) or server-side boundaries
- For detailed auth patterns and security guidance, see the `security-patterns` skill

### Stripe
- Server/client config: `/lib/stripe.js`
- Checkout: `/app/api/checkout/route.js`
- Frontend: `@stripe/react-stripe-js`
- Always verify webhook signatures
- Do not alter billing semantics, price IDs, or webhook handling without spec/tasks

### Google OAuth
- Do not relax OAuth scopes, callback URLs, or token handling without spec reference

## Coding Standards
- React components: **PascalCase**
- Files/folders: **kebab-case** (unless Next.js routing requires otherwise)
- APIs/routes: REST + Next.js App Router conventions
- Zod for all input validation (server-side required, client-side optional)
- ESLint (`next/core-web-vitals`) + Prettier required
- Minimal diff: smallest change that satisfies the ticket
- No unrelated refactors, renames, or formatting-only churn
- If a shared module is touched, add/adjust tests proportional to risk

## Testing Commands
```
npm run lint
npm run test
npm run test:e2e
npm run build
```

## Token/Context Rules
- Default to feature-scoped reads only
- `openspec/product-vision-strategy.md` — Full vision document (~4,400 tokens). Do NOT load unless doing cross-domain analysis or quarterly review.
  - Domain splits (preferred for agents — generated by `/vision:distill`):
    - `openspec/quick-product-vision-strategy.md` — Product vision, pillars, non-goals, AI strategy (§1–§7, §12)
    - `openspec/security-compliance-baseline.md` — Security posture, compliance, AI boundaries, tech non-goals (§10 partial, §11, §12, §15, §16)
    - `openspec/tech-architecture-baseline.md` — Architecture, data, integration, scalability, tech non-goals (§8–§10, §12–§16)
  - Agents should load only their relevant domain split, not the full document
  - To regenerate distilled files: run `/vision:distill`
- Do NOT read `openspec/ui-design-system.md` unless doing UI work
- `design.md` is the architecture reference — load only for architecture decisions
- `openspec/config.yaml` — OpenSpec configuration. Read during artifact creation, not general coding.
- `openspec/changes/` — Active changes. Read only the change you're working on.
- `openspec/specs/` — Canonical specs. Read only specs relevant to the current feature.

## OpenSpec (Spec-Driven Development)

This project uses OpenSpec for structured, spec-driven changes. All feature work follows the artifact pipeline:

```
proposal → specs → design → tasks → apply → archive
```

### Directory Structure
```
openspec/
├── config.yaml          # Schema, context, and per-artifact rules
├── specs/               # Canonical specs (main branch of truth)
└── changes/
    ├── <change-name>/   # Active change work container
    │   ├── proposal.md
    │   ├── specs/<capability>/spec.md
    │   ├── design.md
    │   └── tasks.md
    └── archive/         # Completed changes with decision history
```

### Key Commands

| Command | Purpose |
|---------|---------|
| `/opsx:new` | Start a new change (step-by-step artifacts) |
| `/opsx:ff` | Fast-forward — create all artifacts at once |
| `/opsx:continue` | Create the next artifact for a change |
| `/opsx:apply` | Implement tasks from a change |
| `/opsx:verify` | Verify implementation matches artifacts |
| `/opsx:sync` | Sync delta specs to canonical specs |
| `/opsx:archive` | Archive a completed change |
| `/opsx:explore` | Think through problems without implementing |

### Unified Workflow (Clavix + OpenSpec)

```
/clavix:product (vision) → /clavix:prd (requirements) → /clavix:plan (tasks)
                                      ↓
/opsx:new (change) → /opsx:ff or /opsx:continue (artifacts) → /opsx:apply (implement) → /opsx:verify → /opsx:archive
```

- **Clavix** handles ideation, prompts, PRDs, and high-level planning
- **OpenSpec** handles spec-driven changes with traceable artifacts

### Domain Skill Activation Guide

Activate these skills based on the work context:

| Context | Skill | When |
|---------|-------|------|
| Auth, data access, external APIs | `security-patterns` | Any security-adjacent code |
| Schema changes | `db-migration` | Adding/modifying database tables |
| UI components, interactive features | `ux-states` | Ensuring loading/empty/error/populated states |
| Dashboard/app interfaces | `interface-design` | Complex UI layouts |
| Simple one-off pages | `frontend-design` | Quick UI work |
| API endpoints | `api-contracts` | Adding/modifying route handlers |
| Writing tests | `testing-strategy` | Test architecture decisions |
| Performance work | `performance-patterns` | Bundle, CWV, caching, query optimization |
| Deployments | `deployment-lifecycle` | Feature flags, env vars, rollback |
| Regulated data (PHI/PII) | `compliance-patterns` | HIPAA, audit trails |

### Skill-to-Phase Quick Reference

| Skill | Proposal | Specs | Design | Tasks | Apply |
|-------|----------|-------|--------|-------|-------|
| `security-patterns` | * | ** | ** | ** | ** |
| `testing-strategy` | | ** | ** | ** | ** |
| `api-contracts` | * | ** | ** | ** | ** |
| `performance-patterns` | * | ** | ** | * | ** |
| `deployment-lifecycle` | | | ** | ** | ** |
| `db-migration` | * | * | ** | ** | ** |
| `interface-design` | | * | ** | * | |
| `ux-states` | | * | ** | * | |
| `compliance-patterns` | * | * | ** | * | ** |

`**` = required | `*` = check if relevant

<!-- CLAVIX:START -->
## Clavix Integration

This project uses Clavix for prompt improvement and PRD generation. The following slash commands are available:

> **Command Format:** Commands shown with colon (`:`) format. Some tools use hyphen (`-`): Claude Code uses `/clavix:improve`, Cursor uses `/clavix-improve`. Your tool autocompletes the correct format.

### Prompt Optimization

#### /clavix:improve [prompt]
Optimize prompts with smart depth auto-selection. Clavix analyzes your prompt quality and automatically selects the appropriate depth (standard or comprehensive). Use for all prompt optimization needs.

### PRD & Planning

#### /clavix:prd
Launch the PRD generation workflow. Clavix will guide you through strategic questions and generate both a comprehensive PRD and a quick-reference version optimized for AI consumption.

#### /clavix:plan
Generate an optimized implementation task breakdown from your PRD. Creates a phased task plan with dependencies and priorities.

#### /clavix:implement
Execute tasks or prompts with AI assistance. Auto-detects source: tasks.md (from PRD workflow) or prompts/ (from improve workflow). Supports automatic git commits and progress tracking.

Use `--latest` to implement most recent prompt, `--tasks` to force task mode.

### Session Management

#### /clavix:start
Enter conversational mode for iterative prompt development. Discuss your requirements naturally, and later use `/clavix:summarize` to extract an optimized prompt.

#### /clavix:summarize
Analyze the current conversation and extract key requirements into a structured prompt and mini-PRD.

### Refinement

#### /clavix:refine
Refine existing PRD or prompt through continued discussion. Detects available PRDs and saved prompts, then guides you through updating them with tracked changes.

### Agentic Utilities

These utilities provide structured workflows for common tasks. Invoke them using the slash commands below:

- **Verify** (`/clavix:verify`): Check implementation against PRD requirements. Runs automated validation and generates pass/fail reports.
- **Archive** (`/clavix:archive`): Archive completed work. Moves finished PRDs and outputs to archive for future reference.

**When to use which mode:**
- **Improve mode** (`/clavix:improve`): Smart prompt optimization with auto-depth selection
- **PRD mode** (`/clavix:prd`): Strategic planning with architecture and business impact

**Recommended Workflow:**
1. Start with `/clavix:prd` or `/clavix:start` for complex features
2. Refine requirements with `/clavix:refine` as needed
3. Generate tasks with `/clavix:plan`
4. Implement with `/clavix:implement`
5. Verify with `/clavix:verify`
6. Archive when complete with `/clavix:archive`

**Pro tip**: Start complex features with `/clavix:prd` or `/clavix:start` to ensure clear requirements before implementation.
<!-- CLAVIX:END -->

## Next.js Documentation (Context7 MCP)

This project uses the Context7 MCP server for on-demand Next.js documentation. Instead of bundled local docs, agents query Context7 for up-to-date library references.

- Use the `resolve-library-id` tool to find a library (e.g., "next.js", "react")
- Use the `get-library-docs` tool to fetch documentation for a specific topic
- The MCP server is configured in `.mcp.json` at the project root — no local docs directory needed
- Always consult Context7 for Next.js API questions rather than relying on training data
