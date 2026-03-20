# Domain 2: Tool Design & MCP Integration (18%)

## Overview
This domain focuses on designing effective tools for agent interaction and integrating external systems through the Model Context Protocol (MCP). It covers tool definition best practices, structured output with JSON schemas, MCP server configuration, and resource management.

**Key Learning Outcomes:**
- Design effective tool definitions
- Understand tool_use and tool_choice parameters
- Create JSON schemas for structured output
- Configure and integrate MCP servers
- Manage tools and resources in multi-system environments

---

## 1. Tools and `tool_use`

> Documentation: [Tool Use](https://platform.claude.com/docs/en/build-with-claude/tool-use)

### 1.1 What is `tool_use`

`tool_use` is a mechanism that allows Claude to call external functions. The model does not run code directly—it generates a structured tool call request; your code executes it and returns the result.

### 1.2 Tool Definition

Each tool is defined using a JSON schema:

```json
{
  "name": "get_customer",
  "description": "Finds a customer by email or ID. Returns the customer profile, including name, email, order history, and account status. Use this tool BEFORE lookup_order to verify the customer's identity. Accepts an email (format: user@domain.com) or a numeric customer_id.",
  "input_schema": {
    "type": "object",
    "properties": {
      "email": {"type": "string", "description": "Customer email"},
      "customer_id": {"type": "integer", "description": "Numeric customer ID"}
    },
    "required": []
  }
}
```

**Important:**

1. **The description is the primary selection mechanism.** An LLM chooses tools based on their descriptions. Minimal descriptions ("Retrieves customer information") lead to mistakes when tools overlap.

2. **Include in the description:**
    - What the tool does and returns
    - Input formats and example values
    - Edge cases and constraints
    - When to use this tool vs similar alternatives

3. **Avoid** identical or overlapping descriptions across tools. If `analyze_content` and `analyze_document` have nearly identical descriptions, the model will confuse them.

4. **Built-in tools vs MCP tools:** agents may prefer built-in tools (Read, Grep) over MCP tools with similar functionality. To prevent this, strengthen MCP tool descriptions—highlight concrete advantages, unique data, or context that built-in tools cannot provide.

### 1.3 The `tool_choice` Parameter

`tool_choice` controls how the model selects tools:

| Value | Behavior | When to use |
|---|---|---|
| `{"type": "auto"}` | The model decides whether to call a tool or answer in text | Default for most cases |
| `{"type": "any"}` | The model **must** call some tool | When you need guaranteed structured output |
| `{"type": "tool", "name": "extract_metadata"}` | The model **must** call a specific tool | When you need a forced first step / execution order |

**Important**
- `tool_choice: "any"` + multiple extraction tools → the model picks the best one, but you still get structured output
- Forced selection → when you must guarantee a specific first action (e.g., `extract_metadata` before enrichment)

### 1.4 JSON Schemas for Structured Output

Using `tool_use` with JSON schemas is the **most reliable** way to obtain structured output from Claude. It:
- Guarantees syntactically valid JSON (no missing braces, no trailing commas)
- Enforces the required structure (required fields are present)
- Does **not** guarantee semantic correctness (values can still be wrong)

**Schema design:**

```json
{
  "type": "object",
  "properties": {
    "category": {
      "type": "string",
      "enum": ["bug", "feature", "docs", "unclear", "other"]
    },
    "category_detail": {
      "type": ["string", "null"],
      "description": "Details if category = 'other' or 'unclear'"
    },
    "severity": {
      "type": "string",
      "enum": ["critical", "high", "medium", "low"]
    },
    "confidence": {
      "type": "number",
      "minimum": 0,
      "maximum": 1
    },
    "optional_field": {
      "type": ["string", "null"],
      "description": "Null if the information was not found in the source"
    }
  },
  "required": ["category", "severity"]
}
```

**Schema design rules:**
1. **Required vs optional:** mark fields as required only if the information is always available. Required fields push the model to fabricate values when data is missing.
2. **Nullable fields:** use `"type": ["string", "null"]` for information that may be absent. The model can return `null` instead of hallucinating.
3. **Enums with `"other"`:** add `"other"` + a detail string to avoid losing data outside your predefined categories.
4. **Enum `"unclear"`:** for cases where the model cannot confidently pick a category—honest `"unclear"` is better than a wrong category.

### 1.5 Syntax vs Semantic Errors

| Error type | Example | Mitigation |
|---|---|---|
| **Syntax** | Invalid JSON, wrong field type | `tool_use` with a JSON schema (eliminates) |
| **Semantic** | Totals don't add up, value in wrong field, hallucination | Validation checks, retry with feedback, self-correction |

---

## 2. Model Context Protocol (MCP)

> Documentation: [MCP](https://modelcontextprotocol.io/) | [Tools](https://modelcontextprotocol.io/docs/concepts/tools) | [Resources](https://modelcontextprotocol.io/docs/concepts/resources) | [Servers](https://modelcontextprotocol.io/docs/concepts/servers)

### 2.1 What is MCP

The Model Context Protocol (MCP) is an open protocol for connecting external systems to Claude. MCP defines three primary resource types:

1. **Tools** — functions the agent can call to perform actions (CRUD operations, API calls, command execution)
2. **Resources** — data the agent can read for context (documentation, database schemas, content catalogs)
3. **Prompts** — predefined prompt templates for common tasks

### 2.2 MCP Servers

An MCP server is a process that implements the MCP protocol and provides tools/resources. When you connect to an MCP server:
- All tools are discovered automatically
- Tools from all connected servers are available at once
- Tool descriptions determine how the model will use them

### 2.3 Configuring MCP Servers

**Project configuration (`.mcp.json`)** — for team usage:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "mcp-server-jira"],
      "env": {
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    }
  }
}
```

**Key points:**
- `.mcp.json` is stored at the project root and managed in version control
- Environment variables (`${GITHUB_TOKEN}`) are used for secrets—tokens themselves are not committed
- Available to all project contributors

**User configuration (`~/.claude.json`)** — for personal/experimental servers:
- Stored in the user's home directory
- Not shared via version control
- Suitable for personal experiments and testing

**Choosing servers:**
- For standard integrations (Jira, GitHub, Slack), prefer existing community MCP servers
- Build your own servers only for unique, team-specific workflows

### 2.4 The `isError` Flag in MCP

When an MCP tool encounters an error, it uses `isError: true` in the response. This signals to the agent that the call failed.

**Structured error (good):**

```json
{
  "isError": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "The service is temporarily unavailable. Timeout while calling the orders API.",
    "attempted_query": "order_id=12345",
    "partial_results": null
  }
}
```

**Generic error (anti-pattern):**

```json
{
  "isError": true,
  "content": "Operation failed"
}
```

A generic error gives the agent no information for decision-making—should it retry, change the query, or escalate?

### 2.5 MCP Resources

Resources are data that an agent can request to get context without taking actions:

- Content catalogs (e.g., a list of all project tasks, hierarchical navigation)
- Database schemas (understanding data structure)
- Documentation (API references, internal guides)
- Issue/task summaries

**Resource advantage:** the agent does not need exploratory tool calls to understand what data exists. A resource provides an immediate "map."

---

## 3. Claude API Fundamentals for Tool Integration

### 3.1 API Request Structure

The Claude API follows a request–response model. Each request to the Claude Messages API includes:

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "system": "You are a helpful assistant.",
  "messages": [
    {"role": "user", "content": "Hi!"},
    {"role": "assistant", "content": "Hello!"},
    {"role": "user", "content": "How are you?"}
  ],
  "tools": [...],
  "tool_choice": {"type": "auto"}
}
```

**Key fields:**
- `model` — model selection (`claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`)
- `max_tokens` — maximum number of tokens in the response
- `system` — the system prompt (defines model behavior)
- `messages` — conversation history (**you must send the full history** to maintain coherence)
- `tools` — definitions of available tools
- `tool_choice` — tool selection strategy

### 3.2 Message Roles

The `messages` array uses three roles:
- `user` — user messages
- `assistant` — model responses (included when sending history)
- `tool` — tool call results (the role is not explicitly set; this appears as a `tool_result` content block)

**Important:** on every API request you must send the **full conversation history**. The model does not persist state between requests—each call is independent.

### 3.3 The `stop_reason` Field in the Response

The Claude API response includes `stop_reason`, which indicates why the model stopped generating:

| Value | Description | Action |
|---|---|---|
| `"end_turn"` | The model finished its response | Show the result to the user |
| `"tool_use"` | The model wants to call a tool | Execute the tool and return the result |
| `"max_tokens"` | Token limit reached | The response is truncated; you may need to increase the limit |
| `"stop_sequence"` | A stop sequence was encountered | Handle based on your application logic |

For agentic systems, `"tool_use"` and `"end_turn"` are the most important—they control the agent loop.

---

## Key Takeaways

1. **Tool descriptions** are the primary selection mechanism—make them explicit, detailed, and differentiated
2. **JSON schemas** guarantee syntactic correctness but not semantic accuracy
3. **`tool_choice`** parameter controls whether tool calling is optional, mandatory, or forced to a specific tool
4. **MCP** provides a standardized protocol for connecting external systems (tools and resources)
5. **Error handling** in MCP requires structured error responses with retry hints
6. **Resources** allow agents to understand data structure without exploratory calls
7. **Tool vs built-in tools**: strengthen MCP tool descriptions to compete with built-in alternatives

---

## Missing Content for Future Enhancement

- [ ] Custom MCP server development best practices
- [ ] Tool versioning and backward compatibility strategies
- [ ] Performance optimization for tool-heavy workflows
- [ ] Tool discovery and capability negotiation in dynamic environments
- [ ] Security considerations for tool exposure (input validation, output sanitization)
- [ ] Testing strategies for tools and MCP servers
- [ ] Tool naming conventions and categorization patterns
- [ ] Resource caching and invalidation strategies

