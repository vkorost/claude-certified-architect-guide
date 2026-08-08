# Chapter 12: Escalation, Provenance, and Error Propagation

**Summary:** *A reliable agentic system knows when not to act. Valid escalation triggers are objective and verifiable: an explicit customer request for a human agent, a genuine policy gap, an inability to make progress, a business threshold violation, or an ambiguous match requiring clarification. Negative sentiment and model self-confidence are not valid triggers. Error propagation must be structured and distinguishable: an access failure and a valid empty result look identical to a coordinator that receives only an empty array, yet the downstream consequences differ entirely. Provenance tracking must survive synthesis steps, because a claim without its source citation is not a claim you can act on. The DataWithProvenance pattern preserves claim-source mappings through synthesis by pairing each extracted value with its source, confidence category, and retrieval timestamp. Stratified accuracy metrics reveal per-category failure rates that aggregate numbers mask; a pipeline at 97% aggregate accuracy may be failing 30% of one document type. Structured handoff summaries ensure that human agents receiving escalated tickets have the specific facts they need without re-interviewing the customer.*

---

## The Ticket That Should Have Closed Itself

The human agent opens the ticket. Billing discrepancy. A straightforward overcharge. The support agent had routed it to the human queue with a note that said “negative sentiment detected.”

The overcharge is real. It is also trivially within the agent’s own resolution authority: two tool calls, get_customer then process_refund. The whole interaction would have resolved without escalation. Instead, a human is now involved, the customer waited in the escalation queue, and the agent’s first-contact resolution rate took a hit.

This is the trap the exam calls out directly: sentiment as a proxy for complexity.<sup>[1]</sup> The customer was frustrated. The task was simple. Those two things are independent, and the system conflated them. This chapter is about the patterns that prevent that conflation: knowing when to escalate, how to communicate failure, and how to preserve the chain of evidence that makes a decision auditable.

---

## Knowing When Not to Act Autonomously

The named concept for this chapter is “Knowing when not to act autonomously.” It sounds passive. It is actually one of the harder design problems in agentic systems, because the failure mode is invisible. An agent that escalates too much costs human time and degrades resolution rate metrics. An agent that escalates too little resolves things it should not. Both failures hurt real customers. The exam focuses on the triggers that distinguish them.

### Valid Escalation Triggers

The exam source material identifies five valid escalation triggers.<sup>[2]</sup> Each one is objective: it can be evaluated without inspecting the model’s internal state or reading emotional tone from message text.

**The customer explicitly requests a human agent.** This is the most important trigger. Honor it immediately, without first attempting investigation or offering resolution. The code pattern is straightforward: if *context.customer_requested_human is true*, escalate and return the reason. Do not try to resolve the issue first to “save” the escalation. That is a design choice that prioritizes the metric over the customer’s stated preference, and the exam treats it as an anti-pattern.<sup>[2]</sup>

**Policy is ambiguous or silent on the specific situation.** Consider a customer asking about competitor price matching when the company’s policy only addresses price adjustments for the company’s own previous sales. The policy document does not cover this case. That silence is a policy gap, and a policy gap is a valid escalation trigger. The agent cannot invent policy. Escalation is the correct behavior.<sup>[2]</sup>

**The agent cannot make meaningful progress.** The clearest version of this is exhausting retry attempts. A transient failure that persists beyond the retry limit represents a capability ceiling the agent cannot cross locally. This is distinct from the sentiment-based anti-pattern: the agent is not escalating because it is uncertain, it is escalating because an objective condition (retry count) has been met.<sup>[2]</sup>

**A business threshold is exceeded.** The $500 refund limit example from the customer support scenario is the canonical case.<sup>[1,3]</sup> An agent has a defined financial authority. Requests above that limit are not within the agent’s authority to resolve, regardless of the agent’s assessment of the request’s legitimacy. The threshold is objective. It is either exceeded or it is not.

**Multiple customer matches requiring clarification.** When a query returns multiple matching records, the agent must ask for additional identifiers. Selecting heuristically from ambiguous results is an anti-pattern. The heuristic might be right most of the time. It will be wrong at the worst possible moments, on accounts where the stakes are highest.<sup>[2]</sup>

### Invalid Escalation Triggers

Two specific anti-patterns appear in the exam source material and show up in scenario questions.<sup>[1,2,3]</sup>

**Negative sentiment.** An angry customer asking to update their address does not need a human agent. The emotional state of the message is not correlated with the complexity of the task. Routing based on sentiment conflates two unrelated signals.

**Self-reported model confidence below a threshold.** Model confidence scores are not reliable proxies for actual case complexity. A model might report low confidence on a routine task that it handles well, and high confidence on an edge case where it is about to make an error. Building escalation logic on self-reported confidence means the confidence number is doing work it cannot actually do.

The should_escalate function in the exam source makes this distinction explicit in code: the valid triggers are checked against objective context fields; the anti-patterns are commented out with the labels WRONG!.<sup>[3]</sup> The exam is essentially testing whether you can read that comment correctly.

### The Frustration Case

There is a specific nuance the exam tests separately from the binary “escalate or not” decision. When a customer expresses frustration but the issue is within the agent’s capability, the correct behavior is to acknowledge the frustration while offering resolution, and escalate only if the customer reiterates the preference for a human agent.<sup>[2]</sup>

This is different from ignoring the customer’s emotional state. Acknowledgment is appropriate. Automatic escalation is not. The customer’s stated preference for a human agent is what triggers immediate escalation, not the emotional register of the message.

---

## Structured Handoff Summaries

When escalation does happen, the agent has an obligation to the human receiving the ticket. That obligation is information.

Human agents do not have access to the conversation transcript.<sup>[4]</sup> This is the architectural reality the handoff pattern is designed around. If the agent escalates with a generic status like “*customer issue escalated*” and nothing else, the human agent receives a ticket with no context. They have to start the conversation over from scratch. The customer, who just spent time explaining the situation to the automated agent, now has to explain it again.

The ***structured handoff summary*** solves this. When compiling an escalation, the agent includes: *customer ID, root cause analysis, the amounts involved, *and a* recommended action*.<sup>[4]</sup> These are not summaries in the progressive-summarization sense, where details get lost in compression. They are structured extractions: specific, identifiable fields that a human can act on without re-interviewing the customer.

The handoff summary is the bridge between the agent’s context and the human’s context. Build it deliberately.

---

## Self-Critique: The Evaluator-Optimizer Pattern

There is a failure mode that looks like an escalation problem but is not. The agent resolves the case correctly. The customer is not asking for a human. No threshold is exceeded, no policy gap exists, no match is ambiguous. And satisfaction still drops, because the explanation is inconsistent: one resolution omits the relevant policy, the next omits the timeline, a third omits the next steps. The resolution is right; the communication around it is unreliable, and the specific gap is different every time.

None of the escalation triggers fire here, and none of them should. This is not a "when to hand off" problem. It is an output-quality problem, and the tool for it is the evaluator-optimizer pattern.

The pattern is a same-model second pass. After the agent drafts a response, it evaluates that draft against an explicit completeness rubric before presenting it: Does this address the customer’s actual concern? Does it include the relevant policy context? Does it state the timeline and the next steps? Does it anticipate the obvious follow-up question? If the draft fails the rubric, the model revises and re-checks. Only the passing version reaches the customer.

Distinguish this carefully from the same-session self-review anti-pattern in Chapter 8. Those are not the same operation, and conflating them is an exam trap. The Chapter 8 anti-pattern is a generator reviewing its own work for correctness: the session that wrote the code is asked whether the code is right, and it cannot answer honestly because it retains all the reasoning that produced the code and is biased toward confirming its own decisions. The fix there is an independent instance with no shared context. The evaluator-optimizer pattern is a different operation applied to a different target. It checks a draft for completeness against a rubric, not for correctness against the model’s own prior reasoning. There is no confirmation-bias problem because the model is not being asked to second-guess a conclusion; it is being asked to check a response against a checklist. The same model can do that reliably in the same session.

This is also why self-critique, not few-shot, is the correct intervention when the defect varies case to case. Few-shot examples teach a fixed pattern. When the omission is a moving target (policy this time, timeline next time, next-steps the time after), a finite example set cannot enumerate the space of possible gaps. A rubric check catches whatever gap appears in this specific output, because the rubric is defined abstractly over the dimensions of completeness rather than over a list of pre-seen cases. The intervention classifier in Chapter 9 turns on exactly this discriminator: the word "vary" in the stem eliminates few-shot and selects self-critique.

Keep this separate from the two review architectures the exam groups under Task 4.6: multi-instance and multi-pass review. Multi-instance review is a separate model instance with no shared reasoning context, the independent-reviewer pattern from Chapter 8; it is correct when the target is correctness and the generator’s retained context would bias the review. Multi-pass review, also covered in Chapter 8, splits a large multi-file review into per-file passes plus a cross-file integration pass to counter attention dilution; it is a decomposition move, not a self-critique move. The evaluator-optimizer pattern described here is a third, distinct operation: the same model running a second-pass check of a single draft against a completeness rubric. Three patterns, three failures. Per-case quality that varies unpredictably calls for the second-pass self-critique here; a generator biased toward confirming its own reasoning calls for multi-instance review; attention diluted across many files calls for multi-pass review. Read the stem for the failure it actually describes; the options will usually offer more than one, and the wrong choice is the right pattern aimed at the wrong failure.

---

## Access Failure vs Valid Empty Result

This is the most-tested distinction in Domain 5, and the failure mode is catastrophically silent.<sup>[5]</sup>

Consider a subagent that queries a customer database. Two different things can happen that result in an empty response:

1. The database query succeeded and returned no matching records.
2. The database was unreachable due to a timeout, and the query never ran.

In both cases, if the subagent returns {"customers": []}, the coordinator sees an empty array. The coordinator concludes: no customers found. In the first case, that conclusion is correct. In the second case, it is wrong. The coordinator does not know the search failed. It proceeds on the assumption that the search succeeded and found nothing.

The consequences depend on what happens next. In a billing dispute scenario, “no matching customers” might cause the agent to tell the customer it cannot find their account. In a research system, “no results” might cause the synthesis agent to conclude that no relevant sources exist on a topic that is actually well-documented.

The correct pattern separates these two outcomes with distinct response structures.<sup>[5]</sup>

An access failure is reported as: - *isError*: true - *errorCategory* field indicating the failure type (transient, validation, permission, or business) - *isRetryable* *boolean* so the coordinator can decide whether to retry

A valid empty result is reported as: - *isError*: false - *customers*: [] (the empty array) - Metadata confirming the query parameters that were searched

These are structurally different responses. The coordinator can distinguish them. It cannot distinguish them if both cases return only [].

The *errorCategory* field matters for recovery decisions. **A transient timeout is retryable**. **A permission error is not.** A validation error means the query itself was malformed and retrying without modification will not help. Generic errors like “*search unavailable*” strip this information out before it reaches the coordinator, making intelligent recovery impossible.<sup>[5]</sup>

---

## Local Recovery Before Coordinator Escalation

A related principle governs how errors move through the system: subagents implement local recovery for transient failures and propagate only errors they cannot resolve.<sup>[5]</sup>

This keeps the coordinator’s decision surface clean. If every subagent propagates every transient error immediately, the coordinator spends its attention on failures that the subagent could have retried successfully on the second attempt. The coordinator exists to manage complex decisions, not to be a retry queue. For the recovery side of this story, including crash recovery manifests and structured state exports, see Chapter 11.

The subagent’s local recovery responsibility has a scope limit: transient failures within the subagent’s authority. When a failure is not transient, when it represents a genuine capability gap or a threshold violation, the subagent propagates the error. When it does, it includes what was attempted and any partial results.<sup>[5]</sup>

Partial results matter. Consider a subagent that retrieved data for most of its requested topics before failing on the remainder. The coordinator needs to know which topics succeeded and which failed, so it can decide whether to retry the failures, fill the gap from another source, or proceed with an annotated gap in coverage. Propagating only the failure without the partial results discards the successful work.

The taxonomy from Task Statement 2.2 covers four error categories: transient (timeouts, service unavailability), validation (invalid input format), business (policy violations), and permission (access denied).<sup>[5]</sup> Each category implies a different coordinator response. 
**Transient**: retry with backoff. 
**Validation**: fix the query. 
**Business**: escalate. 
**Permission**: notify administrator.

---

## Structured Error Context

Generic error propagation is an anti-pattern for the same reason generic status codes are anti-patterns: they hide recovery-relevant information behind a label.

“Search unavailable” tells the coordinator that something went wrong. It does not tell the coordinator what was attempted, which query parameters were used, whether any partial results exist, or whether an alternative approach might succeed. The coordinator receiving “search unavailable” can only decide: retry or abort. It cannot make an informed judgment about which other subagent might cover the gap, or what the cost of proceeding without this data point would be.<sup>[5]</sup>

Structured error context includes:<sup>[5]</sup>

- Failure type (from the taxonomy above)
- What was attempted (the query, the parameters, the approach)
- Partial results (whatever was retrieved before the failure)
- Alternative approaches (other tools or sources that might cover the gap)

This is more tokens. It is also the information that separates a coordinator that can recover intelligently from one that can only guess. The extra context is the interface contract between the subagent and the coordinator.

---

## Coverage Annotations

Synthesis output has the same information-completeness problem as error propagation, in a different form. When a synthesis agent compiles findings from multiple subagents, some topic areas may be well-covered and others may have gaps because sources were unavailable or subagents failed.<sup>[5]</sup>

Without annotations, the synthesis output looks uniformly confident. A reader of the report has no way to know which sections are based on multiple corroborating sources and which sections are based on a single low-confidence signal or, worse, on nothing at all.

Coverage annotations are explicit markers in the synthesis output indicating which findings are well-supported and which topic areas have gaps. They preserve the provenance signal through the synthesis step rather than flattening it into a uniform surface.

The exam tests this in the context of the multi-agent research system scenario.<sup>[1]</sup> A synthesis agent that does not propagate coverage information forces the end reader to trust the output uniformly. That trust is not warranted. Coverage annotations are the honest interface.

---

## Claim-Source Mappings and the DataWithProvenance Pattern

Provenance tracks where information came from. The problem it solves is that synthesis destroys provenance by default. When a synthesis agent takes findings from three subagents and combines them into a summary paragraph, the individual claim-source relationships are lost unless they are explicitly preserved.

The DataWithProvenance pattern is the structural mechanism for explicit preservation.<sup>[6]</sup> This pattern applies specifically in the coordinator-plus-subagents topology from Chapter 2, where subagent findings route through a coordinator synthesis step that can compress or discard source information. Each claim carries its source alongside it: the source URL or document name, a relevant excerpt, and metadata indicating the confidence level and retrieval time. When a synthesis agent processes these claims, it preserves the source mappings through the combination step rather than discarding them.

The pattern is implemented as a dataclass with fields*: value, source, confidence (one of "verified", "extracted", "inferred", "estimated"), retrieved_at, and agent_id*.<sup>[6]</sup> The confidence field is not a probability; it is a categorical indicator of how the value was obtained. “*Verified*” means from an authoritative source like a live database. “*Extracted*” means parsed from a structured document. “*Inferred*” means derived from context. “*Estimated*” is a best guess.

Requiring subagents to output structured claim-source mappings (source URLs, document names, relevant excerpts) is the upstream design decision that makes downstream provenance preservation possible.<sup>[7]</sup> A subagent that returns findings as a prose paragraph has already destroyed the provenance. The coordinator cannot reconstruct it from the summary.

This is why the exam places claim-source mappings at the subagent-to-coordinator interface, not at the final output stage. Provenance must be captured at the point of retrieval and preserved through every transformation. Retrofitting it at synthesis time is not possible.

---

## Conflicting Statistics

Multi-source research systems will produce conflicting data. Two subagents searching different sources will find different numbers for the same metric. The naive resolution strategies are wrong in specific, auditable ways.

**Averaging conflicting values** without understanding why they conflict treats a measurement disagreement as statistical noise. If one source reports a value from a study conducted in 2022 and another reports a value from a study conducted in 2025, averaging them produces a number that corresponds to neither study and may mislead more than either individual value would.

**Arbitrarily selecting one value** removes the conflict from the output without resolving it. The coordinator sees a single number and has no way to know it was contested.

The correct approach is explicit annotation: include both values with their source attribution, confidence levels, and publication or collection dates.<sup>[7]</sup> The synthesis agent does not resolve the conflict; it presents the conflict with full context, allowing downstream humans or system components to make an informed reconciliation decision.

Requiring publication and collection dates in structured outputs is the specific mechanism for distinguishing temporal differences from genuine contradictions.<sup>[7]</sup> Two studies measuring the same quantity at different points in time may both be correct: the quantity changed between measurements. Without dates, that temporal difference looks like a factual disagreement. With dates, it looks like a historical trend, which is a more accurate representation.

---

## Stratified Accuracy Metrics

Aggregate accuracy metrics are comfortable lies.

Consider a document extraction pipeline that processes invoices, receipts, and contracts. An aggregate accuracy of 97% looks excellent. But if invoices are at 70%, receipts at 99%, and contracts at 100%, the aggregate number is masking a significant failure category.<sup>[6]</sup> The invoice pipeline is failing nearly a third of the time. The aggregate number would not tell an operator to look there.

Stratified accuracy metrics track performance per document type and per field rather than across the entire corpus.<sup>[6]</sup> The implementation tracks a dictionary keyed by document type, incrementing correct and total counts separately. Accuracy is computed and reported per category, not just in aggregate.

The exam connects this directly to the decision about when to automate high-confidence extractions. An aggregate 97% accuracy figure is not a reliable basis for automation decisions if the per-category breakdown shows that one category is at 70%. Before reducing human review on any category, validate accuracy by document type and field segment.<sup>[8]</sup>

Field-level accuracy adds a second dimension: a pipeline might perform well on some fields (product name, invoice date) while failing consistently on others (line-item subtotals, tax calculations). Tracking both dimensions reveals where the failures actually live, which is a prerequisite for fixing them.

---

## Practice Question: Putting It Together

Consider this scenario. A multi-agent research system has a web search subagent, a document analysis subagent, and a synthesis subagent. The web search subagent experiences a database timeout while searching for recent statistics. It returns {"results": [], "status": "ok"} to indicate it found nothing. The synthesis subagent receives this result and compiles a report concluding that no recent data exists for the queried topic. The report does not indicate any gaps in source coverage.

There are three separate failures here. The web search subagent returned a valid-empty-result response for what was actually an access failure, hiding the error from the coordinator. The synthesis subagent had no visibility into the failure because the error reporting was absent. And the synthesis output contained no coverage annotations indicating that a source category was unavailable.

The corrected design addresses each independently. The web search subagent reports *isError: true* with *errorCategory: "transient"* and *isRetryable: true*. The coordinator makes a retry decision rather than forwarding a misleading empty result. The synthesis subagent structures its output with explicit coverage annotations for any topic area where source data was unavailable or questionable. The human receiving the report knows exactly where the gaps are.

This is what “knowing when not to act autonomously” actually means in practice: it is not only about escalation thresholds. It is about every design decision that preserves signal through the pipeline rather than silently dropping it.

---

## What the Exam Tests

The exam tests escalation and error propagation as a system of objective decisions, not judgment calls.

For escalation triggers, the exam distinguishes valid triggers (explicit customer request, policy gap, retry exhaustion, threshold exceeded, ambiguous match) from the two named anti-patterns: negative sentiment and self-reported model confidence. Both anti-patterns appear in scenario answer choices labeled as wrong, not as edge cases.

For access failures versus valid empty results, the exam tests whether candidates recognize these as structurally distinct events. A subagent returning {"results": []} when a timeout occurred is a misclassification that propagates silently. *The correct design uses isError: true with errorCategory and isRetryable for failures, and isError: false with metadata for genuine empty results.*

For error propagation, the four-category taxonomy (transient, validation, business, permission) maps directly to coordinator recovery decisions. The exam tests whether each category implies the correct next action: retry with backoff, fix the query, escalate, or notify an administrator.

For structured handoff summaries, the exam tests what must be included when escalating to a human agent who lacks transcript access: customer ID, root cause, amounts involved, and recommended action.

For the DataWithProvenance pattern, the exam tests provenance as a design decision made at the subagent output stage, not at synthesis. Provenance cannot be reconstructed after the synthesis step compresses or discards source mappings.

Coverage annotations and conflicting statistics are tested together: the exam asks what the correct synthesis behavior is when sources disagree or when a source category was unavailable. Annotation with source attribution and dates is the answer; averaging or silently discarding gaps is the anti-pattern.

---

## Key Takeaways

- Valid escalation triggers are objective: explicit customer request (honor immediately), policy gap, capability limit or retry exhaustion, business threshold exceeded, multiple ambiguous matches requiring clarification. Invalid triggers are subjective proxies: negative sentiment does not equal task complexity; self-reported model confidence is not a reliable proxy for actual case complexity.<sup>[1,2,3]</sup>
- When a customer explicitly requests a human agent, honor the request immediately without first attempting resolution. When a customer expresses frustration without explicitly requesting a human, acknowledge the frustration and offer resolution; escalate only if the customer reiterates the preference.<sup>[2]</sup>
- Structured handoff summaries (customer ID, root cause, amounts, recommended action) are the interface contract to human agents, who do not have access to the conversation transcript.<sup>[4]</sup>
- An access failure and a valid empty result are not the same event: an access failure must be reported as isError: true with errorCategory and isRetryable; a valid empty result is isError: false with an empty array and metadata. Subagents implement local recovery for transient failures and propagate only errors they cannot resolve, always including what was attempted and any partial results.<sup>[5]</sup>
- Structured error context enables coordinator recovery: failure type, attempted query, partial results, and alternative approaches. Coverage annotations in synthesis output indicate which topic areas are well-supported and which have gaps; without annotations, synthesis output appears uniformly confident when it is not.<sup>[5]</sup>
- The DataWithProvenance pattern preserves claim-source mappings through synthesis steps; provenance must be captured at retrieval time and cannot be reconstructed after synthesis. Conflicting statistics should be annotated with source attribution and collection dates rather than averaged or arbitrarily selected; publication dates distinguish temporal differences from genuine contradictions.<sup>[6,7]</sup>
- Aggregate accuracy metrics mask per-category failures. Stratified accuracy tracking per document type and per field reveals where failures actually occur, which is a prerequisite for safe automation decisions.<sup>[6,8]</sup>

The system that knows when not to act, and that reports its failures honestly, is the system that a human can actually trust.
