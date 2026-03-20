
# Claude Certified Architect — Foundations Certification

## Register for exam 
Register with your [@corporate.com](https://accounts.skilljar.com/accounts/signup/?next=%2Fauth%2Fendpoint%2Flogin%2Fresult%3Fnext%3D%252Fclaude-code-in-action%26d%3Dcahl60vup5xv&t=3gufixqhei80k&d=cahl60vup5xv) email → [Take the Exam](https://anthropic.skilljar.com/claude-certified-architect-foundations-access-request)


## Foundations & Domain Overview

| Domain | Weight | Focus | Files                                                       |
|--------|--------|-------|-------------------------------------------------------------|
| **Agentic Architecture & Orchestration** | 27% | Agent loops, multi-agent patterns, error handling | [see](./domains/01-agentic-architecture-orchestration.md)   |
| **Tool Design & MCP Integration** | 18% | Tool definitions, JSON schemas, MCP servers | [see](./domains/02-tool-design-mcp-integration.md)          |
| **Claude Code Configuration & Workflows** | 20% | CLAUDE.md, skills, planning mode | [see](./domains/03-claude-code-configuration.md)            |
| **Prompt Engineering & Structured Output** | 20% | Few-shot prompting, validation loops, batch API | [see](./domains/04-prompt-engineering-structured-output.md) |
| **Context Management & Reliability** | 15% | Context windows, provenance, crash recovery | [see](./domains/05-context-management-reliability.md)       |


## Anthropic useful courses 

1. [Claude 101](https://anthropic.skilljar.com/claude-101)
2. [Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api)
3. [Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action)
4. [Introduction to agent skills](https://anthropic.skilljar.com/introduction-to-agent-skills)
5. [Introduction to Model Context Protocol](https://anthropic.skilljar.com/introduction-to-model-context-protocol)
6. [Model Context Protocol: Advanced Topics](https://anthropic.skilljar.com/model-context-protocol-advanced-topics)

## Practice Exam

- Registry for the exam
- Use this link https://anthropic.skilljar.com/anthropic-certification-practice-exam/ , or you may find link in your profile after exam registration.

## Official Documentation

| Resource | URL |
|---|---|
| **Claude API — Messages** | https://platform.claude.com/docs/en/api/messages |
| **Claude API — Tool Use** | https://platform.claude.com/docs/en/build-with-claude/tool-use |
| **Claude API — Message Batches** | https://platform.claude.com/docs/en/build-with-claude/message-batches |
| **Claude Agent SDK — Overview** | https://platform.claude.com/docs/en/agent-sdk/overview |
| **Claude Agent SDK — Hooks** | https://platform.claude.com/docs/en/agent-sdk/hooks |
| **Claude Agent SDK — Subagents** | https://platform.claude.com/docs/en/agent-sdk/subagents |
| **Claude Agent SDK — Sessions** | https://platform.claude.com/docs/en/agent-sdk/sessions |
| **Model Context Protocol (MCP)** | https://modelcontextprotocol.io/ |
| **MCP — Tools** | https://modelcontextprotocol.io/docs/concepts/tools |
| **MCP — Resources** | https://modelcontextprotocol.io/docs/concepts/resources |
| **MCP — Servers** | https://modelcontextprotocol.io/docs/concepts/servers |
| **Claude Code — Documentation** | https://code.claude.com/docs/en/overview |
| **Claude Code — CLAUDE.md and Memory** | https://code.claude.com/docs/en/memory |
| **Claude Code — Skills (incl. slash commands)** | https://code.claude.com/docs/en/skills |
| **Claude Code — Hooks** | https://code.claude.com/docs/en/hooks |
| **Claude Code — Sub-agents** | https://code.claude.com/docs/en/sub-agents |
| **Claude Code — MCP Integration** | https://code.claude.com/docs/en/mcp |
| **Claude Code — GitHub Actions CI/CD** | https://code.claude.com/docs/en/github-actions |
| **Claude Code — GitLab CI/CD** | https://code.claude.com/docs/en/gitlab-ci-cd |
| **Claude Code — Headless (non-interactive mode)** | https://code.claude.com/docs/en/headless |
| **Prompt Engineering Guide** | https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview |
| **Extended Thinking** | https://platform.claude.com/docs/en/build-with-claude/extended-thinking |
| **Anthropic Cookbook (code examples)** | https://github.com/anthropics/anthropic-cookbook |


## Glossary of Key Terms

| Term | Definition |
|------|-----------|
| **Agentic loop** | Model-driven sequence: send request → receive response → check `stop_reason` → repeat until `end_turn` |
| **Coordinator** | Multi-agent hub that decomposes tasks, delegates to subagents, aggregates results |
| **Subagent** | Task-specific agent with isolated context; spawned via `Task` tool |
| **Hook** | Deterministic interception point (preToolUse, postToolUse) |
| **MCP** | Model Context Protocol; open standard for integrating external systems |
| **tool_use** | Mechanism allowing Claude to call external functions |
| **tool_choice** | Parameter controlling whether/when Claude calls tools |
| **CLAUDE.md** | Instruction file for Claude Code with 3-level hierarchy |
| **Skill** | On-demand Claude Code command with frontmatter configuration |
| **Few-shot** | 2-4 examples demonstrating expected behavior |
| **Prompt chaining** | Breaking complex tasks into sequential focused steps |
| **Batch API** | Asynchronous API offering 50% cost savings, up to 24 hours processing |
| **Lost-in-the-middle** | Models miss information in the middle 60% of long inputs |
| **Provenance** | Documentation of data source, date, methodology, confidence |
| **State persistence** | Exporting agent state to disk for crash recovery |


**Good luck with your certification!**

