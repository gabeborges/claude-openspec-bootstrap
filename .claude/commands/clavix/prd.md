# Clavix: Create Your PRD

I'll help you create a lightweight, version-scoped Product Requirements Document (Mini-PRD) through strategic questions. By the end, you'll have clear documentation of what to build for this specific version and why.

---

## What This Does

When you run `/clavix-prd`, I:
1. **Check for product vision** - If `.ops/product-vision-strategy.md` exists, I'll reference it
2. **Establish version scope** - Confirm what version we're documenting (v0, v1, v2, etc.)
3. **Ask focused questions** - About users, problems, features, and success metrics
4. **Help you prioritize** - Separate must-haves from nice-to-haves
5. **Create a versioned Mini-PRD** - Saved to `.ops/build/v{X}/prd.md`

**This PRD is scoped to one version/epic, not the entire product.**

---

## CLAVIX MODE: Planning Only

**I'm in planning mode. Creating your version-scoped PRD.**

**What I'll do:**
- ✓ Check for and reference product vision document (if exists)
- ✓ Confirm version number and scope
- ✓ Guide you through strategic questions
- ✓ Help separate must-haves from nice-to-haves
- ✓ Generate practical, engineer-friendly PRD
- ✓ Keep focus on what to build, not how to build it

**What I won't do:**
- ✗ Write API contracts or database schemas
- ✗ Create detailed architecture diagrams
- ✗ Start implementing anything
- ✗ Include implementation details

**We're documenting what to build for this version, not how to build it.**

For complete mode documentation, see: `.clavix/instructions/core/clavix-mode.md`

---

## Self-Correction Protocol

**DETECT**: If you find yourself doing any of these 7 mistake types:

| Type | What It Looks Like |
|------|--------------------|
| 1. Implementation Details | Writing API contracts, database schemas, system architecture diagrams, deployment pipelines |
| 2. Skipping Version Scope | Not confirming which version (v0, v1, v2) this PRD is for |
| 3. Ignoring Product Vision | Not checking for or referencing .ops/product-vision-strategy.md when it exists |
| 4. Wrong Output Location | Saving to .clavix/outputs/ instead of .ops/build/v{X}/prd.md |
| 5. Missing Key Sections | Not including must-haves, nice-to-haves, or non-goals |
| 6. Solution-Focused Problems | Describing solutions instead of problems in "The Problem We're Solving" |
| 7. Capability Hallucination | Claiming features Clavix doesn't have, inventing workflows |

**STOP**: Immediately halt the incorrect action

**CORRECT**: Output:
"I apologize - I was [describe mistake]. Let me return to PRD development."

**RESUME**: Return to the PRD development workflow with strategic questioning.

---

## State Assertion (REQUIRED)

**Before starting PRD development, output:**
```
**CLAVIX MODE: PRD Development (Mini-PRD)**
Mode: planning
Version: v{X} (to be confirmed)
Purpose: Creating version-scoped PRD that defines what to build and why
Implementation: BLOCKED - I will develop requirements, not implement features
```

---

## What is Clavix Mini-PRD?

Clavix Mini-PRD creates lightweight, version-scoped requirements documents that:
- **Focus on one version/epic** (v0, v1, v2, etc.) - not the entire product
- **Respect product vision** - Reference `.ops/product-vision-strategy.md` for strategic context
- **Stay high-level** - Define what to build, not how to build it
- **Separate priorities** - Must-haves vs nice-to-haves
- **Include non-goals** - Explicitly state what's NOT in scope

Each PRD is practical, engineer-friendly, and actionable.

---

## Instructions

**STEP 0: Check for Product Vision & Strategy Document (REQUIRED)**

Before starting the PRD questions, check if `.ops/product-vision-strategy.md` exists:

```bash
# Check if product vision exists
if [ -f ".ops/product-vision-strategy.md" ]; then
  echo "✓ Found product vision document"
fi
```

**If the file exists:**
- **READ IT COMPLETELY** before asking any PRD questions
- Internalize:
  - Vision and strategic pillars that should guide this PRD
  - Technical platform/tooling already decided (use in Assumptions)
  - Architectural approach to follow
  - Security/compliance requirements (use in Constraints)
  - Data principles
  - Non-goals to respect
- Reference it throughout PRD development
- Technical Assumptions section should reference decisions from product vision
- Ensure PRD aligns with product vision

**If the file does NOT exist:**
- Continue with standard PRD flow
- After PRD generation, suggest: "Consider running `/clavix-product` to establish high-level product strategy that will guide future PRDs."

---

**STEP 1: Establish Version Scope (REQUIRED)**

Before asking questions, confirm:

**Ask:** "Which version or epic is this PRD for? (e.g., v0, v1, v2, or 'initial launch')"

- **If unclear**: Suggest v0 for first version, v1 for next, etc.
- **Store this**: Use it in headings and file path
- **Why important**: PRDs are version-scoped, not whole-product scoped

---

**STEP 2: Guide Through Strategic Questions**

Ask these questions **one at a time** with validation:

### Question 1: What are we building? (Project/feature name + brief description)

- **Validation**: Must be concrete and specific
- **If vague**: Ask:
  - "What's the core capability or feature?"
  - "In one sentence, what does this do?"
- **Good answer example**: "User authentication system" or "Dashboard for sales analytics"

### Question 2: Who is this for?

- **Validation**: Clear primary user identified
- **Ask about**:
  - Primary users
  - Secondary users (if any)
  - Context in which they'll use it
- **If vague**: Probe:
  - "Who specifically will interact with this?"
  - "What's their role or situation?"
- **Good answer format**: "Primary: Sales managers reviewing pipeline. Secondary: Sales reps checking their own deals."

### Question 3: What problem does this solve?

- **Validation**: Problem stated, not solution
- **Ask about**:
  - What's broken or missing today
  - Why existing solutions fall short
  - Why this matters now
- **If user jumps to solutions**: Redirect:
  - "Before we talk about how to solve it, what's the core problem?"
  - "What happens if we don't build this?"
- **Good answer**: Focuses on pain points, not features

### Question 4: What does success look like for this version?

- **Validation**: Outcome-focused goals
- **Ask about**:
  - Primary goal for this version
  - Secondary goals if any
  - How users' behavior or situation changes
- **Avoid**: Task-based goals ("ship feature X")
- **Good answer**: "Sales managers can identify at-risk deals in under 2 minutes instead of 30+ minutes of manual review"

### Question 5: What are the must-have features for this version?

- **Validation**: 3-7 concrete features
- **Ask**: "If you could only ship 3-5 things, what would they be?"
- **For each feature**: Get brief explanation of what it does
- **If too many** (8+): Help prioritize:
  - "Which of these are absolute launch-blockers?"
  - "Which could wait for the next version?"
- **If too few** (1-2): Probe:
  - "What else needs to exist for this to be useful?"
- **Good format**: 
  - "User login with email/password"
  - "Deal health score based on activity"
  - "Filter deals by health status"

### Question 6: What would be nice to have, but isn't required?

- **Optional**: Can skip if unclear
- **Ask**: "What features would add value but could wait?"
- **Why important**: Keeps scope manageable, documents future work
- **Good format**: Short list of 2-4 enhancements

### Question 7: How will we know it's working?

- **Validation**: Concrete success signals
- **Ask about**:
  - User behavior indicators
  - Performance expectations
  - Adoption or usage signals
- **Tie back to goals**: Should relate to Q4 answers
- **If vague**: Ask:
  - "What would users be doing if this succeeds?"
  - "What metrics or signals would prove value?"

### Question 8: Any functional requirements or system behaviors to specify?

- **Optional**: Can skip if covered in features
- **Ask**: "Any specific behaviors the system must support?"
- **Focus on**: What the system does, not how
- **If unclear**: Skip this, features may be enough

### Question 9: Any non-functional requirements? (Performance, accessibility, reliability)

- **Optional**: Only if relevant to this version
- **Ask about**:
  - Performance expectations (load time, responsiveness)
  - Accessibility needs
  - Reliability requirements
- **Avoid**: Boilerplate requirements
- **Good format**: Specific and measurable when possible

### Question 10: What technical assumptions should engineers make?

- **Reference product vision if exists**: Pull from core platform decisions
- **Ask about**:
  - Platform or framework choices already made
  - Authentication approach
  - Hosting environment
- **If product vision exists**: "I see your product vision specifies [X]. Should I include that as an assumption?"
- **If no product vision**: Ask what's already decided

### Question 11: Any constraints affecting how this can be built?

- **Reference product vision if exists**: Pull from security/compliance section
- **Ask about**:
  - Compliance or regulatory requirements
  - Data residency rules
  - Integration limitations
- **Keep high-level**: No implementation details

### Question 12: What does this depend on?

- **Optional**: Only if there are dependencies
- **Ask about**:
  - Third-party services
  - Other product initiatives
  - Internal platform capabilities
- **Avoid**: Deep technical details

### Question 13: Any integration points with external systems?

- **Optional**: Only if integrations are relevant
- **Ask**: "Does this need to connect to other systems?"
- **Keep descriptive**: Not technical

### Question 14: Any open questions or uncertainties?

- **Optional**: Valuable for risk management
- **Ask**: "What's still unclear or undecided?"
- **Examples**:
  - Product decisions
  - UX uncertainties
  - Policy questions
- **Why important**: Surfaces risks early

### Question 15: What's explicitly NOT in scope for this version?

- **Validation**: At least 2-3 clear exclusions
- **Ask**: "What are we definitely NOT building?"
- **Why critical**: Prevents scope creep
- **If stuck**: Suggest based on previous answers:
  - "Are we building mobile apps? Admin dashboards?"
  - "Are we handling [related feature]?"
- **Good format**: Explicit list of excluded capabilities

---

**STEP 3: Verify Minimum Viable Answers**

Before generating PRD, confirm:
- Version number is clear
- Project/feature name exists
- Primary users identified
- Problem described (not solution)
- At least 3 must-have features
- At least 2 non-goals specified

If missing critical info, ask targeted follow-ups.

---

**STEP 4: Generate Mini-PRD Document**

Create the PRD using this exact structure:

```markdown
# PRD: {Project/Feature Name}

**Version:** v{X}  
**Product Vision Reference:** [If exists] See `.ops/product-vision-strategy.md` for strategic context

---

## What We're Building

{Answer from Q1 - what we're building, what it does, core value}

{Keep 1-3 paragraphs, simple and concrete}

---

## Who It's For

{Answer from Q2}

- Primary users: {description}
- Secondary users: {description if any}
- Context: {when/how they use it}

---

## The Problem We're Solving

{Answer from Q3}

{Describe the problem clearly:}
- What's broken or missing today?
- Why existing solutions fall short?
- Why this problem matters now

---

## Goals & Outcomes (This Version)

{Answer from Q4}

**Primary goal(s):**
- {Goal 1}

**Secondary goal(s):**
- {Goal 2 if any}

These goals are outcome-focused and user-centric.

---

## Must-Have Features (v{X})

{Answer from Q5}

These are the **non-negotiables** for this version:

1. **{Feature name}** – {Brief explanation of what it does}
2. **{Feature name}** – {Brief explanation}
3. **{Feature name}** – {Brief explanation}
{... continue for 3-7 features}

---

## Nice-to-Have Features (Later)

{Answer from Q6 - if provided}

Valuable features that are **not required for this version**:

- {Feature or enhancement}
- {Feature or enhancement}

---

## How We'll Know It's Working

{Answer from Q7}

Success signals for this version:

- {User behavior indicator}
- {Performance expectation}
- {Adoption or usage signal}

These tie back to our goals above.

---

## Requirements (More Detail)

### Functional Requirements

{Answer from Q8 - if provided, otherwise can be brief or omit}

{Required system behaviors - what the system must do, not how}

---

### Non-Functional Requirements

{Answer from Q9 - if provided}

{Quality expectations relevant to this version only}

Examples: Performance, accessibility, reliability

{Only include what's relevant - no boilerplate}

---

## Technical Considerations (High Level)

{This section respects the Product Technical Strategy from .ops/product-vision-strategy.md}

### Assumptions

{Answer from Q10}

{Technical decisions already made that engineers should assume:}

- {Platform/framework choices}
- {Authentication approach}
- {Hosting environment}

{If product vision exists, reference it: "See .ops/product-vision-strategy.md for core platform decisions"}

---

### Constraints

{Answer from Q11}

{Limits or rules affecting how this can be built:}

- {Compliance/regulatory requirements}
- {Data residency rules}
- {Integration limitations}

{If product vision exists, reference it: "See .ops/product-vision-strategy.md for security and compliance posture"}

---

### Dependencies

{Answer from Q12 - if provided}

{What this PRD depends on:}

- {Third-party services}
- {Other product initiatives}
- {Internal platform capabilities}

---

## Integration Points (If Applicable)

{Answer from Q13 - only include if integrations are relevant, otherwise omit}

{External systems involved and high-level interaction points}

---

## Open Questions

{Answer from Q14 - if provided}

{What's still unclear or undecided:}

- {Product decisions}
- {UX uncertainties}
- {Policy or compliance questions}

{These should be resolved before or during implementation}

---

## What's NOT In Scope

{Answer from Q15}

{Explicit exclusions for this version:}

- {Features intentionally excluded}
- {User segments not supported}
- {Technical capabilities deferred}

---

## Out-of-Scope Technical Detail

This PRD does **not** define:

- API contracts
- Database schemas
- Detailed system architecture
- Deployment pipelines

Those belong in:
- Technical design docs
- Architecture documents
- Implementation specs

---

*Generated with Clavix Planning Mode*  
*Version: v{X}*  
*Generated: {ISO timestamp}*
```

---

## File-Saving Protocol (For AI Agents)

**As an AI agent, follow these exact steps to save PRD files:**

### Step 1: Determine Version Number

```bash
# Check for existing versions in .ops/build/
if [ -d ".ops/build" ]; then
  # Find highest version number
  LATEST=$(ls -d .ops/build/v* 2>/dev/null | sed 's/.*v//' | sort -n | tail -1)
  VERSION=$((LATEST + 1))
else
  VERSION=0
fi
```

**Version Logic:**
- First PRD: `v0`
- Each new PRD: Increment version (v1, v2, v3, etc.)
- **IMPORTANT**: If user specified a version in Q1 (e.g., "this is for v2"), use that version instead of auto-incrementing
- Versions are sequential and never reused
- Example: If v0, v1, v2 exist → create v3 (unless user specified otherwise)

### Step 2: Create Output Directory

```bash
mkdir -p .ops/build/v${VERSION}
```

**Handle errors**:
- If directory creation fails: Check write permissions
- If `.ops/` doesn't exist: Create it first: `mkdir -p .ops/build/v${VERSION}`

### Step 3: Save PRD

**File path**: `.ops/build/v{VERSION}/prd.md`

**CRITICAL**: 
- Save to ROOT level `.ops/build/v{X}/` directory
- NOT inside `.clavix/`
- Single file named `prd.md`

**Content**: Use the complete template structure above with all answers filled in

### Step 4: Verify File Was Created

**Verification Protocol:**
1. **Immediately after Write**, use Read tool to verify:
   - Read `.ops/build/v{VERSION}/prd.md`
   - Confirm content matches what you wrote
   - Verify all sections are present

2. **If Read fails**: STOP and report error to user

### Step 5: Communicate Success

Display to user:
```
✓ Mini-PRD generated successfully!

File saved:
  • Version: v{VERSION}
  • Location: .ops/build/v{VERSION}/prd.md
  • Scope: {Feature name} - version-scoped for v{VERSION}
  • [If product vision exists] Aligned with: .ops/product-vision-strategy.md

Next steps:
  • Review and edit PRD if needed
  • Run /clavix-plan to generate implementation tasks
  • [If no product vision] Consider running /clavix-product for high-level strategy
```

### Error Handling

**If file write fails**:
1. Check error message
2. Common issues:
   - Permission denied: Inform user to check directory permissions
   - Disk full: Inform user about disk space
   - Path too long: Suggest shorter project name

**If version conflicts**:
- Never overwrite existing versions without confirmation
- If v{X} exists and user wants it: Ask for confirmation first
- Suggest creating next version (v{X+1}) instead

---

## Agent Transparency

### Clarifying Questions Protocol

**When to use** (confidence < 95%):
- User's version scope is ambiguous
- Problem description is vague
- Feature list is unclear or too broad
- Non-goals are missing or incomplete

**How to ask**:
1. State what you understand: "I understand you want to build {X}..."
2. State what's unclear: "...but I'm not sure about {Y}"
3. Ask specific question: "Could you clarify {Z}?"

**Template**:
> "I want to make sure I understand correctly. You're building {project} for {users}, which will {core capability}. 
>
> I'm unclear on {specific point}. Could you help me understand {specific question}?"

---

## Troubleshooting

### Issue: User's answers jump to solutions instead of problems

**Cause**: Natural tendency to think about features
**Solution** (inline):
- Redirect: "That's a great solution! But first, what's the problem it solves?"
- Probe: "What happens today that's broken or missing?"
- Don't proceed until problem is clearly stated

### Issue: User lists 10+ must-have features

**Cause**: Unclear priorities or scope creep
**Solution** (inline):
- Help prioritize: "If you could only ship 3-5 things for v{X}, which would they be?"
- Separate: "Which are launch-blockers vs nice-to-haves?"
- Document extras in "Nice-to-Have Features" section

### Issue: User says "I don't know" to critical questions

**Cause**: Genuine uncertainty or needs exploration
**Solution**:
- For problem: Ask about current pain points, what triggered this need
- For features: Walk through user journey step-by-step
- For non-goals: Suggest common exclusions based on project type
- Consider suggesting `/clavix-start` for conversational exploration first

### Issue: No clear version scope

**Cause**: User hasn't thought about versioning
**Solution**:
- Ask directly: "Is this for your first release or a later version?"
- Suggest: "Let's call this v0 (initial version)" or "v1 (next major version)"
- If still unclear: Default to v0 for new projects

### Issue: PRD conflicts with product vision document

**Cause**: User's answers contradict established product strategy
**Solution**:
- Point out specific conflicts: "I see this conflicts with [strategic pillar] in your product vision"
- Ask: "Should we:
  1. Update the PRD to align with the vision
  2. Update the product vision (rare - vision should be stable)
  3. Document this as an intentional exception"
- Ensure alignment before proceeding

### Issue: Product vision exists but wasn't referenced

**Cause**: AI forgot to check for or read the product vision document
**Solution** (self-correction):
- STOP immediately
- Read `.ops/product-vision-strategy.md` completely
- Review current PRD draft for conflicts
- Revise PRD to align with vision
- Add product vision reference to PRD

### Issue: User includes API specs or database schemas in answers

**Cause**: Thinks PRD needs implementation details
**Solution** (inline):
- Redirect: "That's helpful technical detail! For now, let's keep the PRD high-level"
- Explain: "Implementation details go in technical design docs, not the PRD"
- Ask for the 'what' not the 'how': "What should the system do, without specifying how?"

### Issue: Non-goals section is empty or vague

**Cause**: User hasn't thought about exclusions
**Solution**:
- Make it concrete: "Are we building [related feature]? Mobile apps? Admin dashboards?"
- Probe based on features: "You mentioned X, does that mean we're NOT doing Y?"
- Emphasize importance: "This helps prevent scope creep later"

### Issue: Versioning conflicts (v{X} already exists)

**Cause**: Filesystem race condition or manual file creation
**Solution**:
- Check for next available version number
- If v0, v1, v2 exist and user tries to create v1 again → use v3
- Never overwrite existing versions without explicit confirmation
- If user wants to replace a version: "Version v{X} already exists. Should I:
  1. Create a new version (v{Y})
  2. Archive v{X} and replace it
  3. Cancel"

### Issue: Generated PRD feels too detailed or too vague

**Cause**: Balance between completeness and brevity
**Solution**:
- Too detailed: Strip out implementation specifics, keep high-level
- Too vague: Ask follow-up questions for clarity
- Goal: Engineer can understand what to build, but has flexibility on how

---

## Quality Checklist

Before finalizing the PRD, verify:

- [ ] Version number is specified (v0, v1, v2, etc.)
- [ ] Problem is stated clearly (not jumping to solutions)
- [ ] Must-have features are concrete (3-7 items)
- [ ] Non-goals are explicit and clear (at least 2-3)
- [ ] Technical considerations stay high-level (no API specs or schemas)
- [ ] If product vision exists, PRD references it and aligns with it
- [ ] Document is actionable - engineer knows what to build
- [ ] Document is flexible - engineer has room to decide how
