# [FEATURE NAME]

## 1. Purpose (WHY at feature-level)

- [Why this feature exists - 2-3 bullet points]
- [What user/business problem it solves]
- [How it fits into the larger system context]

## 2. Goals (WHAT)

- [Goal 1: Specific, measurable objective]
- [Goal 2: What users/integrators should be able to accomplish]
- [Goal 3: System capabilities this feature provides]

## 3. Non-Goals (WHAT NOT)

- [Explicitly out of scope item 1]
- [Explicitly out of scope item 2]
- [What this feature will NOT do or handle]

## 4. High-Level Flow

1. [Step 1: Simple English explanation of how this feature works]
2. [Step 2: Key interaction or process]
3. [Step 3: Expected outcome]

## 5. User Story

### 5.1 [Persona 1 - e.g., Integrator, Admin, Editor]

- As a **[Persona]**, I can **[action/capability]**, so that **[benefit/outcome]**
- As a **[Persona]**, I can **[action/capability]**, so that **[benefit/outcome]**

### 5.2 [Persona 2]

- As a **[Persona]**, I can **[action/capability]**, so that **[benefit/outcome]**

### 5.3 [Persona 3]

- As a **[Persona]**, I can **[action/capability]**, so that **[benefit/outcome]**

## 6. Acceptance Criteria

### 6.1 [Feature Area 1]

#### Scenario: [Specific scenario name]
Given [initial state/precondition]
When [action is performed]
Then [expected outcome]
And [additional expected behavior]

#### Scenario: [Another scenario]
Given [initial state]
When [action]
Then [expected result]

### 6.2 [Feature Area 2]

#### Scenario: [Specific scenario name]
Given [context]
When [action]
Then [outcome]

### 6.3 Edge Cases & Error Handling

#### Scenario: [Edge case description]
Given [unusual or boundary condition]
When [action]
Then [how system handles it]

## 7. Data Model Impact

### 7.1 [New Entity Name]

[Description of the entity]

Fields:
- `field_name`: type, description, constraints
- `field_name`: type, description, constraints

### 7.2 [Modified Entity]

Changes to existing entity:
- Add: `new_field`: type, description
- Modify: `existing_field`: [describe change]

### 7.3 Relationships

- [Describe relationships between entities]
- [Foreign keys, join tables, constraints]

### 7.4 Constraints

- [Unique constraints]
- [Immutable fields]
- [Validation rules]

### 7.5 Migration Considerations

- [How existing data is affected]
- [Migration strategy if needed]

## 8. API Impact

### 8.1 New APIs

#### [HTTP Method] /api/path/to/endpoint
- **Purpose**: [What this endpoint does]
- **Request**: [Request body structure]
- **Response**: [Response structure]
- **Validation**: [Validation rules]
- **Errors**: [Possible error responses]

#### [HTTP Method] /api/another/endpoint
- **Purpose**: [Description]
- **Request**: [Structure]
- **Response**: [Structure]

### 8.2 Modified APIs

#### [Existing endpoint that changes]
- **Changes**: [What's being modified]
- **Breaking**: [Yes/No - if breaking, explain]
- **Migration**: [How clients should adapt]

### 8.3 [Other System] Impact

- [How this feature affects Search/OMS/Inventory/etc.]
- [New data contracts or expectations]

## 9. Decision Points

### 9.1 [Decision topic/question]

- **Option A**: [First approach]
  - Pros: [Advantages]
  - Cons: [Disadvantages]

- **Option B**: [Alternative approach]
  - Pros: [Advantages]
  - Cons: [Disadvantages]

- **Decision**: [Chosen option]
- **Rationale**: [Why this choice was made]

### 9.2 [Another decision]

- **Option A**: [Approach]
- **Option B**: [Alternative]
- **Decision**: [Choice]
- **Rationale**: [Reasoning]

## 10. Open Questions

- [Unresolved question or pending decision]
  - Current approach: [How we're handling it now]
  - Alternative: [Other possible approach]
  - Recommendation: [Suggested path forward]

- [Another open question]
  - Context: [Background]
  - Options: [Possible solutions]
  - Recommendation: [Suggestion]
