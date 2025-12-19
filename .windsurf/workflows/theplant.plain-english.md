---
description: Apply Plain English writing discipline - use clear, concise language that anyone can understand without jargon or unnecessary complexity.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Ensure all spec, plan, and tasks content follows Plain English principles. This workflow guides writing clear, accessible feature specifications, implementation plans, and task lists that stakeholders and developers can understand quickly.

**Before writing ANY spec content, you MUST follow these rules. This is the key difference from `/speckit.specify`.**

## Rationale (PLAIN-ENGLISH)

Complex language creates barriers. Verbose specs waste review time. Unclear tasks cause rework. Plain English ensures everyone—product owners, developers, testers—can understand requirements and act on them confidently.

## Plain English Rules (CRITICAL - READ FIRST)

### Forbidden Phrases → Plain English Replacements

| ❌ Never Use | ✅ Use Instead |
|--------------|----------------|
| "enforce consistent validation" | "all products follow the same rules" |
| "maintain backward compatibility" | "existing products keep their old values" |
| "downstream system protection" | "Search and OMS always get valid data" |
| "improve authoring experience" | "make the form easier to use" |
| "schema-first product structure" | "define fields once, reuse everywhere" |
| "participate in search/filter logic" | "show up in search results and filters" |
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
| "polymorphism" | (don't use this word at all) |
| "middleware" | "code that runs in between" |
| "payload" | "data" or "request body" |
| "endpoint" | "API" or "URL" |
| "serialization" | "convert to JSON" |
| "deserialization" | "read from JSON" |

### Generic Word Replacements

| ❌ Avoid | ✅ Use Instead |
|----------|----------------|
| utilize | use |
| implement | build, create, add |
| leverage | use |
| facilitate | help, enable |
| functionality | feature |
| in order to | to |
| due to the fact that | because |
| prior to | before |
| subsequent to | after |
| in the event that | if |
| with regard to | about |
| has the ability to | can |
| ensure that | ensure |
| in accordance with | per, following |

### Writing Guidelines

1. **Use simple words**: If a 10-year-old can't understand it, rewrite it
2. **Short sentences**: Under 20 words per sentence
3. **Concrete outcomes**: "so that users can see their orders" not "so that visibility is improved"
4. **Active voice**: "System saves the file" not "The file is saved by the system"
5. **No acronyms without explanation**: First use must spell it out
6. **Lead with the main point**: Put the most important information first
7. **Cut filler words**: Remove "very", "really", "basically", "actually", "just"
8. **Be specific**: "soon" → "by Friday", "some" → "three"

## Application to Spec, Plan, and Tasks

### Feature Specifications (spec.md)

**User Stories**:

```markdown
❌ BAD:
As a user, I would like to have the ability to be able to reset my password
in the event that I have forgotten it, so that I can regain access to my account.

✅ GOOD:
As a user, I want to reset my password so I can access my account when I forget it.
```

**Requirements**:

```markdown
❌ BAD:
The system shall provide functionality that enables users to perform the action
of uploading files to the designated storage location.

✅ GOOD:
Users can upload files to storage.
```

**Success Criteria**:

```markdown
❌ BAD:
The feature implementation shall be considered successful when users are able to
complete the checkout process in a timely manner with minimal friction.

✅ GOOD:
Users complete checkout in under 3 minutes.
```

**Acceptance Scenarios**:

```markdown
❌ BAD:
Given that a user has navigated to the login page and has entered their credentials
into the appropriate input fields, when they click on the submit button, then the
system should validate their credentials and redirect them to the dashboard.

✅ GOOD:
Given a user on the login page with valid credentials
When they click "Log in"
Then they see the dashboard
```

### Implementation Plans (plan.md)

**Technical Decisions**:

```markdown
❌ BAD:
After careful consideration of the various available options and taking into account
the specific requirements of this particular feature, we have determined that it would
be most appropriate to utilize PostgreSQL as the database solution.

✅ GOOD:
Database: PostgreSQL
Reason: Supports JSON queries needed for flexible product attributes.
```

**Architecture Notes**:

```markdown
❌ BAD:
The authentication mechanism will be implemented in such a way that it leverages
JWT tokens for the purpose of maintaining user session state across requests.

✅ GOOD:
Auth uses JWT tokens for session management.
```

**Dependencies**:

```markdown
❌ BAD:
This feature has a dependency on the completion of the user management module,
which is currently in the process of being developed by the backend team.

✅ GOOD:
Depends on: User management module (in progress)
```

### Task Lists (tasks.md)

**Task Descriptions**:

```markdown
❌ BAD:
- [ ] T001 [US1] Implement the necessary functionality to enable the creation of
  new user accounts in the system by building out the required service layer
  components in src/services/user_service.py

✅ GOOD:
- [ ] T001 [US1] Add user creation to src/services/user_service.py
```

**Phase Descriptions**:

```markdown
❌ BAD:
## Phase 3: Implementation of User Authentication Functionality
This phase encompasses the development of all components that are necessary
for the purpose of enabling users to authenticate themselves within the system.

✅ GOOD:
## Phase 3: User Authentication
Build login, logout, and session management.
```

**Acceptance Criteria in Tasks**:

```markdown
❌ BAD:
Acceptance: The user should be able to successfully complete the process of
logging into the system using their email address and password combination.

✅ GOOD:
Acceptance: User logs in with email and password.
```

## Checklist for Spec/Plan/Tasks Review

When reviewing content, check:

- [ ] Can a new team member understand this without asking questions?
- [ ] Is every sentence under 25 words?
- [ ] Did I use active voice?
- [ ] Are requirements testable as written?
- [ ] Are task descriptions actionable?
- [ ] Did I cut filler words?
- [ ] Is the main point in the first sentence?

## Common Violations in Specs

### Violation 1: Vague Requirements

```
❌ "The system should handle errors appropriately"
✅ "Show error message when login fails. Include reason (wrong password, account locked)."
```

### Violation 2: Passive Voice in User Stories

```
❌ "The password can be reset by the user"
✅ "Users can reset their password"
```

### Violation 3: Buried Actions in Tasks

```
❌ "In order to support the product listing feature, create the Product model"
✅ "Create Product model for product listing"
```

### Violation 4: Wordy Success Criteria

```
❌ "Users should be able to complete the registration process without experiencing any errors"
✅ "Users complete registration without errors"
```

### Violation 5: Abstract Acceptance Scenarios

```
❌ "The feature should work correctly under normal conditions"
✅ "User clicks 'Save' → sees 'Changes saved' message → data persists on refresh"
```

## Self-Check (Do This Before Finishing)

After writing the spec, plan, or tasks, read every sentence and ask:

1. **Would my non-technical friend understand this?**
2. **Can I say this with fewer words?**
3. **Is there a simpler word I can use?**

If the answer is "no" to any question, rewrite that sentence.

## AI Agent Requirements

- AI agents MUST apply Plain English rules BEFORE writing any spec content
- AI agents MUST check every sentence against the Forbidden Phrases table
- AI agents MUST run the Self-Check on all generated content before finalizing
- AI agents MUST rewrite any content that violates these principles
- AI agents MUST keep user stories under 20 words
- AI agents MUST keep task descriptions under 15 words (excluding file paths)
- AI agents MUST make requirements testable and specific
- AI agents MUST use active voice for all user actions
- AI agents MUST NOT use technical jargon without a plain English alternative

## Quick Reference

**For Specs**:
- User stories: Who + wants what + why (under 20 words)
- Requirements: Subject + verb + object (testable)
- Success criteria: Metric + target (measurable)

**For Plans**:
- Decisions: Choice + reason (one sentence each)
- Dependencies: What + status (no elaboration)

**For Tasks**:
- Format: Verb + object + location (under 15 words)
- One action per task
- Include file path

## Context

$ARGUMENTS
