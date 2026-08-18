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

The pattern has a name: **Coordinator plus subagents**. The exam guide calls the shape hub-and-spoke, and it hands the coordinator agent three things at once: every channel between subagents, the error handling for all of them, and the routing of information from one to the next.<sup>[2]</sup> The coordinator is the hub. Subagents are the spokes. All communication between subagents routes through the coordinator. Subagents do not talk to each other directly. They do not share state. They do not see each other’s outputs unless the coordinator explicitly passes those outputs in a subsequent delegation prompt.

This topology gives the coordinator complete observability. The guide states the reason plainly: routing every subagent exchange through the coordinator is what buys observability, consistent error handling, and controlled information flow.<sup>[2]</sup> Every result, every failure, every partial finding passes through one place. The coordinator can evaluate synthesis quality and re-delegate to search or analysis subagents with targeted follow-up queries before invoking synthesis again. A failure in one spoke is visible at the hub and nowhere else, because there is no channel along which one subagent could learn that another had trouble.

The flat alternative (agents sharing a global state or full conversation history) removes that control surface. When one agent’s noisy output pollutes another agent’s context, the errors are hard to isolate and harder to reproduce.

So the selection rule is not about agent count and it is not about how hard the task looks. Reach for a coordinator when the system needs one place that sees everything: one place where a failure surfaces, one place where results are judged against the original request, one place that decides what happens next. Reach for a flat arrangement when no such judgment is required, which in practice means when there is only one job to do.

The consequence worth internalizing is the absence of a side channel. There is no queue between subagents, no shared store they both read, no callback one registers with another. A design that proposes any of those is proposing infrastructure the pattern does not contain, and it is buying that infrastructure at the price of the single vantage point. If findings have to travel from one subagent to the next, they travel by being written into the next subagent's prompt by the coordinator. That is the whole mechanism.

Two failures follow from forgetting it. The first is spawning when there is nothing to delegate: if the coordinator already holds the data the answer needs, handing that same data to a subagent buys a round trip and returns no capability that the coordinator did not already have. The second is subtler. When a subagent has itself identified a gap and said so, the coordinator's job is to act on that statement, not to re-derive the gap by inspecting the subagent's output for signs of one. The finding has already been made. Re-deriving it discards the most reliable signal in the system and replaces it with an inference.

The wrong fix here is worth naming in advance, because it is architecturally tempting and it does not work. When reports come back thin, the instinct is to add an agent at the front of the pipeline to plan better: decompose the topic into subquestions first, and the coverage problem goes away. It does not. Thin reports usually mean the loop is open, not that the plan was poor. The guide's remedy is a closed loop: the analysis stage names the specific gaps it found, the coordinator re-delegates targeted work against those gaps, and the stage runs again until coverage is sufficient.<sup>[2]</sup> Planning harder at the front changes nothing about a pipeline that never looks back.

The coordinator carries four responsibilities that belong to it alone and to nothing else in the system.<sup>[2]</sup>

**Decomposition.** The coordinator reads the incoming query and decides how to split the work. It identifies which subtasks are independent (can run in parallel), which are sequential (stage B needs stage A’s output), and whether any subtasks are unnecessary for this particular query. A query about a well-documented public API might not need document analysis at all. The coordinator figures that out before delegating anything.

**Delegation.** The coordinator dispatches subtasks to the appropriate subagents. It writes the prompt string for each invocation, deciding exactly what context and instructions to include. This is the most consequential decision the coordinator makes: too little context and the subagent works blind; too much context and you are back to the noise problem the subagent was supposed to solve.

**Result aggregation.** The coordinator receives all subagent outputs and assembles them into a coherent result. In the research system, this means taking structured findings from multiple analysis passes and passing them to the synthesis subagent in a form that synthesis can use. The coordinator does not just concatenate outputs. It curates them.

**Dynamic subagent selection.** The coordinator decides which subagents to invoke based on the query’s actual requirements, not on a fixed pipeline sequence. This is not optional behavior. It is the mechanism by which the hub-and-spoke topology delivers any advantage over a fixed sequential chain.

The coordinator’s decomposition itself follows one of two patterns, and the exam tests the selection rule between them.<sup>[3]</sup>

**Fixed sequential pipeline (prompt chaining).** The stages are known before execution starts. A code-review pipeline that analyzes each file individually, then runs a cross-file integration pass for dependency and data-flow issues, is a fixed pipeline: the number of stages, their order, and their inputs are determined by the PR’s file list, not by intermediate findings. Use this when the work is predictable and multi-aspect (multiple files, multiple review dimensions) but the decomposition itself does not depend on what any single stage discovers.

**Dynamic adaptive decomposition.** The subtasks emerge from findings. A legacy-codebase investigation that starts by mapping the directory structure, identifies high-impact areas from that map, creates a prioritized analysis plan, then adapts as dependency relationships surface is a dynamic decomposition: the second stage cannot be specified until the first stage reports, because what to investigate depends on what exists. Use this when the work is open-ended and the structure of the problem is itself unknown at the start.

The selection rule: if you can enumerate the stages before the first tool call fires, use a fixed pipeline. If the stages depend on intermediate results, use dynamic decomposition. The coordinator’s dynamic subagent selection (choosing which agents to invoke per query) is the runtime expression of the second pattern. The "map structure first, then prioritize, then adapt" sequence that appears in the exam’s open-ended investigation scenarios is the canonical worked example of the adaptive arm.<sup>[3]</sup>

Two secondary tests sharpen that rule when a workflow sits near the boundary. The first is uniformity. If every input needs the same set of concerns examined, whatever the input turns out to contain, the sequence was knowable in advance by definition and chaining is the cheaper vehicle. Dynamic selection is machinery for deciding what to skip, and there is nothing to skip when the answer is always all of it. The second is dependency direction. Ask whether the second stage can be written down before the first stage has reported. If it can, the plan exists at the start and the pipeline is fixed. If it cannot, the plan is something the run produces, and no amount of care at design time will produce it earlier.

The wrong fix on the adaptive arm is the diligent one. Faced with an unfamiliar codebase and a broad instruction, the reflex is to be systematic: read every file first so that nothing is missed, or lay out a schedule that follows the directory tree and gives each module the same attention. Both spend the entire context budget before a single judgment about importance has been made, and both treat an arbitrary ordering as though it were a priority ordering. The guide's own sequence puts structure first: map the shape of the codebase, identify the high-impact areas from that map, then build a prioritized plan and let it revise itself as dependencies surface.<sup>[3]</sup> Structure mapping is cheap, and it is the only thing that makes the prioritization meaningful. Reading everything is expensive and leaves the prioritization exactly where it started.

(Chapter 9’s intervention classifier uses the structural-versus-recognition distinction to separate decomposition fixes from few-shot fixes when the same domain surfaces in an exam stem.)

---

## The Agent Tool

Subagents are invoked via the Agent tool.<sup>[4]</sup> The exam guide states the requirement as a hard one: the tool must be present in the coordinator's allowedTools for the coordinator to invoke subagents at all.<sup>[5]</sup> The current documentation frames the same field as an approval mechanism rather than a capability switch. Listing the tool in allowedTools auto-approves subagent invocations without a permission prompt; leave it out and the invocation falls through to the permission callback instead, or is denied outright under dontAsk mode.<sup>[6]</sup> The practical consequence is the guide's: a coordinator that has not been granted the tool does not delegate on its own.

One version note matters for production code and for the exam. The tool was renamed from "Task" to "Agent" in Claude Code v2.1.63. Current SDK releases emit "Agent" in tool_use blocks, but still use "Task" in the system:init tools list and in result.permission_denials[].tool_name. Checking both values in block.name ensures compatibility across SDK versions.<sup>[4]</sup> The exam guide, written against the earlier name, says Task.<sup>[5]</sup>

The diagnostic worth carrying is the shape of the symptom. A coordinator that reasons through the delegation correctly, names the right specialist for the job, explains what it intends to hand over, and then invokes nothing at all is not confused. It has demonstrated in its own output that it knows the subagents exist and knows which one it wants, which rules out the missing-knowledge explanations: a system prompt that fails to list the available agents, a description field that fails to describe. Correct reasoning with zero invocations is a capability failure, and the authorization layer is where to look first. The neighboring symptom looks nothing like it. A coordinator that delegates enthusiastically to the wrong specialist has the capability and lacks the knowledge, and the fix for that one lives in the description fields.

---

## AgentDefinition: What You Configure

Each subagent is described by an **AgentDefinition object** passed in the agents parameter of your query() call. Two fields are required.<sup>[4]</sup>

**description** is a natural language statement of when to use this agent. Claude reads it and decides whether to invoke the agent. Write it the way you would write a tool description: specific, bounded, clear about what this agent handles versus what it does not. The coordinator selects subagents based on descriptions. A vague description produces unpredictable selection.

**prompt** is the subagent’s system prompt. This is where the agent’s specialized instructions live: its role, its constraints, its output format requirements, its tool-use preferences. The more focused this prompt is on the subagent’s specific job, the better.

The optional fields give you per-agent configuration:<sup>[6]</sup>

- **tools**: array of allowed tool names. If omitted, the subagent inherits every tool available to subagents, which is the parent's pool narrowed by two standing filters rather than the parent's pool entire. Specify this to restrict the subagent to only the tools it needs.
- **disallowedTools**: tools to remove from the agent’s set, useful when inheriting by default but wanting to block specific capabilities. Server-level MCP patterns are accepted here as well, so an entire server's tools can be removed in one entry.
- **model**: override the model for this agent. Accepts aliases (*'fable', 'opus', 'sonnet', 'haiku', 'inherit'*) or a full model ID. Use a lighter model for a fast web-search pass; use a heavier model for the synthesis step that requires careful reasoning.
- **skills**: skill names to preload into the agent’s context at startup. Skills left off the list stay invocable through the Skill tool; preloading only changes what is present before the first turn.
- **memory**: memory source for this agent ('user', 'project', or 'local').
- **mcpServers**: MCP servers available to this agent, by name or inline configuration.
- **initialPrompt**: auto-submitted as the first user turn when this agent runs as the main thread agent. Ignored when the agent is invoked as a subagent, which is the case throughout this chapter.
- **maxTurns**: maximum agentic turns before the agent stops.
- **background**: run this agent as a non-blocking background task when invoked.
- **effort**: reasoning effort level ('low', 'medium', 'high', 'xhigh', 'max', or a number).
- **permissionMode**: permission mode for tool execution within this agent. (The values are defined in Chapter 1.)

Two of those fields shifted underneath the exam's model in Claude Code v2.1.198 and are worth flagging as version notes rather than as corrections to the pattern. Subagents now run in the background by default, so an Agent tool call that says nothing about it launches a background subagent, and Claude asks for a foreground run when it needs the result before it can continue. Setting **background** to true forces background execution regardless of what Claude requests. A subagent also now inherits the main session's extended thinking configuration.<sup>[6]</sup> Neither change touches the routing rules the exam tests; both change what a running system does by default.

The research system example makes the model field concrete. The web-search subagent runs fast and wide; you might use a lighter, faster model and accept some reduction in analysis depth. The synthesis subagent integrates findings from multiple sources and needs to reason carefully; that is a case for a more capable model. The coordinator pays the cost of an extra API call either way. You get to tune that tradeoff per agent.

---

## What a Subagent Inherits (and What It Does Not)

This is the most exam-tested mechanical detail in the chapter. Memorize both sides of the table.<sup>[4]</sup>

A subagent receives:

- Its own system prompt (AgentDefinition.prompt)
- The Agent tool’s prompt string (the task description passed at invocation time)
- The project *CLAUDE.md* (loaded via settingSources)
- Its defined tool set (inherited from parent, or the subset specified in tools, and narrowed further for a background run)

A subagent does not receive:

- The parent’s conversation history
- The parent’s tool results
- The parent’s system prompt
- Any skill content not listed in AgentDefinition.skills

The exam guide states the same constraint from the other direction, and adds the part that catches people out: subagent context has to be supplied in the prompt, because a subagent inherits no parent context automatically and shares no memory from one invocation to the next.<sup>[5]</sup> There is no accumulation across calls. Each invocation begins where the last one began.

The only channel from coordinator to subagent is the Agent tool’s prompt string.<sup>[4]</sup> If the synthesis subagent needs the web search results, the coordinator must pass those results explicitly in the prompt it sends to the synthesis agent. There is no automatic inheritance. There is no shared memory. There is no side channel.

This constraint is a feature, not a limitation. It is what makes context isolation work. The synthesis subagent starts with exactly the context the coordinator decided it should have: the distilled outputs of the search and analysis stages, formatted and targeted for synthesis. No raw tool call logs. No intermediate reasoning artifacts. No irrelevant prior turns.

The guide's own instruction for the research system is to put the complete findings of the prior agents directly into the next agent's prompt: the web search results and the document analysis outputs go into the synthesis agent's prompt, because that is the only place they can go.<sup>[5]</sup> What the coordinator writes there, it pays for twice, once in its own context and once in the subagent's, so the prompt should carry the findings and not the history that produced them.

The failure this produces is diagnostically clean, which is why it gets tested. A subagent that receives a task description and then reports that it has nothing to work with is not describing a capacity problem. It is describing an empty envelope. Context pressure degrades output: the work gets done, and gets done worse, with details dropped from the middle and quality sliding as the window fills. A flat statement that the data was never supplied means the data was never supplied. The remedy is in the coordinator's invocation, not in the subagent's window size, and it is the same remedy in every instance: put the prior stage's output into the prompt string. Reaching instead for a larger context window, or for a shared memory the pattern does not have, treats a routing omission as a capacity shortfall.

---

## Context Passing: The Rule and Why It Exists

The rule is simple to state. **Pass only context specific to each subagent’s task.**<sup>[5]</sup> The rationale is more interesting.

A coordinator running a multi-topic research job may have, by the time it invokes synthesis, a context that includes: the original query, a planning pass, web search results for three distinct subtopics, document excerpts from seven sources, intermediate analysis notes for each subtopic, and several rounds of quality checks. That is a realistic coordinator context for a serious research workflow.

The synthesis subagent does not need the planning pass. It does not need the raw search queries. It does not need the quality check history. It needs the analysis outputs: structured findings with source attribution, organized for synthesis.

Passing the full coordinator history to the synthesis agent would mean the synthesis model attends to the planning pass and the raw search queries alongside the analysis outputs. Those earlier elements do not help synthesis. They add noise. They consume tokens that could go to the actual findings. And they create confusing signal: the synthesis agent might repeat reasoning the coordinator already completed, or hedge against questions that were already resolved upstream.

The synthesis agent does not need to know how the search was done. It needs to know what it found.

The structured data approach makes context passing precise. When passing outputs from one subagent to the next, separate content from metadata. Findings go in one field. Source URLs, document names, and page numbers go in another. The guide gives this as a skill in its own right, and gives the reason: separation is what preserves attribution as the material moves between agents.<sup>[5]</sup>

The mechanism behind that is worth stating, because it explains why the obvious alternative fails. Attribution that lives inside the prose lives only as long as the prose does. A summarizing stage rewrites the sentence, and the citation that was part of the sentence goes with it. Attribution that lives in its own field is not part of the sentence, so nothing that happens to the sentence can happen to it: the content field can be condensed, restructured, merged with another agent's findings, or rewritten entirely, and the source list arrives at the report generator intact. Every stage that touches the content is a chance to lose an inline reference and no chance at all to lose a sibling field.

Which makes the recovery instinct the wrong one to indulge. When the report generator turns out to be unable to attribute a claim, the appealing move is to send it back upstream to ask where the claim came from, and the appealing move is unavailable: the earlier agents have returned, their contexts are gone, and re-running them re-derives findings rather than recovering the ones already made. Attribution is not something a pipeline can reconstruct at the end. It is something each stage either carries forward or drops, and the design decision that determines which is made at the first handoff, not the last.

---

## Parallel Execution

Multiple subagents can run concurrently.<sup>[4]</sup> The mechanism is straightforward: emit multiple Agent tool calls in a single coordinator response. The SDK runs them in parallel. The coordinator receives all results before continuing.

The research system has a natural parallel phase. Web search and document analysis can run simultaneously. Neither depends on the other’s output. Synthesis depends on both. The coordinator emits Agent tool calls for the search subagent and the analysis subagent in the same response turn; both run in parallel; the coordinator receives both results; then it invokes synthesis with both outputs as input.

The contrast is sequential invocation, where the coordinator emits one Agent call, waits for the result, then emits the next. Sequential invocation is correct when stage B genuinely depends on stage A’s output. For stages that are independent, parallel invocation reduces end-to-end latency proportionally to the number of agents running in parallel.

The guide is explicit about the mechanism, and states it as a skill: the coordinator spawns subagents in parallel by putting several tool calls into one response, not by spreading them over consecutive turns.<sup>[5]</sup> That phrasing is doing real work. Parallelism here is a property of how the calls are batched into one turn, not a property of a dedicated method, a configuration flag, or a model tier. A coordinator that emits one call per turn runs sequentially no matter how capable the model underneath it is and no matter what its instructions say about working in parallel.

The consequence is that the fix for accidental serialization lives inside the conversation rather than around it. Two independent stages running one after the other are not evidence that the SDK lacks concurrency; they are evidence that the coordinator turned twice. Wrapping the agent in an external orchestration layer to run the two calls concurrently reaches for infrastructure to supply something the invocation mechanism already supplies, and it moves the spawning authority out of the coordinator, which is the one place the pattern wants it. The whole benefit of centralized delegation is that one component sees every invocation; an outside scheduler that issues its own is a second hub.

The background field in AgentDefinition adjusts this further. From Claude Code v2.1.198 a subagent runs in the background by default and Claude asks for a foreground run when it needs the result before it can continue, which inverts the earlier arrangement. Setting background to true forces the background path for a given agent whatever Claude requests.<sup>[6]</sup> A background subagent does not hold up the coordinator's own execution, which suits a long analysis whose output is not needed for the next immediate step; the coordinator retrieves the result when it is ready to move to synthesis. Note that a background run also narrows the subagent's built-in tool set, so it is a scheduling decision with a capability consequence attached.<sup>[7]</sup>

The detection side of parallel execution matters for orchestration code that needs to track which subagents have been invoked. Subagents are invoked via the Agent tool, so detecting subagent invocation means checking for tool_use blocks where name is "Agent". Messages originating from within a subagent’s context include a parent_tool_use_id field that links back to the parent’s invocation. In Python, content blocks are accessed directly via message.content. In TypeScript, SDKAssistantMessage wraps the Claude API message, so content is accessed via message.message.content.<sup>[4]</sup>

---

## Nesting and the Depth Limit

This is a place where the platform has moved and the pattern has not, and the two need to be held apart.

The platform permits nesting. A subagent can spawn subagents of its own, by default down to three layers below the main conversation. At the depth limit Claude Code withholds the Agent tool from the subagents sitting at the bottom, so an agent there does its delegated work itself and returns a single summary rather than failing. The limit is set by `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`, and setting it to 1 turns nesting off entirely. To stop one particular subagent from delegating without changing the global depth, leave the Agent tool out of its tools list or name it in disallowedTools.<sup>[7]</sup> Two neighboring limits bound the same growth from other directions: a default ceiling of twenty subagents running at once, and a spend cap that counts every subagent's requests against the query's total.<sup>[6]</sup>

The pattern is a different question. Hub-and-spoke is the architecture the exam guide describes, and it puts every inter-subagent channel, all error handling, and all information routing in the coordinator.<sup>[2]</sup> A subagent that delegates on its own opens a channel the coordinator cannot see: work is dispatched, results return, and none of it passes through the hub. The observability that justified the topology is gone at exactly the depth where the tree stops being one layer deep, and it is gone silently, because the coordinator's view of a nested run is a single summary from its own direct child.

So the flat architecture is now a design decision rather than a platform constraint, and it is one you enforce rather than one you inherit. If a subagent needs work done that itself warrants delegation, the coordinator decomposes that work into separate invocations and manages the dependency between them explicitly. Where a scenario hands a research subagent the Agent tool so that it can spawn search agents of its own, the objection is not that the system will refuse. It is that the coordinator has just given away the one thing it was built to hold.

---

## Coordinator Prompt Design

The exam guide’s task statements are precise about what a coordinator prompt should contain.<sup>[5]</sup> It specifies **research goals and quality criteria**. It does not enumerate step-by-step procedural instructions. The guide gives the reason as adaptability: a subagent working from a goal can respond to what it finds, and a subagent working from a numbered procedure can only respond to what the procedure anticipated.

The distinction matters operationally. A coordinator prompt that says “*first call the web-search subagent, then call the document-analysis subagent, then call synthesis*” is a static pipeline. It runs the same sequence regardless of the query. A simple factual question gets the full pipeline including a document analysis pass that contributes nothing. A query where the web search returns no useful results still routes to synthesis with empty inputs.

A coordinator prompt that specifies goals and quality criteria lets the coordinator decide which subagents to invoke based on what the query actually requires. For a simple factual question, the coordinator might invoke only the web-search subagent and skip document analysis entirely. For a query where preliminary synthesis reveals gaps, the coordinator re-delegates to search and analysis with targeted follow-up queries before re-invoking synthesis.<sup>[2]</sup>

There is a repair that looks like the right one here and is not. When a procedural prompt breaks on an input its author did not foresee, the natural response is to foresee harder: add a clause for the case that broke, then a clause for the next one. Each addition fixes exactly the failure that prompted it. The set of failures it does not fix is the set nobody has hit yet, which is the larger set and the one that matters. Converting the prompt to goals and quality criteria is not a bigger version of that repair; it is a different kind of repair, because it removes the need to enumerate contingencies rather than enumerating one more. A prompt that says what a good result looks like covers inputs its author never imagined. A prompt that says what to do covers the inputs its author imagined, and no others.

Dynamic selection is the coordinator’s core capability. The coordinator does not route all queries through all subagents. It reads the query, reads the description fields of available subagents, and selects the subagents the query requires.

Note where that selection lives. It happens inside the coordinator, at the moment the query arrives, using the same reasoning that will handle the query. The alternative design puts a classifier in front of the agent: train something on past traffic, have it decide which specialists a request needs, and hand the coordinator a plan. That builds a second decision-making component with its own training set, its own drift, and its own failure mode, and it degrades precisely when the traffic starts to look different from the traffic it learned on, which is the moment adaptive routing was supposed to earn its place. The coordinator already reads the query. Giving it the authority to decide what the query needs is cheaper than building something else to decide on its behalf, and it keeps the decision where the observability is.

---

## Two Anti-Patterns the Exam Tests

Both anti-patterns come straight from the exam guide's Domain 1 task statements. One is listed there as a risk, the other as the thing a well-designed coordinator does instead.<sup>[2]</sup>

**Overly narrow task decomposition creating coverage gaps.** The research system has four subagents. If the coordinator assigns each agent a scope so narrow that some aspects of a broad research topic fall between agents, findings have holes. The web-search agent covers recent publications. The document-analysis agent covers the provided source list. If the query asks about a topic where relevant findings exist in both sources that cross-reference each other, and neither agent is configured to notice the cross-reference pattern, the synthesis agent never sees the connection.

Narrow decomposition is sometimes presented as a virtue (focus, specialization). It becomes an anti-pattern when the task boundaries do not align with the natural structure of the work. The fix is to design agent scopes around the shape of the information, not around a desire for symmetry in the agent count. The guide gives the partitioning skill its own line, and it cuts both ways: scope is divided so that agents do not duplicate each other's work, by handing each one a distinct subtopic or a distinct class of source.<sup>[2]</sup> Divide by subject matter and you get gaps where subjects overlap. Divide by source type and you get gaps where a finding lives in two places at once. Neither partition is free, which is why the coordinator has to be looking for the seam rather than trusting the division.

**Routing all queries through all subagents.** The opposite failure. The coordinator always runs the full pipeline regardless of what the query requires. A question that only needs web search runs document analysis anyway. A question that is already well-covered by preliminary synthesis triggers another full search-and-analysis pass. This wastes tokens, increases latency, and can introduce noise when subagents return results for tasks that are not relevant to the query.

The coordinator’s selection logic is not overhead. It is the mechanism by which the hub-and-spoke topology delivers its efficiency.

Both anti-patterns share a root cause: the coordinator is not reasoning about the query. It is executing a fixed procedure. The whole point of putting an LLM in the coordinator role is that it can read the query, assess what the query needs, and choose accordingly.

---

## Decomposing a Multi-Concern Request

Decomposition is not only a multi-agent move. The same principle applies inside a single agent when one user message carries several independent concerns. "I need a refund for order #1234 and I want to update the shipping address on order #5678" is two tasks wearing one sentence. An agent that treats it as a single undifferentiated request tends to address one concern and drop the other, or to cross-wire the parameters between them.

The pattern is: split the message into its distinct concerns, handle each one as a separate unit of work, and synthesize the results into one reply. Critically, the concerns share context rather than being processed in isolation. The customer's verified identity, established once, applies to both the refund and the address change; the agent does not re-verify per concern, and it does not re-fetch the customer record for each task. Shared context established once, applied across every decomposed concern, then a single unified resolution.

This is distinct from the recognition-gap version of the same scenario. If an agent already handles single-concern requests well but its accuracy collapses specifically on multi-concern messages because it fails to notice the second concern, that is a pattern the model can be taught with examples, and few-shot is the cheaper fix. The decomposition pattern here is the structural answer: it applies when the workflow itself is wasteful or error-prone (re-fetching, re-verifying, sequential handling of independent work), not when the model simply needs to recognize a pattern it otherwise executes correctly. Chapter 9's intervention classifier draws this line explicitly; the discriminator is whether the stem quotes a recognition failure or a structural inefficiency.

---

## Three Ways to Create Subagents

The SDK supports three creation paths.<sup>[4]</sup>

**Programmatic definition** (the recommended path): pass AgentDefinition objects in the agents parameter of query(). This is the approach this chapter has been describing. Definitions live in code, are version-controlled, and can be constructed dynamically at runtime based on query characteristics or runtime configuration.

**Filesystem-based definition**: define subagents as markdown files in .claude/agents/ directories, with YAML frontmatter carrying the configuration and the body carrying the system prompt. The agent's name is the required name field in that frontmatter. Claude Code watches the personal and project agent directories and picks up a new or edited file within a few seconds, so a restart is not normally required. Three things defeat the watcher and are worth knowing in that order: a directory that did not exist when the session started is not being watched, so the first file in a brand new agents directory does need a restart; invalid frontmatter or a name another agent already uses will keep a definition from appearing; and a programmatically defined agent takes precedence over a filesystem agent of the same name, so the file loads and is then shadowed.<sup>[6]</sup>

**Built-in general-purpose subagent**: Claude can invoke the built-in general-purpose subagent at any time via the Agent tool without any custom definitions, as long as "Agent" is in allowedTools. This is useful for delegating exploratory research or quick analysis tasks without defining specialized agents upfront.

One practical note on agent creation that appears in the troubleshooting documentation: if Claude completes tasks directly instead of delegating to your defined subagent, verify that the Agent tool is in allowedTools, and verify that the subagent’s description field clearly explains when to use it. The description is how Claude matches tasks to the right subagent. A vague description produces missed delegations.<sup>[4]</sup>

---

## Context Isolation as a Context Window Management Tool

There is a secondary benefit to subagent context isolation that Chapter 11 covers at length. The short version belongs here because it connects directly to why the research system design makes sense.

Intermediate tool calls stay inside the subagent. The parent receives only the subagent’s final message.<sup>[4]</sup> A document-analysis subagent might read and process fifty source files, generating extensive intermediate reasoning across many turns. None of that intermediate content appears in the coordinator’s context. The coordinator receives the analysis summary: structured findings, source attribution, confidence levels. That is a few kilobytes instead of the megabytes of raw file content and processing steps.

This is not just an efficiency gain. It is what makes iterative refinement feasible. The coordinator can invoke synthesis, evaluate the output, identify a gap, re-delegate to document analysis with a targeted query, receive the additional findings, and re-invoke synthesis with the enriched inputs. Each pass keeps the coordinator’s context manageable because the subagents absorb the exploration cost internally.

The guide asks for a loop that keeps re-delegating until coverage is sufficient.<sup>[2]</sup> That loop is affordable only because the coordinator is not paying the context cost of each subagent’s internal work: intermediate calls and results stay where they happened, and only the final message comes back.<sup>[4]</sup>

---

## Session State: Resumption and Forking

A session is a conversation transcript. It persists the exchange of messages and tool calls, not the filesystem. When Claude Code closes, the session stays on disk. What you do with that stored transcript on the next invocation is the subject of Task Statement 1.7, and the exam tests three operations: resume, fork, and start fresh.

**Resumption.** The --resume flag (short form -r) continues a named session: claude --resume “auth-refactor” picks up exactly where that conversation left off, with the full prior transcript restored. The --continue flag (short form -c) resumes the most recent session in the current directory without requiring a name.<sup>[8]</sup> The guide names the flag with a session name as its argument, which is the form worth committing to memory.<sup>[9]</sup> Named resumption is the safer default in any directory that has hosted multiple unrelated tasks, because --continue silently picks up whichever session ran last, which may not be the one you intend. The two flags answer different questions. One asks for a particular conversation and is answered by name. The other asks for the last thing that happened here and is answered by recency. When the name is known, recency is the wrong selector, and it is wrong in a way that produces a plausible-looking session rather than an error.

Names are something you set, which is easy to forget. A session gets one from -n at startup, from /rename during the conversation, or from the picker; the display name and the summary title Claude Code generates for an unnamed session are labels for finding it by eye, not handles that --resume will match. A session ID always resolves, and resolves from any directory.<sup>[8]</sup>

**Forking.** The --fork-session flag, combined with --resume or --continue, branches the session: claude --resume “auth-refactor” --fork-session creates a new session whose transcript starts as a copy of the original, but diverges from that point forward. The original remains untouched, keeps its own ID, and stays independently resumable, so what you end up with is two histories rather than one history with a detour in it. In the Agent SDK the same operation is a boolean set alongside a resume target: fork_session in Python, forkSession in TypeScript.<sup>[10]</sup> Inside a running conversation, /branch does the same thing from the other end, copying the transcript and switching you into the copy.<sup>[8]</sup> Forking beats resuming the original twice when two approaches need to diverge from a shared analysis baseline (comparing two refactoring strategies, for example, or two test approaches that share the same initial codebase analysis). Forking beats copying context into a fresh session because the fork preserves the full tool-call history; a pasted summary loses it.

One asymmetry between those two routes is worth holding on to, because it is the kind of detail that only announces itself at an inconvenient moment. Branching from inside a session keeps running in the same process, so permissions you approved for the session are still approved in the branch. Forking on the command line starts a new process, which begins without those grants and asks again.<sup>[8]</sup>

**Choosing between the three.** The decision has three arms and each one is selected by a different property of the prior session, not by how much work went into it.

Resume when the prior context is mostly still true and you can say precisely what is no longer true. A handful of files changed overnight; the analysis of everything else still holds. The move is to resume and tell the agent which files moved, so re-analysis is aimed rather than blanket. The guide makes this its own skill, and states the converse as the failure: a resumed session that is not told what changed will treat what it already read as current.<sup>[9]</sup>

Start fresh with a structured summary when the prior tool results are stale or the change is broad enough that you cannot enumerate what survived. This is the arm the guide argues for explicitly, and it is the one that feels wrong: more context ought to be safer, and here it is not.<sup>[9]</sup>

Fork when nothing is stale and nothing needs redoing, but two directions need to be explored from the same starting point without either one seeing the other's reasoning.

**Why the transcript loses to the summary.** A transcript is a record of tool calls and their results, and those results carry no expiry date. A file that was read yesterday is in the transcript as its contents, not as its contents-as-of-yesterday. Resuming re-imports every one of those readings as ordinary, unmarked, authoritative context, and the agent has no way to tell a reading that is still accurate from one that was invalidated overnight. A structured summary is written at the moment it is injected. It carries conclusions rather than raw readings, it can state outright what has changed since, and it contains nothing that will be mistaken for a current observation. Past a certain amount of change, the transcript has stopped being context and become misinformation, and the fact that there is a lot of it makes that worse rather than better.

Two repairs suggest themselves here and neither one works. The first is to resume the full history and add an instruction telling the agent to prefer recent tool results over older ones. That does not remove the stale results; it leaves them in the window and adds an ambiguity about which ones the instruction meant. The second is to lift the findings out into a file, then start two clean sessions that each read it. That throws away the tool-call history the fork would have kept and buys back a re-read cost on both branches, in exchange for a hand-picked subset of what was already there.

**Three senses of one word.** The book uses “fork” in three unrelated ways and they are worth separating once. The context: fork field in a skill’s SKILL.md frontmatter (Chapter 7) isolates that skill’s execution in a sub-agent context so intermediate exploration does not pollute the parent conversation. A fork subagent, started with /subtask, is a subagent that inherits the whole conversation instead of starting clean, which trades away the input isolation an ordinary subagent gives you in exchange for not having to re-explain the situation; its own tool calls still stay out of the main window.<sup>[7]</sup> The --fork-session flag branches a whole session transcript so two approaches can diverge from the same prior state. Only the third is Task Statement 1.7. When a scenario describes branching a conversation to explore approaches in parallel, session forking is the mechanism, whatever the word looks like elsewhere.

A fork does not fix stale context. If the session you fork from contains outdated tool results, every branch inherits that staleness. Forking is a divergence tool, not a freshness tool. When the problem is stale cached results rather than wanting to explore two paths, start fresh. And a fork branches the conversation, not the working tree: edits a forked agent makes are real edits, visible to any session working in the same directory, so two branches exploring two approaches are exploring them against one filesystem.<sup>[10]</sup>

---

## The Research System as a Design Walkthrough

Scenario 3 is worth walking through as a complete design exercise, because it is the scenario where all the concepts in this chapter intersect. The guide's own framing of it is short: a coordinator delegating to specialists that search, analyze, synthesize and report, over topics, producing cited reports. It lists the scenario's primary domains as agentic architecture and orchestration, tool design and MCP integration, and context management and reliability, which is a fair map of what a question set drawn from it can reach.<sup>[1]</sup>

The system has four subagents: a web-search agent, a document-analysis agent, a synthesis agent, and a report-generation agent. The coordinator receives a research topic and produces a comprehensive cited report.

The web-search and document-analysis agents are independent. They can run in parallel. The coordinator emits both Agent tool calls in the same response turn. Each agent runs in its own fresh context, with only the tools it needs (the web-search agent has web access; the document-analysis agent has file read access and no web tools) and a prompt that includes only the research topic and any source constraints specific to that agent’s job. Neither agent sees the other’s work while it runs.

The synthesis agent depends on both prior agents. It cannot run until the coordinator has both sets of results. The coordinator collects the web-search findings and the document-analysis findings, formats them as structured inputs (findings with source attribution separated from raw excerpts), and passes that prepared context in the prompt to the synthesis agent. The synthesis agent receives exactly the distilled outputs it needs. It does not receive the raw search query, the intermediate tool call logs from the document analysis, or any other coordinator history.

The report-generation agent depends on synthesis output only. It receives the synthesis result and is responsible for formatting: citation formatting, section organization, executive summary generation. Its prompt specifies output format requirements in detail so the coordinator does not need to post-process the report.

Now consider what the coordinator does when the synthesis output is incomplete. The coordinator evaluates the synthesis result against the research topic. If it identifies a gap (a subtopic not covered, a claim not substantiated), it re-delegates to the web-search or document-analysis agent with a targeted follow-up query, waits for the result, and re-invokes synthesis with the original findings plus the supplementary findings. That loop, run until coverage is sufficient, is what the guide asks for.<sup>[2]</sup> It is feasible because the coordinator’s context window stays manageable: each subagent invocation absorbs its own tool call costs internally, returning only a concise structured result to the coordinator.<sup>[4]</sup>

The design decisions that make this work:

- Parallel execution for independent stages reduces latency without coordination overhead.
- Targeted context passing for each subagent prevents noise and keeps each agent’s context window focused on the work it is doing.
- Iterative refinement rather than a single-pass pipeline improves result quality without requiring the coordinator to manage the full search-and-analysis context across multiple passes.
- Dynamic subagent selection lets the coordinator skip stages that are not needed for a specific query, rather than running the full pipeline every time.

This is hub-and-spoke applied correctly. The coordinator thinks. The subagents execute. The results flow back through the hub.

---

## What the Exam Actually Tests

Four of the six scenarios are drawn at random for any given sitting, so the research system is a coin toss rather than a certainty; what is certain is that Domain 1 carries the largest weight of the five, and that this scenario is where its task statements are most densely exercised.<sup>[1]</sup> Three traps recur.

Sharing full coordinator context with subagents is wrong. Subagent context is supplied deliberately, in the prompt, and consists of what the task needs.<sup>[5]</sup>

Silently dropping subagent failures is wrong. Error handling is one of the three things the guide assigns to the coordinator, and consistent handling is part of what routing everything through one place is supposed to buy.<sup>[2]</sup> Structured error propagation requires reporting what was attempted, what failed, and whether the failure was an access error or a valid empty result. Chapter 12 covers the access-failure-versus-empty-result distinction in detail.

Ignoring provenance when resolving conflicting findings from different subagents is always wrong. When two subagents return conflicting statistics, the coordinator annotates the conflict with source attribution and publication dates rather than arbitrarily selecting one value. Chapter 12 works through the record structure that makes this possible, which this book calls DataWithProvenance. The name and the concrete shape are the book's own; what the exam guide requires is the claim-source mapping itself, produced upstream and preserved through synthesis.

The hub-and-spoke architecture questions on the exam follow a pattern. The correct answers involve a coordinator that selects subagents dynamically, passes targeted context, routes all inter-agent communication through itself, and handles failures with structured error reporting. The wrong answers involve flat architectures, full context sharing, static pipelines, and silent error swallowing.

The topology is simple. The discipline required to implement it correctly is not.

---

## Key Takeaways

- The coordinator-plus-subagents pattern emerges when a task decomposes into subtasks that would contaminate each other if they shared a context; context isolation is the primary reason to reach for this topology, not task complexity alone.
- The Agent tool (formerly Task before Claude Code v2.1.63) is the invocation mechanism; it must appear in the coordinator’s allowedTools, checking both "Agent" and "Task" in block.name ensures cross-version compatibility, and correct delegation reasoning with zero invocations points at that field rather than at the system prompt.
- A subagent receives only its AgentDefinition.prompt, the Agent tool’s prompt string, the project CLAUDE.md, and its defined tool set; it receives no parent conversation history, no parent system prompt, and no parent tool results.
- Pass only the context each subagent needs for its specific task; a subagent reporting that it was given nothing to work with is an omission in the coordinator's prompt string, not a context window that needs enlarging.
- Emit multiple Agent tool calls in a single coordinator response to run subagents in parallel; sequential invocation across turns is the wrong mechanism for independent stages, an external scheduler is the wrong place to fix it, and subagents nest three layers deep by default, so keeping the topology flat means leaving the Agent tool out of a subagent's tools array rather than assuming the platform forbids it.
- Session state has three arms: resume with a stated list of what changed when prior context is mostly valid, start fresh with a structured summary when tool results are stale, fork when two approaches must diverge from one valid baseline.
- The two named anti-patterns are overly narrow decomposition (creating coverage gaps between agents) and routing all queries through all subagents (static pipeline instead of dynamic selection).
