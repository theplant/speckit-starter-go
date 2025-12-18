---
description: Create or update the project-level specification from a project description, providing vision, architecture, and cross-cutting concerns.
handoffs:
  - label: Build Feature Specification
    agent: speckit.specify
    prompt: Create a feature spec for this project. I want to build...
  - label: Update Constitution
    agent: speckit.constitution
    prompt: Update the constitution based on the project spec principles.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Plain English Rules (CRITICAL - READ FIRST)

**Before writing ANY spec content, you MUST follow these rules.**

### Forbidden Phrases → Plain English Replacements

| ❌ Never Use | ✅ Use Instead |
|--------------|----------------|
| "enforce consistent validation" | "all items follow the same rules" |
| "maintain backward compatibility" | "existing data keeps working" |
| "downstream system protection" | "other systems always get valid data" |
| "schema-first structure" | "define fields once, reuse everywhere" |
| "extensibility" | "add new features later" |
| "idempotent" | "running twice gives the same result" |
| "atomicity" | "all-or-nothing" |
| "data integrity" | "data stays correct" |
| "scalability" | "handle more users/data" |
| "latency" | "response time" or "how fast" |
| "throughput" | "how many requests per second" |
| "resilience" | "keep working when things fail" |
| "orchestration" | "coordinate multiple steps" |
| "abstraction" | "hide the details" |
| "encapsulation" | "keep things separate" |
| "middleware" | "code that runs in between" |
| "payload" | "data" or "request body" |
| "endpoint" | "API" or "URL" |
| "serialization" | "convert to JSON" |
| "deserialization" | "read from JSON" |

### Writing Guidelines

1. **Use simple words**: If a 10-year-old can't understand it, rewrite it
2. **Short sentences**: Under 20 words per sentence
3. **Concrete outcomes**: "so that users can see their orders" not "so that visibility is improved"
4. **Active voice**: "System saves the file" not "The file is saved by the system"
5. **No acronyms without explanation**: First use must spell it out

### Self-Check (Do This Before Finishing)

After writing the spec, read every sentence and ask:
- Would my non-technical friend understand this?
- Can I say this with fewer words?
- Is there a simpler word I can use?

If the answer is "no" to any question, rewrite that sentence.

## Outline

The text the user typed after `/speckit.projectspec` in the triggering message **is** the project description. Assume you always have it available in this conversation even if `$ARGUMENTS` appears literally below. Do not ask the user to repeat it unless they provided an empty command.

Given that project description, do this:

1. Load `.specify/templates/project-spec-template.md` to understand required sections.

2. **Read the constitution for plain English rules**: Load `.specify/memory/constitution.md` and find Principle XIV (Plain English Writing Style). Follow those rules strictly.

3. Follow this execution flow:

    1. Parse user description from Input
       If empty: ERROR "No project description provided"
    2. Extract key concepts from description
       Identify: problem domain, stakeholders, goals, constraints, system boundaries
    3. For unclear aspects:
       - Make informed guesses based on context and industry standards
       - Only mark with [NEEDS CLARIFICATION: specific question] if:
         - The choice significantly impacts project scope or architecture
         - Multiple reasonable interpretations exist with different implications
         - No reasonable default exists
       - **LIMIT: Maximum 5 [NEEDS CLARIFICATION] markers total**
       - Prioritize clarifications by impact: scope > architecture > integration > details
    4. Fill Problem Statement section
       If no clear problem: ERROR "Cannot determine problem statement"
    5. Generate Vision & Guiding Principles
       Each principle must be actionable and testable
    6. Define Goals and Non-Goals
       Goals must be specific and measurable
       Non-goals explicitly state what is out of scope
    7. Define Scope Boundaries
       Clear separation of responsibilities
    8. Document High-Level Flow and Key Use Cases
       Plain English descriptions of core workflows
    9. Define System Contracts
       Core guarantees the system provides
    10. Set Success Metrics
        Measurable, technology-agnostic outcomes
    11. Identify Dependencies and Risks
    12. Return: SUCCESS (project spec ready)

4. Write the specification in PLAIN ENGLISH to PROJECT_SPEC_FILE using the template structure, replacing placeholders with concrete details derived from the project description while preserving section order and headings.

5. **Specification Quality Validation**: After writing the initial spec, validate it against quality criteria:

   a. **Create Spec Quality Checklist**: Generate a checklist file at `specs/checklists/project-spec-quality.md` with these validation items:

      ```markdown
      # Project Specification Quality Checklist: [PROJECT NAME]

      **Purpose**: Validate project specification completeness and quality
      **Created**: [DATE]
      **Project Spec**: [Link to project-spec.md]

      ## Content Quality

      - [ ] No implementation details (languages, frameworks, specific tools)
      - [ ] Focused on business value and user needs
      - [ ] Written for non-technical stakeholders
      - [ ] All mandatory sections completed
      - [ ] **Written in plain English (no jargon)**

      ## Plain English Check

      - [ ] No forbidden phrases from the Plain English Rules table
      - [ ] All sentences under 20 words
      - [ ] Outcomes are concrete (not abstract)
      - [ ] A non-technical person could understand every section

      ## Section Completeness

      - [ ] Problem Statement clearly defines the problem and stakeholders
      - [ ] Vision is compelling and achievable
      - [ ] Guiding Principles are actionable and testable
      - [ ] Goals are specific and measurable
      - [ ] Non-Goals explicitly state what is out of scope
      - [ ] Scope Boundaries clearly separate responsibilities
      - [ ] High-Level Flow describes core workflows
      - [ ] Key Use Cases cover primary scenarios
      - [ ] System Contracts define core guarantees
      - [ ] Success Metrics are measurable and technology-agnostic
      - [ ] Dependencies are identified
      - [ ] Risks have mitigations

      ## Project Readiness

      - [ ] No [NEEDS CLARIFICATION] markers remain
      - [ ] All sections are internally consistent
      - [ ] Project can be broken into feature specs
      - [ ] Architecture overview provides sufficient guidance

      ## Notes

      - Items marked incomplete require spec updates before proceeding
      ```

   b. **Run Validation Check**: Review the spec against each checklist item:
      - For each item, determine if it passes or fails
      - **Pay special attention to Plain English Check items**
      - Document specific issues found (quote relevant spec sections)

   c. **Handle Validation Results**:

      - **If all items pass**: Mark checklist complete and proceed to step 6

      - **If items fail (excluding [NEEDS CLARIFICATION])**:
        1. List the failing items and specific issues
        2. Update the spec to address each issue
        3. Re-run validation until all items pass (max 3 iterations)
        4. If still failing after 3 iterations, document remaining issues in checklist notes and warn user

      - **If [NEEDS CLARIFICATION] markers remain**:
        1. Extract all [NEEDS CLARIFICATION: ...] markers from the spec
        2. **LIMIT CHECK**: If more than 3 markers exist, keep only the 3 most critical (by scope/architecture impact) and make informed guesses for the rest
        3. For each clarification needed (max 3), present options to user in this format:

           ```markdown
           ## Question [N]: [Topic]

           **Context**: [Quote relevant spec section]

           **What we need to know**: [Specific question from NEEDS CLARIFICATION marker]

           **Suggested Answers**:

           | Option | Answer | Implications |
           |--------|--------|--------------|
           | A      | [First suggested answer] | [What this means for the project] |
           | B      | [Second suggested answer] | [What this means for the project] |
           | C      | [Third suggested answer] | [What this means for the project] |
           | Custom | Provide your own answer | [Explain how to provide custom input] |
           **Your choice**: _[Wait for user response]_
           ```

        4. Number questions sequentially (Q1, Q2, Q3 - max 3 total)
        5. Present all questions together before waiting for responses
        6. Wait for user to respond with their choices
        7. Update the spec by replacing each [NEEDS CLARIFICATION] marker with the user's selected or provided answer
        8. Re-run validation after all clarifications are resolved

   d. **Update Checklist**: After each validation iteration, update the checklist file with current pass/fail status

6. Report completion with spec file path, checklist results, and readiness for the next phase (creating feature specs with `/speckit.specify`).

## Output Location

The project specification should be written to: `specs/main-spec.md`

## General Guidelines

### Quick Guidelines

- Focus on **WHAT** the project achieves and **WHY** it matters.
- Avoid HOW to implement (no tech stack, APIs, code structure in problem/vision sections).
- Written for business stakeholders, not developers.
- **USE PLAIN ENGLISH - this is the key requirement of this workflow**
- Architecture Overview section CAN include high-level technical patterns but should remain technology-agnostic where possible.

### Section Requirements

- **Mandatory sections**: Problem Statement, Vision & Guiding Principles, Goals, Non-Goals, Scope Boundaries, High-Level Flow, Key Use Cases, Success Metrics
- **Optional sections**: System Contracts, Open Questions, Decision Points, Dependencies, Risks, Architecture Overview, Links
- When a section doesn't apply, remove it entirely (don't leave as "N/A")

### For AI Generation

When creating this spec from a user prompt:

1. **Make informed guesses**: Use context, industry standards, and common patterns to fill gaps
2. **Document assumptions**: Record reasonable defaults in the Open Questions section if needed
3. **Limit clarifications**: Maximum 3 [NEEDS CLARIFICATION] markers - use only for critical decisions that:
   - Significantly impact project scope or architecture
   - Have multiple reasonable interpretations with different implications
   - Lack any reasonable default
4. **Prioritize clarifications**: scope > architecture > integration > details
5. **Think holistically**: Consider how features will interact and what cross-cutting concerns exist
6. **WRITE IN PLAIN ENGLISH**: Re-read every sentence and simplify

### Success Metrics Guidelines

Success metrics must be:

1. **Measurable**: Include specific metrics (time, percentage, count, rate)
2. **Technology-agnostic**: No mention of frameworks, languages, databases, or tools
3. **User-focused**: Describe outcomes from user/business perspective
4. **Verifiable**: Can be tested/validated without knowing implementation details
5. **Written in plain English**: No jargon

**Good examples**:
- "Users can complete their primary task in under 5 minutes"
- "System supports 10,000 concurrent users"
- "95% of operations complete in under 2 seconds"
- "User satisfaction score above 4.5/5"

**Bad examples** (implementation-focused):
- "API response time is under 200ms" (too technical)
- "Database can handle 1000 TPS" (implementation detail)
- "Microservices communicate efficiently" (architecture-specific)
