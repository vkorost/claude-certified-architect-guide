# Chapter 2: Hub-and-Spoke

**Summary:** *A single agentic loop handles a surprising range of problems. Then it hits a wall. The task decomposes. Different subtasks need different tools, different context windows, or different permission boundaries. **The coordinator-plus-subagents** **pattern** is the documented answer to that wall.*

*This chapter establishes the hub-and-spoke topology: a coordinator agent that owns decomposition, delegation, and result aggregation, surrounded by specialized subagents that each execute one focused job in an isolated context. It covers the **Agent tool** (previously Task) that makes subagent invocation work, the **AgentDefinition** fields that configure each subagent, and the precise mechanics of what context a subagent inherits and does not inherit from its parent. It also names and dissects the two failure modes the exam reliably tests: overly narrow decomposition creating gaps between agents, and the coordinator that routes every query through every subagent instead of selecting dynamically.*

*The multi-agent research system from Scenario 3 of the exam guide serves as the recurring lens throughout.*

---

## The Moment Decomposition Happens

The scenario is specific.<sup>[1]</sup> A coordinator agent receives a research topic. It needs to search the web, analyze source documents, synthesize findings, and generate a formatted report. Four distinct jobs. Different tool sets for each. Document analysis benefits from deep focus on a narrow set of sources without the noise of raw web results. Synthesis needs the outputs of both prior stages but must not re-run the searches itself. Report generation needs only the synthesis, clean.

One agent doing all four jobs in a single loop accumulates context from every step. By the time synthesis runs, the context window holds raw search results, document excerpts, intermediate analysis notes, tool call logs, and the original query. Most of that is noise for the synthesis task. The model attends to all of it anyway.

*You cannot unsee context once it is there.* The only way to keep synthesis clean is to ensure it starts with a clean context. The only way to do that is a subagent.

This is the moment decomposition happens. Not when the task is complex. When different stages of the same task would contaminate each other if they shared a context.

---

## The Hub-and-Spoke Topology

The pattern has a name: **Coordinator plus subagents**.<sup>[1,2]</sup> The coordinator is the hub. Subagents are the spokes. All communication between subagents routes through the coordinator. Subagents do not talk to each other directly. They do not share state. They do not see each other’s outputs unless the coordinator explicitly passes those outputs in a subsequent delegation prompt.

This topology gives the coordinator complete observability. Every result, every failure, every partial finding passes through one place. The coordinator can evaluate synthesis quality and re-delegate to search or analysis subagents with targeted follow-up queries before invoking synthesis again. It can handle a subagent failure without the other agents knowing anything went wrong.<sup>[2]</sup>

The flat alternative (agents sharing a global state or full conversation history) removes that control surface. When one agent’s noisy output pollutes another agent’s context, the errors are hard to isolate and harder to reproduce.

The coordinator carries four responsibilities that belong to it alone and to nothing else in the system.<sup>[2]</sup>

**Decomposition.** The coordinator reads the incoming query and decides how to split the work. It identifies which subtasks are independent (can run in parallel), which are sequential (stage B needs stage A’s output), and whether any subtasks are unnecessary for this particular query. A query about a well-documented public API might not need document analysis at all. The coordinator figures that out before delegating anything.

**Delegation.** The coordinator dispatches subtasks to the appropriate subagents. It writes the prompt string for each invocation, deciding exactly what context and instructions to include. This is the most consequential decision the coordinator makes: too little context and the subagent works blind; too much context and you are back to the noise problem the subagent was supposed to solve.

**Result aggregation.** The coordinator receives all subagent outputs and assembles them into a coherent result. In the research system, this means taking structured findings from multiple analysis passes and passing them to the synthesis subagent in a form that synthesis can use. The coordinator does not just concatenate outputs. It curates them.

**Dynamic subagent selection.** The coordinator decides which subagents to invoke based on the query’s actual requirements, not on a fixed pipeline sequence. This is not optional behavior. It is the mechanism by which the hub-and-spoke topology delivers any advantage over a fixed sequential chain.

---

## The Agent Tool

Subagents are invoked via the Agent tool.<sup>[3]</sup> For a coordinator to spawn subagents, "Agent" must appear in its allowedTools. Without it, the coordinator has no mechanism to delegate.

One version note matters for production code and for the exam. The tool was renamed from "Task" to "Agent" in Claude Code v2.1.63. Current SDK releases emit "Agent" in tool_use blocks, but still use "Task" in the system:init tools list and in result.permission_denials[].tool_name. Checking both values in block.name ensures compatibility across SDK versions.<sup>[3]</sup>

When you see an exam question about why a coordinator is not spawning subagents, the first thing to verify is whether "Agent" (or "Task", depending on SDK version) appears in allowedTools. It is the most common single-field oversight.

---

## AgentDefinition: What You Configure

Each subagent is described by an **AgentDefinition object** passed in the agents parameter of your query() call. Two fields are required.<sup>[3]</sup>

**description** is a natural language statement of when to use this agent. Claude reads it and decides whether to invoke the agent. Write it the way you would write a tool description: specific, bounded, clear about what this agent handles versus what it does not. The coordinator selects subagents based on descriptions. A vague description produces unpredictable selection.

**prompt** is the subagent’s system prompt. This is where the agent’s specialized instructions live: its role, its constraints, its output format requirements, its tool-use preferences. The more focused this prompt is on the subagent’s specific job, the better.

The optional fields give you per-agent configuration:<sup>[3]</sup>

- **tools**: array of allowed tool names. If omitted, the subagent inherits all tools available to the coordinator. Specify this to restrict the subagent to only the tools it needs.
- **disallowedTools**: tools to remove from the agent’s set, useful when inheriting by default but wanting to block specific capabilities.
- **model**: override the model for this agent. Accepts aliases (*'sonnet', 'opus', 'haiku', 'inherit'*) or a full model ID. Use a lighter model for a fast web-search pass; use a heavier model for the synthesis step that requires careful reasoning.
- **skills**: skill names to preload into the agent’s context at startup.
- **memory**: memory source for this agent ('user', 'project', or 'local').
- **mcpServers**: MCP servers available to this agent, by name or inline configuration. gd
- **maxTurns**: maximum agentic turns before the agent stops.
- **background**: run this agent as a non-blocking background task.
- **effort**: reasoning effort level ('low', 'medium', 'high', 'xhigh', 'max', or a number).
- **permissionMode**: permission mode for tool execution within this agent. (The values are defined in Chapter 1.)

The research system example makes the model field concrete. The web-search subagent runs fast and wide; you might use a lighter, faster model and accept some reduction in analysis depth. The synthesis subagent integrates findings from multiple sources and needs to reason carefully; that is a case for a more capable model. The coordinator pays the cost of an extra API call either way. You get to tune that tradeoff per agent.

---

## What a Subagent Inherits (and What It Does Not)

This is the most exam-tested mechanical detail in the chapter. Memorize both sides of the table.<sup>[3]</sup>

A subagent receives:

- Its own system prompt (AgentDefinition.prompt)
- The Agent tool’s prompt string (the task description passed at invocation time)
- The project *CLAUDE.md* (loaded via settingSources)
- Its defined tool set (inherited from parent, or the subset specified in tools)

A subagent does not receive:

- The parent’s conversation history
- The parent’s tool results
- The parent’s system prompt
- Any skill content not listed in AgentDefinition.skills

The only channel from coordinator to subagent is the Agent tool’s prompt string.<sup>[3]</sup> If the synthesis subagent needs the web search results, the coordinator must pass those results explicitly in the prompt it sends to the synthesis agent. There is no automatic inheritance. There is no shared memory. There is no side channel.

This constraint is a feature, not a limitation. It is what makes context isolation work. The synthesis subagent starts with exactly the context the coordinator decided it should have: the distilled outputs of the search and analysis stages, formatted and targeted for synthesis. No raw tool call logs. No intermediate reasoning artifacts. No irrelevant prior turns.

The research system walkthrough from the exam scenarios puts it directly: pass only the context relevant to each subagent’s specific task. Sharing the full coordinator conversation history wastes tokens and introduces irrelevant information into the subagent’s reasoning.<sup>[1]</sup>

---

## Context Passing: The Rule and Why It Exists

The rule is simple to state. **Pass only context specific to each subagent’s task.**<sup>[</sup><sup>1,2]</sup> The rationale is more interesting.

A coordinator running a multi-topic research job may have, by the time it invokes synthesis, a context that includes: the original query, a planning pass, web search results for three distinct subtopics, document excerpts from seven sources, intermediate analysis notes for each subtopic, and several rounds of quality checks. That is a realistic coordinator context for a serious research workflow.

The synthesis subagent does not need the planning pass. It does not need the raw search queries. It does not need the quality check history. It needs the analysis outputs: structured findings with source attribution, organized for synthesis.

Passing the full coordinator history to the synthesis agent would mean the synthesis model attends to the planning pass and the raw search queries alongside the analysis outputs. Those earlier elements do not help synthesis. They add noise. They consume tokens that could go to the actual findings. And they create confusing signal: the synthesis agent might repeat reasoning the coordinator already completed, or hedge against questions that were already resolved upstream.

The synthesis agent does not need to know how the search was done. It needs to know what it found.

The structured data approach makes context passing precise. When passing outputs from one subagent to the next, separate content from metadata. Findings go in one field. Source URLs, document names, and page numbers go in another. This preserves attribution through synthesis without embedding source metadata inline in the findings text, where it disrupts reading and confuses the model’s summarization logic.<sup>[2]</sup>

---

## Parallel Execution

Multiple subagents can run concurrently.<sup>[3]</sup> The mechanism is straightforward: emit multiple Agent tool calls in a single coordinator response. The SDK runs them in parallel. The coordinator receives all results before continuing.

The research system has a natural parallel phase. Web search and document analysis can run simultaneously. Neither depends on the other’s output. Synthesis depends on both. The coordinator emits Agent tool calls for the search subagent and the analysis subagent in the same response turn; both run in parallel; the coordinator receives both results; then it invokes synthesis with both outputs as input.

The contrast is sequential invocation, where the coordinator emits one Agent call, waits for the result, then emits the next. Sequential invocation is correct when stage B genuinely depends on stage A’s output. For stages that are independent, parallel invocation reduces end-to-end latency proportionally to the number of agents running in parallel.

The exam question on this topic tests whether you know the mechanism. Parallel subagents require multiple Agent tool calls in a single coordinator response, not multiple calls across separate turns.<sup>[2]</sup>

The background field in AgentDefinition extends this further. Setting background: true runs a subagent as a non-blocking background task. The coordinator does not wait for the background subagent’s result before continuing its own execution. This is useful when a long-running analysis can proceed while the coordinator handles other coordination work, as long as the coordinator does not need that subagent’s output for the next immediate step. The coordinator can retrieve the result later when it is ready to proceed to synthesis.<sup>[3]</sup>

The detection side of parallel execution matters for orchestration code that needs to track which subagents have been invoked. Subagents are invoked via the Agent tool, so detecting subagent invocation means checking for tool_use blocks where name is "Agent". Messages originating from within a subagent’s context include a parent_tool_use_id field that links back to the parent’s invocation. In Python, content blocks are accessed directly via message.content. In TypeScript, SDKAssistantMessage wraps the Claude API message, so content is accessed via message.message.content.<sup>[3]</sup>

---

## The No-Nesting Rule

**Subagents cannot spawn their own subagents.** The Agent tool must not be in a subagent’s tools array.<sup>[3]</sup>

The architecture is flat at depth one. The coordinator is the only agent that delegates. Subagents execute and return results. If you need a subagent to do something that itself requires a subagent, the correct design is to have the coordinator decompose that work into two separate subagent invocations and manage the dependency explicitly.

This constraint matters for exam questions that present a scenario where a deep-research subagent is given the Agent tool so it can spawn its own search agents. The answer is: do not do this. The coordinator owns delegation. Giving delegation capability to a subagent removes the observability that makes hub-and-spoke valuable in the first place.

---

## Coordinator Prompt Design

The exam guide’s task statements are precise about what a coordinator prompt should contain.<sup>[2]</sup> It specifies **research goals and quality criteria**. It does not enumerate step-by-step procedural instructions.

The distinction matters operationally. A coordinator prompt that says “*first call the web-search subagent, then call the document-analysis subagent, then call synthesis*” is a static pipeline. It runs the same sequence regardless of the query. A simple factual question gets the full pipeline including a document analysis pass that contributes nothing. A query where the web search returns no useful results still routes to synthesis with empty inputs.

A coordinator prompt that specifies goals and quality criteria lets the coordinator decide which subagents to invoke based on what the query actually requires. For a simple factual question, the coordinator might invoke only the web-search subagent and skip document analysis entirely. For a query where preliminary synthesis reveals gaps, the coordinator re-delegates to search and analysis with targeted follow-up queries before re-invoking synthesis.<sup>[2]</sup>

Dynamic selection is the coordinator’s core capability. The coordinator does not route all queries through all subagents. It reads the query, reads the description fields of available subagents, and selects the subagents the query requires.

---

## Two Anti-Patterns the Exam Tests

Both anti-patterns have names in the registry, and both come from the exam guide’s task statements.<sup>[2]</sup>

**Overly narrow task decomposition creating coverage gaps.** The research system has four subagents. If the coordinator assigns each agent a scope so narrow that some aspects of a broad research topic fall between agents, findings have holes. The web-search agent covers recent publications. The document-analysis agent covers the provided source list. If the query asks about a topic where relevant findings exist in both sources that cross-reference each other, and neither agent is configured to notice the cross-reference pattern, the synthesis agent never sees the connection.

Narrow decomposition is sometimes presented as a virtue (focus, specialization). It becomes an anti-pattern when the task boundaries do not align with the natural structure of the work. The fix is to design agent scopes around the shape of the information, not around a desire for symmetry in the agent count.

**Routing all queries through all subagents.** The opposite failure. The coordinator always runs the full pipeline regardless of what the query requires. A question that only needs web search runs document analysis anyway. A question that is already well-covered by preliminary synthesis triggers another full search-and-analysis pass. This wastes tokens, increases latency, and can introduce noise when subagents return results for tasks that are not relevant to the query.

The coordinator’s selection logic is not overhead. It is the mechanism by which the hub-and-spoke topology delivers its efficiency.

Both anti-patterns share a root cause: the coordinator is not reasoning about the query. It is executing a fixed procedure. The whole point of putting an LLM in the coordinator role is that it can read the query, assess what the query needs, and choose accordingly.

---

## Three Ways to Create Subagents

The SDK supports three creation paths.<sup>[3]</sup>

**Programmatic definition** (the recommended path): pass AgentDefinition objects in the agents parameter of query(). This is the approach this chapter has been describing. Definitions live in code, are version-controlled, and can be constructed dynamically at runtime based on query characteristics or runtime configuration.

**Filesystem-based definition**: define subagents as markdown files in .claude/agents/ directories. The file name becomes the agent name. These are loaded at startup. If you create a new agent file while the session is running, restart the session to load it. Programmatically defined agents take precedence over filesystem-based agents with the same name.

**Built-in general-purpose subagent**: Claude can invoke the built-in general-purpose subagent at any time via the Agent tool without any custom definitions, as long as "Agent" is in allowedTools. This is useful for delegating exploratory research or quick analysis tasks without defining specialized agents upfront.

One practical note on agent creation that appears in the troubleshooting documentation: if Claude completes tasks directly instead of delegating to your defined subagent, verify that the Agent tool is in allowedTools, and verify that the subagent’s description field clearly explains when to use it. The description is how Claude matches tasks to the right subagent. A vague description produces missed delegations.<sup>[3]</sup>

---

## Context Isolation as a Context Window Management Tool

There is a secondary benefit to subagent context isolation that Chapter 11 covers at length. The short version belongs here because it connects directly to why the research system design makes sense.

Intermediate tool calls stay inside the subagent. The parent receives only the subagent’s final message.<sup>[3]</sup> A document-analysis subagent might read and process fifty source files, generating extensive intermediate reasoning across many turns. None of that intermediate content appears in the coordinator’s context. The coordinator receives the analysis summary: structured findings, source attribution, confidence levels. That is a few kilobytes instead of the megabytes of raw file content and processing steps.

This is not just an efficiency gain. It is what makes iterative refinement feasible. The coordinator can invoke synthesis, evaluate the output, identify a gap, re-delegate to document analysis with a targeted query, receive the additional findings, and re-invoke synthesis with the enriched inputs. Each pass keeps the coordinator’s context manageable because the subagents absorb the exploration cost internally.

The research system’s “re-delegate until coverage is sufficient” loop only works because the coordinator is not paying the context cost of each subagent’s internal work.<sup>[2]</sup>

---

## The Research System as a Design Walkthrough

Scenario 3 is worth walking through as a complete design exercise, because it is the scenario where all the concepts in this chapter intersect.<sup>[1,2]</sup>

The system has four subagents: a web-search agent, a document-analysis agent, a synthesis agent, and a report-generation agent. The coordinator receives a research topic and produces a comprehensive cited report.

The web-search and document-analysis agents are independent. They can run in parallel. The coordinator emits both Agent tool calls in the same response turn. Each agent runs in its own fresh context, with only the tools it needs (the web-search agent has web access; the document-analysis agent has file read access and no web tools) and a prompt that includes only the research topic and any source constraints specific to that agent’s job. Neither agent sees the other’s work while it runs.

The synthesis agent depends on both prior agents. It cannot run until the coordinator has both sets of results. The coordinator collects the web-search findings and the document-analysis findings, formats them as structured inputs (findings with source attribution separated from raw excerpts), and passes that prepared context in the prompt to the synthesis agent. The synthesis agent receives exactly the distilled outputs it needs. It does not receive the raw search query, the intermediate tool call logs from the document analysis, or any other coordinator history.

The report-generation agent depends on synthesis output only. It receives the synthesis result and is responsible for formatting: citation formatting, section organization, executive summary generation. Its prompt specifies output format requirements in detail so the coordinator does not need to post-process the report.

Now consider what the coordinator does when the synthesis output is incomplete. The coordinator evaluates the synthesis result against the research topic. If it identifies a gap (a subtopic not covered, a claim not substantiated), it re-delegates to the web-search or document-analysis agent with a targeted follow-up query, waits for the result, and re-invokes synthesis with the original findings plus the supplementary findings. This iterative refinement loop is feasible because the coordinator’s context window stays manageable: each subagent invocation absorbs its own tool call costs internally, returning only a concise structured result to the coordinator.<sup>[2]</sup>

The design decisions that make this work:

- Parallel execution for independent stages reduces latency without coordination overhead.
- Targeted context passing for each subagent prevents noise and keeps each agent’s context window focused on the work it is doing.
- Iterative refinement rather than a single-pass pipeline improves result quality without requiring the coordinator to manage the full search-and-analysis context across multiple passes.
- Dynamic subagent selection lets the coordinator skip stages that are not needed for a specific query, rather than running the full pipeline every time.

This is hub-and-spoke applied correctly. The coordinator thinks. The subagents execute. The results flow back through the hub.

---

## What the Exam Actually Tests

Scenario 3 is the hardest scenario in the exam, according to the walkthrough documentation.<sup>[1]</sup> The traps are documented explicitly.

Sharing full coordinator context with subagents is always wrong. The correct approach is to pass only context relevant to each subagent’s task.<sup>[1]</sup>

Silently dropping subagent failures is always wrong. Structured error propagation requires reporting what was attempted, what failed, and whether the failure was an access error or a valid empty result. Chapter 12 covers the access-failure-versus-empty-result distinction in detail.

Ignoring provenance when resolving conflicting findings from different subagents is always wrong. When two subagents return conflicting statistics, the coordinator annotates the conflict with source attribution and publication dates rather than arbitrarily selecting one value. Chapter 12 covers this as the DataWithProvenance pattern.

The hub-and-spoke architecture questions on the exam follow a pattern. The correct answers involve a coordinator that selects subagents dynamically, passes targeted context, routes all inter-agent communication through itself, and handles failures with structured error reporting. The wrong answers involve flat architectures, full context sharing, static pipelines, and silent error swallowing.

The topology is simple. The discipline required to implement it correctly is not.

---

## Key Takeaways

- The coordinator-plus-subagents pattern emerges when a task decomposes into subtasks that would contaminate each other if they shared a context; context isolation is the primary reason to reach for this topology, not task complexity alone.
- The Agent tool (formerly Task before Claude Code v2.1.63) is the invocation mechanism; it must appear in the coordinator’s allowedTools, and checking both "Agent" and "Task" in block.name ensures cross-version compatibility.
- A subagent receives only its AgentDefinition.prompt, the Agent tool’s prompt string, the project CLAUDE.md, and its defined tool set; it receives no parent conversation history, no parent system prompt, and no parent tool results.
- Pass only the context each subagent needs for its specific task: sharing the full coordinator history wastes tokens, introduces noise, and is the most common wrong answer on Scenario 3 questions.
- Emit multiple Agent tool calls in a single coordinator response to run subagents in parallel; sequential invocation across separate turns is the wrong mechanism for independent stages.
- Subagents cannot spawn subagents; the Agent tool must not appear in a subagent’s tools array.
- The two named anti-patterns are overly narrow decomposition (creating coverage gaps between agents) and routing all queries through all subagents (static pipeline instead of dynamic selection).
