# Chapter 1: The Agentic Loop

**Summary:** *Every agentic system built on Claude reduces to the same primitive: a loop that runs until **stop_reason** signals termination. The model does not decide to stop. The **stop_reason** field in the API response drives termination logic in the orchestrator. This chapter establishes that mental model, then builds the complete operational picture: how turns accumulate into sessions, what each stop_reason value means and demands from your code, how **ResultMessage** subtypes convey termination state in the SDK, and which knobs (**max_turns, max_budget_usd, effort, permissionMode**) give the orchestrator governance over a running loop. The chapter closes by naming the two **anti-patterns** the exam tests repeatedly: parsing natural language output to detect task completion, and substituting iteration caps for proper stop_reason-driven termination logic. Both failures stem from the same misunderstanding. The model does not signal completion in prose. It signals it in a field.*

---

## The Scene That Explains Everything

Two functions. One field. The contrast between them is the whole chapter.

```
# ANTI-PATTERN: parsing natural language
while True:
    response = get_response()
    text = response.content[0].text
    if "task complete" in text.lower():
        break
    if "I'm done" in text.lower():
        break

# CORRECT: check stop_reason field
while True:
    response = get_response()
    if response.stop_reason == "end_turn":
        break   # Claude decided it's done
    if response.stop_reason == "tool_use":
        execute_and_continue()
```

The anti-pattern is checking assistant text content to determine loop termination.<sup>[1]</sup> It feels reasonable until it isn’t: the model rephrases its completion signal, your substring match misses it, and the loop spins indefinitely. Or the model produces “I’ve completed the task as requested” in a tool-call response that needs further processing, and your matcher fires early.

The correct pattern reads a typed field. **stop_reason** is part of every successful Messages API response.<sup>[2]</sup> It does not appear in the prose. It is structural. The model does not embed completion status in language. The API emits it as a discrete value. This distinction is the foundation of every agentic architecture decision discussed in this book, and it is the first thing the exam tests.<sup>[1]</sup>

Name this: **stop_reason-driven termination**. The orchestrator loop continues when stop_reason is "**tool_use**" and exits when **stop_reason** is anything else. The exit branch may require different handling depending on which value arrived. But the fundamental control structure is keyed on that field, not on parsed text.

---

## The Loop at a Glance

Before the values, the shape. Every agent session follows the same cycle.<sup>[3]</sup>

1. **Receive prompt.** Claude receives the prompt, system prompt, tool definitions, and conversation history.
2. **Evaluate and respond.** Claude evaluates the current state and determines how to proceed. It may respond with text, request one or more tool calls, or both.
3. **Execute tools.** The SDK or your code runs each requested tool and collects results. Results feed back to Claude for the next decision.
4. **Repeat.** Steps 2 and 3 cycle. Each full cycle is one turn. Claude continues until it produces a response with no tool calls.
5. **Return result.** The loop ends. The SDK or your code delivers the final output.

A turn is one complete round trip: Claude produces output that includes tool calls, those tools execute, results return to Claude. Turns continue until Claude produces output with no tool calls, at which point the loop ends.<sup>[3]</sup>

The SDK handles this cycle automatically. A complex task (“refactor the authentication module and update the tests”) can chain dozens of tool calls across many turns, with the model adjusting its approach based on each result, without your orchestration code doing anything between turns.<sup>[3]</sup> This automation is the value. The loop manages itself. What you configure is its boundaries and permissions.

---

## The stop_reason Values

Seven values. Each demands a different response from your code.<sup>[2]</sup>

**"end_turn"**

The most common value. Claude finished its response naturally. The conversation can continue with a new user message, or you can take the final output and move on. This is the happy path.<sup>[2]</sup>

One edge case: sometimes the model returns an empty response with stop_reason: "end_turn" and no content. This typically occurs after tool results, when Claude interprets the assistant turn as already complete. The documented cause is mixing text content into the same user message as a tool_result block.<sup>[2]</sup> The fix is addressed in the next section on tool_result placement.

**"tool_use"**

Claude is requesting one or more tool calls and expects your code to execute them. The response contains tool_use blocks with the tool name and a JSON object of arguments. Your application extracts those arguments, runs the operation, and sends the output back in a tool_result block on the next request.<sup>[4]</sup> The loop continues.

This value drives the core loop continuation logic. While stop_reason == "tool_use", execute tools and continue. On any other value, exit (or handle the specific condition).<sup>[2]</sup>

**"max_tokens"**

Claude reached the max_tokens limit specified in the request. The response is truncated. If the truncated response contains an incomplete tool_use block, a retry with a higher max_tokens value is required to get the full tool call.<sup>[2]</sup> Your code must handle this as a boundary condition, not as task completion.

**"stop_sequence"**

Claude encountered one of your custom stop sequences. The response is complete up to that point. Useful for protocol-based conversations where delimiters signal state transitions.<sup>[2]</sup>

**"pause_turn"**

Returned when the server-side sampling loop reaches its iteration limit while executing server tools like web search or web fetch. The default limit is 10 iterations per request.<sup>[2]</sup> The work is not finished. Re-send the conversation, including the paused response, to let Claude continue where it left off. Any agent loop using server tools should handle this value.<sup>[4]</sup>

**"refusal"**

Claude declined to generate a response due to safety considerations. Your application should handle this case explicitly rather than treating it as a generic error.<sup>[2]</sup>

**"model_context_window_exceeded"**

Claude stopped because the model’s context window limit was reached. This value is available by default in Sonnet 4.5 and newer models; earlier models require a beta header to enable it.<sup>[2]</sup> The response is valid but limited by the context window, not by max_tokens. Knowing which ceiling was hit matters for the recovery strategy.

A complete dispatcher looks like this:

```python
def handle_response(response):
    if response.stop_reason == "tool_use":
        return handle_tool_use(response)
    elif response.stop_reason == "max_tokens":
        return handle_truncation(response)
    elif response.stop_reason == "model_context_window_exceeded":
        return handle_context_limit(response)
    elif response.stop_reason == "pause_turn":
        return handle_pause(response)
    elif response.stop_reason == "refusal":
        return handle_refusal(response)
    else:
        # end_turn and stop_sequence land here
        return response.content[0].text
```

*The same logic in TypeScript, for the SDK layer:*

```typescript
async function runLoop(prompt: string): Promise<string> {
  for await (const message of query({ prompt })) {
    if (message.type === "result") {
      if (message.subtype === "success") {
        return message.result;
      }
      throw new Error(`Loop ended: ${message.subtype}`);
    }
  }
  throw new Error("Stream ended without result");
}
```

The TypeScript SDK exposes termination state through ResultMessage.subtype, which provides a different vocabulary from the Messages API’s stop_reason. The two layers are related but distinct. The next section addresses the SDK layer specifically.

---

## ResultMessage and SDK Termination State

When using the Agent SDK, the loop’s termination state arrives in a **ResultMessage**, the final message emitted when the loop ends. The **subtype** field is the primary way to check termination state.<sup>[3]</sup>

**Five** subtypes:

| Subtype | Meaning | result field available? |
| --- | --- | --- |
| "success" | Claude finished the task normally | Yes |
| "error_max_turns" | Hit the maxTurns limit before finishing | No |
| "error_max_budget_usd" | Hit the maxBudgetUsd limit before finishing | No |
| "error_during_execution" | An error interrupted the loop (API failure, cancelled request) | No |
| "error_max_structured_output_retries" | Structured output validation failed after configured retry limit | No |

The result field (final text output) is only present on the "success" subtype. Always check the subtype before reading it.<sup>[3]</sup> All result subtypes carry **total_cost_usd, usage, num_turns, and session_id** so cost tracking and session resumption remain available even after an error.

A small note the docs flag explicitly: trailing system events such as **prompt_suggestion** can arrive after the ResultMessage. Iterate the stream to completion rather than breaking on the result.<sup>[3]</sup>

The ResultMessage also includes a **stop_reason** field (*str | None in Python, string | null in TypeScript*) indicating why the model stopped generating on its final turn. On error subtypes, stop_reason carries the value from the last assistant response before the loop ended. The two fields serve complementary purposes: subtype tells you whether the SDK loop succeeded or failed; stop_reason tells you what the model was doing when it ended.

---

## The tool_result Block Placement Rule

This is a short rule with a sharp consequence. After each tool call, the result must be appended to conversation history as a tool_result block in a new user-role message. Do not mix text content into the same user message as a tool_result block.<sup>[2]</sup>

The anti-pattern:

```
# INCORRECT: Adding text immediately after tool_result
{
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": "toolu_123", "content": "6912"},
        {"type": "text", "text": "Here's the result"},  # Don't do this
    ],
}
```

*The correct form:*

```
# CORRECT: Send tool results directly without additional text
{
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": "toolu_123", "content": "6912"}
    ],
}  # Just the tool_result, no additional text
```

Mixing text into the same block causes Claude to interpret the assistant turn as already complete, producing the empty end_turn response described earlier.<sup>[2]</sup> The SDK handles this correctly when you use its built-in tool execution. If you are driving the Messages API loop manually, this rule is a necessary discipline.

---

## Loop Control: Turns, Budget, and Effort

Three options govern how long a loop runs and how deeply it reasons. All three are fields on the options object passed to the SDK *query()* function.<sup>[3]</sup>

### max_turns and max_budget_usd

*max_turns (Python: max_turns; TypeScript: maxTurns)* caps the number of tool-use round trips. It counts tool-use turns only. Without a limit, the loop runs until Claude finishes on its own, which is fine for well-scoped tasks but can run long on open-ended prompts.<sup>[3]</sup> When the limit is reached, the SDK emits a ResultMessage with subtype *"error_max_turns".*

*max_budget_usd (Python: max_budget_usd; TypeScript: maxBudgetUsd) *caps accumulated cost. When the spend threshold is reached, the loop stops and emits *"error_max_budget_usd".* Setting a budget is a practical default for production agents.<sup>[3]</sup>

Here is the anti-pattern this pair corrects: using an iteration cap as the primary stopping mechanism. The reasoning is superficially similar to *max_turns,* but the intent is different. An iteration cap substitutes for proper stop_reason-driven control. The agent loop should exit because the model signals completion via *stop_reason,* not because your orchestrator counted to ten. *max_turns* is a safety net, not the primary exit condition.<sup>[1]</sup>

The exam tests this distinction.

### The effort Option

Effort controls how much reasoning Claude applies per turn. Lower effort levels use fewer tokens per turn and reduce cost.<sup>[3]</sup>

| Level | Behavior | Good for |
| --- | --- | --- |
| "low" | Minimal reasoning, fast responses | File lookups, listing directories |
| "medium" | Balanced reasoning | Routine edits, standard tasks |
| "high" | Thorough analysis | Refactors, debugging |
| "xhigh" | Extended reasoning depth | Coding and agentic tasks; recommended on Opus 4.7 |
| "max" | Maximum reasoning depth | Multi-step problems requiring deep analysis |

The Python SDK leaves effort unset when not specified, deferring to the model’s default behavior. The TypeScript SDK defaults to "high".<sup>[3]</sup>

Extended thinking is a separate feature from effort. They are independent: effort: "low" with extended thinking enabled is a valid configuration, as is effort: "max" without it.<sup>[3]</sup>

Effort can be set at the session level in query() options, or per subagent via the effort field on AgentDefinition to override the session level. The use of subagents as a composition pattern is covered in Chapter 2 (Coordinator plus subagents).

---

## Permission Mode

The permissionMode option (Python: permission_mode; TypeScript: permissionMode) controls whether the agent asks for approval before using tools that are not covered by explicit allow or deny rules.<sup>[3]</sup>

Four modes:

**"default"**

Tools not covered by allow rules trigger the approval callback. No callback means deny. This is the **interactive mode: a human-in-the-loop configuration** **where the agent surfaces ambiguous actions for review.**<sup>[3]</sup>

**"acceptEdits"**

**Auto-approves file** **edits and common filesystem commands** (*mkdir, touch, mv, cp,* and similar). Other Bash commands follow default rules. The practical choice for autonomous agents on a development machine where file modification is expected but arbitrary shell execution is not.<sup>[3]</sup>

**"plan"**

**Read-only tools run.** Claude explores and produces a plan without editing source files. The agent can read, search, and analyze, but write operations are blocked. Use this when you want Claude’s analysis and recommendations before committing to changes. This value is applied in plan mode workflows covered in Chapter 7 (Explicit-intent execution modes).

**"bypassPermissions"**

Never prompts. **All tool calls execute without approval regardless of other settings.** Reserve this for CI environments, containers, or other isolated environments where no human is available for approval.<sup>[3]</sup> This value is the **recommended configuration for CI/CD integration**, covered in Chapter 8 (Review-session isolation).

Note that *permissionMode*: "acceptEdits" does not auto-approve MCP tools; permissionMode: "bypassPermissions" does, because it disables all safety prompts.<sup>[3]</sup> Chapter 5 covers the specific implication for MCP tool permissions.

---

## What the Loop Looks Like End-to-End

Here is the complete Python loop with all the concepts from this chapter integrated. This is the pattern the exam expects you to recognize and evaluate.

```python
import anthropic
from claude_agent_sdk import query, ResultMessage

client = anthropic.Anthropic()
tools = [{"name": "lookup_customer", "description": "...", "input_schema": {}}]
messages = [{"role": "user", "content": "Find customer John Smith"}]

# The Agentic Loop
while True:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        tools=tools,
        messages=messages
    )

    # KEY: Check stop_reason to control the loop
    if response.stop_reason == "end_turn":
        break  # Claude is done

    if response.stop_reason == "tool_use":
        tool_block = next(
            b for b in response.content if b.type == "tool_use"
        )
        result = execute_tool(tool_block.name, tool_block.input)

        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": [{"type": "tool_result",
                         "tool_use_id": tool_block.id,
                         "content": result}]
        })
```

*The equivalent TypeScript loop using the Agent SDK:*

```typescript
import { query, ResultMessage } from "@anthropic-ai/claude-agent-sdk";

async function runAgent(prompt: string): Promise<void> {
  for await (const message of query({
    prompt,
    options: {
      maxTurns: 20,
      maxBudgetUsd: 5.0,
      permissionMode: "acceptEdits",
      effort: "high",
    },
  })) {
    if (message.type === "assistant") {
      // Handle progress: see which tools Claude called this turn
    } else if (message.type === "result") {
      const result = message as ResultMessage;
      if (result.subtype === "success") {
        console.log(result.result);
      } else {
        console.error(`Loop ended: ${result.subtype}`);
      }
      break;
    }
  }
}
```

Three things to observe in both versions. First: loop continuation is keyed on stop_reason (Messages API) or subtype (SDK). Second: tool results are returned as isolated tool_result blocks in new user-role messages. Third: the loop does not inspect the text of Claude’s responses to decide what to do next. The control flow is structural, not semantic.

---

## The Two Anti-Patterns, Named

The exam names these explicitly and tests them repeatedly. They have distinct mechanics, but both originate from the same misunderstanding: the belief that the model communicates completion intent through the content of its text output.

### Anti-Pattern 1: Parsing Natural Language for Completion Signals

```
# This will eventually fail
if "task complete" in response.content[0].text.lower():
    break
if "I'm done" in text.lower():
    break
```

The model does not guarantee consistent phrasing. Any phrase you match, Claude will eventually rephrase. Worse: Claude might produce completion-sounding language mid-task, in a response that also contains tool calls. The structural stop_reason field is the reliable signal. It is a typed enum value, not a substring in prose.<sup>[1,2]</sup>

### Anti-Pattern 2: Arbitrary Iteration Caps as Primary Control

```
max_iterations = 10
for i in range(max_iterations):
    response = get_response()
    # process...
```

Iteration caps are reasonable safety rails. They are not loop termination logic. A cap that fires before the model signals completion interrupts a working session. A cap set high enough to never interfere provides no safety value. The correct pattern uses stop_reason for primary control and max_turns (or max_budget_usd) as boundary conditions: the loop exits normally on "end_turn" and receives a ResultMessage with "error_max_turns" only if the task ran unexpectedly long.<sup>[1,3]</sup>

Both anti-patterns appear as exam distractors. They look reasonable to candidates with incomplete knowledge of the API. Knowing why they fail is the knowledge the exam is measuring.

---

## The Architecture Implication

The loop is not Claude’s loop. The model generates responses; the orchestrator runs the loop.

This is a meaningful distinction for system design. Claude does not execute tools. It emits a structured request describing what tool to call with what arguments. Your application (or the SDK on your behalf) executes the tool and returns the result. Claude never sees your implementation; it only sees the schema you provided and the result you returned.<sup>[4]</sup>

That execution model means the **orchestrator owns the termination logic.** Claude cannot break out of your loop. It cannot decide the session is over in any way that bypasses your code. The stop_reason field is Claude’s communication channel for expressing completion intent. Your code reads it and acts.

This separation is why the two anti-patterns are architectural failures, not just coding mistakes. Parsing text for completion signals outsources loop control logic to the model’s prose style. That is the wrong layer. The stop_reason field exists precisely to give the orchestrator a reliable, structured, parseable signal.

Everything else in this book builds on that separation: hooks that intercept tool calls in the orchestrator layer (Chapter 3), multi-agent systems where one orchestrator drives subagent loops (Chapter 2), context management that shapes what Claude sees on each turn (Chapter 11). The primitives are all correct loop mechanics at the foundation.

---

## Key Takeaways

- **stop_reason-driven termination** is the fundamental control pattern: continue on* "tool_use", exit on "end_turn", handle "max_tokens", "pause_turn", "refusal", "stop_sequence", and "model_context_window_exceeded"* as distinct conditions.
- The **ResultMessage.subtype** field in the Agent SDK provides termination state at the session level: "success" carries the final text output; the four error subtypes do not include a result field.
- **tool_result** blocks belong in isolated user-role messages; mixing text content into the same block causes empty end_turn responses.
- **max_turns** and **max_budget_usd** are safety bounds, not primary loop control; using an iteration cap as the main exit mechanism is a named anti-pattern.
- The **effort** option controls reasoning depth per turn across five levels; the **TypeScript SDK defaults to "*high*", the Python SDK leaves it *unset*.**
- The four permissionMode values ***("default", "acceptEdits", "plan", "bypassPermissions"***) map to four distinct trust contexts: interactive human approval, autonomous dev machine operation, read-only planning, and CI/container execution.
- **The model does not execute tools and cannot break out of the orchestrator’s loop; stop_reason is the only structural channel for signaling completion intent.**
