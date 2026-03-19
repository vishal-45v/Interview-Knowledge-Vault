# Contributing Guide

Thank you for contributing to the Interview Knowledge Vault. This guide explains how to add questions, improve answers, add diagrams, and expand sections.

---

## Table of Contents

1. [Types of Contributions](#types-of-contributions)
2. [File Structure Conventions](#file-structure-conventions)
3. [Adding Questions](#adding-questions)
4. [Improving Answers](#improving-answers)
5. [Adding Diagrams](#adding-diagrams)
6. [Adding Production Scenarios](#adding-production-scenarios)
7. [Code Examples](#code-examples)
8. [Quality Standards](#quality-standards)
9. [Pull Request Process](#pull-request-process)

---

## Types of Contributions

### High Priority
- New scenario-based questions with detailed answers
- Production failure case studies
- ASCII diagrams for complex architectural patterns
- Java code examples illustrating concepts
- Corrections to inaccurate content

### Medium Priority
- Additional analogy explanations
- New debugging scenarios
- Extended follow-up trap questions
- Improvements to clarity of existing answers

### Lower Priority
- Grammar and formatting fixes
- Additional resource links
- Typo corrections

---

## File Structure Conventions

### Core Java Chapters
Each chapter directory must contain exactly these 6 files:

```
chapter-name/
├── theory-questions.md      # Conceptual questions about the topic
├── scenario-questions.md    # Real-world scenario-based questions
├── follow-up-traps.md       # Trick/follow-up questions interviewers ask
├── structured-answers.md    # Detailed answers with explanations
├── analogy-explanations.md  # Simple analogies for complex concepts
└── diagram-explanations.md  # ASCII diagrams and visual explanations
```

### Spring Boot Subfolders
Each Spring Boot subfolder must contain:

```
subfolder/
├── scenarios.md             # Interview scenarios
├── answers.md               # Detailed answers
├── architecture-notes.md    # How the feature works internally
├── trap-questions.md        # Follow-up and trap questions
└── diagram-explanations.md  # Visual explanations
```

---

## Adding Questions

### Theory Questions Format

```markdown
## Q{N}: {Question title in sentence case}

**Question:** {The full question text}

**Difficulty:** {Beginner | Intermediate | Advanced | Expert}

**Topics covered:** {comma-separated list of concepts}

**Answer:**

{Detailed answer — minimum 3 paragraphs for intermediate/advanced questions}

**Key Points:**
- {Point 1}
- {Point 2}
- {Point 3}

**Follow-up questions:**
- {Follow-up 1}
- {Follow-up 2}
```

### Scenario Questions Format

```markdown
## Scenario {N}: {Scenario title}

**Context:** {Background — describe the system, team, or code situation}

**Problem:** {What specific issue or challenge is being presented}

**Question:** {What the interviewer asks}

**Expected Approach:**
1. {Step 1 — what the candidate should identify first}
2. {Step 2}
3. {Step 3}

**Ideal Answer:**

{Full detailed answer with code examples where relevant}

**What interviewers look for:**
- {Criterion 1}
- {Criterion 2}

**Common mistakes candidates make:**
- {Mistake 1}
- {Mistake 2}
```

### Numbering Convention
- Questions within a file are numbered sequentially starting at 1
- New questions append to the end of the file
- Do not renumber existing questions when inserting

---

## Improving Answers

When improving an existing answer:

1. **Do not replace** the original — add a "Deeper Explanation" section below it
2. **Cite the Java version** when behavior differs between versions (e.g., Java 8 vs Java 17)
3. **Add code examples** to any answer that lacks them and would benefit from one
4. **Mark deprecated behavior** clearly with `> **Deprecated since Java X:** ...`

### Answer Quality Checklist
- [ ] Answer addresses the exact question asked
- [ ] Answer is technically accurate as of Java 17+
- [ ] Code examples compile and run correctly
- [ ] Edge cases and gotchas are mentioned
- [ ] Related concepts are cross-referenced

---

## Adding Diagrams

### ASCII Diagram Standards

All diagrams must use consistent box-drawing characters:

```
┌─────────────────────┐
│   Component Name    │
│                     │
│  ┌───────────────┐  │
│  │  Sub-component│  │
│  └───────────────┘  │
└─────────────────────┘
```

Arrows:
- Horizontal flow: `──────►`
- Vertical flow: `│` with `▼` or `▲`
- Bidirectional: `◄──────►`

### Diagram File Format

```markdown
## {Diagram Title}

**Purpose:** {One sentence explaining what this diagram shows}

**When to use this diagram in interviews:** {Context}

```
{ASCII diagram here}
```

**Key components:**
1. **{Component 1}:** {Explanation}
2. **{Component 2}:** {Explanation}

**How to explain this in an interview:**
> "{Example verbal explanation}"
```

### Diagram Categories
Place diagrams in the appropriate file:
- `diagrams/system-design-patterns.md` — load balancers, API gateways, service meshes
- `diagrams/distributed-systems-patterns.md` — message queues, event-driven, CQRS, Saga
- `diagrams/backend-architecture-patterns.md` — monolith vs microservices, layered, hexagonal
- `diagrams/database-scaling-patterns.md` — replication, sharding, caching tiers

---

## Adding Production Scenarios

Production scenarios in sections 07 and 08 must follow this template:

```markdown
## Scenario: {Title}

### Situation
{2–3 sentences describing the system and what happened}

### Symptoms
- {Observable symptom 1}
- {Observable symptom 2}
- {Alert that fired}

### Investigation Steps
1. **Check {tool/metric}:** {What to look for and why}
2. **Examine {log/trace}:** {What patterns indicate this issue}
3. **Narrow down:** {How to confirm the root cause}

### Root Cause
{Clear explanation of the underlying technical cause}

### Fix
**Immediate mitigation:**
{Quick action to stop the bleeding}

**Permanent fix:**
{Code or configuration change to prevent recurrence}

**Code example (if applicable):**
```java
// Before (problematic)
...

// After (fixed)
...
```

### Post-Mortem Notes
- **What went wrong:** {Brief summary}
- **Contributing factors:** {Environment, process, or code factors}
- **Prevention:** {What monitoring, testing, or process would catch this earlier}
- **Related issues to watch for:** {Similar failure modes}
```

---

## Code Examples

### Java Code Standards

All Java code examples should:
1. Target Java 11+ unless demonstrating legacy behavior
2. Use meaningful variable names
3. Include comments explaining non-obvious behavior
4. Show imports when the class source is not obvious
5. Be formatted consistently

```java
// Good example — well-named, commented, shows the concept clearly
public class UserService {
    private final UserRepository repository;

    // Constructor injection preferred over field injection
    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public Optional<User> findById(Long id) {
        // readOnly = true optimizes: no dirty checking, Hibernate flush mode NEVER
        return repository.findById(id);
    }
}
```

### Code Block Language Tags

Always specify the language:
- ` ```java ` for Java code
- ` ```sql ` for SQL queries
- ` ```yaml ` for YAML configuration
- ` ```bash ` for shell commands
- ` ```dockerfile ` for Dockerfile content
- ` ```json ` for JSON

---

## Quality Standards

### Before Submitting

Every question must meet these standards:

1. **Accuracy:** Technically correct for the stated Java/Spring version
2. **Completeness:** Answers cover edge cases, not just the happy path
3. **Depth:** Senior-level answers — not surface-level definitions
4. **Clarity:** A competent engineer can understand without external references
5. **Uniqueness:** Not a duplicate of an existing question

### Questions to Avoid
- Questions with trivially-Googleable answers (e.g., "What does System.out.println do?")
- Opinion questions without technical substance
- Questions about deprecated APIs (unless demonstrating evolution)
- Questions already well-covered in the existing files

---

## Pull Request Process

1. **Fork** the repository
2. **Create a branch** named `add/{topic-name}` or `fix/{issue-description}`
3. **Make changes** following the conventions above
4. **Validate** your content with the quality checklist
5. **Submit PR** with:
   - Title: `Add: {N} new questions for {section}`
   - Body: List of files changed and question count added
6. **Respond** to review comments within 48 hours

### PR Review Criteria
Reviewers will check:
- Technical accuracy
- Format consistency
- Question count per file meets minimums
- No placeholder or TODO content
- Code examples are syntactically correct
