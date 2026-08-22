# Chapter 12: Escalation, Provenance, and Error Propagation

**Summary:** *A reliable agentic system knows when not to act. Valid escalation triggers are objective and verifiable: an explicit customer request for a human agent, a genuine policy gap, an inability to make progress, and an ambiguous match requiring clarification. A business ceiling such as a refund limit is enforced deterministically before the call rather than left to the agent's judgment. Negative sentiment and model self-confidence are not valid triggers, though self-reported confidence has a legitimate use elsewhere: prioritizing a human review queue after the fact. Error propagation must be structured and distinguishable: an access failure and a valid empty result look identical to a coordinator that receives only an empty array, yet the downstream consequences differ entirely. Provenance tracking must survive synthesis steps, because a claim without its source citation is not a claim anyone can act on. A provenance record, which this book calls DataWithProvenance, preserves claim-source mappings through synthesis by pairing each extracted value with its source, a category describing how it was obtained, and a retrieval timestamp. Accuracy broken out by document type and by field reveals per-category failure rates that an aggregate number hides; a pipeline reported at 97% overall may be failing nearly a third of one document type. Structured handoff summaries ensure that human agents receiving escalated tickets have the specific facts they need without re-interviewing the customer.*

## The Ticket That Should Have Closed Itself

The human agent opens the ticket. Billing discrepancy. A straightforward overcharge. The support agent had routed it to the human queue with a note that said “negative sentiment detected.”

The overcharge is real. It is also trivially within the agent’s own resolution authority: two tool calls, get_customer then process_refund. The whole interaction would have resolved without escalation. Instead, a human is now involved, the customer waited in the escalation queue, and the agent’s first-contact resolution rate took a hit.

This is the trap the exam calls out directly. The exam guide puts two signals side by side, routing driven by sentiment and routing driven by the model's own confidence, and calls both unreliable stand-ins for how complex a case actually is. One of the guide's own worked items places a sentiment-threshold router among the wrong answers.<sup>[1]</sup> The customer was frustrated. The task was simple. Those two things are independent, and the system conflated them. This chapter is about the patterns that prevent that conflation: knowing when to escalate, how to communicate failure, and how to preserve the chain of evidence that makes a decision auditable.

The scene above is the book's own illustration. No source prints it, no ticket number or customer is named in it, and nothing in the argument depends on its details. It exists to put a shape on a failure that is otherwise hard to see, because the system that produced it reported success.

## Knowing When Not to Act Autonomously

The named concept for this chapter is “Knowing when not to act autonomously.” It sounds passive. It is actually one of the harder design problems in agentic systems, because the failure mode is invisible. An agent that escalates too much costs human time and degrades resolution rate metrics. An agent that escalates too little resolves things it should not. Both failures hurt real customers. The exam focuses on the triggers that distinguish them.

### Valid Escalation Triggers

Task Statement 5.2 names four escalation triggers, and a fifth belongs to the list for a different reason and by a different mechanism.<sup>[1,2]</sup> The four have a property in common: each is evaluated against something observable in the world outside the model. None of them requires inspecting the model’s internal state or reading emotional tone from message text.

That property is the whole discrimination, and it is worth stating before the list rather than after it. A valid trigger names a condition. An invalid trigger names a correlate of a condition. "The customer asked for a person" is a fact about the transcript that either holds or does not. "The customer sounds upset" is an inference about a mental state, produced by a classifier that was never validated against the thing anyone actually cares about, which is whether this case exceeds what the agent can do. The two families fail differently, too. A definitional trigger that misfires does so because the case genuinely was ambiguous. A proxy trigger misfires in both directions at once: it escalates simple cases from angry customers and resolves hard cases from polite ones, and neither error leaves a trace that looks like an error.

**The customer explicitly requests a human agent.** This is the most important trigger. Honor it immediately, without first attempting investigation or offering resolution. The code pattern is straightforward: if *context.customer_requested_human is true*, escalate and return the reason. Do not try to resolve the issue first to “save” the escalation. That is a design choice that prioritizes the metric over the customer’s stated preference, and the exam treats it as an anti-pattern.<sup>[1]</sup>

**Policy is ambiguous or silent on the specific situation.** Consider a customer asking about competitor price matching when the company’s policy only addresses price adjustments for the company’s own previous sales. The policy document does not cover this case. That silence is a policy gap, and a policy gap is a valid escalation trigger. The agent cannot invent policy. Escalation is the correct behavior.<sup>[1]</sup>

**The agent cannot make meaningful progress.** The clearest version of this is exhausting retry attempts. A transient failure that persists beyond the retry limit represents a capability ceiling the agent cannot cross locally. This is distinct from the sentiment-based anti-pattern: the agent is not escalating because it is uncertain, it is escalating because an objective condition (retry count) has been met.<sup>[1]</sup>

**A business threshold is exceeded.** This is the fifth trigger, and it sits outside Task Statement 5.2. The guide raises it under hook design instead, as a compliance rule that intercepts an outgoing tool call and redirects it to a human escalation workflow, with a refund above $500 as the worked example.<sup>[2]</sup> An agent has a defined financial authority. Requests above that limit are not within its authority to resolve, regardless of its assessment of the request’s legitimacy. The threshold is objective. It is either exceeded or it is not.

The placement matters more than it looks. The other four triggers are decisions the agent makes; this one is a decision made for the agent. A refund ceiling encoded in the system prompt is a request the model can misread, and a model that misreads it once will do so again under a longer context. A refund ceiling encoded in a PreToolUse hook is arithmetic that runs before `process_refund` executes, and it either returns a deny or it does not. Chapter 3 sets the rule this chapter inherits: PreToolUse enforces policy because it is the only interception point ahead of the side effect, and PostToolUse normalizes data because by then the money has already moved. The same chapter records the two limits on what a hook can do here. A denial issued by the hook survives a session running in bypass mode, so the ceiling is not something a permissive configuration can be talked past. An approval issued by the hook does no more than skip the interactive prompt, and cannot grant a refund that the permission rules forbid. The escalation is the redirect, not the block. Blocking alone leaves the customer with nothing.

**Multiple customer matches requiring clarification.** When a query returns multiple matching records, the agent must ask for additional identifiers. Selecting heuristically from ambiguous results is an anti-pattern. The heuristic might be right most of the time. It will be wrong at the worst possible moments, on accounts where the stakes are highest.<sup>[1]</sup>

### Invalid Escalation Triggers

Two specific anti-patterns are named in the exam guide, and both show up in scenario questions.<sup>[1]</sup>

**Negative sentiment.** An angry customer asking to update their address does not need a human agent. The emotional state of the message is not correlated with the complexity of the task. Routing based on sentiment conflates two unrelated signals.

**Self-reported model confidence below a threshold.** Model confidence scores are not reliable proxies for actual case complexity. A model might report low confidence on a routine task that it handles well, and high confidence on an edge case where it is about to make an error. The guide's own reasoning on this is sharper than the general statement, and it is worth carrying intact: the failure mode is not that the confidence number is noisy but that it is wrong in the specific direction that matters. An agent that is mishandling hard cases is, by construction, confident about them. Routing on low confidence therefore filters out exactly the cases that were never going to fail, and passes through the ones that will.<sup>[1]</sup>

A useful way to hold both anti-patterns in one place is to picture the escalation decision as a small predicate over the case: a function that returns true when the customer has asked for a person, when no policy covers the situation, or when the retry budget is spent, and that has nothing to say about tone or self-assessment. Written that way, the two invalid triggers are visible as the two lines that are not in the function. This book uses that framing as an illustration and nothing more. The guide does not print such a function, does not name one, and does not specify its fields; what the guide supplies is the list of conditions and the reasoning about why the two proxies fail. The illustration is a way to hold the list in mind, not a structure to reproduce.

There is a further move available here that the escalation discussion tends to skip, because it looks too modest to be the answer. When an agent's escalation behavior is miscalibrated in both directions at once, over-escalating the routine and attempting the exceptional, the guide's stated first response is not new infrastructure. It is writing the escalation criteria into the system prompt explicitly, with a few worked examples showing the boundary in both directions.<sup>[1]</sup> The root cause of a miscalibrated boundary is usually an unstated one. A confidence router, a sentiment classifier, or a separate model trained on historical tickets all add machinery on top of a decision rule nobody has written down. That ordering, state the rule before building the apparatus that infers it, recurs across the whole exam and is worth recognizing as a shape rather than memorizing case by case.

### The Frustration Case

There is a specific nuance the exam tests separately from the binary “escalate or not” decision. When a customer expresses frustration but the issue is within the agent’s capability, the correct behavior is to acknowledge the frustration while offering resolution, and escalate only if the customer reiterates the preference for a human agent.<sup>[1]</sup>

This is different from ignoring the customer’s emotional state. Acknowledgment is appropriate. Automatic escalation is not. The customer’s stated preference for a human agent is what triggers immediate escalation, not the emotional register of the message.

### Ambiguity That Is Not an Escalation

Task Statement 5.2 is titled for escalation *and* ambiguity resolution, and the second half is easy to lose behind the first. Not every case the agent cannot resolve cleanly is a case for a human. Most ambiguity is resolved inside the conversation, and the design question is which resolution the situation calls for.

The guide is explicit on one instance. When a tool result comes back with several matching customer records, the agent asks for an additional identifier. It does not pick the closest match on a heuristic.<sup>[1]</sup> The reasoning generalizes past customer lookup: a heuristic tie-break has a hit rate, and the cases where it misses are not distributed at random. Records collide when they are similar, and the records most likely to be similar are the ones belonging to the same household, the same company, or the same person with two accounts. Those are the accounts where a wrong pick costs most.

The book's own reading of the surrounding shape, offered as a reading rather than as something the guide states: the choice between asking and proceeding tracks reversibility, not difficulty. An ambiguous read is cheap to get wrong, so an agent that has a defensible interpretation should state the interpretation it is acting on and proceed, leaving the correction path open. An ambiguous delete or an ambiguous refund is not cheap to get wrong, so the interpretation gets confirmed before the call rather than after it. Two consequences fall out of that. First, more clarifying questions is not automatically safer; each one is a place the interaction can be abandoned, and an agent that opens with a list of questions has delivered nothing yet. Second, when a destructive action does misfire on the wrong record, the fix is disambiguation at the moment of confirmation, showing the fields on which the candidates differ. A longer undo window is a real mitigation but it addresses the consequence rather than the cause, and the rate of wrong identifications under it is unchanged.

## Structured Handoff Summaries

When escalation does happen, the agent has an obligation to the human receiving the ticket. That obligation is information.

In the architecture the guide describes, the human agent receiving the escalation has no access to the conversation transcript.<sup>[3]</sup> That is the constraint the handoff pattern is designed around, and it is a constraint worth checking rather than assuming, because a deployment where the transcript does travel with the ticket has a different problem: too much context rather than none. If the agent escalates with a generic status like “*customer issue escalated*” and nothing else, the human agent receives a ticket with no context. They have to start the conversation over from scratch. The customer, who just spent time explaining the situation to the automated agent, now has to explain it again.

The ***structured handoff summary*** solves this. When compiling an escalation, the agent includes: *customer ID, root cause analysis, the amounts involved, *and a* recommended action*.<sup>[3]</sup> These are not summaries in the progressive-summarization sense, where details get lost in compression. They are structured extractions: specific, identifiable fields that a human can act on without re-interviewing the customer.

The handoff summary is the bridge between the agent’s context and the human’s context. Build it deliberately.

## Self-Critique: The Evaluator-Optimizer Pattern

There is a failure mode that looks like an escalation problem but is not. The agent resolves the case correctly. The customer is not asking for a human. No threshold is exceeded, no policy gap exists, no match is ambiguous. And satisfaction still drops, because the explanation is inconsistent: one resolution omits the relevant policy, the next omits the timeline, a third omits the next steps. The resolution is right; the communication around it is unreliable, and the specific gap is different every time.

None of the escalation triggers fire here, and none of them should. This is not a "when to hand off" problem. It is an output-quality problem, and the tool for it is the evaluator-optimizer pattern.

The pattern is a same-model second pass. After the agent drafts a response, it evaluates that draft against an explicit completeness rubric before presenting it: Does this address the customer’s actual concern? Does it include the relevant policy context? Does it state the timeline and the next steps? Does it anticipate the obvious follow-up question? If the draft fails the rubric, the model revises and re-checks. Only the passing version reaches the customer.

Distinguish this carefully from the same-session self-review anti-pattern in Chapter 8. Those are not the same operation, and conflating them is an exam trap. The Chapter 8 anti-pattern is a generator reviewing its own work for correctness: the session that wrote the code is asked whether the code is right, and it cannot answer honestly because it retains all the reasoning that produced the code and is biased toward confirming its own decisions. The fix there is an independent instance with no shared context. The evaluator-optimizer pattern is a different operation applied to a different target. It checks a draft for completeness against a rubric, not for correctness against the model’s own prior reasoning. There is no confirmation-bias problem because the model is not being asked to second-guess a conclusion; it is being asked to check a response against a checklist. The same model can do that reliably in the same session.

This is also why self-critique, not few-shot, is the correct intervention when the defect varies case to case. Few-shot examples teach a fixed pattern. When the omission is a moving target (policy this time, timeline next time, next-steps the time after), a finite example set cannot enumerate the space of possible gaps. A rubric check catches whatever gap appears in this specific output, because the rubric is defined abstractly over the dimensions of completeness rather than over a list of pre-seen cases. The intervention classifier in Chapter 9 turns on exactly this discriminator: the word "vary" in the stem eliminates few-shot and selects self-critique.

Keep this separate from the two review architectures the exam groups under Task 4.6: multi-instance and multi-pass review. Multi-instance review is a separate model instance with no shared reasoning context, the independent-reviewer pattern from Chapter 8; it is correct when the target is correctness and the generator’s retained context would bias the review. Multi-pass review, also covered in Chapter 8, splits a large multi-file review into per-file passes plus a cross-file integration pass to counter attention dilution; it is a decomposition move, not a self-critique move. The evaluator-optimizer pattern described here is a third, distinct operation: the same model running a second-pass check of a single draft against a completeness rubric. Three patterns, three failures. Per-case quality that varies unpredictably calls for the second-pass self-critique here; a generator biased toward confirming its own reasoning calls for multi-instance review; attention diluted across many files calls for multi-pass review. Read the stem for the failure it actually describes; the options will usually offer more than one, and the wrong choice is the right pattern aimed at the wrong failure.

## Access Failure vs Valid Empty Result

This is the most-tested distinction in Domain 5, and the failure mode is catastrophically silent.<sup>[4]</sup>

Consider a subagent that queries a customer database. Two different things can happen that result in an empty response:

1. The database query succeeded and returned no matching records.
2. The database was unreachable due to a timeout, and the query never ran.

In both cases, if the subagent returns {"customers": []}, the coordinator sees an empty array. The coordinator concludes: no customers found. In the first case, that conclusion is correct. In the second case, it is wrong. The coordinator does not know the search failed. It proceeds on the assumption that the search succeeded and found nothing.

The consequences depend on what happens next. In a billing dispute scenario, “no matching customers” might cause the agent to tell the customer it cannot find their account. In a research system, “no results” might cause the synthesis agent to conclude that no relevant sources exist on a topic that is actually well-documented.

Notice what makes this worse than an ordinary bug. An ordinary bug produces something visibly wrong: a crash, a malformed field, an answer that reads oddly. This produces an answer that reads perfectly. The reasoning downstream of the empty array is sound, the tone is confident, the citation trail is intact, and the conclusion is false. Nothing in the output distinguishes it from the same output in the case where the query genuinely matched nothing. The system has not made a mistake in reasoning; it has reasoned correctly from a fact that was manufactured by the transport layer. That is why the exam treats the distinction as structural rather than as a quality-of-error-message question, and why no amount of downstream prompting fixes it. The information required to tell the two cases apart was destroyed before the coordinator saw anything.

The correct pattern separates these two outcomes with distinct response structures.<sup>[4]</sup>

An access failure is reported as:

- The failure flag set, so the result is marked as a failed call rather than as data
- A category naming the failure type: transient, validation, permission, or business
- A retryability boolean, so the coordinator can decide whether repeating the call is worth anything
- A human-readable description of what went wrong

A valid empty result is reported as:

- The failure flag not set
- The empty collection itself
- Metadata confirming the query parameters that were actually searched

These are structurally different responses. The coordinator can distinguish them. It cannot distinguish them if both cases return only an empty array.

The category field matters for recovery decisions. **A transient timeout is retryable**. **A permission error is not.** A validation error means the query itself was malformed and retrying without modification will not help. Generic errors like “*search unavailable*” strip this information out before it reaches the coordinator, making intelligent recovery impossible.<sup>[4]</sup>

### Which Of Those Fields Are Real

The four bullets above come from two different places, and the exam does not distinguish them but an implementation has to.

The failure flag is a protocol field. In the Agent SDK it is `isError` in TypeScript and `is_error` in Python, returned by a tool handler alongside its content, and the documentation is direct about what it buys: it marks the result as a failed call rather than as data that happens to look odd, so Claude can say what went wrong instead of treating a failure as a normal result.<sup>[5]</sup> That sentence is the access-failure distinction stated as a tool-authoring rule.

`errorCategory` and `isRetryable` are not protocol fields. They are the exam guide's names for application-level metadata, and no MCP result schema reserves them.<sup>[4,5]</sup> They travel in the payload the tool composes. That is not a criticism of the guide, which is describing a discipline rather than an API, but it changes what a correct implementation looks like. A tool result can carry a machine-readable JSON object next to its content, and that object is where categorized error metadata belongs, because it arrives as exact fields rather than as text the model has to parse back out.<sup>[5]</sup>

Two live details bite here and are the kind of thing an implementation question can turn on. When a handler sets the structured JSON object, text blocks in the content array are not forwarded to Claude, on the assumption that they duplicate the structured data. A human-readable description written only into a text block therefore disappears at exactly the moment the structured metadata appears; it belongs inside the object. And the Python `@tool` decorator forwards only content and the error flag from a handler's return value, so an in-process Python SDK server cannot return the structured object at all. Getting categorized error metadata out of Python means running a standalone MCP server.<sup>[5]</sup>

There is also a boundary the guide's four categories sit inside rather than span. A tool handler receives arguments that have already been validated against its schema, so a call missing a required parameter never reaches the handler and never produces a tool result of any kind.<sup>[5]</sup> That failure is a protocol-level rejection: the problem was found before execution. Everything the four categories describe, transient through business, is an execution outcome: the call ran and the outcome was failure. Two different layers, two different reporting mechanisms, and an item that contrasts a missing parameter with an API returning a domain error is testing exactly that line.

### The Same Bug, In Anthropic's Own Product

This is not a hypothetical failure that only appears in exam scenarios. Claude Code hit it in its own MCP handling and shipped a fix for it.

An MCP server can notify Claude Code that its tool list has changed, at which point Claude Code re-requests the server's capabilities. Before version 2.1.214, a transient error during that refresh replaced the server's tools, prompts, and resources with an empty list. The current behavior keeps the previously discovered capabilities until a later refresh succeeds.<sup>[6]</sup>

Read that as the chapter's own distinction and it is exact. A refresh that fails is an access failure. An empty list is a valid empty result meaning the server offers nothing. The old code returned the second for the first, and the visible symptom would have been an agent that calmly reported it had no tool for the job, which is the same shape as the synthesis agent concluding that no research exists. The fix was not better error text. It was refusing to let a failure be represented as data.

## Local Recovery Before Coordinator Escalation

A related principle governs how errors move through the system: subagents implement local recovery for transient failures and propagate only errors they cannot resolve.<sup>[4]</sup>

This keeps the coordinator’s decision surface clean. If every subagent propagates every transient error immediately, the coordinator spends its attention on failures that the subagent could have retried successfully on the second attempt. The coordinator exists to manage complex decisions, not to be a retry queue. For the recovery side of this story, including crash recovery manifests and structured state exports, see Chapter 11.

The subagent’s local recovery responsibility has a scope limit: transient failures within the subagent’s authority. When a failure is not transient, when it represents a genuine capability gap or a threshold violation, the subagent propagates the error. When it does, it includes what was attempted and any partial results.<sup>[4]</sup>

Partial results matter. Consider a subagent that retrieved data for most of its requested topics before failing on the remainder. The coordinator needs to know which topics succeeded and which failed, so it can decide whether to retry the failures, fill the gap from another source, or proceed with an annotated gap in coverage. Propagating only the failure without the partial results discards the successful work.

There is a reason the propagated report has to be this complete, and it is topological. Chapter 2 established that a coordinator's view of a nested run is a single summary from its own direct child. Whatever the subagent chose not to put in that summary does not exist as far as the coordinator is concerned; there is no transcript underneath to consult, no tool output the coordinator can go back and read. The subagent's error report is not a notification that the coordinator can follow up on. It is the entire record. A subagent that reports a failure without saying what it attempted has not given the coordinator a lead to chase, it has given it a dead end.

### The Two Anti-Patterns Are Opposite Errors

Task Statement 5.3 names two failure modes and neither is a sensible default.<sup>[4]</sup> They are worth setting against each other, because a system tuned away from one tends to land on the other.

**Silent suppression** returns an empty result as though the query had succeeded. The workflow completes, the report is produced, and the failure is invisible. This is the access-failure confusion from the previous section, seen from the propagation side.

**Cascade termination** halts the whole workflow when any single subagent fails. Nothing is hidden, which is the improvement, but the cost is that one unreachable source destroys the work of every other subagent, including the ones that succeeded. In a research system with five subagents, a timeout on one of them should produce a report with four topic areas covered and one annotated as a gap, not no report.

The correct behavior sits between them, and it is not a compromise between them. It is a different move: report the failure with enough structure that the coordinator can decide what the workflow does next, and let the coordinator decide. Halting is one of the decisions available to it. What is wrong is the subagent making that decision unilaterally, in either direction.

The platform's default leans the same way. In the Agent SDK, an error raised inside a tool handler does not stop the agent loop; the in-process MCP server converts it into an error result and the loop continues, so Claude can retry, reach for a different tool, or explain the failure.<sup>[5]</sup> The framing is worth borrowing: how a failure is reported determines what the model reads, not whether the run survives. There is one real difference between letting an exception escape and catching it. An uncaught exception reaches Claude as the raw exception message. A caught error returned with the failure flag reaches Claude as whatever the handler composed, which can name the request that failed and what to try instead.<sup>[5]</sup> Same outcome for the loop, very different quality of information at the point of recovery. That is structured error context, arrived at from the tool side rather than the subagent side.

All four of those categories presuppose that the failure reached the result channel in the first place. Chapter 5 covers the prior question, which is whether a given failure belongs in that channel at all or is refused at the protocol layer before the tool runs. Everything in this section applies to failures that arrived as results.

The taxonomy from Task Statement 2.2 covers four error categories: transient (timeouts, service unavailability), validation (invalid input format), business (policy violations), and permission (access denied).<sup>[4]</sup> Each category implies a different coordinator response. 
**Transient**: retry with backoff. 
**Validation**: fix the query. 
**Business**: escalate. 
**Permission**: notify administrator.

### Where the Retryable Line Actually Falls

Categories are easy to recite and harder to apply, because the useful question is not what to call an error but whether repeating the call could change the outcome. Claude Code's own MCP connection handling draws that line explicitly, and it is the clearest worked example available of the discipline this task statement describes.

When a remote MCP server fails its initial connection, Claude Code retries up to three times with exponential backoff on transient conditions: a 5xx response, a connection refused, a timeout. Authentication failures and not-found responses are not retried. The stated reason is the operative one: those failures require a configuration change to resolve, so a retry cannot succeed.<sup>[6]</sup> The same split governs capability discovery after a connection is established, where transient network and server errors are retried and authentication errors, 4xx responses and request timeouts are not. A mid-session disconnect gets its own budget: up to five reconnection attempts, starting at a one-second delay and doubling.<sup>[6]</sup>

The test is not severity and it is not category membership. It is whether anything outside the call has to change first. A timeout might resolve on its own; nobody has to do anything. An expired credential will not resolve on its own no matter how many times it is asked, and each attempt costs a round trip while making the failure look intermittent rather than structural. That is why the guide's business-rule case takes an explicit non-retryable flag paired with an explanation the agent can give the customer.<sup>[4]</sup> A refund denied by policy is not a call that failed. It is a call that succeeded and returned a no, and an agent that retries it is asking the same question of a system that has already answered.

There is a third state that neither category covers and it is a genuine trap. Claude Code reports MCP server status as one of pending, connected, failed, needs-auth, or disabled, and the documentation says directly that pending is not a failure: it commonly means a server has not connected yet or that its tool list was served from cache with the connection deferred to first use.<sup>[7]</sup> Not-yet is a third outcome alongside succeeded and failed, and collapsing it into either one produces a wrong decision. Treated as failure, a working server gets written off. Treated as success, the coordinator concludes that a server with no tools listed has nothing to offer.

## Structured Error Context

Generic error propagation is an anti-pattern for the same reason generic status codes are anti-patterns: they hide recovery-relevant information behind a label.

“Search unavailable” tells the coordinator that something went wrong. It does not tell the coordinator what was attempted, which query parameters were used, whether any partial results exist, or whether an alternative approach might succeed. The coordinator receiving “search unavailable” can only decide: retry or abort. It cannot make an informed judgment about which other subagent might cover the gap, or what the cost of proceeding without this data point would be.<sup>[4]</sup>

Structured error context includes:<sup>[4]</sup>

- Failure type (from the taxonomy above)
- What was attempted (the query, the parameters, the approach)
- Partial results (whatever was retrieved before the failure)
- Alternative approaches (other tools or sources that might cover the gap)

This is more tokens. It is also the information that separates a coordinator that can recover intelligently from one that can only guess. The extra context is the interface contract between the subagent and the coordinator.

## Coverage Annotations

Synthesis output has the same information-completeness problem as error propagation, in a different form. When a synthesis agent compiles findings from multiple subagents, some topic areas may be well-covered and others may have gaps because sources were unavailable or subagents failed.<sup>[4]</sup>

Without annotations, the synthesis output looks uniformly confident. A reader of the report has no way to know which sections are based on multiple corroborating sources and which sections are based on a single low-confidence signal or, worse, on nothing at all.

Coverage annotations are explicit markers in the synthesis output indicating which findings are well-supported and which topic areas have gaps. They preserve the provenance signal through the synthesis step rather than flattening it into a uniform surface.

The exam guide lists coverage annotation among the skills under error propagation across multi-agent systems, and the scenario it fits most naturally is the multi-agent research system, where a coordinator delegates to a web search subagent, a document analysis subagent, a synthesis subagent and a report generator, and the deliverable is a cited report.<sup>[4]</sup> A synthesis agent that does not propagate coverage information forces the end reader to trust the output uniformly. That trust is not warranted. Coverage annotations are the honest interface.

Worth noticing where the annotation has to originate. It cannot be produced at synthesis by inspection, because by then the failure looks like an absence, and an absence is exactly what a successful query with no matches also looks like. The synthesis agent can only annotate a gap that something upstream told it about. Coverage annotation is therefore not an output-formatting decision; it is the last visible consequence of the reporting discipline in the preceding three sections. Get the error contract wrong and the annotation cannot be written, however carefully the synthesis prompt asks for it.

The platform provides a small demonstration of the same dependency. Claude Code tells the model which MCP server failed to connect and what the connection error was, including when a tool search comes back with no matching tool, so the model can report the connection failure rather than concluding the capability does not exist. That reporting depends on tool search being active, and in configurations without it, which include a custom API base URL, tool search switched off, a model that does not support it, and several hosted deployments, failed server connections are not reported to the model at all.<sup>[6]</sup> In those configurations, an unreachable server and a nonexistent capability are indistinguishable from inside the conversation. The mechanism that would carry the annotation is absent, so no amount of instruction produces one.

## Claim-Source Mappings and the DataWithProvenance Pattern

Provenance tracks where information came from. The problem it solves is that synthesis destroys provenance by default. When a synthesis agent takes findings from three subagents and combines them into a summary paragraph, the individual claim-source relationships are lost unless they are explicitly preserved.

The structural mechanism for explicit preservation is the claim-source mapping. Task Statement 5.6 requires subagents to output structured mappings, naming source URLs, document names and relevant excerpts, and requires downstream agents to preserve them through synthesis.<sup>[8]</sup> The requirement applies specifically in the coordinator-plus-subagents topology from Chapter 2, where subagent findings route through a coordinator synthesis step that can compress or discard source information. Each claim carries its source alongside it. When a synthesis agent processes these claims, it preserves the source mappings through the combination step rather than discarding them.

**A note on the name.** This book calls the record that carries a claim and its origin together *DataWithProvenance*, and the label appears in the glossary and in Chapter 2's forward reference. The name is the book's own, and so is the field list below. The exam guide does not name a pattern here, does not specify a class, and does not enumerate its fields. What the guide specifies is the requirement: mappings produced upstream and preserved downstream, with source identification and dates attached. The concrete shape is one way to satisfy that requirement, offered because a requirement stated abstractly is hard to hold in mind and easy to satisfy badly. Anything about the illustration that is not in the guide is not on the exam.

With that said, here is the shape. A provenance record pairs a value with the source it came from, a category describing how it was obtained, the time it was retrieved, and the identity of the subagent that produced it. The category is the interesting field, and it is not a probability. It records the manner of acquisition: read directly from an authoritative system, parsed out of a structured document, derived from surrounding context, or estimated. A number that came from a live database and a number that a model inferred from a paragraph are different kinds of thing, and a synthesis step that sees only the two numbers has no way to know that.

**The wrong fix, and it is a tempting one.** Once every value carries a category describing how reliable its acquisition was, the obvious next move is to rank the categories and, when two subagents disagree, keep the value with the better provenance and log the rest. This is wrong, and it is wrong in the way this whole task statement is about. Ranking by provenance is still selection: one value survives into the synthesis output and the disagreement does not. A log entry is not an annotation; nobody reading the report sees it. The guide's position is that conflicting values from credible sources are completed and explicitly annotated, and passed to the coordinator to reconcile.<sup>[8]</sup> Provenance metadata exists to make a conflict legible, not to adjudicate it. An automatic resolver built on top of a provenance field turns the field into the mechanism that destroys the thing it was added to preserve, which is the sort of failure that survives review because every individual step in it looks reasonable.

Requiring subagents to output structured claim-source mappings is the upstream design decision that makes downstream provenance preservation possible.<sup>[8]</sup> A subagent that returns findings as a prose paragraph has already destroyed the provenance. The coordinator cannot reconstruct it from the summary.

This is why the exam places claim-source mappings at the subagent-to-coordinator interface, not at the final output stage. Provenance must be captured at the point of retrieval and preserved through every transformation. Retrofitting it at synthesis time is not possible. The distinction to hold onto is between preservation and reconstruction. A mapping that flows through the pipeline as a field cannot be silently dropped, because dropping it requires deleting something. A mapping rebuilt afterwards by a citation-repair agent reading back over transcripts is an inference about which claim came from which source, and an inference that is right most of the time is indistinguishable, in the finished report, from one that is right every time.

### Where the Schema Does the Work

The same preservation-versus-resolution split appears one layer down, in extraction, and the wrong answer there is instructive because it is not obviously wrong.

Consider a document whose field has been amended: an original value and a later revision, both present in the source. If the extraction schema allows one value per field, the extraction has to choose, and the output is inconsistent across runs because nothing in the document says which one the schema wanted. The tempting fix is instructional: tell the model to always take the most recent amendment. It will mostly comply. But instructions steer behavior probabilistically, and a schema with one slot has already decided that only one value exists, so the instruction is patching an output shape that cannot represent the truth. The guide's requirement points the other way: complete the analysis with the conflicting values included and explicitly annotated, and let the coordinator decide how to reconcile them before anything reaches synthesis, with collection or publication dates attached so the ordering is recoverable.<sup>[8]</sup> The shape that satisfies that requirement holds more than one value per field, each carrying where in the document it came from and when it took effect, and leaves the choice to whichever downstream system has rules about it. The requirement is then encoded structurally instead of requested behaviorally, and the amendment history survives as data rather than being resolved away at extraction time.

Two more preservation requirements sit under the same task statement and are easy to overlook because they read as presentation concerns.

The first is sectioning. When sources genuinely disagree, a synthesis agent tends to fail in one of two opposite ways: a single confident sentence that hides the disagreement, or hedged vagueness that communicates nothing. Both come from trying to produce one uniform level of certainty. The structural answer is to separate the report into well-established findings and contested ones, keeping each source's original characterization and its methodological context intact.<sup>[8]</sup> Certainty then varies by section, which is what the evidence actually does.

The second is rendering. Synthesis pulls in content of different kinds, and converting all of it to one format costs something on at least one of them. Financial figures belong in tables, news findings in prose, technical results in structured lists.<sup>[8]</sup> Standardizing everything to a common intermediate representation before synthesis does not avoid this, because a common representation is itself a uniform format and imposes the same flattening one step earlier. Rendering by content type at synthesis is the move.

## Conflicting Statistics

Multi-source research systems will produce conflicting data. Two subagents searching different sources will find different numbers for the same metric. The naive resolution strategies are wrong in specific, auditable ways.

**Averaging conflicting values** without understanding why they conflict treats a measurement disagreement as statistical noise. If one source reports a value from a study conducted in 2022 and another reports a value from a study conducted in 2025, averaging them produces a number that corresponds to neither study and may mislead more than either individual value would.

**Arbitrarily selecting one value** removes the conflict from the output without resolving it. The coordinator sees a single number and has no way to know it was contested.

The correct approach is explicit annotation: include both values with their source attribution, confidence levels, and publication or collection dates.<sup>[8]</sup> The synthesis agent does not resolve the conflict; it presents the conflict with full context, allowing downstream humans or system components to make an informed reconciliation decision.

Requiring publication and collection dates in structured outputs is the specific mechanism for distinguishing temporal differences from genuine contradictions.<sup>[8]</sup> Two studies measuring the same quantity at different points in time may both be correct: the quantity changed between measurements. Without dates, that temporal difference looks like a factual disagreement. With dates, it looks like a historical trend, which is a more accurate representation.

## Stratified Accuracy Metrics

Aggregate accuracy metrics are comfortable lies.

The guide states the risk in one line: an aggregate figure, and it uses 97% overall as its example, may be concealing poor performance on a particular document type or a particular field.<sup>[9]</sup> The line is short enough to skim past, so what follows is the book's own worked illustration of it, with figures chosen to be internally consistent rather than taken from any source.

Take a pipeline that processes 1,000 documents in a period: 610 contracts, 300 receipts, 90 invoices. Contracts extract correctly every time, 610 of 610. Receipts run at 99%, so 297 of 300. Invoices run at 70%, so 63 of 90. Total correct: 970. Aggregate accuracy: 97.0%, which is the figure that goes on the dashboard and the figure that gets quoted in the meeting where somebody proposes reducing review.

Notice which category is failing. The invoice segment is wrong nearly a third of the time and contributes 27 of the 30 errors in the whole period, and it is the smallest of the three. That is not a coincidence in the example; it is the general mechanism. A segment's power to move the aggregate is proportional to its volume, so the segments most capable of hiding behind a good aggregate are precisely the low-volume ones. A category can fail outright without the headline number moving enough to be noticed, and the smaller it is, the more completely it can fail before anyone sees it in that number.

Tracking accuracy per document type and per field, rather than across the corpus, is what surfaces this.<sup>[9]</sup> Field level adds the second dimension: a pipeline might do well on product names and dates while failing consistently on line-item subtotals and tax calculations, and a document-type breakdown alone would show invoices at 70% without saying which part of an invoice is wrong. Both dimensions are named in the guide, and both are prerequisites for the decision they gate.

That decision is automation. The guide is specific: verify consistent performance across all segments *before* reducing human review.<sup>[9]</sup> The ordering is the tested point. A pilot that routes a slice of high-confidence extractions to downstream systems and watches what happens is a reasonable-sounding alternative and it is the wrong one, because it measures aggregate behavior again, in production, on live data, with the failing segment still hidden inside the average. The segment breakdown is available now, from data already collected, before anything is exposed.

### Sampling Is a Different Instrument

The guide names stratified random sampling alongside segment analysis, and the two are easy to fuse into one idea because they share a word. They answer different questions.

Segment analysis breaks known results down by category. It tells an operator where the errors already recorded are concentrated. Stratified random sampling pulls a fixed proportion of high-confidence extractions for human review on an ongoing basis, and its stated purposes are measuring the error rate inside that population and detecting novel error patterns.<sup>[9]</sup> The second purpose is the one that cannot be obtained any other way. A high-confidence output that is wrong is, by definition, not flagged, so nothing in the pipeline's own signals will surface it. Only looking at some of them will.

This is also where the tempting wrong fixes cluster, and there are two. If high-confidence extractions carry a real error rate, the reflex is to lower the confidence threshold, which pushes an entire category back into the review queue, costs reviewer capacity in proportion to volume, and still cannot detect the next unfamiliar failure. The subtler reflex is a rule set that flags the document features known to cause errors. Rules of that kind work, on the errors that motivated them. A rule is a description of a failure that has already been seen, and the population being sampled is the one where the unseen failures live. Random selection has no such blind spot, which is its entire value here; it finds what nobody thought to look for because it was not looking for anything.

### Where Self-Reported Confidence Is Actually Useful

This chapter has now said twice that self-reported model confidence is not a reliable signal, and the guide elsewhere builds review routing on top of confidence scores. That looks like a contradiction and is not, and an exam item can be written on either side of it, so the reconciliation is worth stating plainly.

Task Statement 5.2 rejects self-reported confidence as an escalation trigger: as a gate, evaluated by the agent, deciding autonomously whether to act.<sup>[1]</sup> Task Statement 5.5 endorses field-level confidence scores for routing review attention, with a condition attached: the thresholds are calibrated against labeled validation sets, and low-confidence or internally contradictory documents are routed to human review to make the best use of limited reviewer capacity.<sup>[9]</sup> Task Statement 4.6 adds a third use, verification passes in which the model reports confidence alongside each finding so that review can be routed on it.<sup>[10]</sup>

Three properties separate the endorsed uses from the rejected one, and any of them is enough on its own. The endorsed uses calibrate the number against labeled ground truth before trusting it, which converts a self-assessment into a measured one. They put a human at the far end, so a miscalibrated score costs reviewer attention rather than a wrong action. And they use the score to order a queue rather than to decide, which is a weaker demand: ranking survives a score that is badly scaled as long as it is roughly monotonic, while a threshold does not.

The rejected use has none of the three. An uncalibrated number gates an irreversible action with no human in the loop. The signal is the same in both cases; what differs is how much weight it is asked to carry, and the exam's apparent inconsistency is really a question about that.

Which leads to the output shape that a well-designed extraction pipeline actually emits. When both failure directions are present at once, low-confidence extractions passing through unreviewed and correct ones consuming review time, neither is a prompting defect and few-shot examples will not fix either. The pipeline output has to carry three things: the extracted fields with their confidence scores, a review flag whose threshold was set against labeled data, and the reasons that flag was raised. The scores alone leave the threshold decision to whatever reads the output next, which is how the first failure happens. The flag alone tells a reviewer that something needs attention without saying which field, which is how the second one persists. Both directions close only when the calibrated decision and the field-level explanation travel together.

## Practice Question: Putting It Together

Consider this scenario. A multi-agent research system has a web search subagent, a document analysis subagent, and a synthesis subagent. The web search subagent experiences a database timeout while searching for recent statistics. It returns {"results": [], "status": "ok"} to indicate it found nothing. The synthesis subagent receives this result and compiles a report concluding that no recent data exists for the queried topic. The report does not indicate any gaps in source coverage.

There are three separate failures here. The web search subagent returned a valid-empty-result response for what was actually an access failure, hiding the error from the coordinator. The synthesis subagent had no visibility into the failure because the error reporting was absent. And the synthesis output contained no coverage annotations indicating that a source category was unavailable.

The corrected design addresses each independently. The web search subagent sets the failure flag on its result and puts the category and the retryability boolean into the structured payload, so the failure arrives as a marked failure and not as data. The coordinator makes a retry decision rather than forwarding a misleading empty result. The synthesis subagent structures its output with explicit coverage annotations for any topic area where source data was unavailable or questionable. The human receiving the report knows exactly where the gaps are.

The three failures are worth separating because fixing only the first is the common half-measure. A subagent that reports honestly into a pipeline with no coverage annotation still produces a confident report, because the honest signal has nowhere to go. Reporting, propagation and annotation are one chain, and the chain has the strength of its weakest link rather than the average of the three.

This is what “knowing when not to act autonomously” actually means in practice: it is not only about escalation thresholds. It is about every design decision that preserves signal through the pipeline rather than silently dropping it.

## What the Exam Tests

The exam tests escalation and error propagation as a system of objective decisions, not judgment calls.

For escalation triggers, the exam distinguishes valid triggers (explicit customer request, policy gap, inability to make progress, ambiguous match requiring clarification) from the two named anti-patterns: negative sentiment and self-reported model confidence. Both anti-patterns appear in scenario answer choices labeled as wrong, not as edge cases. A business ceiling such as a refund limit also routes to escalation, but it belongs to hook design rather than to this task statement, and it is enforced before the call rather than decided by the agent. The proportionate first fix for a miscalibrated escalation boundary is stating the criteria explicitly with worked examples, not adding a classifier or a confidence router on top of an unstated rule.

For access failures versus valid empty results, the exam tests whether candidates recognize these as structurally distinct events. A subagent that returns an empty collection with a success status when a timeout occurred is a misclassification that propagates silently. The correct design marks the failure with the error flag and carries a category and a retryability boolean in structured metadata, while a genuine empty result carries no error flag and includes metadata describing what was searched.

For error propagation, the four-category taxonomy (transient, validation, business, permission) maps directly to coordinator recovery decisions. The exam tests whether each category implies the correct next action: retry with backoff, fix the query, escalate, or notify an administrator. It also tests the two named anti-patterns as a pair. Suppressing an error silently and terminating the whole workflow on one subagent failure are opposite mistakes, and neither is the default; the subagent recovers locally from transient failures and propagates the rest with structure.

For structured handoff summaries, the exam tests what must be included when escalating to a human agent who lacks transcript access: customer ID, root cause, amounts involved, and recommended action.

For provenance, the exam tests it as a design decision made at the subagent output stage, not at synthesis. A claim-source mapping carried forward as a field of the record cannot be silently dropped; attribution reconstructed after synthesis is approximate by construction. The same preservation-over-resolution logic decides the extraction cases: a field with an amendment gets a schema that holds multiple values with source location and effective date, not an instruction about which one to prefer.

Coverage annotations and conflicting statistics are tested together: the exam asks what the correct synthesis behavior is when sources disagree or when a source category was unavailable. Annotation with source attribution and dates is the answer; averaging, ranking by provenance, or silently discarding gaps is the anti-pattern.

For human review and confidence calibration, the exam tests the ordering. Validate accuracy by document type and by field before reducing review, not after; a production pilot measures the aggregate again and leaves the failing segment hidden. Stratified random sampling of high-confidence outputs measures the error rate inside them and finds novel patterns, which heuristic rules over known error features cannot do. And confidence scores are acceptable for routing a review queue once calibrated against labeled data, while remaining unacceptable as an autonomous escalation gate.

## Key Takeaways

- Valid escalation triggers are objective: explicit customer request (honor immediately), policy gap, inability to make meaningful progress, multiple ambiguous matches requiring clarification. Invalid triggers are subjective proxies: negative sentiment does not equal task complexity; self-reported model confidence is not a reliable proxy for actual case complexity. A business ceiling such as a refund limit belongs to hook design and is enforced in a PreToolUse hook ahead of the call, not decided by the agent.<sup>[1,2]</sup>
- When a customer explicitly requests a human agent, honor the request immediately without first attempting resolution. When a customer expresses frustration without explicitly requesting a human, acknowledge the frustration and offer resolution; escalate only if the customer reiterates the preference. When a lookup returns several matches, ask for another identifier rather than picking on a heuristic.<sup>[1]</sup>
- Structured handoff summaries (customer ID, root cause, amounts, recommended action) are the interface contract to human agents, who do not have access to the conversation transcript.<sup>[3]</sup>
- An access failure and a valid empty result are not the same event: an access failure carries the error flag plus a category and a retryability boolean in structured metadata, while a valid empty result carries no error flag and includes metadata on what was searched. The error flag is a protocol field; the category and retryability names are the guide's application-level metadata, not a reserved schema. Subagents implement local recovery for transient failures and propagate only errors they cannot resolve, always including what was attempted and any partial results.<sup>[4,5]</sup>
- Structured error context enables coordinator recovery: failure type, attempted query, partial results, and alternative approaches. Silent suppression and cascade termination are opposite anti-patterns and neither is the default. Coverage annotations in synthesis output indicate which topic areas are well-supported and which have gaps; without annotations, synthesis output appears uniformly confident when it is not.<sup>[4]</sup>
- Claim-source mappings preserved through synthesis are the mechanism; provenance must be captured at retrieval and cannot be reconstructed afterwards. Conflicting statistics are annotated with source attribution and collection dates rather than averaged, arbitrarily selected, or resolved by ranking provenance; publication dates distinguish temporal differences from genuine contradictions. A field with an amendment needs a schema holding multiple values with source location and effective date, not an instruction about which to prefer.<sup>[8]</sup>
- Aggregate accuracy masks per-category failures, and the smallest segments hide the best. Accuracy tracked per document type and per field must be validated before review is reduced, not after. Stratified random sampling of high-confidence outputs is a separate instrument that measures the error rate inside them and detects novel patterns. Confidence scores calibrated against labeled data are legitimate for routing a review queue and remain illegitimate as an autonomous escalation gate.<sup>[9,10]</sup>

The system that knows when not to act, and that reports its failures honestly, is the system that a human can actually trust.
