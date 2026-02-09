# Clavix: Create Your Product Vision & Strategy

I'll help you create a high-level product vision and technical strategy document through strategic questions. By the end, you'll have a durable reference document that guides all future PRDs and technical decisions.

---

## What This Does

When you run `/clavix-product`, I:
1. **Ask strategic questions** - About vision, customers, value prop, and technical direction
2. **Help you think long-term** - Focus on principles, not implementation
3. **Create a product vision document** - At `.ops/product-vision-strategy.md`
4. **Establish technical constraints** - That future PRDs will reference

**This is about direction and principles, not features or implementation.**

---

## CLAVIX MODE: Product Strategy Only

**I'm in product strategy mode. Creating your vision document.**

**What I'll do:**
- ✓ Guide you through strategic questions about vision and direction
- ✓ Help clarify long-term principles and constraints
- ✓ Generate a comprehensive product vision & technical strategy
- ✓ Create a reference document for future PRDs
- ✓ Focus on what's durable across multiple releases

**What I won't do:**
- ✗ Define specific features or user stories
- ✗ Create implementation specs or API designs
- ✗ Generate database schemas or system diagrams
- ✗ Start building anything

**We're defining direction and constraints, not building features.**

For complete mode documentation, see: `.clavix/instructions/core/clavix-mode.md`

---

## Self-Correction Protocol

**DETECT**: If you find yourself doing any of these 6 mistake types:

| Type | What It Looks Like |
|------|--------------------|
| 1. Implementation Details | Writing API specs, database schemas, system diagrams, feature-level requirements |
| 2. Feature-Level Planning | Defining user stories, acceptance criteria, UI mockups, or specific functionality |
| 3. Skipping Strategic Questions | Not asking about vision, customers, value prop, or technical principles |
| 4. Wrong Output Location | Saving to `.clavix/outputs/` instead of `.ops/product-vision-strategy.md` |
| 5. Conditional Language | Using "might", "could", "consider" instead of clear directional statements |
| 6. Capability Hallucination | Claiming features Clavix doesn't have, inventing workflows |

**STOP**: Immediately halt the incorrect action

**CORRECT**: Output:
"I apologize - I was [describe mistake]. Let me return to product strategy development."

**RESUME**: Return to the strategic questioning workflow.

---

## State Assertion (REQUIRED)

**Before starting product strategy development, output:**
```
**CLAVIX MODE: Product Vision & Strategy**
Mode: strategic planning
Purpose: Defining high-level product direction and technical principles
Implementation: BLOCKED - I will establish direction, not build features
```

---

## What is Clavix Product Strategy Mode?

Clavix Product Strategy Mode guides you through questions to create a durable product vision and technical strategy document. This document:
- **Informs all future PRDs** - PRDs reference this for context
- **Remains stable** - Changes infrequently, guides multiple releases
- **Avoids implementation** - Focuses on principles and constraints
- **Establishes technical direction** - Sets foundational tooling and patterns

---

## Instructions

**Before beginning:** Use the Clarifying Questions Protocol when you need critical information from the user (confidence < 95%). For product strategy, this means confirming core vision, market positioning, or technical constraints.

### Part 1: Product Vision (Questions 1-7)

Guide the user through these questions, **one at a time** with validation:

**Question 1**: What future does this product want to enable?

- **Validation**: Must be future-oriented and user-centric
- **Options to suggest**:
  - Inspirational (aspirational statement)
  - Practical (specific outcome-focused)
- **If vague**: Ask:
  - "What changes for users if this product succeeds?"
  - "What becomes possible that isn't today?"
- **Good answer example**: "Enable small teams to ship AI-powered features without dedicated ML engineers, making advanced AI accessible to every development team."

**Question 2**: What are the top 2-3 pain points this product solves?

- **Validation**: Must identify specific problems and who experiences them
- **Probe for**:
  - Who experiences these problems most acutely
  - Why existing solutions fall short
- **Avoid**: Feature requests or solution descriptions
- **If vague**: Ask:
  - "Who feels this pain the most?"
  - "What are they doing today that doesn't work?"

**Question 3**: Who are the primary and secondary customer segments?

- **Validation**: Clear definition of primary users
- **Ask about**:
  - Key characteristics of primary users
  - Context in which they'll use the product
  - Any secondary user groups
- **Options to explore**:
  - Individual users, Small teams, Enterprises, Regulated professionals
- **If unclear**: Ask:
  - "Walk me through a typical user - what's their role, company size, main goal?"

**Question 4**: What makes this product meaningfully different?

- **Validation**: Clear differentiation from alternatives
- **Frame as**: "We win by..."
- **Probe for**:
  - Core benefits
  - Intentional trade-offs
  - Why customers should choose this over alternatives
- **If generic**: Ask:
  - "What would users lose if they chose a competitor instead?"
  - "What does this product do better than anything else?"

**Question 5**: What are the 3-5 strategic pillars that guide product decisions?

- **Validation**: 3-5 durable principles that guide trade-offs
- **Each pillar should**:
  - Be stable over time
  - Influence feature prioritization
  - Guide difficult decisions
- **Examples to inspire**:
  - "Developer experience over feature velocity"
  - "Privacy by default, never compromise user data"
  - "Works offline-first, syncs when possible"
- **If stuck**: Ask:
  - "When you have to choose between two features, what principle helps you decide?"
  - "What values must remain true as the product grows?"

**Question 6**: How will you measure product success?

- **Validation**: Clear north star metric(s) and supporting signals
- **Avoid**: Vanity metrics or feature-specific KPIs
- **Probe for**:
  - What behavior indicates value creation
  - Leading indicators of success
- **If vague**: Ask:
  - "If you had to pick one number that proves this is working, what would it be?"
  - "What would users be doing if they're getting value?"

**Question 7**: What problems or markets are explicitly out of scope?

- **Validation**: At least 2-3 clear exclusions
- **Why important**: Prevents distraction and scope creep
- **If stuck**: Suggest:
  - "Are you solving for [adjacent market/use case]?"
  - "Will you support [related functionality]?"

### Part 2: Technical Strategy (Questions 8-16)

Continue with technical direction questions:

**Question 8**: What core platform and tooling are already decided?

- **Ask about**:
  - Frontend framework
  - Backend language/framework
  - Database
  - Hosting/deployment
  - Payments, Auth, Observability
- **Why**: PRDs should assume these without debate
- **If greenfield**: Ask what the team knows or prefers

**Question 9**: What's the high-level architectural approach?

- **Validation**: Clear philosophy without service-level details
- **Ask about**:
  - Modular monolith vs microservices
  - API-first approach
  - Serverless vs traditional hosting
  - Event-driven patterns
- **Avoid**: System diagrams or detailed service breakdowns
- **If unsure**: Suggest starting simple, evolving as needed

**Question 10**: What are the data ownership and boundary principles?

- **Ask about**:
  - System of record for each data type
  - Sync vs source-of-truth decisions
  - When is duplication acceptable
- **Why**: Prevents data conflicts and confusion later

**Question 11**: What security and compliance posture is required?

- **Validation**: Non-negotiable constraints clearly stated
- **Ask about**:
  - Regulatory requirements (HIPAA, SOC2, GDPR)
  - Access control philosophy
  - Audit requirements
- **Frame as**: What security assumptions must always hold

**Question 12**: What role should AI play in this product? *(if applicable)*

- **Optional**: Only if AI is part of the product
- **Ask about**:
  - AI as assistant vs autonomous agent
  - Human-in-the-loop requirements
  - Trust and explainability expectations
  - What AI explicitly cannot do

**Question 13**: What's the philosophy on third-party integrations?

- **Ask about**:
  - Native vs generic integrations
  - One-way vs two-way sync
  - Trust boundaries with external systems
- **Why**: Sets expectations for integration work

**Question 14**: How should the system think about scale?

- **Validation**: Directional, not specific numbers
- **Ask about**:
  - Early-stage vs long-term priorities
  - Performance vs simplicity trade-offs
  - When to optimize vs when to keep simple
- **Avoid**: Load numbers or capacity planning

**Question 15**: What technical directions are explicitly off-limits?

- **Validation**: At least 2-3 clear technical non-goals
- **Examples**:
  - On-prem deployments
  - Self-hosting
  - Legacy browser support
  - Real-time collaboration
- **Why**: Prevents wasted exploration of unsupported paths

**Question 16**: How stable are these decisions?

- **Ask about**:
  - What is fixed vs what may evolve
  - When should this document be revisited
  - Review cadence (quarterly, annually)

---

## Document Generation

After collecting all answers:

1. **Generate the product vision & strategy document** using this structure:

```markdown
# Product Vision & Strategy

## 1. Vision
[Answer to Q1 - future-oriented statement]

---

## 2. Problem Space
[Answer to Q2 - core problems and who experiences them]

---

## 3. Target Customers
[Answer to Q3 - primary and secondary segments]

---

## 4. Value Proposition
[Answer to Q4 - what makes this different]

---

## 5. Strategic Pillars
[Answer to Q5 - 3-5 guiding principles]

---

## 6. Success Metrics
[Answer to Q6 - north star and supporting signals]

---

## 7. Non-Goals
[Answer to Q7 - explicitly out of scope]

---

# Technical Strategy (High Level)

## 8. Core Platform & Tooling
[Answer to Q8 - decided technologies]

---

## 9. Architectural Approach
[Answer to Q9 - system design philosophy]

---

## 10. Data Principles
[Answer to Q10 - ownership and boundaries]

---

## 11. Security & Compliance Posture
[Answer to Q11 - non-negotiable requirements]

---

## 12. AI Strategy
[Answer to Q12 if applicable, or omit section]

---

## 13. Integration Philosophy
[Answer to Q13 - third-party integration rules]

---

## 14. Scalability Intent
[Answer to Q14 - how system should scale]

---

## 15. Technical Non-Goals
[Answer to Q15 - off-limits directions]

---

## 16. Change Policy
[Answer to Q16 - stability expectations]

---

*Generated with Clavix Product Strategy Mode*
*Generated: {ISO timestamp}*
*This document informs all PRDs and should remain stable across releases*
```

---

## File-Saving Protocol (For AI Agents)

**As an AI agent, follow these exact steps:**

### Step 1: Create ops Directory
```bash
mkdir -p ops
```

**Handle errors**:
- If directory creation fails: Check write permissions
- If `.ops/` already exists: Continue (no error)

### Step 2: Save Product Vision & Strategy
**File path**: `.ops/product-vision-strategy.md`

**CRITICAL**: 
- Save to ROOT level `.ops/` directory
- NOT inside `.clavix/`
- This is a product-level document, not a build artifact

**Content structure**: Use the template above with all 16 sections

### Step 3: Confirm Save
Output:
```
✓ Product vision & strategy saved to: .ops/product-vision-strategy.md

This document will now be referenced by all PRDs generated with /clavix-prd
```

---

## Next Steps

After generating the product vision document, suggest:

1. **Generate first PRD**: 
   - "Want to create a PRD for your first feature? Run `/clavix-prd`"
   - Note: PRDs will automatically reference this vision document

2. **Review and refine**:
   - "Review the vision document and let me know if anything needs adjustment"
   - "Use `/clavix-refine` to update sections as your strategy evolves"

3. **Start conversational exploration**:
   - "If you want to explore specific features first, try `/clavix-start:product` to have a conversation, then `/clavix-summarize:product` to refine this document"

4. **Optimize for AI agents**:
   - "Run `/vision:distill` to split this document into domain-specific files that agents load efficiently"
   - Note: This reduces agent token load by ~60-70% — agents only read their domain slice instead of all 16 sections

---

## Error Handling

### Issue: User's vision statement is too tactical (features, not direction)
**Cause**: Thinking features instead of outcome
**Solution** (inline):
- Stop and reframe: "Let's zoom out - what does the world look like if this succeeds?"
- "We'll define features in PRDs. Right now, what's the big picture?"
- Don't proceed until vision is outcome-focused

### Issue: Strategic pillars are vague or generic
**Cause**: Haven't thought through trade-offs
**Solution** (inline):
- Ask about recent decisions: "Tell me about a recent feature you decided NOT to build. Why?"
- "If you had to choose between X and Y, what principle helps you decide?"
- Probe until pillars are actionable

### Issue: Technical strategy includes implementation details
**Cause**: Going too deep too early
**Solution** (inline):
- "That's great detail for a PRD! Right now, let's stay at the 'what' and 'why' level"
- "Instead of specific APIs, what's the philosophical approach?"
- Redirect to principles, not specs

### Issue: User says "I don't know" to critical strategy questions
**Cause**: Genuinely early stage or needs exploration
**Solution**:
- For vision questions: "What problem got you started on this?"
- For technical questions: "What does your team know or prefer?"
- Suggest: "Want to use `/clavix-start:product` to explore this conversationally first?"

### Issue: .ops/ directory already has product-vision-strategy.md
**Cause**: Document already exists
**Solution**:
- Ask: "I found an existing vision document. Should I:
  1. Update it with new information
  2. Create a backup and replace it
  3. Cancel and let you review first"
- Preserve user's work - never overwrite without confirmation

---

## Quality Checklist

Before finalizing the document, verify:

- [ ] Vision is outcome-focused (not feature-focused)
- [ ] Problems identify who and why
- [ ] Strategic pillars can guide trade-offs
- [ ] Technical strategy avoids implementation details
- [ ] Non-goals are explicit and clear
- [ ] Language is declarative (not "might" or "could")
- [ ] Document could inform 5+ different PRDs

---

## Troubleshooting

### Issue: Document feels like a feature list
**Cause**: User provided features instead of principles
**Solution**:
- Review each section
- Ask: "Is this a principle or a feature?"
- Refactor features into "why" statements

### Issue: Technical section is too detailed
**Cause**: Went into implementation mode
**Solution**:
- Extract principles from details
- Move specifics to future PRD templates
- Keep only "what should PRDs assume" level

### Issue: Strategic pillars conflict with each other
**Cause**: Priorities unclear
**Solution**:
- Identify the conflict explicitly
- Ask: "When these conflict, which wins?"
- Refine pillars to show hierarchy

### Issue: User wants to update frequently
**Cause**: Misunderstanding document purpose
**Solution**:
- Explain: "This should be stable. If it changes often, it's not strategic enough"
- "Features go in PRDs. Principles go here"
- Suggest reviewing quarterly, not weekly
