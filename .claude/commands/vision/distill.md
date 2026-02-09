---
name: vision:distill
description: Split product-vision-strategy.md into domain-specific files for AI-efficient agent consumption
---

# Vision Distill

Split `.ops/product-vision-strategy.md` into 3 domain-scoped files under `.ops/`. Agents load only their domain (~30-40% of the full doc) instead of the entire vision document.

## Output Files

| File | Sections | Domain |
|------|----------|--------|
| `quick-product-vision-strategy.md` | §1, §2, §3, §4, §5, §6, §7, §12 | Product: vision, customers, value prop, pillars, metrics, non-goals, AI strategy |
| `security-compliance-baseline.md` | §10 (partial), §11, §12, §15, §16 | Security: data compliance subsections, security posture, AI boundaries, technical non-goals, change policy |
| `tech-architecture-baseline.md` | §8, §9, §10 (full), §12, §13, §14, §15, §16 | Architecture: platform, arch approach, data principles, AI strategy, integrations, scale, non-goals, change policy |

**§10 special handling:** Security file gets only the "Compliance & Auditability Guarantees" and "Data Flow Rules" subsections. Architecture file gets full §10.

**Intentional overlaps:** §12 in all 3 (AI boundaries span all domains). §15, §16 in security + architecture. §10 partial in security, full in architecture.

## Usage

```
/vision:distill              # Distill from .ops/product-vision-strategy.md
/vision:distill $ARGUMENTS   # Optional: path to a different vision doc
```

## Process

### Step 1: Read Source

Read `.ops/product-vision-strategy.md` (or path from `$ARGUMENTS` if provided).

**If missing:** Stop with: "No vision document found at `.ops/product-vision-strategy.md`. Run `/clavix:product` first."

### Step 2: Parse Sections

Identify all `## N.` headings (e.g., `## 1. Vision`, `## 8. Core Platform & Tooling`).

**Matching strategy:** Match by heading number first (`## 1.`, `## 2.`, etc.). If a number isn't found, fall back to heading text match (e.g., "Vision", "Security & Compliance Posture"). This handles docs where numbering differs.

Build a map: `section_number → { heading, content, status }`.

### Step 3: Assess Section Quality

For each section, determine if it's usable. A section is **included** only when ALL 3 criteria pass:

1. **Exists** — Heading is present in the source
2. **Substantive** — Content goes beyond the heading; not just "TBD", "TODO", "[Fill in later]", or a single generic sentence
3. **Concrete** — Contains at least one product-specific statement (names a technology, defines a constraint, states a principle, identifies a customer)

Mark sections that fail:
- `skip:empty` — Heading exists, body is empty/whitespace only
- `skip:placeholder` — Body is entirely placeholder text
- `skip:missing` — Heading not found in source

**Short sections (1-2 lines):** Still qualify if the content is a concrete, product-specific statement. Length alone doesn't disqualify.

### Step 4: Build Output Files

For each of the 3 output files, collect the mapped sections that passed quality assessment.

**If ALL sections for a file were skipped:** Do NOT create that file. Report it as skipped.

**§10 partial extraction for security file:**
From the full §10 content, extract only:
- The "Compliance & Auditability Guarantees" subsection (bold heading or `###` through next subsection/section break)
- The "Data Flow Rules" subsection (bold heading or `###` through next subsection/section break)

### Step 5: Write Files

Write each output file to `.ops/` with this structure:

```markdown
<!-- Source: .ops/product-vision-strategy.md -->
<!-- Included: §1, §2, §3, §4, §5, §6, §7, §12 -->
<!-- Skipped: §4 (placeholder) — or "none" -->
<!-- Regenerate: /vision:distill -->

[Verbatim section content from source]
```

**Rules:**
- Copy content exactly — preserve all markdown formatting, horizontal rules, emphasis, lists, code blocks
- Do NOT summarize, reword, reformat, or editorialize
- Separate sections with `---` between them (matching source style)
- Omit skipped sections entirely (no empty headings)

### Step 6: Account for All Sections

Verify every section found in the source is either:
- Included in at least one output file, OR
- Reported as skipped with a reason

**Extra sections (§17+, or unnumbered):** Report as "unaccounted" — don't silently drop them.

**The canonical 16-section structure:**
§1 Vision, §2 Problem Space, §3 Target Customers, §4 Value Proposition, §5 Strategic Pillars, §6 Success Metrics, §7 Non-Goals, §8 Core Platform & Tooling, §9 Architectural Approach, §10 Data Principles, §11 Security & Compliance Posture, §12 AI Strategy, §13 Integration Philosophy, §14 Scalability Intent, §15 Technical Non-Goals, §16 Change Policy

### Step 7: Completion Report

Output a summary table:

```
## Vision Distill Complete

| File | Status | Included | Skipped |
|------|--------|----------|---------|
| quick-product-vision-strategy.md | created | §1, §2, §3, §4, §5, §6, §7, §12 | none |
| security-compliance-baseline.md | created | §10 (partial), §11, §12, §15, §16 | none |
| tech-architecture-baseline.md | created | §8, §9, §10, §12, §13, §14, §15, §16 | none |

All 16 sections accounted for: yes/no
```

**If any sections were skipped**, add:

```
Sections to revisit (flesh out in canonical doc, then rerun /vision:distill):
- §N Title: reason (empty/placeholder/missing)
```

## Behavior

- **Idempotent** — Safe to rerun. Overwrites existing split files.
- **Non-destructive** — NEVER modifies `.ops/product-vision-strategy.md`.
- **Verbatim** — No summarizing or rewording. Exact copy.
- **Deterministic** — Same input always produces same output.

## Error Handling

| Problem | Action |
|---------|--------|
| Source file missing | Stop. Tell user to run `/clavix:product` |
| Source has no `##` headings | Stop. "Source doc doesn't match expected structure (no ## headings found)." |
| All sections for one output file skipped | Skip that file. Report in summary. |
| All sections for ALL output files skipped | Stop. "No usable sections found. The vision doc may be a skeleton — flesh it out first." |
| Extra sections beyond §16 | Include in report as "unaccounted". Don't drop silently. |
