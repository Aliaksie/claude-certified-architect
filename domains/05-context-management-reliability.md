# Domain 5: Context Management & Reliability (15%)

## Overview
This domain covers managing context window limitations, preserving data provenance, ensuring system reliability, and designing for production use. It includes context compression strategies, structured state persistence, error annotation, and data attribution practices.

**Key Learning Outcomes:**
- Manage context window limitations effectively
- Extract and preserve fact structures
- Trim unnecessary tool results
- Apply position-aware input principles
- Preserve data provenance and attribution
- Design for crash recovery and state persistence
- Annotate coverage gaps and partial results

---

## 1. Context Management in Production Systems

> Documentation: [Context Window Management](https://platform.claude.com/docs/en/build-with-claude/context-management)

### 1.1 Extract Facts into a Separate Block

Instead of relying on conversation history (which degrades during summarization), extract key facts into a structured block:

```
=== CASE FACTS (updated whenever a new fact appears) ===
Customer ID: CUST-12345
Order ID: ORD-67890
Order Date: 2025-01-15
Order Amount: $89.99
Issue: Damaged item on delivery
Customer Request: Full refund
Status: Pending manager approval
===
```

Include this block in every prompt, regardless of how history is summarized.

**Why this matters:**
- Summarization loses precision (dates, numeric values, specific details become vague)
- Structured facts prevent hallucination based on corrupted history
- Facts survive context compression intact

### 1.2 Trimming Tool Results

If `lookup_order` returns 40+ fields but you only need 5 for the current task:

```python
# PostToolUse hook: keep only relevant fields
@hook("PostToolUse", tool="lookup_order")
def trim_order_fields(result):
    return {
        "order_id": result["order_id"],
        "status": result["status"],
        "total": result["total"],
        "items": result["items"],
        "return_eligible": result["return_eligible"]
    }
```

This conserves context and reduces noise.

**Cost benefit:**
- Each tool call can return hundreds of fields
- Most are irrelevant to the current task
- Trimming can reduce context usage by 30-50% without losing critical information

### 1.3 Position-aware Input

Place critical information with the lost-in-the-middle effect in mind:

```
[KEY FINDINGS — at the top]
Found 3 critical vulnerabilities...

[DETAILED RESULTS — middle]
=== File auth.ts ===
...
=== File database.ts ===
...

[ACTION ITEMS — at the end]
Priority: fix auth.ts vulnerabilities before merge.
```

**Lost-in-the-middle effect:**
- Models reliably process information at the start and end of a long input
- Details in the middle can be missed, especially in dense context
- Place priorities, key decisions, and action items at the start or end

### 1.4 Scratchpad Files

In long investigations, the agent can write key findings to a scratchpad file:

```
# investigation-scratchpad.md
## Key findings
- PaymentProcessor in src/payments/processor.ts inherits from BaseProcessor
- refund() is called from 3 places: OrderController, AdminPanel, CronJob
- External PaymentGateway API has a rate limit of 100 req/min
- Migration #47 added refund_reason (NOT NULL) — 2024-12-01
```

When context degrades (or in a new session), the agent can consult the scratchpad instead of re-running discovery.

### 1.5 Delegating to Subagents to Protect Context

```
Main agent: "Investigate dependencies of the payments module"
  -> Subagent (Explore): reads 15 files, traces imports
  -> Returns: "Payments depends on AuthService, OrderModel, and the external PaymentGateway API"

Main agent: keeps one line in context instead of 15 files
```

**Separate context layer:**
In multi-agent systems, each subagent operates within a limited context budget—it receives only the information required for its task. The coordinator acts as a separate context layer: it aggregates subagent outputs, stores global state, and allocates context. This prevents "context leakage," where one agent consumes the window with information irrelevant to others.

**Constrained context budgets for subagents:**
- Send minimal context: a specific task + necessary data
- Instruct the subagent to return structured results, not raw data dumps
- Use `allowedTools` to limit the subagent's toolset—fewer tools means fewer distractions and lower context cost

### 1.6 Structured State Persistence (for crash recovery)

Each agent exports its state to a known location:

```json
// agent-state/web-search-agent.json
{
  "status": "completed",
  "queries_executed": ["AI music 2024", "AI music composition"],
  "results_count": 12,
  "key_findings": [...],
  "coverage": ["music composition", "music production"],
  "gaps": ["music distribution", "music licensing"]
}
```

The coordinator loads a manifest on resume:

```json
// agent-state/manifest.json
{
  "web-search": "completed",
  "doc-analysis": "in_progress",
  "synthesis": "not_started"
}
```

**Benefits:**
- On crash or network failure, resume from the last completed agent (not from the start)
- No duplicate work
- Coordinator knows which agents still need work

---

## 2. Preserving Provenance

### 2.1 The Attribution Loss Problem

When summarizing results from multiple sources, the "claim → source" link can be lost:

```
Bad: "The AI music market is estimated at $3.2B." (No source, no year.)

Good:
{
  "claim": "The AI music market is estimated at $3.2B.",
  "source_url": "https://example.com/report",
  "source_name": "Global AI Music Report 2024",
  "publication_date": "2024-06-15",
  "confidence": 0.9
}
```

**Why attribution matters:**
- Verifiability: readers can check the source
- Accountability: authors are credited
- Temporal context: dates matter for trending data
- Trust: readers assess source credibility

### 2.2 Handling Conflicting Data

When two sources provide different values:

```json
{
  "claim": "Share of AI-generated music on streaming platforms",
  "values": [
    {
      "value": "12%",
      "source": "Spotify Annual Report 2024",
      "date": "2024-03",
      "methodology": "Automated classification"
    },
    {
      "value": "8%",
      "source": "Music Industry Association Survey",
      "date": "2024-07",
      "methodology": "Survey of 500 labels"
    }
  ],
  "conflict_detected": true,
  "possible_explanation": "Difference in methodology and time period"
}
```

Do not arbitrarily choose one value. Preserve both with attribution and let the coordinator decide.

### 2.3 Include Dates for Correct Interpretation

Without dates, temporal differences can be misinterpreted as contradictions:

```
Bad: "Source A says 10%, source B says 15%. Contradiction."
Good: "Source A (2023) says 10%, source B (2024) says 15%. Likely +5% growth over a year."
```

### 2.4 Render by Content Type

Don't force everything into one format:
- Financial data -> tables
- News and analysis -> prose
- Technical findings -> structured lists
- Time series -> chronological ordering

---

## 3. Reliability Patterns

### 3.1 Partial Failure Handling

When a system can only partially complete a task, structure the response to indicate coverage:

```json
{
  "status": "partial_success",
  "sections": [
    {
      "name": "Introduction",
      "status": "complete",
      "content": "..."
    },
    {
      "name": "Methodology",
      "status": "complete",
      "content": "..."
    },
    {
      "name": "Results",
      "status": "partial",
      "content": "...",
      "coverage_percentage": 75,
      "gaps": ["Statistical significance testing", "Regression analysis"]
    },
    {
      "name": "Conclusion",
      "status": "not_started",
      "reason": "Awaiting results section completion"
    }
  ]
}
```

### 3.2 Coverage Annotations

Always annotate the scope and limitations of results:

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

### 3.3 Graceful Degradation

When a critical tool fails:
1. **Attempt alternative approaches** (different query, different data source)
2. **Use partial results** (annotated as incomplete)
3. **Inform the user** about gaps and limitations
4. **Suggest workarounds** (manual research, escalation, retry later)

---

## 4. System Context Window Fundamentals

### 4.1 Context Window Composition

The context window includes:
- The system prompt
- The full message history
- Tool definitions
- Tool results
- Any included resources or documents

**Total must stay within the model's limit:**
- `claude-opus-4-6`: 200K tokens
- `claude-sonnet-4-6`: 200K tokens
- `claude-haiku-4-5`: 100K tokens

### 4.2 Lost-in-the-Middle Effect

From research studies:
- Models reliably process the **first 20%** of a long input
- Models reliably process the **last 20%** of a long input
- The **middle 60%** is often missed or skimmed

**Mitigation strategies:**
1. **Place critical information at the start or end**
2. **Split very long inputs** (multiple requests or documents)
3. **Use explicit markers**: `### IMPORTANT: ...`
4. **Chunk by topic** rather than chronologically

### 4.3 Key Context-window Problems

1. **Lost-in-the-middle effect:** models miss details in long inputs
2. **Accumulation of tool results:** each tool call adds output; if tools return 40+ fields but only 5 matter, most context is wasted
3. **Progressive summarization:** when compressing history, numeric values, percentages, and dates often get lost and become vague ("about", "roughly", "a few")

---

## Key Takeaways

1. **Structured fact blocks** survive context compression and prevent hallucination
2. **Tool result trimming** can reduce context usage by 30-50%
3. **Position-aware input** compensates for the lost-in-the-middle effect
4. **Scratchpad files** allow recovery from context degradation
5. **Subagent delegation** creates separate context layers and prevents leakage
6. **State persistence** enables crash recovery without losing work
7. **Provenance preservation** maintains attribution and source credibility
8. **Coverage annotations** inform readers about gaps and limitations
9. **Partial failure handling** is better than aborting entire workflows

---

## Missing Content for Future Enhancement

- [ ] Context compression algorithms and techniques
- [ ] Real-time context budget tracking and alerting
- [ ] Cache eviction policies for tool results
- [ ] Streaming and pagination for large result sets
- [ ] Audit trail design for compliance (healthcare, finance)
- [ ] Multi-language context management
- [ ] Context optimization for different model sizes
- [ ] Historical context retrieval and reuse patterns
- [ ] Privacy-preserving context filtering

