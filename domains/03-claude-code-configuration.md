# Domain 3: Claude Code Configuration & Workflows (20%)

## Overview
This domain covers Claude Code configuration, workspace management, skill development, and integrated workflows. It includes CLAUDE.md hierarchies, slash commands, planning modes, and CI/CD integration patterns.

**Key Learning Outcomes:**
- Configure CLAUDE.md files at different hierarchy levels
- Create and manage custom skills and commands
- Apply planning mode for complex tasks
- Use built-in tools effectively (Glob, Grep, Read, Edit)
- Integrate Claude Code into CI/CD pipelines

---

## 1. Claude Code — Configuration and Workflows

> Documentation: [Claude Code](https://code.claude.com/docs/en/overview) | [Memory / CLAUDE.md](https://code.claude.com/docs/en/memory) | [Skills](https://code.claude.com/docs/en/skills) | [MCP](https://code.claude.com/docs/en/mcp) | [Hooks](https://code.claude.com/docs/en/hooks) | [Sub-agents](https://code.claude.com/docs/en/sub-agents) | [GitHub Actions](https://code.claude.com/docs/en/github-actions) | [Headless](https://code.claude.com/docs/en/headless)

### 1.1 The CLAUDE.md Hierarchy

CLAUDE.md is the instruction file(s) for Claude Code. There is a three-level hierarchy:

```
1. User-level: ~/.claude/CLAUDE.md
   - Applies only to that user
   - NOT shared via VCS
   - Personal preferences and working style

2. Project-level: .claude/CLAUDE.md or a root CLAUDE.md
   - Applies to all project contributors
   - Managed via VCS
   - Coding standards, testing standards, architectural decisions

3. Directory-level: CLAUDE.md in subdirectories
   - Applies when working with files in that directory
   - Conventions specific to that part of the codebase
```

**Common mistake:** a new team member does not receive project instructions because they were placed in `~/.claude/CLAUDE.md` (user-level) instead of `.claude/CLAUDE.md` (project-level).

### 1.2 `@path` Syntax (File Imports)

CLAUDE.md can reference external files using `@path`, making configuration modular:

```markdown
# Project CLAUDE.md

Coding standards are described in @./standards/coding-style.md
Test requirements are in @./standards/testing-requirements.md
Project overview is in @README.md and dependencies are in @package.json
```

**Rules for `@path`:**
- Use `@` immediately before the file path (no space)
- Relative and absolute paths are supported
- Relative paths are resolved relative to the file that contains the import
- Maximum import nesting depth is 5

This avoids duplication and lets each package include only relevant standards.

### 1.3 The `.claude/rules/` Directory

`.claude/rules/` is an alternative to a monolithic CLAUDE.md, used to organize rules by topic:

```
.claude/rules/
  testing.md          -- testing conventions
  api-conventions.md  -- API conventions
  deployment.md       -- deployment rules
  react-patterns.md   -- React patterns
```

**Key feature: YAML frontmatter with `paths` for conditional loading:**

```yaml
---
paths: ["src/api/**/*"]
---

For API files, use async/await with explicit error handling.
Each endpoint must return a standard response wrapper.
```

```yaml
---
paths: ["**/*.test.tsx", "**/*.test.ts"]
---

Tests must use describe/it blocks.
Use data factories instead of hardcoding.
Do not mock the database—use a test database.
```

**How it works:**
- A rule is loaded **only** when Claude Code edits a file matching the `paths` pattern
- This saves context and tokens—irrelevant rules are not loaded
- Glob patterns let you apply conventions by file type regardless of location (ideal for tests scattered across the codebase)

**When to use `.claude/rules/` with `paths` vs directory-level CLAUDE.md:**
- `.claude/rules/` with `paths` — when conventions apply to files spread across many directories (tests, migrations)
- Directory-level CLAUDE.md — when conventions are tied to a specific directory and are not needed elsewhere

### 1.4 Custom Slash Commands and Skills

> **Note:** in the current Claude Code version, custom commands (`.claude/commands/`) are unified with skills (`.claude/skills/`). Both formats create `/name` commands. The exam guide references `.claude/commands/`—that format is still supported.

Slash commands are reusable prompt templates invoked via `/name`:

**`.claude/commands/` format (legacy, supported):**

```
.claude/commands/
  review.md        -- /review -- standard code review
  test-gen.md      -- /test-gen -- test generation
```

**`.claude/skills/` format (current):**

```
.claude/skills/
  review/SKILL.md  -- /review -- with frontmatter configuration
  test-gen/SKILL.md
```

**Project commands** (`.claude/commands/` or `.claude/skills/`):
- Stored in VCS and available to everyone when cloning the repo
- Ensure consistent workflows across the team

**User commands** (`~/.claude/commands/` or `~/.claude/skills/`):
- Personal commands not shared via VCS
- For individual workflows

### 1.5 Skills — `.claude/skills/`

Skills are advanced commands configured via SKILL.md frontmatter:

```yaml
---
context: fork
allowed-tools: ["Read", "Grep", "Glob"]
argument-hint: "Path to the directory to analyze"
---

Analyze the code structure in the specified directory.
Output a report on dependencies and architectural patterns.
```

**Frontmatter parameters:**

| Parameter | Description |
|---|---|
| `context: fork` | Runs the skill in an isolated subagent. Verbose output does not pollute the main session |
| `allowed-tools` | Restricts which tools are available (security—e.g., the skill cannot delete files if not allowed) |
| `argument-hint` | Hint that asks for an argument when invoked without parameters |

**When to use a skill vs CLAUDE.md:**
- **Skill** — on-demand invocation for a specific task (review, analysis, generation)
- **CLAUDE.md** — always-loaded general standards and conventions

**Personal skills (`~/.claude/skills/`):**
- Create personal variants under different names so you don't affect teammates

---

## 2. Planning Mode vs Direct Execution

### 2.1 Planning Mode

**Planning mode:**
- The model only investigates and plans; it does not make changes
- Uses Read, Grep, Glob to explore the codebase
- Produces an implementation plan that the user approves
- Safe exploration with no side effects

**When to use planning mode:**
- Large changes (dozens of files)
- Multiple plausible approaches (microservices: how to define boundaries?)
- Architectural decisions (which framework? what structure?)
- Unfamiliar codebase (you must understand before changing)
- Library migrations affecting 45+ files

**When to use direct execution:**
- Single-file fixes with a clear stack trace
- Adding one validation check
- Well-understood, unambiguous changes

**Combined approach:**
1. Planning mode for investigation and design
2. User approves the plan
3. Direct execution to implement the approved plan

### 2.2 Explore Subagent

**Explore subagent** — a specialized subagent for exploring the codebase:
- Isolates verbose output from the main context
- Returns only a summary
- Prevents context-window exhaustion in multi-phase tasks

---

## 3. Built-in Tools in Claude Code

### 3.1 Tool Selection Reference

| Task | Tool | Example |
|---|---|---|
| Find files by name/pattern | **Glob** | `**/*.test.tsx`, `src/components/**/*.ts` |
| Search within files | **Grep** | Function name, error message, import |
| Read a file in full | **Read** | Load a file for analysis |
| Write a new file | **Write** | Create a file from scratch |
| Edit an existing file precisely | **Edit** | Replace a specific snippet via unique text match |
| Run a shell command | **Bash** | git, npm, run tests, build |

### 3.2 Incremental Investigation Strategy

Do not read all files at once. Build understanding incrementally:

```
1. Glob: find entry points (function definition, export)
2. Read: read the found files
3. Grep: find usages (import, calls)
4. Read: read consumer files
5. Repeat until you have a complete picture
```

### 3.3 Fallback: Read + Write Instead of Edit

When Edit fails due to a non-unique text match:
1. Read — load the full file content
2. Modify the content programmatically
3. Write — write the updated version

---

## 4. Context Management Commands

### 4.1 The `/compact` Command

`/compact` is a built-in command for compressing context:
- Summarizes prior history to free up the context window
- Used in long investigation sessions when the context fills up with verbose tool output
- Risk: exact numeric values, dates, and specific details can be lost during summarization

### 4.2 The `/memory` Command

`/memory` is a built-in command for managing memory between sessions:
- Opens the `CLAUDE.md` file for editing, allowing you to save notes, preferences, and context
- Information persists across sessions and is automatically loaded on startup
- Useful for storing project conventions, user preferences, frequently used commands, and current work context
- Alternative to re-explaining the same instructions in every session

---

## 5. Claude Code CLI and CI/CD Integration

### 5.1 Claude Code CLI for CI/CD

**The `-p` (or `--print`) flag:**

```bash
claude -p "Analyze this pull request for security issues"
```

- Non-interactive mode: processes the prompt, prints to stdout, exits
- Does not wait for user input
- The only correct way to run Claude in CI/CD pipelines

**Structured output for CI:**

```bash
claude -p "Review this PR" --output-format json --json-schema '{"type":"object",...}'
```

- `--output-format json` — output in JSON
- `--json-schema` — validate output against a schema
- The result can be parsed to automatically post inline PR comments

### 5.2 Session Context Isolation

The same Claude session that generated code is often less effective at reviewing it (the model retains its reasoning context and is less likely to challenge its own decisions). Use an independent instance for review.

**Preventing duplicate comments:**
When re-reviewing after new commits, include prior review results in context and instruct Claude to report only new or unresolved issues.

---

## 6. Advanced Features

### 6.1 Hooks in Claude Code

Hooks allow interception of Claude Code operations at specific points in the workflow:

```python
@hook("PreCodeEdit")
def validate_changes(edit_context):
    # Custom validation before applying changes
    return edit_context

@hook("PostCodeEdit")
def run_formatter(edit_result):
    # Run formatter after changes are applied
    return edit_result
```

### 6.2 Sub-agents in Claude Code

Claude Code supports spawning sub-agents for parallel task execution:
- Sub-agents inherit context from the parent
- Useful for parallel analysis (split a large codebase into chunks)
- Each sub-agent can have its own `allowed-tools` restrictions

---

## Key Takeaways

1. **CLAUDE.md hierarchy** ensures project standards are shared via VCS (project-level, not user-level)
2. **`@path` imports** keep configuration modular and DRY
3. **`.claude/rules/` with `paths`** applies conventions by file type, not directory location
4. **Skills vs CLAUDE.md**: skills are on-demand tasks; CLAUDE.md is always-loaded standards
5. **Planning mode** is essential for large, unfamiliar, or high-risk changes
6. **Incremental investigation** (Glob → Grep → Read) prevents context exhaustion
7. **CLI `-p` flag** is the correct way to run Claude in CI/CD pipelines
8. **Session isolation** ensures independent review without confirmation bias

---

## Missing Content for Future Enhancement

- [ ] Advanced hook patterns for complex workflows
- [ ] Sub-agent coordination and parallel execution strategies
- [ ] Integration with additional CI/CD platforms (beyond GitHub/GitLab)
- [ ] Custom tool development for Claude Code
- [ ] Performance optimization for large codebase exploration
- [ ] Template and scaffold generation workflows
- [ ] Automated refactoring patterns
- [ ] Security policies for Claude Code in enterprise environments

