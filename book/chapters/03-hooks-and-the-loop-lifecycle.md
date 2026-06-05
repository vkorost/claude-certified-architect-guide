# Chapter 3: Hooks and the Loop Lifecycle

**Summary:** *The agentic loop has two distinct behavioral layers: what happens inside the model call (governed by system prompts and context), and what happens around it (governed by hooks). System prompts are probabilistic. Hooks execute as code. The exam tests whether you can correctly identify which layer a business requirement belongs to, and the answer determines whether your agent’s compliance is a suggestion or a guarantee. This chapter defines the complete hook event taxonomy for both Python and TypeScript SDKs, explains the semantics of PreToolUse and PostToolUse in depth, covers priority rules when multiple hooks apply, and establishes the “Prompt or Hook?” decomposition framework that underlies every enforcement question in the exam.*

---

## The Scene That Starts Every Discussion

A customer support agent processes refunds. The product requirements document says the agent must never approve a refund above five hundred dollars. So someone writes this into the system prompt:

```
IMPORTANT: Never process refunds above $500. If a refund is above $500, escalate to a human.
```

Clean. Clear. Documented. And wrong.

Not wrong as in incorrect policy. Wrong as the chosen enforcement mechanism. The model reads that instruction the same way it reads everything else: as context that influences its probabilistic response. On most requests, it complies. Occasionally, under the right combination of conversation history, phrasing, and token sampling, it does not. The failure mode is not dramatic. It does not announce itself. A $700 refund processes, the customer receives it, and no one discovers the violation until the audit report arrives three weeks later.<sup>[1]</sup>

Now consider the hook version. A ***PostToolUse*** callback fires every time process_refund completes. The callback reads *tool_input["amount"].* If the amount exceeds 500, the hook returns a block and routes to escalation. No exception path exists. No conversation framing can talk the hook out of it. The rule runs as code.<sup>[2]</sup>

The distinction between these two implementations is the central argument of this chapter. Call it the **Prompt or Hook boundary**: the architectural decision point where you determine whether a business requirement belongs inside the model call (as context) or outside it (as code).

---

## How Hooks Fit the Loop

The agent loop, as established in Chapter 1, is a cycle: Claude evaluates context, produces tool calls, the SDK executes those tools, results feed back, repeat. The loop terminates on *stop_reason: "end_turn".*

Hooks occupy the seams of this cycle. They are callback functions registered in the hooks field of your agent options, keyed to named events.<sup>[3]</sup> When an event fires, the SDK collects every hook registered for that event type, runs them, and applies their outputs before proceeding. The model never knows a hook ran. The model sees only the result of what the hook decided to allow, block, or transform.

This is the key architectural fact: **hooks execute in the orchestrator layer, not inside the model call**. A system prompt instruction lives in the context window. A hook lives in your application code. They have completely different execution guarantees.

```
options.hooks = {
  PreToolUse: [{ matcher: "process_refund", hooks: [refundLimitHook] }],
  PostToolUse: [{ matcher: "get_customer",   hooks: [timestampNormHook] }]
}
```

The matcher field is a regex pattern matched against the tool name. "process_refund" matches exactly. "Write|Edit" matches either file modification tool. "mcp__" (with the double underscore prefix) matches any MCP tool call. A hook without a matcher fires for every event of that type.<sup>[4]</sup>

---

## The Complete Taxonomy

Both the Python and TypeScript SDKs share a common set of hook events. The TypeScript SDK adds several more for session and workspace lifecycle management.<sup>[5]</sup>

### Shared Events (Both SDKs)

**PreToolUse** Fires before a tool executes. This is where you make permission decisions. The hook receives *tool_name, tool_input, session_id, cwd, *and* hook_event_name.* When the hook fires inside a subagent, *agent_id *and* agent_type* are also populated. Return a *permissionDecision* to control what happens next.

**PostToolUse** Fires after a tool returns, before the result reaches the model. This is where you normalize, augment, or replace tool outputs. The hook receives the full tool result, and you can return *additionalContext* (appends information to the result) or *updatedToolOutput* (replaces the result entirely).<sup>[6]</sup>

**PostToolUseFailure** Fires when a tool execution fails. Use this to log errors, trigger alerts, or implement fallback logic at the orchestrator level.

**UserPromptSubmit** Fires when a prompt is submitted to the agent. Use this to inject additional context into every user prompt, enforce access control at the prompt level, or validate prompt content before it reaches the model.

**Stop** Fires when agent execution finishes. Use this to save session state, emit completion metrics, or trigger downstream workflows.

**SubagentStart** and **SubagentStop** Fire when a subagent initializes and completes. The coordinator-plus-subagents pattern from Chapter 2 runs through the Agent tool; these hooks let you track, log, and aggregate parallel task activity from outside the subagent’s context.<sup>[7]</sup>

**PreCompact** Fires before automatic context compaction. Receives a trigger field with value "manual" or "auto". Use this to archive the full conversation transcript before the summarization pass discards it. The full compaction strategy belongs to Chapter 11; the hook’s callback interface is defined here.<sup>[8]</sup>

**PermissionRequest** Fires when a permission dialog would normally be displayed. Enables custom permission handling without surface-level UI.

**Notification** Fires for agent status messages: permission_prompt (Claude needs approval), idle_prompt (Claude is waiting for input), auth_success, and elicitation_dialog. Each notification includes a message field and optionally a title. Forward these to Slack, PagerDuty, or any webhook endpoint without disrupting the loop.

### TypeScript-Only Additions

The TypeScript SDK adds hooks for session and workspace lifecycle events not yet available in Python:<sup>[5]</sup>

- **SessionStart** and **SessionEnd**: session initialization and termination
- **PostToolBatch**: fires once per batch of tool calls before the next model call; use this to inject conventions for the entire batch rather than once per tool
- **TeammateIdle**: a teammate in a multi-agent workspace becomes idle
- **TaskCompleted**: a background task completes
- **ConfigChange**: a configuration file changes; reload settings dynamically
- **WorktreeCreate** and WorktreeRemove: git worktree lifecycle management
- **Setup**: session setup and maintenance tasks

Note: SessionStart and SessionEnd can be registered as SDK callback hooks in TypeScript. In Python, they are only available as shell command hooks defined in settings files like .claude/settings.json.<sup>[9]</sup>

---

## PreToolUse in Depth: Permission Decisions

**PreToolUse** is the enforcement hook. It intercepts the tool call before any side effects occur. The hookSpecificOutput you return contains a permissionDecision field with four possible values:<sup>[10]</sup>

**"allow"** The operation proceeds. The model receives the tool result as normal.

**"deny"** The operation is blocked. The model receives a rejection message as the tool result. Claude typically attempts a different approach or reports it cannot proceed. Pair deny with systemMessage to explain the reason and prevent retry loops (more on this shortly).

**"ask"** The SDK surfaces an approval prompt to the human operator. Used when an action requires human judgment before proceeding.

**"defer" (TypeScript only)** Ends the current query and defers it for later resumption. Not available in the Python SDK.

### Modifying Input: updatedInput

PreToolUse is not just a gate. It can transform what actually reaches the tool. Return updatedInput alongside permissionDecision: "allow" to substitute different arguments before execution. A practical example: all Write tool calls have their file_path argument rewritten to prepend a /sandbox prefix, redirecting every write to a controlled directory. The model issues the same call; the hook transparently redirects it.<sup>[11]</sup>

There is one constraint: you must always return a new object for **updatedInput**. Never mutate the original **tool_input** object. And when using updatedInput, the permissionDecision: "*allow*" must also be present, or the modification does not take effect.

### The systemMessage Field

Every hook output supports a top-level systemMessage field. This field injects a message into the conversation that the model can see, independent of any tool result. When you block an operation with deny, the model knows the call was rejected but may not know why. If it does not know why, it will try again with a different phrasing. systemMessage prevents this by injecting a clear explanation:<sup>[12]</sup>

```
return {
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "Refund exceeds agent limit"
    },
    "systemMessage": "Refunds above $500 require human approval. "
                     "Use escalate_to_human with the refund details."
}
```

The model reads this, understands the constraint, and calls escalate_to_human instead of retrying process_refund. One hook, three outcomes: block the bad action, explain the constraint, redirect to the correct action.

### The continue_ Field

The continue_ field (spelled **continue** in TypeScript, **continue**_ in Python to avoid the reserved keyword) determines whether the agent keeps running after the hook completes. When False, the agent stops after the hook fires, regardless of other pending tool calls or turns. Use this to halt execution entirely on critical violations rather than just blocking the immediate tool call.<sup>[13]</sup>

---

## PostToolUse in Depth: Output Transformation

**PostToolUse** *fires **after** the tool returns but **before** the model sees the result.* Two output fields control what the model receives:<sup>[6]</sup>

**additionalContext** Appends information to the tool result. The original result is preserved; additional content is concatenated. Use this to inject metadata, warnings, or supplementary data alongside what the tool returned.

**updatedToolOutput** Replaces the tool’s output entirely before it reaches Claude. The model never sees what the tool actually returned. Use this for wholesale transformation.

### Data Normalization: The Canonical Use Case

Consider a customer support agent that calls three different MCP tools: one returns timestamps as Unix epoch integers, one returns them as ISO 8601 strings, one returns HTTP status codes as numbers. The model has to reason across all three tool results. If the formats differ, the model must internally reconcile the representations before doing anything useful with them.<sup>[14]</sup>

A PostToolUse hook with updatedToolOutput solves this cleanly. The hook intercepts every tool result, detects the format, normalizes it to a consistent schema, and sends that normalized result to the model. Claude sees uniform output regardless of which tool produced it. The tools do not need to be modified. The model does not need prompting to handle inconsistency.

```python
def normalize_timestamps(tool_name, tool_input, tool_output):
    if isinstance(tool_output.get("timestamp"), int):
        # Unix epoch to ISO 8601
        from datetime import datetime, timezone
        ts = datetime.fromtimestamp(tool_output["timestamp"], tz=timezone.utc)
        tool_output["timestamp"] = ts.isoformat()
    return {
        "hookSpecificOutput": {
            "hookEventName": "PostToolUse",
            "updatedToolOutput": tool_output
        }
    }
```

This is Task Statement 1.5 from the exam guide in concrete form: PostToolUse for normalization is not an optimization. It is an architectural choice that prevents the model from needing to compensate for infrastructure inconsistency.<sup>[2]</sup>

---

## Priority Rules: When Multiple Hooks Apply

When multiple hooks register for the same event and the same call triggers more than one of them, the **SDK runs all matching hooks in parallel**. Completion order is non-deterministic. Write each hook to act independently. Do not design hooks that depend on another hook having already run.<sup>[15]</sup>

For permission decisions, the SDK applies a strict priority rule:

**Deny beats defer, which beats ask, which beats allow.**

A single deny from any hook blocks the operation, regardless of what every other hook returns. The SDK does not average or vote on permission decisions. One deny is final.<sup>[16]</sup>

This rule has an important design implication. If you register a “log everything” hook alongside a “block dangerous operations” hook for the same event, the logger can return {} (empty object, meaning no decision) without interfering with the security hook. The deny wins. The log still runs. The hooks compose cleanly because the most restrictive result always wins.

---

## Asynchronous Hooks: Side Effects Without Blocking

By default, the agent waits for each hook to return before proceeding. This is the correct behavior when the hook’s output influences the operation: you need the permission decision or transformed output before the loop continues.

For hooks that only need to perform side effects, this wait is unnecessary latency. Logging, metrics, webhooks, Slack notifications: none of these need to block the agent. Return async: true (or async_: True in Python) to tell the agent to continue immediately without waiting:<sup>[17]</sup>

```
return {
    async: true,
    asyncTimeout: 5000  // optional millisecond timeout for background work
}
```

The constraint is absolute: **async hooks cannot block, modify, or inject context**. The agent has already moved on. Async mode is for fire-and-forget side effects only. If your hook needs to influence what happens next, it must be synchronous.

---

## MCP Tool Names in Hook Matchers

MCP tools follow the naming pattern mcp__<server>__<action>. A tool called get_customer on an MCP server named support_db is addressed as mcp__support_db__get_customer throughout the hook system.<sup>[18]</sup>

This pattern matters for matcher design. Consider a hook that should audit every MCP tool call. The matcher "mcp__" (with the trailing double underscore treated as a prefix pattern in the regex context) covers all MCP tools regardless of server or action name. The matcher "mcp__support_db__" narrows to one server. The matcher "mcp__support_db__process_refund" targets a single tool.

The full treatment of MCP server configuration, including how servers are defined and how tools are discovered, is in Chapter 5. The hook matcher syntax is established here.

---

## The Prompt or Hook Boundary

The exam will present you with a business requirement and ask you to choose the right enforcement mechanism. The decision comes down to one question: does this requirement need a guarantee?

Prompt instructions are probabilistic. They influence the model’s behavior by adding context to the input the model reasons over. Under typical conditions, a clear system prompt instruction will be followed reliably. But “reliably” is not “always.” The model can be argued out of an instruction through conversation framing. It can misweigh the instruction when the context window is long. It can fail to apply it in edge cases the author did not anticipate.<sup>[1]</sup>

Hooks execute as code. They do not reason. They do not weigh competing considerations. They are functions: given input, return output. If the input matches the condition, the hook applies. Every time. Regardless of what the model was thinking, what the user said, or how the conversation got there.

Here is the decomposition framework. Apply it to every enforcement question:

**Use a prompt when:** - The requirement is a soft preference: style, tone, formatting conventions - Partial compliance is acceptable: “prefer shorter responses” does not need a hard stop on long responses - The requirement depends on context that only the model can evaluate: “be empathetic when the customer is upset” requires model judgment - A violation is recoverable without business consequence

**Use a hook when:** - The requirement is a business rule with a defined threshold: refund limits, transaction amounts, record counts - Compliance must be deterministic: identity verification before financial operations, access control gates - The failure mode is a compliance violation or safety issue, not just a suboptimal response - The requirement applies regardless of conversation framing or model confidence

The customer support scenario from the beginning of this chapter lands squarely in the hook column. “Never process refunds above $500” is a business rule with a defined threshold. Its failure mode is a compliance violation. Probabilistic compliance is not acceptable. A PostToolUse hook on process_refund is the correct mechanism, full stop.<sup>[2]</sup>

This distinction appears explicitly in Task Statement 1.4 of the exam guide: “when deterministic compliance is required, prompt instructions alone have a non-zero failure rate.”<sup>[1]</sup> The exam is not asking whether prompts usually work. It is asking whether you understand that “usually” is architecturally insufficient for certain requirements.

---

## Decomposing Requirements: Worked Examples

The Prompt or Hook boundary is the named concept. Decomposition is the skill. Here are three requirement patterns and how they map:

### Pattern 1: Workflow Ordering

**Requirement:** The agent must call get_customer and receive a verified customer ID before calling process_refund.

**Wrong approach:** Add to system prompt: “Always call get_customer first and wait for a verified customer ID before processing any refund.”

**Right approach:** PreToolUse hook on process_refund that checks whether a verified customer ID is available in session state. If not, return deny with a systemMessage instructing the model to retrieve the customer first.

**Why:** This is a prerequisite gate. The failure mode if skipped is processing a refund against an unverified customer, which is a business logic violation. Code enforces it; a prompt suggests it.<sup>[1]</sup>

### Pattern 2: Output Format Normalization

**Requirement:** Multiple MCP tools return timestamps in different formats. The model should receive consistent ISO 8601 timestamps.

**Wrong approach:** Add to system prompt: “If a tool returns a Unix timestamp, convert it to ISO 8601 before reasoning about it.”

**Right approach:** PostToolUse hook with updatedToolOutput that normalizes all timestamp fields before Claude sees them.

**Why:** Asking the model to perform format conversion is adding cognitive overhead to a task that code handles trivially. The hook runs in the orchestrator; the model processes clean data.<sup>[14]</sup>

### Pattern 3: Tone and Style Preferences

**Requirement:** Responses to customers should be warm and avoid technical jargon.

**Right approach:** System prompt. This is a style preference that requires model judgment to apply appropriately. No threshold, no determinism requirement, no compliance risk. A hook enforcing “warmth” would be both impossible and absurd.

**Wrong approach:** Writing a hook that pattern-matches output for technical terms and blocks responses containing them. This produces a brittle, false-positive-heavy system that cannot handle legitimate uses of technical language in technical contexts.

The hook is not smarter than the model on questions of warmth. Stop before you write one that tries to be.

---

## SubagentStart and SubagentStop

The SubagentStart and SubagentStop events connect the hook system to the multi-agent patterns from Chapter 2. When the coordinator spawns a subagent using the Agent tool, SubagentStart fires. When the subagent completes, SubagentStop fires.

These hooks give the parent orchestrator visibility into parallel subagent activity without having access to the subagent’s internal context. A SubagentStop hook can aggregate results, detect failures, or trigger coordinator logic when all parallel subagents have completed.<sup>[7]</sup>

One practical implication: subagents do not automatically inherit parent agent permissions. If the parent agent has a PreToolUse hook auto-approving certain tools, those hooks apply to the parent’s tool calls. The subagent runs in its own execution context with its own permission checks. When multiple subagents are spawned in parallel, each one may independently request permissions. Handle this by configuring permission rules that apply at the subagent session level, or use PreToolUse hooks scoped to run inside subagent sessions using the agent_type field on the hook input.<sup>[19]</sup>

---

## The PreCompact Hook

Context compaction is covered in depth in Chapter 11. The PreCompact hook’s place in this chapter is its event mechanics.

The hook fires before compaction occurs, whether triggered automatically (when the context window approaches its limit) or manually (when /compact is sent as a prompt). The trigger field on the hook input tells you which case you are in: "auto" or "manual".<sup>[8]</sup>

The canonical use case is archiving the full conversation transcript before the summarization pass discards it. The agent loop will continue with a summarized history. The PreCompact hook gives you a window to capture the full record before that happens: write it to a file, push it to an audit log, send it to an analytics endpoint.

An async hook works correctly here. Archiving is a side effect. The compaction does not need to wait for the archive to complete.

```python
def archive_transcript(hook_input):
    transcript = hook_input.get("messages", [])
    trigger = hook_input.get("trigger")
    write_to_audit_log(transcript, trigger=trigger)
    return {"async_": True}
```

---

## Configuring Hooks: The Registration Pattern

Hooks are passed in the hooks field of your agent options. The structure is consistent across both SDKs: keys are event names, values are arrays of matcher configurations, each containing an optional pattern and the callback functions to run when it matches.<sup>[3]</sup>

```python
from claude_agent_sdk import ClaudeAgentOptions
options = ClaudeAgentOptions(
    tools=[lookup_customer, check_order, process_refund, escalate_to_human],
    hooks={
        "PreToolUse": [
            {
                "matcher": "process_refund",
                "hooks": [refund_limit_hook],
                "timeout": 30
            }
        ],
        "PostToolUse": [
            {
                "matcher": "mcp__support_db__.*",
                "hooks": [normalize_timestamps_hook]
            }
        ],
        "Stop": [
            {
                "hooks": [save_session_state_hook]
            }
        ]
    }
)
```

The timeout field on a matcher defaults to 60 seconds. Increase it for hooks that perform external API calls or database writes. Use AbortSignal in TypeScript to handle cancellation gracefully when a hook times out.

Hook event names are case-sensitive. PreToolUse works. preToolUse does not register correctly.<sup>[20]</sup>

---

## What the Exam Specifically Tests

The exam draws from Task Statements 1.4 and 1.5. Both task statements are built around the same conceptual axis: knowing when to put business logic in a prompt versus a hook.

Here is the question pattern you will see most often:

*A customer support agent has a policy rule about [threshold or prerequisite]. The current implementation uses [prompt instruction or hook]. The agent [sometimes violates / always enforces] the policy. What is the correct implementation?*

The answer key every time: hooks for thresholds, prerequisites, and compliance requirements; prompts for preferences, style, and requirements that require model judgment to apply contextually.

Secondary patterns:

- **PostToolUse for data normalization**: when multiple tools return different formats, the hook normalizes before the model processes. The prompt-based alternative (asking the model to convert formats) adds unnecessary reasoning load.
- **systemMessage to prevent retry loops**: pair with deny to explain why an operation was blocked. Without the explanation, the model retries. With it, the model redirects.
- **Priority rules for multiple hooks**: one deny overrides any number of allow returns from other hooks. Write hooks to act independently; do not rely on hook execution order.
- **Async hooks for logging**: side effects that do not need to influence agent behavior should return async: true to avoid adding latency to every tool call.
- **MCP tool name pattern in matchers**: mcp__<server>__<action> is the format; use it in regex matchers to target specific servers or specific tools.

One original sample question to test the concept:

**Q: A compliance team requires that all database write operations be preceded by a successful call to verify_access. An architect adds “always call verify_access before any write operation” to the system prompt. During testing, the agent occasionally performs writes without calling verify_access when the user provides authorization context in the conversation. What should the architect do?**

A. Add more detail to the system prompt instruction about when verify_access is required. 
B. Implement a PreToolUse hook on write operations that checks for a verified access token and denies the call if one is not present. 
C. Implement a PostToolUse hook on verify_access that blocks subsequent write calls until it fires. 
D. Use tool_choice: "any" to force the model to call a tool on every turn.
***
The correct answer is B.** The requirement is a deterministic prerequisite gate. Hook-based enforcement is the correct mechanism. Option A fails because the problem is not vague instructions: it is probabilistic enforcement. Option C misapplies PostToolUse (it fires after verify_access, not before write operations). Option D forces tool use but does not address the ordering requirement.*

*When hooks are absent, tool descriptions become the only routing mechanism. Chapter 4 is about getting that mechanism right.*

---

## Key Takeaways

- Hooks execute in the orchestrator layer, outside the model call. Prompts execute inside the model call, as context. These are different execution environments with different guarantees.
- PreToolUse handles permission decisions (allow, deny, ask; defer in TypeScript only) and input modification via updatedInput. The systemMessage field injects explanations that prevent the model from retrying blocked operations.
- PostToolUse handles output transformation via additionalContext (append) and updatedToolOutput (replace). The canonical use case is normalizing heterogeneous data formats from multiple MCP tools before the model processes them.
- When multiple hooks apply to the same event, they run in parallel. Deny wins over defer over ask over allow. A single deny blocks regardless of other hooks.
- Async hooks return async: true for fire-and-forget side effects. They cannot block, modify, or inject context; the agent has already proceeded by the time they complete.
- The PreCompact hook fires before context compaction, receiving a trigger field of "manual" or "auto". Use it to archive transcripts before summarization. The Prompt or Hook decision: use hooks when deterministic compliance is required; use prompts when model judgment is required to apply the requirement contextually.
- MCP tools are addressed in hook matchers using the pattern mcp__<server>__<action>. Regex patterns like mcp__support_db__.* target all tools on a specific server. 

The model is not your enforcement layer. Your code is.
