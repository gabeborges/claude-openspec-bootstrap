# claude-openspec-bootstrap

Bootstrap template for Next.js 15 + Supabase + Stripe projects with Claude Code, OpenSpec, and Clavix.

## What This Is

This is not a runnable app — it's a configuration template. It copies skills, commands, specs config, documentation, and project rules into your existing Next.js project. The result is a fully configured spec-driven development (SDD) environment with AI-assisted workflows powered by Claude Code.

## Target Stack

Next.js 15 (App Router), TypeScript, React 19, Tailwind CSS, shadcn/ui, Headless UI, Supabase, Stripe, Google OAuth, Zod, Vitest + Playwright

## What's Included

| Item | Path | Purpose |
|------|------|---------|
| Project rules | `CLAUDE.md` | Security, autonomy, coding standards, stack config |
| Command reference | `AGENTS.md` | Available slash commands summary |
| OpenSpec config | `openspec/config.yaml` | Artifact pipeline rules and phase gates |
| Domain skills (21) | `.claude/skills/` | Security, testing, API, performance, compliance, UI, OpenSpec workflow |
| Slash commands (26) | `.claude/commands/` | OPSX (10), Clavix (11), Interface Design (4), Vision (1) |
| Agent permissions | `.claude/settings.local.json` | Pre-configured tool permissions |
| Expert agent | `.claude/agents/` | OpenSpec/Claude configuration subagent |
| MCP servers | `.mcp.json` | Context7, Playwright, Supabase, Resend, Vercel |
| Git ignores | `.gitignore` | Excludes node_modules, .env, .next, .ops, .clavix |

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- [Clavix CLI](https://github.com/clavix) (`clavix`)
- [OpenSpec CLI](https://github.com/openspec) (`openspec`)
- Node.js + npm

## Installation

Run these steps from your target project root.

### Step 1: Initialize Clavix

Sets up the `.clavix/` directory and config for prompt optimization and PRD workflows.

```bash
clavix init
```

### Step 2: Initialize OpenSpec

Creates the `openspec/` directory with a default config for spec-driven development.

```bash
openspec init
```

### Step 3: Replace OpenSpec config

Copy `openspec/config.yaml` from this bootstrap into your project, replacing the default one generated in Step 2. This adds phase gates, skill activation rules, and artifact pipeline configuration.

### Step 4: Install agent skills

Installs Vercel reference skills for React and web development patterns. Skip react-native skills if not needed.

```bash
npx skills add vercel-labs/agent-skills
```

### Step 5: Copy `.mcp.json`

Copy `.mcp.json` to your project root. This configures MCP servers that extend Claude's capabilities. Claude Code will prompt for approval on first use.

#### MCP Servers

| Server | Package | Purpose |
|--------|---------|---------|
| Context7 | `@upstash/context7-mcp` | On-demand library docs (Next.js, React, etc.) |
| Playwright | `@playwright/mcp` | Browser automation and testing |
| Supabase | `@supabase/mcp-server-supabase` | Database operations, auth, project management |
| Resend | `@bugzy-ai/resend-mcp-server` | Email sending and management |
| Vercel | `mcp-remote` → `mcp.vercel.com` | Deployment management, logs, project config |

#### Required Environment Variables

Set these before using servers that need auth:

```bash
# Supabase — personal access token (dashboard.supabase.com/account/tokens)
SUPABASE_ACCESS_TOKEN=sbp_xxxxxxxxxxxxx

# Resend — API key (resend.com/api-keys)
RESEND_API_KEY=re_xxxxxxxxxxxxx
RESEND_FROM_EMAIL=You <notifications@yourdomain.com>
```

**Vercel** uses OAuth — it will prompt you to authenticate in the browser on first use. No env vars needed.

**Context7** and **Playwright** require no authentication.

### Step 6: Copy `CLAUDE.md` and `AGENTS.md`

Copy both files to your project root. `CLAUDE.md` contains project rules (security, autonomy, coding standards, stack config) that Claude loads every session. `AGENTS.md` provides a quick reference of all available slash commands.

### Step 7: Copy `.gitignore`

Copy the `.gitignore` file, or merge its entries into your existing one. Key exclusions: `node_modules/`, `.env*`, `.next/`, `.ops/`, `.clavix/`.

### Step 8: Replace `.claude/skills/` folder

Copy the `.claude/skills/` directory into your project, replacing any existing skills. This adds 21 domain-specific skills covering security patterns, testing strategy, API contracts, performance, compliance, database migrations, UI design, and OpenSpec workflows.

### Step 9: Replace `.claude/commands/` folder

Copy the `.claude/commands/` directory into your project. This adds 26 slash commands across 4 namespaces:

- **opsx/** (10) — OpenSpec change lifecycle (`/opsx:new`, `/opsx:ff`, `/opsx:apply`, etc.)
- **clavix/** (11) — Prompt optimization and PRD workflows (`/clavix:prd`, `/clavix:plan`, `/clavix:implement`, etc.)
- **interface-design/** (4) — UI design system management (`/interface-design:init`, `/interface-design:audit`, etc.)
- **vision/** (1) — Product vision distillation (`/vision:distill`)

### Step 10: Copy `settings.local.json` to `.claude/`

Copy `.claude/settings.local.json` into your project's `.claude/` directory. This provides pre-configured agent permissions so Claude Code can execute commands without repeated approval prompts.

## What's NOT Copied

These directories are generated by their respective tools and should not be copied from the bootstrap:

- **`.clavix/`** — Generated by `clavix init` (Step 1)
- **`.agents/`** — Installed by `npx skills add` (Step 4)

## Workflow Overview

The unified Clavix + OpenSpec pipeline:

```
/clavix:prd → /clavix:plan → /opsx:new → /opsx:ff → /opsx:apply → /opsx:verify → /opsx:archive
```

Start with a PRD, generate a task plan, create a spec-driven change, fast-forward through artifacts, implement, verify against specs, and archive when complete.

See `AGENTS.md` for the full command list and usage details.
