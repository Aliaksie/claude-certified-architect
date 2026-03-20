# Domain 1: Agentic Architecture & Orchestration (27%)

## Overview
This domain covers the design and implementation of autonomous agent systems that can execute complex tasks through iterative decision-making and tool interaction. It represents the core of Claude Certified Architect foundations, covering agent loops, multi-agent patterns, error handling, and human-in-the-loop systems.

**Key Learning Outcomes:**
- Design and implement agentic loops
- Build multi-agent systems with coordinator patterns
- Handle agent errors and recovery
- Implement escalation and human oversight
- Execute complex task decomposition

---

## 1. Fundamentals of the Agentic Loop

### 1.1 What is an Agentic Loop

The agentic loop is the core pattern for autonomous task execution. The model doesn't just answer—it performs a sequence of actions:

```
1. Send a request to Claude with tools
2. Receive a response
3. Check stop_reason:
   - "tool_use" -> execute the tool, append the result to history, go back to step 1
   - "end_turn" -> the task is complete, show the result to the user
4. Repeat until completion
```

**This is a model-driven approach:** Claude decides which tool to call next based on context and prior tool results. This differs from hard-coded decision trees where the action sequence is fixed.

### 1.2 Anti-patterns (avoid)

- Parsing assistant text to detect completion ("Task completed")
- Using an arbitrary iteration limit (e.g., `max_iterations=5`) as the primary stop condition
- Checking whether the assistant produced textual content as a completion signal

**Correct approach:** the only reliable completion signal is `stop_reason == "end_turn"`.

---

## 2. Claude Agent SDK — Building Agentic Systems

> Documentation: [Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) | [Hooks](https://platform.claude.com/docs/en/agent-sdk/hooks) | [Subagents](https://platform.claude.com/docs/en/agent-sdk/subagents) | [Sessions](https://platform.claude.com/docs/en/agent-sdk/sessions)

### 2.1 `AgentDefinition` Configuration

`AgentDefinition` is the agent configuration object in the Claude Agent SDK:

```python
agent = AgentDefinition(
    name="customer_support",
    description="Handles customer requests for returns and order issues",
    system_prompt="You are a customer support agent...",
    allowed_tools=["get_customer", "lookup_order", "process_refund", "escalate_to_human"],
    # For a coordinator:
    # allowed_tools=["Task", "get_customer", ...]
)
```

**Key parameters:**
- `name` / `description` — identification and description of the agent
- `system_prompt` — system prompt with instructions
- `allowed_tools` — list of allowed tools (principle of least privilege)

### 2.2 Hub-and-Spoke: Coordinator and Subagents

A multi-agent architecture is typically built as a hub-and-spoke topology:

```
         Coordinator
        /     |      \
   Subagent1  Subagent2  Subagent3
    (search)   (analysis)   (synthesis)
```

**The coordinator is responsible for:**
- Decomposing the task into subtasks
- Deciding which subagents are needed (dynamic selection)
- Delegating work to subagents
- Aggregating and validating results
- Handling errors and retries
- Communicating results to the user

**Critical principle: subagents have isolated context.**
- Subagents do **not** automatically inherit the coordinator's conversation history
- All required context must be **explicitly passed** in the subagent prompt
- Subagents do not share memory across calls
- All communication flows through the coordinator (for observability and error control)

### 2.3 The `Task` Tool for Spawning Subagents

Subagents are spawned via the `Task` tool:

```python
# The coordinator's allowedTools must include "Task"
coordinator_agent = AgentDefinition(
    allowed_tools=["Task", "get_customer"]
)
```

**Explicit context passing is mandatory:**

```
# Bad: the subagent has no context
Task: "Analyze the document"

# Good: full context in the prompt
Task: "Analyze the following document.
Document: [full document text]
Prior search results: [web search results]
Output format requirements: [schema]"
```

**Parallel spawning:** a coordinator can call multiple `Task`s in one response—subagents run in parallel:

```
# One coordinator response contains:
Task 1: "Search for articles about X"
Task 2: "Analyze document Y"
Task 3: "Search for articles about Z"
# All three run concurrently
```

### 2.4 Hooks in the Agent SDK

Hooks allow interception and transformation at specific points in the agent lifecycle.

**PostToolUse** intercepts a tool result before it is provided to the model:

```python
# Example: normalize date formats from different MCP tools
@hook("PostToolUse")
def normalize_dates(tool_result):
    # Convert Unix timestamp -> ISO 8601
    # Convert "Mar 5, 2025" -> "2025-03-05"
    return normalized_result
```

**Outgoing-call interception hook** blocks actions that violate policy:

```python
# Example: block refunds above $500
@hook("PreToolUse")
def enforce_refund_limit(tool_call):
    if tool_call.name == "process_refund" and tool_call.args.amount > 500:
        return redirect_to_escalation(tool_call)
```

**Key difference: hooks vs prompt instructions**

| Attribute | Hooks | Prompt instructions |
|---|---|---|
| Guarantee | **Deterministic** (100%) | **Probabilistic** (>90%, not 100%) |
| When to use | Critical business rules, financial operations, compliance | General preferences, recommendations, formatting |
| Example | Block refunds > $500 | "Try to solve before escalating" |

**Rule:** when failure has financial, legal, or safety consequences—use hooks, not prompts.

### 2.5 `fork_session` and Session Management

**`--resume <session-name>`** resumes a named session:

```bash
claude --resume investigation-auth-bug
```

- Continues a prior conversation with saved context
- Useful for long investigations across multiple sessions
- Risk: if files changed since the prior session, tool results may be stale

**`fork_session`** creates an independent branch from shared context:

```
Codebase investigation
         |
    fork_session
    /           \
Approach A:      Approach B:
Redux            Context API
```

- Both forks inherit context up to the branch point
- Afterwards, they diverge independently
- Useful for comparing approaches or testing strategies

**When to start a new session instead of resuming:**
- Tool results are stale (files changed)
- A lot of time has passed and context has degraded
- It is better to restart with "Here is a short summary of what we found: ..." than to resume with old tool data

---

## 3. Task Decomposition Strategies

### 3.1 Fixed Pipelines (Prompt Chaining)

Each step is defined in advance:

```
Document -> Metadata extraction -> Data extraction -> Validation -> Enrichment -> Final output
```

**When to use:**
- The task structure is predictable (reviews always follow the same template)
- All steps are known up front
- You need stability and reproducibility

### 3.2 Dynamic Adaptive Decomposition

Subtasks are generated based on intermediate results:

```
1. "Add tests for a legacy codebase"
2. -> First: map the structure (Glob, Grep)
3. -> Found: 3 modules with no tests, 2 with partial coverage
4. -> Prioritize: start with the payments module (high risk)
5. -> During work: discovered a dependency on an external API
6. -> Adapt: add a mock for the external API before writing tests
```

**When to use:**
- Open-ended investigative tasks
- When the full scope is unknown up front
- When each step depends on the results of the previous step

### 3.3 Multi-pass Code Review

For pull requests with 10+ files:

```
Pass 1 (per-file): Analyze auth.ts -> list local issues
Pass 1 (per-file): Analyze database.ts -> list local issues
Pass 1 (per-file): Analyze routes.ts -> list local issues
...
Pass 2 (integration): Analyze relationships between files
  -> Cross-file issues: inconsistent types, circular dependencies
```

**Why a single pass over 14 files is bad:**
- Attention dilution: deep analysis for some files, shallow for others
- Inconsistent comments: a pattern is flagged in one file but approved in another
- Missed bugs: obvious errors get skipped due to cognitive overload

---

## 4. Escalation and Human-in-the-Loop

### 4.1 When to Escalate to a Human

**Escalation triggers (clear rules):**

| Situation | Action |
|---|---|
| The customer explicitly asks "get me a manager" | Escalate immediately; do not attempt to solve |
| Policy does not cover the request | Escalate (e.g., competitor price matching when policy is silent) |
| The agent cannot make progress | Escalate after a reasonable number of attempts |
| Financial operation above a threshold | Escalate (preferably enforced via a hook, not a prompt) |
| Multiple matches when searching for a customer | Ask for additional identifiers; do not guess |

**What is NOT a reliable trigger:**

| Unreliable method | Why it fails |
|---|---|
| Sentiment analysis | Customer mood does not correlate with case complexity |
| Model self-rated confidence (1–10) | The model can be confidently wrong; calibration is poor |
| An automatic classifier | Overengineering; may require training data you don't have |

### 4.2 Escalation Patterns

**Immediate escalation:**

```
Customer: "I want to speak to a manager"
Agent: [immediately calls escalate_to_human]
NOT: "I can help with your issue, let me..."
```

**Escalation after an attempt to resolve:**

```
Customer: "My refrigerator broke two days after purchase"
Agent: [checks the order, offers a warranty replacement]
If the customer is not satisfied -> escalate
```

**Nuanced escalation (acknowledge → resolve → escalate on reiteration):**

```
Customer: "This is outrageous, I'm very unhappy with the quality!"
Agent: [acknowledges frustration] "I understand your frustration."
       [offers resolution] "I can offer a replacement or a refund."
Customer: "No, I want to talk to someone!"
Agent: [customer insists again -> immediate escalation]
```

Key principle: acknowledge emotion first, then propose a concrete solution, and only escalate if the customer reiterates the desire for a human. Do not escalate on the first expression of dissatisfaction (that is not the same as requesting a manager).

**Escalation for a policy gap:**

```
Customer: "Competitor X has this item 30% cheaper—give me a discount"
Policy: covers price adjustments only on your own site
Agent: [escalates — policy does not cover competitor price matching]
```

### 4.3 Structured Handoff Protocols

On escalation, the agent should pass a structured summary to a human:

```json
{
  "customer_id": "CUST-12345",
  "customer_name": "Ivan Petrov",
  "issue_summary": "Refund request for a damaged item",
  "order_id": "ORD-67890",
  "root_cause": "Item arrived damaged; photos attached",
  "actions_taken": [
    "Verified customer via get_customer",
    "Confirmed order via lookup_order",
    "Offered a standard replacement — customer insists on a refund"
  ],
  "refund_amount": "$89.99",
  "recommended_action": "Approve a full refund",
  "escalation_reason": "Customer requested to speak with a manager"
}
```

The human operator does not have access to the full conversation transcript—they only see this summary. Therefore it must be complete and self-contained.

### 4.4 Confidence Calibration and Human Oversight

For data extraction systems:

1. **Field-level confidence scores:** the model outputs a confidence score per extracted field
2. **Calibration:** use labeled validation sets to tune thresholds
3. **Routing:**
    - High confidence + stable accuracy -> automated processing
    - Low confidence or ambiguous sources -> human review

**Stratified random sampling:**
- Even for high-confidence extractions, regularly audit a sample
- An aggregate 97% accuracy can hide 40% errors for a particular document type
- Analyze accuracy by document type and by field, not only overall

---

## 5. Error Handling in Multi-agent Systems

### 5.1 Error Categories

| Category | Examples | Retryable | Agent action |
|---|---|---|---|
| **Transient** | Timeout, 503, network failure | Yes | Retry with exponential backoff |
| **Validation** | Invalid input format, missing required field | No (fix input) | Modify request and retry |
| **Business** | Policy violation, threshold exceeded | No | Explain to the user; propose an alternative |
| **Permission** | Access denied | No | Escalate |

### 5.2 Error-handling Anti-patterns

| Anti-pattern | Problem | Correct approach |
|---|---|---|
| Generic status "search unavailable" | The coordinator can't decide how to recover | Return error type, query, partial results, alternatives |
| Silent suppression (empty result = success) | Coordinator thinks there were no matches, but it was a failure | Distinguish "no results" from "search failure" |
| Aborting the whole workflow on one failure | You lose all partial results | Continue with partial results; annotate gaps |
| Infinite retries inside a subagent | Latency and wasted resources | Local recovery (1–2 retries), then propagate to coordinator |

### 5.3 A Structured Subagent Error

```json
{
  "status": "partial_failure",
  "failure_type": "timeout",
  "attempted_query": "AI impact on music industry 2024",
  "partial_results": [
    {"title": "AI Music Generation Report", "url": "...", "relevance": 0.8}
  ],
  "alternative_approaches": [
    "Try a narrower query: 'AI music composition tools'",
    "Use an alternative data source"
  ],
  "coverage_impact": "Not covered: AI impact on music production"
}
```

This provides the coordinator with the information needed to decide:
- Retry with a modified query?
- Use partial results?
- Delegate to a different subagent?
- Continue without this section and annotate the gap?

### 5.4 Coverage Annotations in the Final Synthesis

```markdown
## Report: AI Impact on Creative Industries

### Visual Art (FULL COVERAGE)
[research results]

### Music (PARTIAL COVERAGE — search agent timeout)
[partial results]
⚠️ Note: coverage for this section is limited due to a timeout in the search agent.

### Literature (FULL COVERAGE)
[research results]
```

---

## Key Takeaways

1. **Agentic loops** are model-driven: the only reliable stop signal is `stop_reason == "end_turn"`
2. **Multi-agent systems** use coordinator patterns with explicit context passing to subagents
3. **Hooks** enforce deterministic compliance; prompts are probabilistic
4. **Task decomposition** can be fixed (pipelines) or dynamic (adaptive)
5. **Escalation** requires clear triggers and structured handoff information
6. **Error handling** must distinguish transient (retryable) from business (non-retryable) errors
7. **Multi-pass approaches** improve quality for complex tasks (code review, analysis)

---

## Missing Content for Future Enhancement

- [ ] Multi-agent coordination patterns (service mesh, event-driven)
- [ ] Agent monitoring and observability metrics
- [ ] Error recovery strategies beyond retry-with-feedback
- [ ] Performance optimization for coordinator patterns
- [ ] Testing strategies for agentic systems
- [ ] Production deployment patterns for multi-agent workflows

