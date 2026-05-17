# Coverage Map

This map shows how the book's chapters align to the five CCA-F exam domains and their task statements. Use it to identify which chapters to review when drilling a specific domain.

## Domain 1: Agentic Architecture (27%)

**Chapters:** 1, 2, 3

**Task statements covered:**

- Identify the correct stop_reason and its implications for loop continuation or termination
- Distinguish between end_turn, tool_use, max_tokens, and pause_turn as termination signals
- Design a coordinator-subagent delegation pattern for a given task decomposition requirement
- Select the appropriate hook type (PreToolUse, PostToolUse, PreCompact, Notification) for a given lifecycle intervention
- Diagnose a runaway or prematurely terminating agentic loop given a scenario description
- Explain the difference between a system prompt hook and a UserPromptSubmit hook in terms of execution context

## Domain 2: Tool Design (18%)

**Chapters:** 4, 5

**Task statements covered:**

- Write or evaluate a tool description for routing precision and ambiguity risk
- Identify why a model selected the wrong tool given a set of tool descriptions and a user prompt
- Distinguish between tool_choice modes: auto, any, and tool
- Select the appropriate built-in Claude Code tool for a given file or shell task
- Evaluate the scope and trust implications of a given MCP server configuration
- Apply incremental exploration principles to MCP tool design

## Domain 3: Configuration and Customization (20%)

**Chapters:** 6, 7, 8

**Task statements covered:**

- Describe the three-layer CLAUDE.md hierarchy and explain inheritance and override semantics
- Identify which configuration layer should own a given policy or instruction
- Distinguish between a slash command and a skill in terms of invocation and scope
- Explain what plan mode changes about Claude's execution behavior and when to enable it
- Configure a CI/CD pipeline run using the --bare flag and -p flag correctly
- Identify session isolation risks in a multi-job pipeline and select the correct remediation
- Explain the permissionMode options and their appropriate use cases

## Domain 4: Prompting and Output (20%)

**Chapters:** 9, 10

**Task statements covered:**

- Identify vague prompt instructions and rewrite them using explicit criteria
- Distinguish between prompting strategies that produce reliable structured output versus those that do not
- Describe the tool_use-as-JSON-contract pattern and explain why it enforces schema compliance
- Design a validation loop for structured output including retry logic and failure conditions
- Identify prompt patterns that increase hallucination risk in tool-calling contexts
- Select the appropriate output format for a given downstream consumer

## Domain 5: Context Management and Reliability (15%)

**Chapters:** 11, 12

**Task statements covered:**

- Explain the lost-in-the-middle effect and its architectural consequences for long-running agents
- Select a context management strategy (summarization, truncation, partitioning) for a given scenario
- Identify the conditions that should trigger escalation rather than autonomous retry
- Describe the DataWithProvenance pattern and explain what it prevents
- Diagnose an error propagation failure in a multi-agent system given a scenario description
- Explain what context window exhaustion looks like from a stop_reason perspective and how to handle it
