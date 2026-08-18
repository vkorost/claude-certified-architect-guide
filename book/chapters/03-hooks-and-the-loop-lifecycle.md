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

Now consider the hook version. A ***PreToolUse*** callback fires every time the agent asks for process_refund, before the tool runs at all. The callback reads *tool_input["amount"].* If the amount exceeds 500, the hook returns a deny decision and points the model at the escalation tool instead. No exception path exists. No conversation framing can talk the hook out of it. The rule runs as code, in the orchestrator, on every call.<sup>[2]</sup>

Which hook fires, and when, is the whole of the mechanism, and it is worth stating plainly before anything else in this chapter, because the two hooks that bracket a tool call are the most confused pair of objects in Domain 1. PreToolUse intercepts an outgoing tool call and can stop it: that is where policy is enforced. PostToolUse intercepts a tool result on its way back and can rewrite it: that is where data is normalized. A rule that has to prevent something must run before the thing it prevents. A hook that fires after process_refund returns can log the violation, alert on it, and file it neatly, and the money has already moved.<sup>[2]</sup>

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

The matcher field filters which callbacks fire, and it does not always mean what a reader who assumes "regex" expects. The characters in the string select the evaluation path. A matcher containing only letters, digits, underscores, hyphens, spaces, commas and pipes is compared as an exact string, or as a list of exact strings separated by pipes or commas: "process_refund" matches that tool and nothing else, "Write|Edit" matches either file modification tool exactly. A matcher containing any other character is evaluated as an unanchored regular expression, so "^Notebook" matches every tool whose name begins with Notebook, and "Edit.*" matches NotebookEdit as well as Edit. Anchor with ^ and $ when a whole-string match is what you meant. An empty matcher, an asterisk, or no matcher at all fires for every event of that type.<sup>[4]</sup>

The consequence that catches people sits at the boundary between the two paths: "mcp__" contains only exact-match characters, so it is compared against tool names as a literal and matches nothing at all. Reaching every tool on a server means forcing the regular-expression path, as in "mcp__support_db__.*". Matching against the tool name is also specific to tool events. Notification hooks match the notification type, subagent hooks the agent type, and several events, Stop and UserPromptSubmit and PostToolBatch among them, accept no matcher and run every time they fire.<sup>[4]</sup>

---

## The Complete Taxonomy

Both the Python and TypeScript SDKs share a common set of hook events. The TypeScript SDK adds a long list of further events for session, workspace and message lifecycle management. One caution before the list: the mechanics in this chapter are stable, but the membership of the two lists is a snapshot, and events land in TypeScript first and reach Python later.<sup>[5]</sup>

### Shared Events (Both SDKs)

**PreToolUse** Fires before a tool executes. This is where you make permission decisions. The hook receives *tool_name, tool_input, session_id, cwd, *and* hook_event_name.* When the hook fires inside a subagent, *agent_id *and* agent_type* are also populated. Return a *permissionDecision* to control what happens next.

**PostToolUse** Fires after a tool returns, before the result reaches the model. This is where you normalize, augment, or replace tool outputs. The hook receives the full tool result, and you can return *additionalContext* (appends information to the result) or *updatedToolOutput* (replaces the result entirely).<sup>[6]</sup>

**PostToolUseFailure** Fires when a tool execution fails. Use this to log errors, trigger alerts, or implement fallback logic at the orchestrator level.

**UserPromptSubmit** Fires when a prompt is submitted to the agent. Use this to inject additional context into every user prompt, enforce access control at the prompt level, or validate prompt content before it reaches the model.

**Stop** Fires when agent execution finishes. Use this to save session state, emit completion metrics, or trigger downstream workflows.

**SubagentStart** and **SubagentStop** Fire when a subagent initializes and completes. The coordinator-plus-subagents pattern from Chapter 2 runs through the Agent tool; these hooks let you track, log, and aggregate parallel task activity from outside the subagent’s context.<sup>[7]</sup>

**PreCompact** Fires before automatic context compaction. Receives a trigger field with value "manual" or "auto". Use this to archive the full conversation transcript before the summarization pass discards it. The full compaction strategy belongs to Chapter 11; the hook’s callback interface is defined here.<sup>[8]</sup>

**PermissionRequest** Fires when a tool call needs a permission decision. Enables custom permission handling without surface-level UI.

**Notification** Fires for agent status messages, and the notification type is what a matcher filters on here: permission_prompt (Claude needs approval), idle_prompt (Claude is waiting for input), auth_success, and the elicitation events. Each notification includes a message field and optionally a title, which makes forwarding to Slack, PagerDuty or any webhook endpoint a few lines of code. One caveat matters for anyone building on the SDK rather than driving the terminal: in a headless SDK session, only the elicitation completion and response events reach this hook. The rest are emitted by interactive UI an SDK session never runs, and a permission request goes to the canUseTool callback instead.<sup>[9]</sup>

### TypeScript-Only Additions

The TypeScript SDK adds hooks for session, workspace and message lifecycle events not available as Python SDK callbacks:<sup>[5]</sup>

- **SessionStart**, **SessionEnd** and **Setup**: session initialization, termination, and setup or maintenance work
- **PostToolBatch**: fires once per batch of tool calls before the next model call; use this to inject conventions for the entire batch rather than once per tool
- **UserPromptExpansion**: a typed command or an MCP prompt expands before it reaches Claude, though not when Claude invokes a skill itself. The place to block a command from direct invocation
- **MessageDisplay**: an assistant message with text completes. Redact or reformat what is displayed without touching the transcript
- **StopFailure**: the turn ended on an API error rather than a normal stop
- **PostCompact**: compaction finished. The counterpart to PreCompact, and where the generated summary gets logged
- **PermissionDenied**: auto mode denied a tool call, including denials carrying no classifier verdict
- **Elicitation** and **ElicitationResult**: an MCP server asks for user input mid-task, and a user answers
- **TaskCreated** and **TaskCompleted**: a task is created through the task tool, and a task is marked complete. The second is where a gate such as "the tests pass before this closes" belongs
- **InstructionsLoaded**: a CLAUDE.md or rules file is loaded, for auditing which instruction files a session picked up
- **ConfigChange**, **FileChanged**, **CwdChanged** and **DirectoryAdded**: configuration files, watched files, the working directory, and directories added mid-session
- **TeammateIdle**, **WorktreeCreate** and **WorktreeRemove**: a workspace teammate goes idle, and git worktree lifecycle

Note: SessionStart and SessionEnd can be registered as SDK callback hooks in TypeScript. In Python, they are only available as shell command hooks defined in settings files like .claude/settings.json.<sup>[10]</sup>

---

## PreToolUse in Depth: Permission Decisions

**PreToolUse** is the enforcement hook. It intercepts the tool call before any side effects occur. The hookSpecificOutput you return contains a permissionDecision field with four possible values:<sup>[11]</sup>

**"allow"** The operation proceeds. The model receives the tool result as normal.

**"deny"** The operation is blocked. The model receives a rejection message as the tool result. Claude typically attempts a different approach or reports it cannot proceed. Pair deny with permissionDecisionReason to explain the reason and prevent retry loops (more on this shortly).

**"ask"** The SDK surfaces an approval prompt to the human operator. Used when an action requires human judgment before proceeding.

**"defer" (TypeScript only)** Ends the current query and defers it for later resumption. The Python SDK's decision type carries three values and this is not one of them.<sup>[12]</sup>

There is an asymmetry in that list worth holding onto. A hook can stop a call, but silence from a hook is not approval. Returning an empty object is a declaration that this hook has no opinion, and the call carries on into the rest of the permission evaluation to be settled by something else.<sup>[12]</sup>

### Modifying Input: updatedInput

PreToolUse is not just a gate. It can transform what actually reaches the tool. Return updatedInput alongside permissionDecision: "allow" to substitute different arguments before execution. A practical example: all Write tool calls have their file_path argument rewritten to prepend a /sandbox prefix, redirecting every write to a controlled directory. The model issues the same call; the hook transparently redirects it.<sup>[13]</sup>

There is one hard constraint here and one widely believed non-constraint. The hard one: always return a new object for **updatedInput**, never mutating the original **tool_input**. The non-constraint: permissionDecision: "*allow*" is not required for the substitution to take effect. Pairing updatedInput with "allow" auto-approves the rewritten call, pairing it with "ask" shows the rewritten call to the operator, and omitting permissionDecision still applies the modification, which then flows through the normal permission evaluation like any other call. The pairing that quietly discards the work is "defer", which drops updatedInput on the floor. The field also has to sit inside hookSpecificOutput; at the top level it is ignored.<sup>[14]</sup>

### Explaining a Block: permissionDecisionReason and systemMessage

Two fields carry an explanation out of a hook, and they go to different readers. Getting them the wrong way round is a quiet failure: the hook works, the block holds, the operator sees a tidy log line, and the model spends the next three turns rephrasing the same call.<sup>[12]</sup>

**permissionDecisionReason** sits inside hookSpecificOutput and is the field the model reads. A denied model knows it was rejected and does not know why, and a model that does not know why will try again with different phrasing. The reason turns a wall into a signpost.

**systemMessage** is a top-level field accepted on every event, and it shows a message to the user. It is the operator's channel, not the model's. To hand the model context outside a permission decision, the field is additionalContext.

Both together look like this:<sup>[12]</sup>

```
return {
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "Refunds above $500 require human "
                                    "approval. Use escalate_to_human "
                                    "with the refund details."
    },
    "systemMessage": "Blocked a refund above the agent limit and "
                     "routed it to escalation."
}
```

The model reads the reason, understands the constraint, and calls escalate_to_human instead of retrying process_refund. The operator reads the system message and knows a block happened. One hook, three outcomes: stop the bad action, explain the constraint to the party whose behavior has to change, and tell the human watching that it happened.

### The continue_ Field

The continue_ field (spelled **continue** in TypeScript, **continue**_ in Python to avoid the reserved keyword) determines whether the agent keeps running after the hook completes. When False, the agent stops after the hook fires, regardless of other pending tool calls or turns. Use this to halt execution entirely on critical violations rather than just blocking the immediate tool call.<sup>[15]</sup>

---

## PostToolUse in Depth: Output Transformation

**PostToolUse** *fires **after** the tool returns but **before** the model sees the result.* Two output fields control what the model receives:<sup>[6]</sup>

**additionalContext** Appends information to the tool result. The original result is preserved; additional content is concatenated. Use this to inject metadata, warnings, or supplementary data alongside what the tool returned.

**updatedToolOutput** Replaces the tool’s output entirely before it reaches Claude. The model never sees what the tool actually returned. Use this for wholesale transformation. The field works for any tool in both SDKs. An older field, updatedMCPToolOutput, replaces MCP tool output only and is deprecated, so a hook written today should not reach for it.<sup>[12]</sup>

### Data Normalization: The Canonical Use Case

Consider a customer support agent that calls three different MCP tools: one returns timestamps as Unix epoch integers, one returns them as ISO 8601 strings, one returns HTTP status codes as numbers. The model has to reason across all three tool results. If the formats differ, the model must internally reconcile the representations before doing anything useful with them.<sup>[16]</sup>

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

This is Task Statement 1.5 in concrete form. The skill the guide names is normalizing heterogeneous formats from different tools before the agent processes them, and the ordering inside that phrase is the whole point: the reconciliation happens outside the model, or it happens inside it. Normalization in a hook is not an optimization. It moves a correctness concern out of the layer that reasons probabilistically and into the layer that does not.<sup>[2]</sup>

---

## Priority Rules: When Multiple Hooks Apply

When multiple hooks register for the same event and the same call triggers more than one of them, the **SDK runs all matching hooks in parallel**. Completion order is non-deterministic. Write each hook to act independently. Do not design hooks that depend on another hook having already run.<sup>[17]</sup>

For permission decisions, the SDK applies a strict priority rule:

**Deny beats defer, which beats ask, which beats allow.**

A single deny from any hook blocks the operation, regardless of what every other hook returns. The SDK does not average or vote on permission decisions. One deny is final.<sup>[18]</sup>

This rule has an important design implication. If you register a “log everything” hook alongside a “block dangerous operations” hook for the same event, the logger can return {} (empty object, meaning no decision) without interfering with the security hook. The deny wins. The log still runs. The hooks compose cleanly because the most restrictive result always wins.

The same ordering reaches past the hooks themselves. Hooks are the first step of permission evaluation, ahead of the deny rules, the ask rules, the permission mode and the allow rules, and that position produces an asymmetry worth memorizing. A hook deny is final: it holds in every permission mode, bypassPermissions included. A hook allow is not its mirror image, because it does not skip the deny and ask rules underneath it, which are evaluated regardless of what the hook returned. A hook can take away what the settings grant. It cannot grant what the settings forbid.<sup>[19]</sup>

That ordering also draws the line between the two ways to gate a tool call, and the line is not about strictness. A rule that can be written down as a pattern over a tool name and its arguments belongs in the permission rules, declaratively, in a settings file: shorter, auditable by someone who does not read the codebase, no code to maintain. A rule whose answer depends on something no pattern can see, session state, an account balance, the result of an earlier call, belongs in a PreToolUse hook, because arriving at the decision means running a program. Neither replaces the other, and the mistake runs in both directions: reaching for a hook to express a fixed command pattern, or reaching for a rule to express a condition that has to be computed.<sup>[19]</sup>

---

## Asynchronous Hooks: Side Effects Without Blocking

By default, the agent waits for each hook to return before proceeding. This is the correct behavior when the hook’s output influences the operation: you need the permission decision or transformed output before the loop continues.

For hooks that only need to perform side effects, this wait is unnecessary latency. Logging, metrics, webhooks, Slack notifications: none of these need to block the agent. Return async: true (or async_: True in Python) to tell the agent to continue immediately without waiting:<sup>[20]</sup>

```
return {
    async: true,
    asyncTimeout: 5000  // optional millisecond timeout for background work
}
```

The constraint is absolute: **async hooks cannot block, modify, or inject context**. The agent has already moved on. Async mode is for fire-and-forget side effects only. If your hook needs to influence what happens next, it must be synchronous.

---

## MCP Tool Names in Hook Matchers

MCP tools follow the naming pattern mcp__<server>__<action>. A tool called get_customer on an MCP server named support_db is addressed as mcp__support_db__get_customer throughout the hook system.<sup>[21]</sup>

This pattern matters for matcher design, and it is where the two evaluation paths from earlier in the chapter bite. Consider a hook that should audit every MCP tool call. The matcher has to be forced onto the regular-expression path: "^mcp__" does it with an anchor, "mcp__.*" does it with a wildcard. "mcp__" on its own does neither, because it contains only exact-match characters and is therefore compared against tool names as a literal, matching none of them. Narrowing to one server takes "mcp__support_db__.*" for exactly the same reason. Only the fully qualified name, "mcp__support_db__process_refund", works as written, because a single named tool is the case an exact-string matcher exists for.<sup>[4]</sup>

The full treatment of MCP server configuration, including how servers are defined and how tools are discovered, is in Chapter 5. The hook matcher syntax is established here.

---

## The Prompt or Hook Boundary

The exam will present you with a business requirement and ask you to choose the right enforcement mechanism. The decision comes down to one question: does this requirement need a guarantee?

Prompt instructions are probabilistic. They influence the model’s behavior by adding context to the input the model reasons over. Under typical conditions, a clear system prompt instruction will be followed reliably. But “reliably” is not “always.” The model can be argued out of an instruction through conversation framing. It can misweigh the instruction when the context window is long. It can fail to apply it in edge cases the author did not anticipate.<sup>[1]</sup>

Hooks execute as code. They do not reason. They do not weigh competing considerations. They are functions: given input, return output. If the input matches the condition, the hook applies. Every time. Regardless of what the model was thinking, what the user said, or how the conversation got there.

Here is the decomposition framework. Apply it to every enforcement question:

**Use a prompt when:** - The requirement is a soft preference: style, tone, formatting conventions - Partial compliance is acceptable: “prefer shorter responses” does not need a hard stop on long responses - The requirement depends on context that only the model can evaluate: “be empathetic when the customer is upset” requires model judgment - A violation is recoverable without business consequence

**Use a hook when:** - The requirement is a business rule with a defined threshold: refund limits, transaction amounts, record counts - Compliance must be deterministic: identity verification before financial operations, access control gates - The failure mode is a compliance violation or safety issue, not just a suboptimal response - The requirement applies regardless of conversation framing or model confidence

The customer support scenario from the beginning of this chapter lands squarely in the hook column. “Never process refunds above $500” is a business rule with a defined threshold. Its failure mode is a compliance violation. Probabilistic compliance is not acceptable. A PreToolUse hook on process_refund is the mechanism, and it has to be PreToolUse, because the hook has to run before the money moves.<sup>[2]</sup>

Note what the hook column does not say. It does not say hooks are better than prompts, and it does not say every hook is a gate. The axis is advisory against deterministic. A prompt instruction is a request, weighed by the model against everything else in its context and then acted on or not. A hook is a function, which runs whether or not the model agrees with it. A documented failure rate, whatever the number, is the tell: the instruction is being weighed rather than applied, and no amount of emphasis converts a weight into a guarantee. Capitalizing the instruction, repeating it, and adding examples of compliance all move the rate and none of them reach zero, because all three operate on the layer that produced the failures.<sup>[1]</sup>

The second axis runs inside the hook column, and it is the one this chapter opened on. Prevention and post-processing are different jobs that take different events. A rule that must stop something takes PreToolUse, because that is the only interception point that exists before the side effect. A rule that must hold of every artifact after the fact takes PostToolUse, because that is the only interception point that exists after the tool has produced something to fix. A formatting convention violated in some fraction of generated files is a post-processing problem, and running the formatter after every write reaches full compliance not by persuading the model but by making the model's compliance irrelevant to the outcome. Trying to prevent a formatting error before it exists is the wrong shape of fix. So is trying to prevent a refund that has already been paid.<sup>[2]</sup>

This distinction appears explicitly in Task Statement 1.4, which pairs programmatic enforcement against prompt-based guidance for workflow ordering, and states that where compliance has to be deterministic, an instruction on its own carries a failure rate above zero.<sup>[1]</sup> The exam is not asking whether prompts usually work. It is asking whether you understand that “usually” is architecturally insufficient for certain requirements.

There is a rung above the hook, and the same task statement puts it there deliberately. Enforcement strength is a function of distance: how far the rule sits from anything the agent's environment can reach. A prompt instruction lives in the context window, the layer the conversation is made of. A hook lives outside the model call and inside the agent's configuration, a layer someone can register another handler into, misconfigure, or fail to load. A threshold check written into the body of process_refund lives inside the tool, and there is no path from the agent to it at all: the call arrives, the function evaluates its own precondition, and a request over the limit returns a refusal instead of a payment. Where the requirement is that the rule holds no matter how the agent is prompted or configured, that is the placement being asked for, and the hook is the second-best answer rather than the answer.<sup>[22]</sup>

None of which retires the hook. Tool-internal enforcement protects one tool; a hook holds a rule across a set of them, including an MCP server whose implementation you do not own. Put the invariant that defines a tool inside the tool, and the policy that spans tools in the hook.

The redirect at the end of a block has a second half, and Task Statement 1.4 is specific about it. Denying a call and pointing the model at escalate_to_human is where enforcement ends and handoff begins, and a handoff carrying only a pointer is not a handoff. The human receiving the escalation does not have the conversation transcript. What the escalation call has to carry is the material that lets someone act without reading anything: the customer identity, the root cause, the amount or scope in question, and a recommended action. Persisting the transcript somewhere and passing along a reference to it moves the reading problem rather than solving it, and it is a durable wrong answer precisely because it looks like diligence.<sup>[22]</sup>

The same asymmetry governs when to escalate. Escalation is the right move once the authorization boundary is known to have been crossed, and it is right at that moment rather than after an attempt has been made and rejected: an attempt the agent already knows is unauthorized is not a check, it is a violation with a retry attached. Escalation is not the right move where the boundary was never the problem. If the agent has established eligibility and explained the charges and then the backend call fails, the resolved work is real. Handing back what was resolved, saying plainly that the last step did not complete, and offering the next options preserves that value. Discarding all of it because the final call failed does not.<sup>[22]</sup>

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

**Why:** Asking the model to perform format conversion is adding cognitive overhead to a task that code handles trivially. The hook runs in the orchestrator; the model processes clean data.<sup>[16]</sup>

### Pattern 3: Tone and Style Preferences

**Requirement:** Responses to customers should be warm and avoid technical jargon.

**Right approach:** System prompt. This is a style preference that requires model judgment to apply appropriately. No threshold, no determinism requirement, no compliance risk. A hook enforcing “warmth” would be both impossible and absurd.

**Wrong approach:** Writing a hook that pattern-matches output for technical terms and blocks responses containing them. This produces a brittle, false-positive-heavy system that cannot handle legitimate uses of technical language in technical contexts.

The hook is not smarter than the model on questions of warmth. Stop before you write one that tries to be.

---

## SubagentStart and SubagentStop

The SubagentStart and SubagentStop events connect the hook system to the multi-agent patterns from Chapter 2. When the coordinator spawns a subagent using the Agent tool, SubagentStart fires. When the subagent completes, SubagentStop fires.

These hooks give the parent orchestrator visibility into parallel subagent activity without having access to the subagent’s internal context. A SubagentStop hook can aggregate results, detect failures, or trigger coordinator logic when all parallel subagents have completed.<sup>[7]</sup>

One practical implication: subagents do not automatically inherit parent agent permissions. If the parent agent has a PreToolUse hook auto-approving certain tools, those hooks apply to the parent’s tool calls. The subagent runs in its own execution context with its own permission checks. When multiple subagents are spawned in parallel, each one may independently request permissions. Handle this by configuring permission rules that apply at the subagent session level, or use PreToolUse hooks scoped to run inside subagent sessions using the agent_type field on the hook input.<sup>[23]</sup>

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

The timeout field on a matcher is set in seconds. When it is omitted, the event's own default applies, and the defaults are not uniform: 600 seconds for most events, 30 for UserPromptSubmit, 10 for MessageDisplay, and a much shorter shutdown budget for SessionEnd, one and a half seconds unless configured otherwise. Raise it for hooks that make external API calls or database writes, and use AbortSignal in TypeScript to handle the cancellation gracefully.<sup>[24]</sup>

What happens when a callback overruns is also event-specific, and PreToolUse is the case with teeth. The callback is cancelled, its output discarded, and the tool call does not run: Claude receives a result saying the hook did not answer in time. PreToolUse fails closed, which is the right direction for an enforcement point and a surprise to anyone who assumed a slow logger would be skipped rather than block the call it was only observing. PostToolUse fails the other way, since the tool has already run and there is nothing left to withhold, and the result is kept.<sup>[24]</sup>

Hook event names are case-sensitive. PreToolUse works. preToolUse does not register correctly.<sup>[25]</sup>

---

## What the Exam Specifically Tests

The exam draws from Task Statements 1.4 and 1.5. Both task statements are built around the same conceptual axis: knowing when to put business logic in a prompt versus a hook.

Here is the question pattern you will see most often:

*A customer support agent has a policy rule about [threshold or prerequisite]. The current implementation uses [prompt instruction or hook]. The agent [sometimes violates / always enforces] the policy. What is the correct implementation?*

The answer key every time: hooks for thresholds, prerequisites, and compliance requirements; prompts for preferences, style, and requirements that require model judgment to apply contextually.

Secondary patterns:

- **PostToolUse for data normalization**: when multiple tools return different formats, the hook normalizes before the model processes. The prompt-based alternative (asking the model to convert formats) adds unnecessary reasoning load.
- **permissionDecisionReason to prevent retry loops**: pair it with deny to tell the model why an operation was blocked. Without the explanation the model retries; with it the model redirects. systemMessage is a separate channel that speaks to the operator.
- **Priority rules for multiple hooks**: one deny overrides any number of allow returns from other hooks. Write hooks to act independently; do not rely on hook execution order. A hook deny also survives bypassPermissions, and a hook allow does not defeat a deny rule in settings.
- **Async hooks for logging**: side effects that do not need to influence agent behavior should return async: true to avoid adding latency to every tool call.
- **MCP tool name pattern in matchers**: mcp__<server>__<action> is the format, and a bare server prefix takes the exact-string path and matches nothing, so a wildcard or an anchor is what turns it into a pattern.

One original sample question to test the concept:

**Q: A compliance team requires that all database write operations be preceded by a successful call to verify_access. An architect adds “always call verify_access before any write operation” to the system prompt. During testing, the agent occasionally performs writes without calling verify_access when the user provides authorization context in the conversation. What should the architect do?**

A. Add more detail to the system prompt instruction about when verify_access is required.  
B. Implement a PreToolUse hook on write operations that checks for a verified access token and denies the call if one is not present.  
C. Implement a PostToolUse hook on verify_access that blocks subsequent write calls until it fires.  
D. Use tool_choice: "any" to force the model to call a tool on every turn.

***The correct answer is B.** The requirement is a deterministic prerequisite gate. Hook-based enforcement is the correct mechanism. Option A fails because the problem is not vague instructions: it is probabilistic enforcement. Option C misapplies PostToolUse (it fires after verify_access, not before write operations). Option D forces tool use but does not address the ordering requirement.*

*When hooks are absent, tool descriptions become the only routing mechanism. Chapter 4 is about getting that mechanism right.*

---

## Key Takeaways

- Hooks execute in the orchestrator layer, outside the model call. Prompts execute inside the model call, as context. These are different execution environments with different guarantees.
- PreToolUse enforces policy, because it is the only interception point that exists before the side effect. It handles permission decisions (allow, deny, ask; defer in TypeScript only) and input modification via updatedInput. permissionDecisionReason is the field the model reads and the one that stops a denied call becoming a retry loop; systemMessage speaks to the operator instead.
- PostToolUse normalizes data and runs deterministic post-processing, because it is the only interception point that exists after the tool has produced something to fix. It transforms output via additionalContext (append) and updatedToolOutput (replace), and the canonical use case is normalizing heterogeneous data formats from multiple MCP tools before the model processes them. A rule that must prevent an action cannot live in the hook that fires once the action is done.
- When multiple hooks apply to the same event, they run in parallel. Deny wins over defer over ask over allow. A single deny blocks regardless of other hooks, and it holds even under bypassPermissions, because hooks are evaluated ahead of the rules and the mode. A hook allow is weaker than it looks: the deny and ask rules are still evaluated underneath it.
- Async hooks return async: true for fire-and-forget side effects. They cannot block, modify, or inject context; the agent has already proceeded by the time they complete.
- The PreCompact hook fires before context compaction, receiving a trigger field of "manual" or "auto". Use it to archive transcripts before summarization. The Prompt or Hook decision: use hooks when deterministic compliance is required; use prompts when model judgment is required to apply the requirement contextually. Above both sits the tool's own logic, which is where a rule belongs when it has to hold regardless of how the agent is prompted or configured, and beside the hook sits the declarative permission rule, which is where a condition belongs when it can be written down as a pattern rather than computed.
- MCP tools are addressed in hook matchers using the pattern mcp__<server>__<action>. Regex patterns like mcp__support_db__.* target all tools on a specific server. 

The model is not your enforcement layer. Your code is.
