# Glossary

Terms are listed alphabetically. Each definition covers the concept as it applies to Claude architecture and the CCA-F exam. Where a term has a specific technical meaning that differs from its colloquial use, the technical meaning is given.

**--bare flag** A Claude Code CLI flag that suppresses interactive UI elements and produces machine-readable output. Used in CI/CD contexts where Claude is invoked as a pipeline step rather than an interactive session.

**-p flag** The "print" flag in Claude Code CLI invocations. Enables non-interactive, single-turn execution. Combined with --bare for fully automated pipeline runs.

**agentic loop** The execution cycle in which Claude receives a message, produces a response that may include tool calls, executes tools, receives tool results, and continues until a terminal stop_reason is reached.

**coordinator** In a hub-and-spoke multi-agent system, the coordinator is the top-level Claude instance responsible for task decomposition, subagent delegation, and result aggregation. It does not perform the leaf-level work itself.

**context window** The total token budget available in a single Claude API call, encompassing the system prompt, conversation history, tool definitions, tool results, and the current response being generated.

**DataWithProvenance** An architectural pattern in which each piece of data produced by a subagent is wrapped with metadata identifying its source, the subagent that produced it, and the conditions under which it was generated. Prevents silent data corruption in multi-agent pipelines.

**end_turn** A stop_reason value indicating that Claude has completed its response and does not require further tool execution. The loop terminates normally.

**escalation trigger** A condition or threshold that causes an agent to stop autonomous action and route the task to a human or a higher-authority system. Defined explicitly in system prompts or hooks.

**CLAUDE.md hierarchy** The three-layer configuration system in Claude Code: global (user-level, applies to all projects), project (repository root, applies to all sessions in the project), and folder (subdirectory-level, applies when Claude operates within that folder). Lower layers can override or extend upper layers.

**hook** A mechanism for injecting behavior at specific points in the Claude Code agentic loop lifecycle. Hooks are not model instructions; they are executable scripts that run in response to lifecycle events.

**hub-and-spoke** A multi-agent architecture pattern in which a central coordinator delegates subtasks to specialized subagents. The coordinator aggregates results. Subagents do not communicate with each other directly.

**JSON schema** A declarative specification of the structure, types, and constraints of a JSON object. Used in tool definitions to specify input parameters and in structured output patterns to enforce response format.

**lost-in-the-middle effect** The empirical tendency for language models to underweight information that appears in the middle of a long context window relative to information at the beginning or end. A known reliability risk in long-running agents with large histories.

**max_tokens** A stop_reason value indicating that Claude's response was cut off because it reached the token limit set by the caller. Requires explicit handling; the response is incomplete.

**MCP (Model Context Protocol)** An open protocol for connecting Claude to external tool servers. MCP servers expose tools, resources, and prompts over a standardized interface. Claude Code supports local and remote MCP servers.

**pause_turn** A stop_reason value indicating that the agentic loop has paused and is waiting for external input or a condition to be resolved before continuing. Distinct from end_turn in that the loop is not complete.

**permissionMode** A Claude Code configuration setting that controls how the agent handles file and shell operations. Values include default (requires confirmation for destructive actions), acceptEdits (auto-approves file edits), and bypassPermissions (skips all permission checks, intended for trusted CI environments).

**plan mode** A Claude Code execution mode in which Claude generates a plan for a task without executing any actions. The user reviews and approves the plan before execution begins. Reduces risk of irreversible autonomous actions.

**PreCompact hook** A lifecycle hook that fires before Claude Code compresses the conversation history to free context window space. Used to inject summary instructions or preserve specific content before compaction.

**PreToolUse hook** A lifecycle hook that fires before Claude executes a tool call. Can inspect, modify, or block the tool call. The primary mechanism for implementing tool-level policy enforcement.

**PostToolUse hook** A lifecycle hook that fires after a tool call completes and before the result is returned to Claude. Can transform or annotate tool results.

**Notification hook** A lifecycle hook that fires when Claude Code emits a notification event, such as a long-running task status update. Used for routing alerts to external systems.

**session isolation** The property of a pipeline or CI/CD configuration in which each Claude invocation runs with a clean context and no shared state from previous runs. Prevents cross-job context bleed.

**skill** A reusable, named instruction set stored in a CLAUDE.md file or configuration that can be invoked by name. Skills encapsulate multi-step procedures. Distinct from slash commands in scope and invocation mechanism.

**slash command** A Claude Code invocation pattern using a leading slash, such as /review or /test. Slash commands trigger predefined behaviors or skill sequences. They are explicit intent signals, not natural language prompts.

**stop_reason** The field in a Claude API response that indicates why the model stopped generating. Values include end_turn, tool_use, max_tokens, and pause_turn. Correct handling of each stop_reason is required for a robust agentic loop.

**structured output** Output from Claude that conforms to a predefined schema, typically JSON. Achieved most reliably through the tool_use mechanism, which enforces schema compliance at the API level.

**subagent** In a hub-and-spoke architecture, a Claude instance that receives a delegated subtask from the coordinator, executes it, and returns a result. Subagents typically have narrower context and more specific tool access than the coordinator.

**tool_choice** An API parameter that controls how Claude selects tools. Values: auto (model decides whether to use a tool), any (model must use one of the available tools), tool (model must use a specific named tool).

**tool_use** A stop_reason value indicating that Claude has generated a tool call and is waiting for the result before continuing. Also used to refer to the content block type that represents a tool call in the API response.

**UserPromptSubmit hook** A Claude Code lifecycle hook that fires when the user submits a prompt, before Claude processes it. Used to inject context, timestamps, or policy checks at the start of each turn.

**validation loop** An architectural pattern in which Claude generates structured output, an external validator checks it against a schema, and the result is fed back to Claude for correction if validation fails. Continues until output passes or a retry limit is reached.
