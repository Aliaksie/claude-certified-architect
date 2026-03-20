# Domain 4: Prompt Engineering & Structured Output (20%)

## Overview
This domain covers advanced prompt engineering techniques, structured output validation, prompt chaining strategies, and techniques for obtaining reliable, accurate results from Claude. It includes few-shot prompting, validation loops, self-correction patterns, and batch processing for non-interactive workflows.

**Key Learning Outcomes:**
- Apply few-shot prompting effectively
- Design prompts with explicit criteria
- Implement prompt chaining for complex tasks
- Use validation and retry-with-feedback patterns
- Utilize batch API for asynchronous processing
- Achieve semantic accuracy beyond syntax validation

---

## 1. Prompt Engineering — Advanced Techniques

> Documentation: [Prompt Engineering](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview) | [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook)

### 1.1 Few-shot Prompting

Few-shot prompting is the inclusion of 2–4 input/output examples in a prompt to demonstrate the expected behavior.

**Why few-shot is more effective than textual descriptions:**
- A vague instruction like "be more precise" can be interpreted in many ways
- An example unambiguously shows the expected format and decision logic
- The model generalizes the pattern to new cases (it does not just repeat the examples)

### 1.2 Types of Few-shot Examples and When to Use Them

**Examples for ambiguous scenarios:**

```
Request: "My order is broken"
Action: Call get_customer -> lookup_order -> check status.
Rationale: "broken" may mean a damaged item; you need order details.

Request: "Get me a manager"
Action: Immediately call escalate_to_human.
Rationale: The customer explicitly requests a human. Do not attempt to solve autonomously.
```

**Examples for output formatting:**

```
Finding example:
{
  "location": "src/auth/login.ts:42",
  "issue": "SQL injection in the username parameter",
  "severity": "critical",
  "suggested_fix": "Use a parameterized query"
}
```

**Examples to separate acceptable vs problematic code:**

```
// Acceptable (do not flag):
const items = data.filter(x => x.active);

// Problem (flag):
const items = data.filter(x => x.active == true); // Use strict equality ===
```

**Examples for extraction from different document formats:**

```
Document with inline citations:
"As shown in the study (Smith, 2023), the rate is 42%."
-> {"value": "42%", "source": "Smith, 2023", "type": "inline_citation"}

Document with bibliography references:
"The rate is 42%. [1]"
-> {"value": "42%", "source": "reference_1", "type": "bibliography"}
```

**Examples for informal measurements:**

```
Text: "about two handfuls of rice"
-> {"amount": "~100g", "original_text": "two handfuls", "precision": "approximate"}

Text: "a pinch of salt"
-> {"amount": "~1g", "original_text": "a pinch", "precision": "approximate"}
```

Few-shot is especially effective for extracting informal and non-standard measurement units that are too diverse for purely rule-based instructions.

### 1.3 Format Normalization Rules in Prompts

When using strict JSON schemas for structured output, add normalization rules in the prompt:

```
Normalization:
- Dates: always ISO 8601 (YYYY-MM-DD); "yesterday" -> compute an absolute date
- Currency: numeric amount + currency code; "five bucks" -> {"amount": 5, "currency": "USD"}
- Percentages: decimal fraction; "half" -> 0.5
```

This prevents semantic errors where the JSON is syntactically valid but values are inconsistent.

---

## 2. Explicit Criteria vs Vague Instructions

### 2.1 Design Explicit Criteria

**Bad (vague):**

```
Check code comments for accuracy.
Be conservative—report only high-confidence findings.
```

**Good (explicit criteria):**

```
Flag a comment as problematic ONLY if:
1. The comment describes behavior that CONTRADICTS the actual code behavior
2. The comment references a non-existent function or variable
3. A TODO/FIXME comment refers to a bug that has already been fixed in code

Do NOT flag:
- Comments that are merely stylistically outdated
- Comments with minor wording inaccuracies
- Missing comments (that is a separate category)
```

### 2.2 Define Severity Criteria with Examples

```
CRITICAL: Runtime failure for users
  Example: NullPointerException while processing a payment

HIGH: Security vulnerability
  Example: SQL injection, XSS, missing authorization checks

MEDIUM: Logic bug without immediate impact
  Example: Wrong sorting, off-by-one error

LOW: Code quality
  Example: Duplication, suboptimal algorithm for small data
```

---

## 3. Prompt Chaining

### 3.1 Breaking Down Complex Tasks

Prompt chaining breaks a complex task into a sequence of focused steps:

```
Step 1: Analyze auth.ts (local issues only)
       -> Output: list of issues in auth.ts

Step 2: Analyze database.ts (local issues only)
       -> Output: list of issues in database.ts

Step 3: Integration pass (cross-file dependencies)
       -> Output: issues at module boundaries
```

**Why this matters:**
- Avoids **attention dilution**—when the model receives too many files at once, it may miss bugs in some files while providing shallow commentary on others
- Ensures consistent analysis quality per file
- Allows separate analysis of cross-file interactions

### 3.2 When to Use Prompt Chaining vs Dynamic Decomposition

- **Prompt chaining** — predictable, repeatable tasks (code review, file migrations)
- **Dynamic decomposition** — open-ended investigations where subtasks become clear only during execution

---

## 4. The "Interview" Pattern

### 4.1 Clarification Before Implementation

Before implementing a solution, Claude asks clarifying questions:

```
Claude: "Before implementing caching for the API, a few questions:
1. Which cache invalidation strategy do you prefer—TTL or event-based?
2. Is stale data acceptable when the cache is unavailable?
3. Should caching be per-user or global?
4. What is the expected data volume to cache?"
```

**When this is useful:**
- Unfamiliar domain (fintech, healthcare, legal systems)
- Tasks with non-obvious implications (cache strategies, failure modes)
- Multiple viable approaches where the best choice depends on context

---

## 5. Validation and Retry-with-Feedback

### 5.1 Validation Loop Pattern

When extracted data fails validation:

```
Step 1: Extract data from the document
Step 2: Validate (Pydantic, JSON Schema, business rules)
Step 3: If there's an error—retry with context:
  - The original document
  - The previous (incorrect) extraction
  - The specific error: "Field 'total' = 150, but sum(line_items) = 145. Re-check values."
```

### 5.2 When Retry Will Be Effective

- Format errors (date in the wrong format)
- Structural errors (a field placed in the wrong location)
- Arithmetic inconsistencies (the model can re-check)

### 5.3 When Retry Will NOT Help

- The information is absent from the source document
- The required context is external (the data is in another document not provided)
- Business logic errors—set `isRetryable: false`

### 5.4 Pydantic as a Validation Tool

Pydantic is a Python library for schema-based data validation. For the exam, the key points are:
- **Structural validation:** types, requiredness, enum constraints checked in code after receiving JSON from Claude
- **Semantic validation:** custom validators enforce business logic (sum of items equals total; start_date < end_date)
- **Validate–retry loops:** on Pydantic validation failure, construct an error message and re-prompt Claude with the error context
- **JSON Schema generation:** Pydantic models can generate JSON Schema for `tool_use`, providing a single source of truth

---

## 6. Self-correction

### 6.1 Detecting Internal Contradictions

A pattern for detecting internal contradictions:

```json
{
  "stated_total": "$150.00",
  "calculated_total": "$145.00",
  "conflict_detected": true,
  "line_items": [
    {"name": "Widget A", "price": 75.00},
    {"name": "Widget B", "price": 70.00}
  ]
}
```

The model extracts both the stated value and a computed value—if they differ, `conflict_detected` allows you to handle the discrepancy.

---

## 7. Message Batches API

> Documentation: [Message Batches](https://platform.claude.com/docs/en/build-with-claude/message-batches)

### 7.1 Overview

The Message Batches API lets you submit batches of requests for asynchronous processing:

| Attribute | Value |
|---|---|
| Savings | **50%** compared to synchronous calls |
| Processing window | Up to **24 hours** (no latency SLA guarantee) |
| Multi-turn tool calling | **Not supported** (one request = one response) |
| Correlation | `custom_id` field to link request and response |

### 7.2 When to Use Batch API vs Synchronous API

| Task | API | Why |
|---|---|---|
| Pre-merge PR check | **Synchronous** | The developer is waiting; 24 hours is unacceptable |
| Overnight tech-debt report | **Batch** | Result is needed by morning; 50% savings |
| Weekly security audit | **Batch** | Not urgent; 50% savings |
| Interactive code review | **Synchronous** | Immediate response required |
| Processing 10,000 documents | **Batch** | Bulk processing; savings are significant |

### 7.3 Using `custom_id`

```json
{
  "custom_id": "doc-invoice-2024-001",
  "params": {
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Extract data from: ..."}]
  }
}
```

`custom_id` allows you to:
- Link the result to the original document
- On failure, re-submit only the failed documents
- Avoid re-processing successful documents

### 7.4 Handling Failures in Batches

1. Submit a batch of 100 documents
2. 95 succeed; 5 fail (context limit exceeded)
3. Identify failures by `custom_id`
4. Modify strategy (e.g., split long documents into chunks)
5. Re-submit only the 5 failed documents

### 7.5 SLA Planning

If you need a result in 30 hours and the Batch API can take up to 24 hours:
- Submission window: 30 - 24 = **6 hours**
- Batches must be submitted no later than 24 hours before the deadline
- For frequent submissions, split into 4-hour windows

---

## 8. Extended Thinking

> Documentation: [Extended Thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

Extended Thinking allows the model to reason internally before providing an answer. This is particularly useful for:
- Complex reasoning tasks
- Multi-step problem solving
- Verification of correctness before output

**Trade-off:** Extended Thinking uses more tokens and is slower but provides higher-quality reasoning for complex problems.

---

## Key Takeaways

1. **Few-shot prompting** is more effective than textual descriptions for demonstrating expected behavior
2. **Explicit criteria** with examples prevent ambiguity in task specifications
3. **Prompt chaining** avoids attention dilution by breaking large tasks into focused steps
4. **Validation-retry loops** fix format and structural errors but cannot recover missing information
5. **Pydantic** provides structural and semantic validation with a single source of truth
6. **Batch API** provides 50% cost savings for non-urgent, large-scale processing
7. **Self-correction** patterns detect conflicts between stated and computed values
8. **Extended Thinking** improves reasoning quality at the cost of tokens and latency

---

## Missing Content for Future Enhancement

- [ ] Prompt optimization techniques (cost reduction, latency improvement)
- [ ] A/B testing methodologies for prompt variants
- [ ] Model-specific prompt adaptations
- [ ] Chain-of-thought vs tree-of-thought vs other reasoning patterns
- [ ] Handling ambiguous user input and disambiguation strategies
- [ ] Multi-language prompt engineering
- [ ] Bias detection and mitigation in prompts
- [ ] Real-time vs batch processing decision frameworks

