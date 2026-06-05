# Chapter 11: Context Window Discipline

**Summary:** *The context window is finite, and its middle is unreliable. The lost-in-the-middle effect means information placed in the center of long inputs is less likely to influence model output, a structural property of how attention works across long sequences. That constraint has architectural consequences: how to position critical facts, how to manage tool output accumulation, when to compact or delegate, and how to design for sessions that outlive a single context window. This chapter covers all of it. The case facts block is the primary defense against progressive summarization: a structured, immutable reference section placed at the start of context and never compressed through summarization passes. Tool output accumulation is the dominant context cost in long sessions; a single verbose tool result can persist and consume space across every subsequent turn. The ToolSearch mechanism addresses MCP server overhead by loading tool schemas on demand rather than preloading all definitions. Subagent delegation keeps the coordinator’s context lean by absorbing exploration transcripts internally, returning only concise findings to the parent.*

---

## The Session That Ate Its Own Memory

The customer record was simple. John Smith. Account ACC-12345. Order #98765. Charged $150.00 instead of the promotional price of $99.99. An overcharge of exactly $50.01. Seven years as a customer. High priority.<sup>[1]</sup>

By the fifth turn of the conversation, after the first summarization pass, the record looked different. “Customer called about billing issue with promotion.” No name. No account number. No order number. No amounts.

By the tenth turn, after the second pass, it was four words: “Customer has a billing issue.”

The name, the account, the order, the exact dollar figures, the promotion code, the tenure, the priority rating: all gone. Not corrupted. Not misread. Gone, silently, through the perfectly normal operation of progressive summarization compressing context to make room for new turns.<sup>[1]</sup>

This is the problem. The context window is not just a size limit. **It is a degradation engine when left unmanaged.** And the model cannot tell you that it lost something, because it no longer knows the something existed.

---

## The Lost-in-the-Middle Effect

Context windows have a size. That size is real and fixed per model. But size is not the only structural property worth understanding.

Research into model behavior on long-context retrieval tasks reveals a consistent pattern: models reliably recall information positioned at the beginning and end of long inputs, and *they are significantly less reliable at recalling information placed in the middle*.<sup>[2]</sup> The effect is not about compression or summarization. It shows up even when the full context is present. It is a property of how attention mechanisms distribute weight across long sequences. The middle gets less.

This is what the domain calls the *lost-in-the-middle effect*. Name it, because once named it becomes an architectural constraint to design around rather than a vague “context problem” to worry about later.<sup>[1]</sup>

The practical implication: where you put information in context is not neutral. Placing a customer’s account number in turn 3 of a 40-turn conversation and expecting it to remain salient in turn 40 is not a safe assumption. Placing a 3000-token tool output in the middle of a 180K-token context is not the same as placing it at the beginning.

Position-aware ordering is the corrective. Put the most important information at the beginning and end of context. Organize aggregated results with explicit section headers so key findings surface at the top of each section. When preparing inputs for a synthesis step, put the summary first, then the detail.<sup>[2]</sup>

The effect is real. Plan for it.

---

## Progressive Summarization Risk

Summarization is useful. It compresses verbose history into manageable context, extends how long a session can run, and reduces the token cost of subsequent turns. The problem is not summarization. The problem is summarizing the wrong things.

Progressive summarization risk is what happens when numerical values, dates, amounts, account identifiers, and customer-stated expectations get flattened into vague prose through successive compression passes.<sup>[2]</sup> The John Smith example above is the canonical illustration. The summarizer is not broken. It correctly identifies that “*John Smith, ACC-12345, order #98765, charged $150 versus $99.99 promotional price*” is about a billing issue. It preserves the semantic category while discarding the specific facts inside it.

For a conversational system where the outcome is “*resolve this customer’s billing issue*,” those specific facts are not decoration. They are the task. The exact overcharge amount is what the refund calculation requires. The order number is what the lookup tool needs. The account ID is what the authentication check verifies. Strip those and the agent can no longer do its job, even if it still knows the category of job it is supposed to do.

The progressive nature of the risk makes it worse. A single summarization pass might preserve enough. Two passes almost certainly will not. By the third pass, even the category might collapse into something so vague (“billing matter under review”) that the agent cannot act on it at all.

The detection problem is quiet. The model does not announce that it has lost critical facts. It continues to respond confidently, often substituting “typical patterns” for the specific case details it no longer has. A support agent that has lost the order number will ask for it again, or worse, attempt to operate without it, producing plausible-sounding but factually wrong responses.

---

## The Case Facts Block

The architectural response to progressive summarization risk is not to avoid summarization. It is to protect what must not be summarized.

**The case facts block is a structured, immutable reference section placed at the beginning of context, outside the summarized history, that contains all critical transactional facts for the current session**.<sup>[1,2]</sup> It is never summarized. It is never compressed. It is present in every request, at the beginning where recall is highest.

For a customer support scenario, the block looks something like this:

```
## CASE FACTS (Do not summarize — reference directly)

| Field          | Value                        |
|----------------|------------------------------|
| Customer       | John Smith                   |
| Account ID     | ACC-12345                    |
| Order          | #98765                       |
| Expected Price | $99.99 (promotion SUMMER2026)|
| Charged Price  | $150.00                      |
| Overcharge     | $50.01                       |
| Customer Since | 2019                         |
| Priority       | High                         |

## RULES
- Refund amount ($50.01) is within $500 agent limit
- This case qualifies for immediate resolution [1]
```

The block includes the instruction to not summarize, not as a magic string the compactor recognizes, but because putting the intent in writing provides the compactor with context about how to treat it. The compactor reads your CLAUDE.md and your case facts block alike.<sup>[3]</sup>

The structure matters. Plain prose facts are easier to summarize into vague categories than a labeled table. A table resists compression because it presents information as data, not as narrative. The compactor has less to compress it into.

For multi-issue sessions, each distinct issue gets its own structured entry in the block, preserving all relevant specifics as a separate context layer from the summarized conversation history.<sup>[2]</sup>

The case facts block is a position and structure decision, not a prompt instruction. Prompt instructions asking the model to “*remember important details*” will not survive compaction. The block survives compaction because it lives outside the history being compacted, injected afresh into each request from persistent storage.

---

## What Consumes Context in the SDK

Understanding where context goes is prerequisite to managing it. The SDK accumulates context from multiple sources across a session, and not all of them are obvious.<sup>[3]</sup>

The system prompt loads with every request. Small fixed cost, but always present.

CLAUDE.md files load at session start via settingSources. If your project has a large CLAUDE.md or several nested files, they consume context on every request, every turn, before any conversation begins.<sup>[3]</sup>

Tool definitions load with every request. This is where architects most often underestimate cost. Every tool schema, field description, and example is part of the context every time Claude runs. For an agent with eight built-in tools and three MCP servers, the tool definition cost alone can be substantial.

MCP servers are the specific pressure point. Each MCP server adds all its tool schemas to every request.<sup>[3]</sup> A single large MCP server with many tools can consume thousands of tokens before the agent does any work. Two or three such servers, and the tool definition overhead becomes nontrivial. This is the context cost that *ToolSearch* addresses: instead of loading all tool definitions upfront, ToolSearch lets the agent discover and load tools on demand, turn by turn, reducing the baseline overhead to a manageable minimum.<sup>[3,4]</sup>

Conversation history accumulates across turns. Tool inputs and tool outputs both go into the history. This is the dominant cost in long-running sessions. A single verbose tool result from an order lookup that returns 40 fields when only 5 are relevant accumulates in the history and stays there, consuming space on every subsequent turn.<sup>[2]</sup>

The implication: trim tool outputs to relevant fields before they accumulate. A *PostToolUse* hook that strips irrelevant fields from verbose MCP tool responses is doing context hygiene, not just data normalization.<sup>[3]</sup>

Prompt caching handles the static pieces. Content that stays the same across turns, including the system prompt, tool definitions, and CLAUDE.md, is automatically prompt cached, which reduces cost and latency for repeated prefixes.<sup>[3]</sup> Caching does not reduce context window usage, but it does reduce the token cost and round-trip time for those fixed elements.

---

## Automatic Compaction and the compact_boundary Event

When the context window approaches its limit, the SDK automatically compacts the conversation. It summarizes older history to free space, keeping the most recent exchanges and key decisions intact. This is compaction as a default behavior, not something the architect has to implement.<sup>[3]</sup>

The SDK emits a signal when this happens. In Python, it appears as a SystemMessage with subtype: "compact_boundary". In TypeScript, it is a distinct SDKCompactBoundaryMessage type. Listening for this event tells you when compaction occurred, which is useful for logging, audit trails, and state management.<sup>[3]</sup>

The critical operational implication of automatic compaction: specific instructions from early in the conversation may not survive it. If you put deployment restrictions in the first user message and the context later compacts, those restrictions may disappear from the model’s working context. The model will not notice. It will continue running, now missing the constraint.

The fix is to *put persistent rules in CLAUDE.md*, loaded via settingSources, not in the initial prompt. CLAUDE.md content is re-injected on every request because it is loaded from the project configuration on each turn, not stored in conversation history. Instructions in CLAUDE.md survive compaction because they never get compacted.<sup>[3]</sup>

You can also include a summarization instructions section in CLAUDE.md telling the compactor what to preserve. The section header is not a magic string; the compactor matches on intent. The section can say something like “*Always preserve: customer account numbers, order IDs, specific dollar amounts, escalation decisions.*” The compactor reads this alongside everything else and uses it as guidance for what to keep.<sup>[3]</sup>

Manual compaction is available via */compact* sent as a prompt string. It is SDK input, not CLI shorthand. Send /compact as a prompt to trigger compaction on demand, before the automatic threshold hits, when you can see that the session is filling with verbose discovery output that is no longer relevant.<sup>[3]</sup>

The *PreCompact* hook fires before compaction and is where the full transcript can be archived before the SDK summarizes it. Chapter 3 owns the full hook mechanics. The relevant architecture point here is that compaction is a signal, not a failure, and the PreCompact hook is where the system can respond to that signal with preservation logic.<sup>[5]</sup>

---

## Context Degradation Symptoms

Context degradation is insidious because the model does not report it. It degrades gradually and silently, substituting general knowledge for specific session findings as the specific findings become less accessible.<sup>[1,2]</sup>

The two canonical symptoms:

Inconsistent answers. The model gives a different answer to the same question asked at turn 40 than it gave at turn 10. Not because anything changed, but because what was salient at turn 10 has drifted toward the middle of a 150K-token context and is receiving less attention.

“Typical patterns” substitution. Instead of referencing a specific finding from earlier in the session, the model references what is generally true about this kind of system. “Typically in this architecture…” instead of “Based on the auth module analysis from earlier…” This is the model doing the best it can with what is accessible, which is increasingly general knowledge rather than specific session work.<sup>[2]</sup>

Both symptoms are recognizable in practice. Both are architectural failures before they are model failures. The architecture let the context fill without mitigation, and the model’s reliability degraded as a result.

---

## Mitigations

Three structural mitigations, each addressing a different failure mode.

**Scratchpad files.** A scratchpad is a file the agent writes to during exploration, recording key findings as it discovers them. Before a verbose analysis phase, the agent creates a scratchpad. After each major finding, it updates the scratchpad. If context fills and compaction occurs, the agent re-reads the scratchpad to restore the specific findings that might otherwise have been lost to summarization. The scratchpad is not context. It is external state, persistent across context resets.<sup>[1,2]</sup>

The scratchpad pattern requires the agent to know it should use it. That instruction belongs in CLAUDE.md or in the system prompt, where it survives compaction.

**/compact on demand.** When the agent can detect that context is filling with verbose output that served its purpose and is no longer needed, triggering /compact before the automatic threshold clears that material while preserving a summary. Manual compaction gives more control over when compression happens and what the summary covers. Used in combination with summarization instructions in CLAUDE.md, it can produce more targeted compaction than the automatic behavior.<sup>[3]</sup>

**Subagent delegation.** This is the structural mitigation for verbose exploration phases. A subagent starts with a clean conversation: no prior message history, no accumulated tool outputs from the parent session. It does its work, and only its final response returns to the parent agent as a tool result.<sup>[3]</sup> The parent’s context grows by that summary, not by the full exploration transcript.

For a codebase analysis that requires reading dozens of files and following import chains across multiple passes, the verbose output of that exploration should stay inside a subagent. The coordinator gets the findings. The coordinator’s context stays lean, and the coordinator’s recall stays sharp across the full session.<sup>[2]</sup>

Subagent delegation does not mean the coordinator loses visibility. It means the coordinator’s context contains summaries and findings rather than exploration transcripts. That is the right shape.

---

## Crash Recovery Manifests

Long-running agent sessions introduce a failure mode that shorter sessions don’t have: the session can crash after significant work has been done, and none of that work is automatically recoverable.

The architectural response is crash recovery through structured state exports. Each agent exports its state to a known location at meaningful checkpoints, including what it has completed, what findings it has produced, and where it was in the task. The coordinator maintains a manifest: a structured file that describes what has been done and by which agents.<sup>[1,2]</sup>

On session resume, the coordinator loads the manifest and injects the relevant state summaries into each agent’s initial context. The agents do not start from scratch; they start from the checkpoint. The coordinator knows which phases are complete and which need to continue.<sup>[2]</sup>

The manifest is persistent external state, not conversation history. It survives crashes because it lives on disk. It survives context compaction for the same reason. A session that resumes with a well-designed manifest can continue as if the interruption were minor, because the critical state is not inside the context window at all.

Designing for crash recovery is designing against the assumption that sessions are atomic. For any agent session that runs for more than a few minutes and does meaningful work, that assumption will eventually be wrong. When the crash originates from a subagent failure, Chapter 12’s error propagation patterns govern how that failure is communicated before the session terminates.

---

## Context Window Sizes

The size of the ceiling matters when the work is large.

Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 have 1M-token context windows. Other Claude models, including Claude Sonnet 4.5, have 200K-token windows.<sup>[4]</sup>

A 1M-token window does not eliminate the need for context discipline. The lost-in-the-middle effect operates at 1M tokens. Tool output accumulation happens at 1M tokens. Compaction will still fire if the session runs long enough. But the ceiling being five times higher changes what is feasible in a single session and what requires subagent delegation or cross-session state management.

Claude Sonnet 4.6, Claude Sonnet 4.5, and Claude Haiku 4.5 feature context awareness, which means these models receive information about their remaining token budget after each tool call and can adapt their behavior accordingly. A model with context awareness can make informed decisions about depth of analysis as context fills, rather than operating blind until the compaction threshold hits.<sup>[4]</sup>

The model selection decision is also a context architecture decision. If a task genuinely requires holding 500K tokens of accumulated research in memory simultaneously, the choice of model is constrained by which models can accommodate that window.

---

## MCP Server Context Cost and ToolSearch

MCP server context cost deserves separate treatment because it catches architects who understand conversation history accumulation but underestimate the static overhead.

Every configured MCP server contributes all its tool schemas to every request. Not just the tools that will be called in this turn. All of them, every turn, before the conversation begins.<sup>[3]</sup> A server with 12 tools, each with detailed descriptions and parameter schemas, can consume thousands of tokens on every request. Three such servers, and the agent is starting each turn with significant context already spent before the first message.

ToolSearch is the architectural response. Instead of loading all tool definitions from configured MCP servers upfront, ToolSearch allows the agent to discover and load tools on demand. The agent asks ToolSearch what tools are available matching a query, gets back the relevant schema, and then calls the tool. The full schema for the remaining tools never enters the context window on turns where those tools are not needed.<sup>[3,4]</sup>

The tradeoff is one additional ToolSearch call per tool discovery event. For agents that use a predictable subset of a large MCP server’s tools on each turn, the token savings from not loading unused schemas can significantly outweigh the overhead of the discovery call.

For agents that genuinely use all available tools regularly, preloading may be more efficient. ToolSearch is a tool for the case where the tool set is large but usage is selective.

---

## Exam Sample Questions

*These questions are original constructions. They do not reproduce exam content.*

**Question 1.** **A multi-turn support agent is handling a billing dispute. By turn 15, the agent asks the customer for their account number, even though the customer provided it in turn 2. Which of the following best explains this behavior?**

A. The model exceeded its max_tokens limit and reset the conversation. 
B. Progressive summarization compressed the account number into a vague category description, and the case facts block was not used. 
C. The PostToolUse hook stripped the account number from the tool result before it was processed. 
D. The model’s context awareness feature determined the account number was no longer relevant.

***Correct answer: B.** Progressive summarization is the mechanism; absence of a case facts block is the architectural gap. A would manifest as an error, not continued operation. C describes a hook behavior, not a summarization effect. D mischaracterizes context awareness, which reports remaining token budget, not relevance.*

**Question 2.** **A codebase exploration agent is running a long multi-file analysis. After 30 turns, the agent begins referencing “typical patterns in this kind of codebase” rather than specific findings from its earlier file reads. Which mitigation most directly addresses this symptom?**

A. Increase the model’s effort setting to "max" to improve recall. 
B. Enable the PreCompact hook to archive transcripts before compaction fires. 
C. Have the agent maintain a scratchpad file recording specific findings and re-read it periodically. 
D. Reduce the number of tool calls per turn to limit context growth.

***Correct answer: C.** Scratchpad files externalize key findings to persistent state outside the context window, making them accessible even after compaction or middle-of-context drift. A affects reasoning depth, not recall. B archives the transcript but does not make findings accessible after compaction. D slows context growth but does not address the recall problem.*

**Question 3.** **A coordinator agent has access to three MCP servers, each with approximately 15 tool schemas. The coordinator uses only 2-3 tools per session. Which configuration reduces per-turn context overhead most effectively?**

A. Reduce the number of MCP servers from three to one. 
B. Use ToolSearch to load tools on demand rather than preloading all schemas. 
C. Set tool_choice: "none" to prevent tool calls from occurring unnecessarily. 
D. Increase the context window by switching to a 1M-token model.

***Correct answer: B.** ToolSearch loads tool schemas on demand, preventing unused schemas from consuming context on every request. A would require removing servers the agent legitimately needs. C prevents tool use entirely, which contradicts the agent’s function. D increases the ceiling but does not reduce the per-request overhead.*

---

## Key Takeaways

- The lost-in-the-middle effect is structural: information at the center of long inputs receives less attention than information at the beginning and end. Design context layout accordingly with critical facts at the top, key findings summarized at the top of each section, and immutable reference blocks anchored to the beginning.
- Progressive summarization destroys specific facts silently. The case facts block is the architectural counter: an immutable structured block outside summarized history, present in every request at the beginning of context, never compressed through summarization passes.
- Tool output accumulation is the dominant context cost in long sessions. Trim verbose outputs to relevant fields before they accumulate. MCP server schemas are a separate static cost that ToolSearch addresses through on-demand loading rather than preloading all definitions every turn.
- Automatic compaction handles the overflow but strips early instructions. Persistent rules belong in CLAUDE.md, loaded via settingSources, not in initial prompts that will not survive compression.
- **Three structural mitigations for context degradation**: ***scratchpad files*** for persisting findings across context resets, ***/compact on demand*** to clear verbose material when it has served its purpose, and ***subagent delegation ***to keep the coordinator’s context lean while verbose exploration happens elsewhere.
- Crash recovery through structured state exports means sessions can survive interruption. The manifest is external state, not conversation history; design for it when sessions are nontrivial.
- Context window size is a model selection variable. 1M-token models (Claude Opus 4.7, Opus 4.6, Sonnet 4.6) change what is feasible in a single session, but context discipline remains necessary regardless of ceiling height.
