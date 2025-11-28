# How to Write Good Specifications
## A Practical Guide to Requirements Documentation

---

# 🎯 What is a Specification?

A **specification** (spec) is a document that describes:
- **WHAT** the feature should do
- **WHY** it matters to users/business
- **WHO** will use it
- **WHEN** it's considered successful

### What a Spec is NOT:
- ❌ Technical implementation details
- ❌ Code architecture decisions
- ❌ Technology choices (languages, frameworks, databases)
- ❌ API designs or database schemas

---

# 🎪 Why User Stories & Acceptance Scenarios? (1/2)

### The Problem with Traditional Requirements:

❌ "The system shall validate user input"  
- Who is validating? Why? How do we test it?

### The Solution: User-Centered Format

✅ **User Story**: "As a user, I want to upload photos..."  
- Clear **who** and **why**

✅ **Acceptance Scenario**: "Given..., When..., Then..."  
- Clear **how to test**

---

# 🎪 Why User Stories & Acceptance Scenarios? (2/2)

### Benefits:

- **Shared understanding** between business and QA
- **Direct mapping** to automated tests
- **User focus** prevents building wrong features
- **Testable** from day one

---

# 👥 Who is Your Audience?

Specifications are written for **non-technical stakeholders**:

- Product managers
- Business analysts
- Designers
- Clients/customers
- Project sponsors
- QA testers (before implementation)

**Key Rule**: If a business person can't understand it, it's too technical!

---

# 🎨 The Golden Rule: WHAT vs. HOW

### ✅ WHAT (Specification Language)

- "Users can upload photos"
- "The system validates email addresses"
- "Customers receive order confirmation"
- "Data is kept secure"

### ❌ HOW (Implementation Language)

- "React component for file upload"
- "Use regex pattern for email validation"
- "Send email via SendGrid API"
- "Store passwords with bcrypt encryption"

**Remember**: Focus on USER OUTCOMES, not system internals!

---

# 📖 User Stories: The Foundation

User stories describe **who** needs the feature and **why** it matters.

### Standard Format:

> As a [user type],  
> I want to [action/capability],  
> So that [business value/benefit]

### ✅ Good Example (Photo Sharing)

> As a user, I want to upload photos and videos,  
> so that I can store and access my memories from any device.

**Notice**: Clear actor, clear action, clear value proposition!

---

# 🎯 Acceptance Scenarios: Given-When-Then

Acceptance scenarios make user stories **testable** using Given-When-Then format.

### Format:

> **Given** [initial context/state]  
> **When** [action occurs]  
> **Then** [expected outcome]

### Why This Works:

- **Given** = Sets up the test conditions
- **When** = Describes the user action
- **Then** = Defines success (testable!)

---

# ✅ Acceptance Scenario Examples

### Example 1: Photo Upload

**US1-AS1:**  
- **Given** I am a user with valid credentials,
- **When** I select a photo (JPG/PNG/HEIC, 50MB max) and click upload,
- **Then** the photo appears in my gallery within 5 seconds

### Example 2: Album Sharing

**US4-AS5:**  
- **Given** I have an album,
- **When** I share it with another user,
- **Then** they can view all current and future items I add

**Each scenario = One automated test!**

---

# 🔗 User Stories + Acceptance Scenarios (1/2)

A complete user story has:

1. **User Story** - The "what" and "why"
2. **Multiple Acceptance Scenarios** - The "how to verify"
3. **Priority** - Business importance (P1, P2, P3)
4. **Independent Test Statement** - Can it be tested alone?

---

# 🔗 User Stories + Acceptance Scenarios (2/2)

### Example Structure:

**User Story 1 - Upload Personal Media (Priority: P1)**

> As a user, I want to upload photos and videos...

**Acceptance Scenarios:**
- US1-AS1: Upload single photo → appears in gallery
- US1-AS2: Upload video → shows with thumbnail
- US1-AS3: View gallery → see all media by date
- US1-AS4: Click photo → opens full-screen viewer
- US1-AS5: Click video → plays with controls

---

# 📝 Types of Requirements in a Spec

A complete spec has different requirement types, each serving a purpose:

### 1. User Stories (US#)
**Purpose**: Describe WHO needs WHAT and WHY  
Format: *As a [user type], I want [capability], so that [value]*

### 2. Acceptance Scenarios (US#-AS#)
**Purpose**: Define HOW to test the user story  
Format: *Given [context], When [action], Then [outcome]*

### 3. Functional Requirements (FR-###)
**Purpose**: Specify WHAT the system MUST do  
Format: *System MUST [specific capability with details]*

---

# 📝 Types of Requirements (cont.)

### 4. Success Criteria (SC-###)
**Purpose**: Define HOW to measure success  
Format: *[X]% of users [achieve outcome] in [Y time]*

### 5. Edge Cases
**Purpose**: Handle unusual or error situations  
Format: *What happens when [unusual situation]?*

**All types must be**: Testable • Unambiguous • Technology-agnostic

---

# 📝 Functional Requirements Examples

Functional Requirements specify **WHAT the system MUST do**.

### ✅ Good Functional Requirements

- **FR-001**: System MUST accept photo uploads in JPG, PNG, HEIC up to 50MB
- **FR-015**: System MUST display gallery by upload date (newest first)
- **FR-019**: System MUST allow sharing photos by email address
- **FR-034**: System MUST allow delete with confirmation prompt

### ❌ Bad Functional Requirements

- "System should upload files quickly" (vague: how quickly?)
- "System must be fast" (unmeasurable)
- "System must provide good UX" (subjective)
- "System must be secure" (undefined: what does secure mean?)

**Test**: Can you verify this without knowing the code?

---

# 🎯 Success Criteria: The Gold Standard

Success criteria must be:

1. **Measurable** - Include specific metrics
2. **Technology-agnostic** - No implementation details
3. **User-focused** - Outcomes, not system internals
4. **Verifiable** - Testable without code knowledge

---

# ✅ Success Criteria Examples: The Good (1/2)

✓ **"Users complete checkout in under 3 minutes"**  
→ Measurable: 3 minutes  
→ Verifiable: time the process

✓ **"System supports 10,000 concurrent users"**  
→ Measurable: 10,000 users  
→ Verifiable: load testing

---

# ✅ Success Criteria Examples: The Good (2/2)

✓ **"95% of searches return results < 1 second"**  
→ Measurable: 95%, 1 second  
→ Verifiable: performance monitoring

✓ **"Task completion rate improves by 40%"**  
→ Measurable: 40% improvement  
→ Verifiable: analytics comparison

---

# ❌ Success Criteria Examples: The Bad (1/2)

✗ **"API response time is under 200ms"**  
→ Too technical (API)  
→ Better: "Users see results instantly"

✗ **"Database can handle 1000 TPS"**  
→ Implementation detail  
→ Better: "System processes 1000 orders/sec"

---

# ❌ Success Criteria Examples: The Bad (2/2)

✗ **"React components render efficiently"**  
→ Framework-specific  
→ Better: "Pages load in under 2 seconds"

✗ **"Redis cache hit rate above 80%"**  
→ Technology-specific  
→ Better: "Frequently accessed data loads instantly"

**Tip**: Replace tech terms with user outcomes!

---

# 🔗 How Everything Connects

### The Complete Picture:

**User Story** (The "Why")  
↓  
**Acceptance Scenarios** (The "How to Test")  
↓  
**Functional Requirements** (The "What System Must Do")  
↓  
**Success Criteria** (The "How to Measure Success")

### Example Flow:

1. **US3**: "I want to share photos with friends"
2. **US3-AS1**: "When I share, friend can view it"
3. **FR-019**: "System MUST allow sharing by email"
4. **SC-006**: "Shared media accessible < 10 sec"

**Each layer adds detail while staying technology-agnostic!**

---

# 🚫 Common Mistakes to Avoid (1/2)

### 1. Implementation Leakage
❌ "Use S3 and CloudFront for photo storage"  
✅ "System stores photos persistently with fast access"

### 2. Vague Requirements  
❌ "Fast upload times"  
✅ "Photos appear in gallery within 5 seconds"

### 3. Missing Edge Cases
❌ "Users can delete albums"  
✅ "Users can delete albums (unless shared); requires confirmation"

---

# 🚫 Common Mistakes to Avoid (2/2)

### 4. Subjective Language
❌ "Intuitive sharing experience"  
✅ "90% of users share successfully on first try"

### 5. Untestable Scenarios
❌ "When user uploads, then it works properly"  
✅ "When user uploads 50MB photo, then appears in gallery < 5 seconds"

---

# 📋 Specification Quality Checklist (1/3)

Before considering your spec complete, verify:

### Content Quality
- [ ] No implementation details (languages, frameworks, APIs, databases)
- [ ] Focused on user value and business needs
- [ ] Written for non-technical stakeholders
- [ ] All mandatory sections completed

---

# 📋 Specification Quality Checklist (2/3)

### User Stories & Scenarios
- [ ] Each user story has "As a... I want... so that..." format
- [ ] All acceptance scenarios use Given-When-Then format
- [ ] Each scenario is independently testable
- [ ] Scenarios are numbered (US#-AS#)
- [ ] Each "Then" statement is specific and measurable

---

# 📋 Specification Quality Checklist (3/3)

### Requirement Completeness
- [ ] All requirements testable and unambiguous
- [ ] Success criteria are measurable (include numbers!)
- [ ] Success criteria are technology-agnostic
- [ ] All edge cases identified
- [ ] Functional requirements numbered (FR-###)

---

# 🎓 Real-World Example: Bad → Good

### ❌ Original (Poor Spec)

**Feature: User System**

Build authentication with React and Node.js.  
Use JWT tokens and bcrypt for passwords.

**Requirements:**
- Fast login
- Secure storage
- Good UX

**Problems**: 
- All implementation details (React, Node.js, JWT, bcrypt)
- No user story or acceptance scenarios
- Unmeasurable (fast? good UX?)

---

# 🎓 Real-World Example: Bad → Good (cont.)

### ✅ Improved (Good Spec)

**User Story 2 - User Authentication (P1)**

> As a user, I want to create an account and log in,  
> so that my media is private and accessible only to me.

**Acceptance Scenarios:**

**US2-AS1:**
- **Given** I am a new user,
- **When** I provide email and password (8+ chars),
- **Then** my account is created and I'm logged in

---

# 🎓 Real-World Example: Bad → Good (cont. 2)

**US2-AS2:**
- **Given** I have an account,
- **When** I enter correct credentials,
- **Then** I'm logged into my media gallery

---

# 🎓 Real-World Example: Bad → Good (cont. 3)

**US2-AS3:**
- **Given** I am logged in,
- **When** I log out and another user logs in,
- **Then** they see only their own media

**US2-AS4:**
- **Given** I am not logged in,
- **When** I try to access upload/gallery,
- **Then** I am redirected to login page

---

# 🎓 Real-World Example: Bad → Good (cont. 4)

### ✅ Improved Spec (Continued)

**Functional Requirements:**
- FR-008: System allows account creation with email/password
- FR-009: System enforces password minimum (8 chars, letter + number)
- FR-010: System provides secure login
- FR-013: System isolates each user's media

---

# 🎓 Real-World Example: Bad → Good (cont. 5)

**Success Criteria:**
- SC-004: 95% complete account creation and first upload < 3 minutes
- SC-007: Zero unauthorized access incidents
- SC-008: 90% share media successfully on first attempt

**Edge Cases:**
- What happens with wrong password 3 times?
- What happens when session expires during upload?

**Better**: User-focused, testable, technology-agnostic!

---

# 🎯 Key Sections of a Good Spec (1/2)

### Mandatory Sections (Every Feature):

1. **User Stories** - As a [who], I want [what], so that [why]
2. **Acceptance Scenarios** - Given-When-Then test cases (US#-AS#)
3. **Functional Requirements** - Specific, testable "MUST" statements (FR-###)
4. **Success Criteria** - Measurable outcomes (SC-###)
5. **Edge Cases** - What happens when... ?

---

# 🎯 Key Sections of a Good Spec (2/2)

### Optional Sections (Include When Relevant):

6. **Key Entities** - Data objects involved
7. **Scope** - What's included/excluded?
8. **Assumptions** - What are we assuming?
9. **Dependencies** - What does it rely on?

**Tip**: Remove optional sections that don't apply!

---

# 💡 Pro Tips for Spec Writing (1/2)

### 1. One Acceptance Scenario = One Test
Each US#-AS# should map directly to an automated test case.

### 2. Think Like a Tester
Ask: "How would I verify this?"  
If unclear, it's not testable.

### 3. Use the "Explain to Your Boss" Test
If you can't explain it to a non-technical manager,  
it's too technical.

---

# 💡 Pro Tips for Spec Writing (2/2)

### 4. Be Specific with Numbers
"Fast" → "within 5 seconds"  
"Many users" → "1000 concurrent users"  
"Large files" → "50MB photos, 500MB videos"

### 5. Consider Edge Cases Early
"What happens when..." reveals spec gaps.

### 6. Number Everything
- User Stories: US1, US2, US3...
- Acceptance Scenarios: US1-AS1, US1-AS2...
- Functional Requirements: FR-001, FR-002...
- Success Criteria: SC-001, SC-002...

This makes traceability easy!

---

# ✅ Verification Requirements

Every acceptance scenario should be:

### The Four Pillars of Testability:

1. **Testable** - Can be demonstrated and verified
2. **Complete** - Verifies entire expected behavior
3. **Automated** - Can run repeatedly w/o manual work
4. **Independent** - Can be tested separately

### Example: US1-AS1

**Given** I am a user with valid credentials,  
**When** I select a photo (JPG/PNG/HEIC, 50MB max) and click upload,  
**Then** the photo appears in gallery < 5 seconds

**Automated Test**:  
✅ Create user → Upload test.jpg (10MB) →  
   Verify in gallery → Verify time < 5s

---

# 📚 Real-World Exercise

### Transform This Bad Requirement:

❌ "Build a photo sharing system using AWS S3,  
with React frontend and GraphQL API.  
Use Redis for caching and ensure sub-200ms."

### Your Turn:
- Write a user story (As a... I want... so that...)
- Create 2-3 acceptance scenarios (Given-When-Then)
- Add functional requirements (no tech!)
- Define success criteria (measurable)

---

# 📚 Exercise: Answer

### ✅ Good Version:

**User Story 3 - Share Media with Users (P2)**

> As a user, I want to share photos with other users,  
> so that I can share memories with friends.

**Acceptance Scenarios:**

**US3-AS1:**
- **Given** I have uploaded a photo,
- **When** I select "Share" and enter user's email,
- **Then** they receive notification and can view it in "Shared with me"

---

# 📚 Exercise: Answer (cont.)

**US3-AS2:**
- **Given** I have shared a photo with a user,
- **When** I select "Stop sharing",
- **Then** that user no longer has access

**US3-AS3:**
- **Given** Another user shared a photo with me,
- **When** I view it in "Shared with me",
- **Then** I can view but not delete or modify it

---

# 📚 Exercise: Answer (cont. 2)

**Functional Requirements:**
- FR-019: System allows sharing individual photos by email
- FR-020: System notifies users when media is shared
- FR-021: System provides "Shared with me" section
- FR-022: System allows revoking access anytime
- FR-023: System prevents recipients from deleting/modifying content

---

# 📚 Exercise: Answer (cont. 3)

**Success Criteria:**
- SC-006: Shared media accessible < 10 seconds
- SC-008: 90% share media successfully on first try

**Edge Cases:**
- What happens with invalid email?
- What happens when user is removed while viewing?

**Notice**: Complete user story with testable scenarios, no technology mentioned!

---

# 🎯 Summary: The Spec Writing Mindset (1/2)

### Always Remember:

1. Start with **User Stories** (As a... I want... so that...)
2. Make them testable with **Acceptance Scenarios** (Given-When-Then)
3. Write **WHAT** the feature does, not **HOW** it's built
4. Write for **business people**, not developers
5. Be **specific and measurable** (use numbers!)
6. Think **user outcomes**, not system internals
7. **Validate** every requirement is testable
8. When unclear, **make informed guesses** (document them!)

---

# 🎯 Summary: The Spec Writing Mindset (2/2)

### The Ultimate Test:

> "Can a non-technical stakeholder understand this spec  
> and know when the feature is done?"

If yes, you've written a good spec! 🎉

---

# ❓ Questions & Discussion (1/2)

**Q: How many acceptance scenarios per user story?**  
A: Typically 3-6. Cover happy path, errors, and key variations.

**Q: Do I need Given-When-Then for every scenario?**  
A: Yes! It ensures testability and clarity.  
If you can't write it, it's not testable.

**Q: How much detail is too much?**  
A: If it describes implementation (code, frameworks, APIs),  
it's too much.

**Q: What if requirements change?**  
A: Specs are living documents. Update as needed.

---

# ❓ Questions & Discussion (2/2)

**Q: How do I handle technical constraints?**  
A: Note in Dependencies or Assumptions,  
but keep requirements technology-agnostic.

**Q: What if I need 10 clarifications?**  
A: Make informed guesses for all but the 3 most critical.  
Document assumptions.

**Q: Can one scenario test multiple requirements?**  
A: Keep scenarios focused on one main behavior.  
It's okay to have many small scenarios!

---

# 📖 Resources & Next Steps (1/2)

### Practice Makes Perfect:

1. Review existing specs in your projects
2. Write user stories for each feature
3. Create Given-When-Then acceptance scenarios
4. Identify and remove implementation details
5. Make vague requirements specific and measurable
6. Number everything for traceability
7. Create a quality checklist and validate

---

# 📖 Resources & Next Steps (2/2)

### Key Takeaway:

> "A good specification enables everyone—from  
> business to QA—to know what success looks like,  
> without knowing a single line of code."
>
> "Every acceptance scenario becomes an automated test."

---

# 🤖 Doing Specs with AI in Cursor

### The Modern Approach

You can automate specification writing using **Spec-Kit** with AI in Cursor:

**Benefits:**
- Template-driven spec generation
- Consistent structure across projects
- AI-assisted requirement creation
- Built-in validation and quality checks

**Let's see how to set it up...**

---

# 🚀 Step 1a: Install Prerequisites

Before using Spec-Kit, install the required tools:

**Install Homebrew (if not already installed):**

`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`

**Install uv:**

`brew install uv`

---

# 🚀 Step 1b: Initialize Your Project

Create your new project with Spec-Kit:

**Command:**

`uvx --from git+https://github.com/github/spec-kit.git specify init my-awesome-project`

**Navigate to your project:**

`cd my-awesome-project`

**What this does:**
- Creates the base `.specify` folder
- Sets up standard spec-kit structure
- Provides templates for specifications

---

# 🚀 Step 2: Overlay Template Constitution

Overlay comprehensive constitution and templates:

**Commands:**

`git clone --depth 1 git@github.com:theplant/speckit-starter-go.git /tmp/speckit-starter-go-$$ && cp -r /tmp/speckit-starter-go-$$/.specify/* .specify/ && rm -rf /tmp/speckit-starter-go-$$`

**What this does:**
- Adds theplant specific templates
- Includes best practices and patterns
- Provides AI instructions for Cursor

---

# 🤖 Using AI to Write Specs (1/2)

Once set up, you can use Cursor AI to:

1. **Generate user stories** from feature descriptions
2. **Create acceptance scenarios** automatically
3. **Define functional requirements** with proper numbering
4. **Generate success criteria** that are measurable
5. **Identify edge cases** you might have missed

---

# 🤖 Using AI to Write Specs (2/2)

**Command in Cursor:**

`/speckit.specify feature description`

**Example:**

`/speckit.specify create photo video sharing system`

The AI will create a complete spec following all the principles we've learned!

**Your role**: Validate and refine the AI-generated spec

---

# ✅ AI-Generated Specs Follow Best Practices (1/2)

The spec-kit AI ensures:

- ✅ User stories use "As a... I want... so that..." format
- ✅ Acceptance scenarios use Given-When-Then
- ✅ Requirements are testable and unambiguous

---

# ✅ AI-Generated Specs Follow Best Practices (2/2)

The spec-kit AI ensures:

- ✅ Success criteria are measurable and technology-agnostic
- ✅ Edge cases are identified
- ✅ Everything is properly numbered (US#-AS#, FR-###, SC-###)

**The AI does the heavy lifting, you do the validation!**

---

# 🔄 The Complete Workflow

The Spec-Kit workflow has 4 phases:

1. **Setup** (One time)
2. **Create Specs** (Per feature)
3. **Validate** (Review)
4. **Iterate** (Refine)

Let's see each phase...

---

# 🔄 Phase 1: Setup (One Time)

**Install prerequisites:**
- Install Homebrew
- Install uv

**Initialize project:**
- Run Spec-Kit init command
- Overlay templates

**Done once per project!**

---

# 🔄 Phase 2: Create Specs (Per Feature)

**Run the command in Cursor:**

`/speckit.specify feature description`

**AI generates complete spec:**
- User Stories with priorities
- Acceptance Scenarios (Given-When-Then)
- Functional Requirements (FR-###)
- Success Criteria (SC-###)
- Edge Cases

**Takes minutes instead of hours!**

---

# 🔄 Phase 3: Validate (Review)

**Check the AI-generated spec:**

- ✅ User stories are clear and have business value
- ✅ Acceptance scenarios are testable
- ✅ Requirements are technology-agnostic
- ✅ Success criteria are measurable
- ✅ Edge cases are comprehensive

**Your expertise ensures quality!**

---

# 🔄 Phase 4: Iterate (Refine)

**Based on validation:**

- Update spec to address any gaps
- Clarify ambiguous requirements
- Add missing edge cases
- Re-run AI command if major changes needed

**Result**: High-quality, validated specification!

---

# Thank You! 🙏

**Remember**: Great specifications lead to great products!

Questions? Let's discuss!

---

# Appendix: Quick Reference Card (1/3)

## ✅ DO This:

- Start with user stories (As a... I want... so that...)
- Write acceptance scenarios in Given-When-Then format
- Number all elements (US1-AS1, FR-001, SC-001)
- Focus on user value and outcomes
- Use measurable, specific criteria (with numbers!)
- Write for non-technical readers
- Document assumptions
- Consider edge cases ("What happens when...")
- Make every requirement testable

---

# Appendix: Quick Reference Card (2/3)

## ❌ DON'T Do This:

- Mention frameworks, languages, or technologies
- Use vague terms (fast, good, secure, intuitive)
- Assume technical knowledge
- Leave requirements untestable
- Skip acceptance scenarios
- Write "Then it works" (too vague!)
- Over-clarify minor details

## 📏 Quality Bar:

Every requirement must answer:
- Can I test this? (Is it an acceptance scenario?)
- Can a non-developer understand this?
- Is this measuring user outcomes?
- Is this free of implementation details?
- Does it include specific numbers/thresholds?

**If all YES → Good requirement!** ✨

---

# Appendix: Quick Reference Card (3/3)

## 📋 Quick Format Reference:

**User Story:**  
As a [user], I want [action], so that [value]

**Acceptance Scenario:**  
Given [context], When [action], Then [outcome]

**Functional Requirement:**  
System MUST [specific capability]

**Success Criterion:**  
[X]% of users [achieve outcome] in [Y time]

**Edge Case:**  
What happens when [unusual situation]?
