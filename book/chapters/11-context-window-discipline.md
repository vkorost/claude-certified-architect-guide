# Chapter 11: Context Window Discipline

**Summary:** *The context window is finite, its middle is unreliable, and its contents decay in quality before they run out of room. Two distinct effects drive that. Position: information in the center of a long input is less likely to reach the output than information at either end. Volume: accuracy and recall fall as the token count climbs, whatever the ceiling happens to be. Both have architectural consequences, and this chapter works through them: where critical facts belong, how to choose a carry-forward mechanism when history has to be compressed, what actually survives compaction and what only appears to, how tool output and tool schemas accumulate, and how to design sessions that outlive a single window. The case facts block is the standing answer to progressive summarization: transactional facts extracted into a persistent block carried in every request, outside the history being summarized. It is one of several carry-forward mechanisms, and most of the difficulty is in choosing among them. Tool output accumulation is the dominant growth cost in long sessions. Tool schemas are the dominant fixed cost, which is why tool search now withholds them until they are needed. Subagent delegation keeps a coordinator lean by absorbing exploration transcripts elsewhere and returning only the finding. Several reflexes recur and none of them works, two of them chronically: raising the ceiling when the problem is degradation, and forking a session when the problem is staleness.*

---

## The Session That Ate Its Own Memory

The exam guide states the risk in a single line and then leaves it alone. Compress a conversation repeatedly and the figures, the dates, and the things the customer specifically asked for come back as generalities.<sup>[1]</sup> That is the whole of it. The guide does not work an example.

What follows is therefore this book's illustration rather than a documented case. The names and the figures are invented, no source prints them, and nothing in the argument rests on them.

A support session opens with a record that is entirely unambiguous. A customer named John Smith. Account ACC-12345. Order #98765, placed under promotion SUMMER2026 at an advertised price of $99.99 and charged at $150.00, an overcharge of exactly $50.01. A customer since 2019, seven years, which is why the record is flagged high priority.

By the fifth turn of the conversation, after the first summarization pass, the record looks different. “Customer called about billing issue with promotion.” No name. No account number. No order number. No amounts.

By the tenth turn, after the second pass, it is four words: “Customer has a billing issue.”

The name, the account, the order, the exact dollar figures, the promotion code, the tenure, the priority rating: all gone. Not corrupted. Not misread. Gone through the ordinary and entirely correct operation of a summarizer making room for new turns.<sup>[1]</sup>

This is the problem. The context window is not just a size limit. **It is a degradation engine when left unmanaged.** And the model cannot tell you that it lost something, because it no longer knows the something existed.

---

## The Lost-in-the-Middle Effect

Context windows have a size. That size is real and fixed per model. But size is not the only structural property worth understanding.

The exam guide names the effect and states it in one direction: what sits at the two ends of a long input is processed reliably, and what sits in the middle sections may simply not come back.<sup>[1]</sup> The effect is not about compression or summarization. It shows up even when the full context is present and nothing has been discarded. The middle gets less.

This is the *lost-in-the-middle effect*. Naming it is worth the sentence, because a named constraint is something to design around, where an unnamed “context problem” is something to worry about later and never actually address.<sup>[1]</sup>

The practical implication: where information sits in context is not neutral. Placing a customer’s account number in turn 3 of a 40-turn conversation and expecting it to remain salient in turn 40 is not a safe assumption. Placing a 3000-token tool output in the middle of a 180K-token context is not the same as placing it at the beginning.

Position-aware ordering is the corrective. Put the most important information at the beginning and end of context. Organize aggregated results with explicit section headers so key findings surface at the top of each section. When preparing inputs for a synthesis step, put the summary first, then the detail.<sup>[1]</sup>

### Position and volume are two different problems

It is worth separating the position effect from the effect that gets confused with it, because the two have different symptoms and different fixes, and an exam item that describes one will offer the other as a distractor.

Position is about where a fact sits relative to the two ends of the input. Volume is about how much is in the window at all. The API documentation names the second one directly and calls it *context rot*: push the token count up and both accuracy and recall fall off. The conclusion it draws from that is the one worth carrying, namely that curating what is in context matters as much as how much space is available.<sup>[2]</sup>

That last clause is the load-bearing one, and it is the reason this chapter exists. If quality fell only when the window filled, context management would be a capacity-planning exercise and the answer would always be a bigger model. Quality falls before the window fills. A session at 40 percent of a 1M-token window can hold more accumulated noise than a session at 90 percent of a 200K one, and the larger window is the one behaving worse.

The two effects also respond to different interventions. Position is fixed by ordering: move the fact to an end, anchor it in a block that is always injected at the front, put summaries above detail. Volume is fixed by removal: trim tool outputs, delegate verbose work elsewhere, compact, clear. Reordering a bloated context does not reduce it, and trimming a small context does not relocate anything. An architecture that has only one of these tools will be wrong roughly half the time.

Both effects are real. Plan for both.

---

## Progressive Summarization Risk

Summarization is useful. It compresses verbose history into manageable context, extends how long a session can run, and reduces the token cost of subsequent turns. The problem is not summarization. The problem is summarizing the wrong things.

Progressive summarization risk is the exam guide's name for a specific failure: run history through enough compression passes and the exact quantities, the calendar dates, the identifiers, and the things the customer said they wanted all come back as generalities.<sup>[1]</sup> The scene at the top of the chapter is the shape of it. The summarizer is not broken. It correctly identifies that “*John Smith, ACC-12345, order #98765, charged $150 versus $99.99 promotional price*” is about a billing issue. It preserves the semantic category while discarding the specific facts inside it.

For a conversational system where the outcome is “*resolve this customer’s billing issue*,” those specific facts are not decoration. They are the task. The exact overcharge amount is what the refund calculation requires. The order number is what the lookup tool needs. The account ID is what the authentication check verifies. Strip those and the agent can no longer do its job, even if it still knows the category of job it is supposed to do.

The progressive nature of the risk makes it worse. A single summarization pass might preserve enough. Two passes almost certainly will not. By the third pass, even the category might collapse into something so vague (“billing matter under review”) that the agent cannot act on it at all.

The detection problem is quiet. The model does not announce that it has lost critical facts. It continues to respond confidently, often substituting “typical patterns” for the specific case details it no longer has. A support agent that has lost the order number will ask for it again, or worse, attempt to operate without it, producing plausible-sounding but factually wrong responses.

---

## The Case Facts Block

The architectural response to progressive summarization risk is not to avoid summarization. It is to protect what must not be summarized.

**The case facts block is a structured collection of the session's transactional facts, extracted out of the conversation and carried in every prompt, positioned outside the history that gets summarized.**<sup>[1]</sup> The guide's own list of what belongs in it is short and concrete: amounts, dates, order numbers, statuses.<sup>[1]</sup> It is never summarized, because summarization never reaches it. It is present in every request.

Two properties do the work, and they are worth separating because scenarios test them separately. The first is *exclusion*: the block sits outside the summarized history, so no compression pass can touch it. The second is *position*: the block goes at the front, where recall is strongest. Exclusion is what the guide specifies for the case facts pattern directly. Position is what the guide specifies separately, as a mitigation for the effect described in the previous section, and putting the two together is this book's reading rather than a single instruction printed anywhere.<sup>[1]</sup> A block that is excluded from summarization but buried in the middle of a long input has solved one of the two problems.

For a customer support scenario, the block looks something like this:

```
## CASE FACTS (do not summarize; reference directly)

Customer:        John Smith
Account ID:      ACC-12345
Order:           #98765
Expected price:  $99.99 (promotion SUMMER2026)
Charged price:   $150.00
Overcharge:      $50.01
Customer since:  2019
Priority:        High

## RULES
Refund amount ($50.01) is within the $500 agent limit.
This case qualifies for immediate resolution.
```

The block includes the instruction to not summarize, not as a magic string the compactor recognizes, but because putting the intent in writing provides the compactor with context about how to treat it. The compactor reads your CLAUDE.md and your case facts block alike.<sup>[3]</sup>

The structure matters. Plain prose facts are easier to summarize into vague categories than a labeled list of fields. Labeled fields resist compression because they present information as data, not as narrative. The compactor has less to compress them into.

For multi-issue sessions, each distinct issue gets its own structured entry in the block, kept as a separate context layer from the summarized conversation history.<sup>[1]</sup> The layering is the point. One block holding a fused account of three issues is a summary already, and it will lose the boundaries between them before it loses anything else.

The case facts block is a position and structure decision, not a prompt instruction. Prompt instructions asking the model to “*remember important details*” will not survive compaction. The block survives compaction because it lives outside the history being compacted, injected afresh into each request from persistent storage.

---

## Choosing the Carry-Forward Mechanism

The case facts block is one answer. The harder skill, and the one the exam concentrates on, is choosing among several answers when the scenario has already told you which one it wants and has done so obliquely.

Every mechanism in this section carries something forward past a boundary that would otherwise discard it. They are not interchangeable. What separates them is the *kind* of information at risk, and secondarily whether the future demands on that information are known in advance.

### Precision against narrative

The first cut is the sharpest, and most items that look hard are this cut in disguise.

Some information has to survive *exactly*. An amount, a date, an identifier, a status, a dosage, a term the user defined earlier and expects to be used with that meaning. For these, a value that is approximately right is a value that is wrong. Summarization cannot preserve them, and this is not a defect to be tuned out. Summarization compresses by choosing what to say about a passage instead of repeating the passage, and a number restated in prose is a number that has been described rather than carried. Raising the fidelity of the summarizer lowers the probability of loss without ever reaching zero, which is the wrong shape of guarantee for a fact that must be exact. Extraction into a structured block is a different shape of guarantee: the value is never summarized, so it cannot be approximated. Extract first, then summarize everything else.

Other information has to survive *in substance*. Conclusions reached across a long investigation, the reasoning behind a decision, the state of an argument, findings that were accumulated over many sessions and would take many sessions to rebuild. Here compression is the correct behavior and not a compromise, because the value is in the conclusion rather than in the words. Truncation is wrong for the same reason it is tempting: it is cheap, and it deletes the earliest material, which in a long investigation is usually the material that everything since has been built on. Retrieval is wrong for a different reason: it can return the passages but cannot return the conclusion, because the conclusion was never written down anywhere to be retrieved.

So: precision facts get extracted and protected. Narrative conclusions get progressively summarized. A scenario that describes both is describing both mechanisms, and the block and the summary coexist without competing.

### Authoritative state against conversational history

A third kind of information is neither precise nor narrative. It is *current*, and its distinguishing property is that it changes.

A user states a preference at turn 4 and states a different preference at turn 30. Both statements are in the history, both are true records of what was said, and only one of them is the answer. Asked at turn 45, the model resolves the conflict from history and resolves it inconsistently, because nothing in the transcript marks one entry as superseding the other. Two turns of the same conversation can produce two different answers.

The instinct is to remove the stale entry. It is the wrong instinct twice over. It edits a record that other parts of the session may depend on, and it does not actually solve the problem, because a preference is rarely stated once and cleanly: it is implied, revised, hedged and referred back to, and pruning the entry that stated it outright can leave the weaker signals standing without the clear one to override them.

The mechanism that works removes the inference rather than the evidence. Maintain the current state as a structured object outside the conversation, update it as the state changes, and include it with every request. The model no longer has to work out which historical entry wins, because it is handed the answer. History becomes a record of how the state got here rather than the source of what it is.

That distinction generalizes past preferences. Any time a scenario describes a model resolving something inconsistently that the surrounding system already knows for certain, the fix is to supply the certainty rather than improve the inference.

Where the object goes is a smaller question with a consistent answer, and this is the book's reasoning rather than a rule printed anywhere. External system state belongs in the standing, high-authority part of the request rather than in the turn sequence. Injecting it as a synthetic user or assistant turn misrepresents it: nobody said it, and it now sits in a structure that records who said what. Having the model fetch it through a tool call on every turn spends a round trip re-acquiring data the application already had in hand. Placing it in the standing portion of the request costs neither, and it inherits the authority that portion carries. The general shape holds: state the system already knows should be supplied, not requested and not disguised as conversation.

### Known fields against unknown questions

The last cut is about the future rather than the information.

When tool outputs are crowding the window and more calls are coming, two answers present themselves. One is to keep the outputs and make them searchable, moving them into a store the agent can query semantically. The other is to stop keeping most of each output, extracting only the fields the task actually consumes and discarding the rest at the point of arrival.

Which is correct depends on a single question: is it known now what will be needed later? A task with an enumerated set of required fields, where the same handful of fields is read from every response, has already answered it. Extraction is the fit, and it is the stronger fit because it addresses accumulation at the source, where semantic storage addresses it after the fact and adds a retrieval step to every subsequent use. When the future demands are genuinely varied and cannot be listed in advance, the calculation inverts and retrieval earns its overhead.

The same question governs a related case. When retrieved passages rather than tool results are the thing crowding out the conversation, the corrective has to bound how much retrieved material is resident at once, typically by keeping only the results of the most recent few queries. Deduplicating those passages is the plausible wrong answer, and the reason it fails is worth stating plainly: deduplication reduces redundancy within whatever is present, and the problem is not that the present material is redundant, it is that the amount of it grows without limit. Removing repetition from an unbounded set leaves an unbounded set.

There is one category retrieval handles badly no matter how the question is phrased, and it is worth marking because the failure is systematic rather than occasional. A constraint that applies everywhere is, by construction, not *about* anything in particular. A rule prohibiting a class of action, a formatting requirement, a compliance boundary: none of them is topically close to the turn where they need to fire, because their whole character is that they apply regardless of topic. Similarity search ranks by aboutness, so a globally applicable rule scores poorly against every specific query and surfaces reliably for none of them. Summarization fails it for a neighboring reason, treating a rule that has not come up recently as low-salience material to compress. Invariants therefore do not go in a retrieval layer and do not go in the summarized history. They go in a region that is present at full fidelity in every request, on the same reasoning that puts transactional facts in the case facts block. This chapter's argument rather than a documented rule, but it follows directly from what retrieval and summarization each select for.

### Drift is not a window problem

One symptom belongs here and does not belong to any of the mechanisms above, because it is regularly misdiagnosed as one of them.

An agent follows its instructions early in a session and follows them less closely later. The turn count is high, the token count is not: the session is nowhere near the window limit, nothing has compacted, no history has been discarded, and every instruction the agent was given is still verbatim in the request. It is drifting anyway.

That is not a capacity failure and no capacity mechanism will touch it. The instruction has not been removed, it has been *outweighed*. Conversation accumulates, the accumulated material is more recent and more voluminous than the instruction, and the instruction's share of the model's attention falls as a matter of proportion. Starting a fresh session with a summary does correct it, and at the cost of discarding a session that had nothing wrong with it. The proportionate fix is to restate the constraint inside the conversation, at a natural break, close to where it next applies. This costs a few tokens and restores the signal without discarding anything.

The diagnostic is the token count. Degradation with the window near full is a volume problem. Degradation with the window a fraction full is a weighting problem. They present almost identically in the transcript and they take opposite fixes.

---

## What Consumes Context in the SDK

Understanding where context goes is prerequisite to managing it. The context window is everything available to Claude during a session, and it does not reset between turns within that session: the documentation is explicit that everything accumulates.<sup>[3]</sup> The sources are several and not all of them are obvious.

The system prompt loads with every request. Small fixed cost, but always present.

CLAUDE.md files load at session start via settingSources, and their full content is in every request thereafter. If a project has a large CLAUDE.md or several nested files, they consume context on every turn, before any conversation begins.<sup>[3]</sup>

Skill descriptions load at session start through the same setting sources, but only as short summaries. A skill's full body enters context when the skill is actually invoked.<sup>[3]</sup> This is a deliberately cheap default, and it is the same design idea that governs tool schemas below: advertise, then load.

Tool definitions load with every request, and this is where architects most often underestimate cost. Every schema, field description and example is part of the context every time Claude runs. Fifty tools can occupy ten to twenty thousand tokens.<sup>[4]</sup> Built-in tool schemas load on every request. MCP tool schemas are treated differently, and the difference is covered in its own section later in this chapter.<sup>[3]</sup>

Conversation history accumulates across turns. Prompts, responses, tool inputs and tool outputs all go into it. This is the dominant growth cost in long-running sessions, and large tool outputs dominate within it: reading a big file or running a command with verbose output can spend thousands of tokens in a single turn, and every one of those tokens is still there on every later turn.<sup>[3]</sup> The exam guide states the same point from the shape of the data rather than the size, with an order lookup returning forty or more fields when five of them are relevant.<sup>[1]</sup>

The implication: trim tool outputs to relevant fields before they accumulate. A *PostToolUse* hook that strips irrelevant fields from verbose MCP tool responses is doing context hygiene, not just data normalization.<sup>[3]</sup> The point of view worth adopting is that a tool result is not a record to be retained. It is an input to one decision, and the fields the decision does not read are pure carrying cost from the moment they arrive.

The API offers a server-side version of the same idea for callers working below the SDK. Context editing includes a tool-result clearing strategy that removes the oldest tool results once the conversation passes a configured threshold, replacing each with placeholder text so the model knows something was there.<sup>[5]</sup> The clearing happens on the server before the prompt reaches the model, and the client keeps its own full history unmodified.<sup>[5]</sup> Note the ordering rule that follows from this: clearing works oldest-first and chronologically, which means it is a mechanism for discarding what has already been processed, not a mechanism for discarding what is merely large.

Prompt caching handles the static pieces. Content that stays the same across turns, including the system prompt, tool definitions and CLAUDE.md, is automatically prompt cached, which reduces cost and latency for repeated prefixes.<sup>[3]</sup> It is worth being precise about what that buys, because the two things are routinely conflated. Caching changes what those tokens cost, not whether they count.<sup>[2]</sup> A cached prefix occupies exactly as much of the window as an uncached one. Caching is a bill-reduction mechanism that looks like a capacity mechanism, and treating it as the latter is how a session that appeared affordable turns out to have been full all along.

---

## Automatic Compaction and the compact_boundary Event

When the context window approaches its limit, the SDK automatically compacts the conversation. It summarizes older history to free space, keeping the most recent exchanges and key decisions intact. This is compaction as a default behavior, not something the architect has to implement.<sup>[3]</sup>

The SDK emits a signal when this happens. It appears in the stream as a message carrying type "system" and subtype "compact_boundary": in Python that surfaces as a SystemMessage, and in TypeScript as a distinct SDKCompactBoundaryMessage type.<sup>[3]</sup> Listening for the event tells you when compaction occurred, which is useful for logging, audit trails and state management. It is also the natural place to trigger a re-read of whatever external state the session depends on, since the event is the system announcing that the history behind it has changed shape.

The critical operational implication of automatic compaction: specific instructions from early in the conversation may not survive it. If you put deployment restrictions in the first user message and the context later compacts, those restrictions may disappear from the model’s working context. The model will not notice. It will continue running, now missing the constraint.

The fix is to *put persistent rules in CLAUDE.md*, loaded via settingSources, not in the initial prompt. CLAUDE.md content is re-injected on every request rather than being stored in conversation history, so it is never part of what gets compacted.<sup>[3]</sup> That statement carries an important qualification, and the next section is about it.

You can also include a summarization instructions section in CLAUDE.md telling the compactor what to preserve. The section header is free-form rather than a magic string; the compactor matches on intent, because it reads CLAUDE.md the same way it reads any other context.<sup>[3]</sup> The section can name the categories that must come through a pass: account and order identifiers, exact amounts, the current objective and its acceptance criteria, file paths already touched, decisions made and the reasoning behind them, errors seen.<sup>[3]</sup> The compactor uses it as guidance for what to keep.

Manual compaction is available via */compact* sent as a prompt string. It is SDK input, not CLI shorthand. Send /compact as a prompt to trigger compaction on demand, before the automatic threshold hits, when the session is visibly filling with discovery output that has served its purpose.<sup>[3]</sup> A manual pass also accepts a focus, which is the part worth remembering: instructing the compaction toward the work that matters keeps what was chosen rather than what an automatic pass inferred was important.<sup>[6]</sup> An automatic pass has to guess. A manual one does not have to.

Two adjacent controls belong with it. The threshold itself is adjustable, so the automatic pass can be made to run earlier than its default.<sup>[6]</sup> And clearing is a different operation from compacting: when the next task is unrelated to the last one, discarding the conversation outright is correct, because an old conversation crowds out the files the new task needs and costs tokens on every message until it goes.<sup>[6]</sup> Compaction preserves a session at reduced fidelity. Clearing ends it. Choosing between them is a question about whether the next task continues the previous one.

The *PreCompact* hook fires before compaction and is where the full transcript can be archived before the SDK summarizes it. It receives a trigger field distinguishing a manual pass from an automatic one, which matters because the two arrive under different conditions: a manual pass is a decision and an automatic one is a symptom. Chapter 3 owns the full hook mechanics. The architecture point here is that compaction is a signal rather than a failure, and PreCompact is where a system can answer that signal with preservation logic.<sup>[7]</sup>

One further behavior is easy to miss and shows up as a puzzling latency and cost change rather than as an error. The summarization request inherits the session's own extended thinking configuration, so a session running with thinking enabled reasons its way through the summary too. That affects how the summary is produced and leaves the session's settings unchanged afterwards.<sup>[6]</sup>

---

## What Survives Compaction

“Put it in CLAUDE.md and it survives” is the right instinct and an incomplete rule. What actually determines survival is not the file a rule lives in. It is *how the rule was loaded*, and specifically whether it entered the request as standing configuration or entered the message history as a message.

Anything that is not part of message history cannot be summarized away, because compaction operates on message history and nothing else. The system prompt is in that category, and so is an output style. They come through a pass unchanged, not because they are protected but because they were never in scope.<sup>[6]</sup>

Anything re-read from disk on each request comes back too. The project-root CLAUDE.md is re-injected after a pass, along with unscoped rules and auto memory.<sup>[6]</sup> This is what the previous section was describing, and for a single top-level file the rule as usually stated is accurate.

The exception is the one that catches people, and Chapter 6 already established it from the memory side. Nested CLAUDE.md files in subdirectories, and rules carrying `paths:` frontmatter, do not come back. They are conditional loads: they enter message history at the moment their trigger file is read, which puts them squarely inside the material compaction summarizes. They reload the next time Claude reads a file that matches.<sup>[6]</sup> The consequence in practice is a session that follows a subdirectory's conventions, compacts, and then stops following them until it happens to touch that subdirectory again. Nothing is broken and no error is raised. If a rule has to hold regardless of which file is open, it does not belong in a scoped file at all: drop the path scoping or move the rule to the project root.<sup>[6]</sup>

Invoked skill bodies are re-injected, with limits. Each is capped at 5,000 tokens and the total at 25,000, and once the total is exceeded the oldest invoked skills are dropped.<sup>[6]</sup> Truncation keeps the beginning of the file, which turns into a straightforward authoring rule: the instructions that must survive belong near the top of a skill, not at the bottom.

Hooks are not a context question at all. They run as code, so compaction has nothing to do with them.<sup>[6]</sup>

Discovered tool definitions sit at the boundary. When the conversation is long enough to compact, tools that were loaded on demand earlier may be removed along with everything else, and the agent searches for them again when it next needs them.<sup>[4]</sup> That is correct behavior rather than a fault, but it means a tool being available at turn 20 is not evidence that it will be available at turn 80.

The generalization worth carrying out of this section: *durability is a property of the loading mechanism, not of the content*. A critical rule written into a conditionally loaded file is a critical rule with a conditional lifetime, and no amount of emphasis inside the file changes that.

---

## Context Degradation Symptoms

Context degradation is insidious because the model does not report it. It degrades gradually and quietly, substituting general knowledge for specific session findings as the specific findings become less accessible. The exam guide names two symptoms for extended sessions and it names them together, which is the useful part: the model starts answering inconsistently, and it starts referring to what is typical of this sort of system instead of to the classes it actually found earlier.<sup>[1]</sup>

The two canonical symptoms:

Inconsistent answers. The model gives a different answer to the same question asked at turn 40 than it gave at turn 10. Not because anything changed, but because what was salient at turn 10 has drifted toward the middle of a 150K-token context and is receiving less attention.

“Typical patterns” substitution. Instead of referencing a specific finding from earlier in the session, the model references what is generally true about this kind of system. “Typically in this architecture…” instead of “Based on the auth module analysis from earlier…” This is the model doing the best it can with what is accessible, which is increasingly general knowledge rather than specific session work.<sup>[1]</sup>

The second symptom deserves more weight than its length here suggests, because it is the most reliable trigger in the whole domain. Generic language where specific language is expected is not a prompting failure and not a sign that the question was badly asked. It is a report on the state of the context, delivered in the only way the model has available. Once it appears, no further exploration in that context is trustworthy, and the useful response is to stop adding to the context rather than to try harder within it.

Both symptoms are recognizable in practice. Both are architectural failures before they are model failures. The architecture let the context fill without mitigation, and the model’s reliability degraded as a result.

---

## Four Reflexes That Do Not Fix Degradation

Before the mitigations that work, the ones that present themselves first. Each is a reasonable response to a misreading of the symptom, and each is the plausible wrong answer in a scenario that describes the symptom accurately.

**Raise the ceiling.** The context is degrading, the context has a size, so buy more size. This is the reflex the domain is largely built to defeat, and it fails on its own terms rather than on cost. A larger window does nothing to the position effect, which is about relative position within whatever is loaded and applies identically at any scale. It does nothing to the volume effect either, because quality falls with the token count and a bigger ceiling permits a bigger token count. Where the immediate problem is a hard boundary discarding earlier material that the user keeps referring back to, enlarging the boundary postpones the same event rather than preventing it; converting that older material to compressed form and keeping recent turns intact is what actually resolves it, and it resolves it at any window size. The honest statement of what a larger window buys is scope, not reliability: it changes what fits in one session, and it changes nothing about how well the contents are used. Discipline is not a workaround for a small window that a large window makes unnecessary.

**Reach for a stronger model.** A close relative, and it fails for the same reason with an extra step. Retention across a long session is not a reasoning capability, so a model that reasons better does not retain better. The information the agent has lost is not information it needs more capability to interpret; it is information that is no longer present in the request. What restores it is putting it somewhere the request can reach, which is a file, and any model can read a file.

**Fork the session.** When a session has gone bad, branching from it feels like a fresh start with the useful history retained. It is not. Chapter 2 established the mechanic: a fork begins as a copy of the original's history and inherits everything in it, including tool results that have gone stale, and it does not clear anything. Applied to a degraded session, forking produces a second degraded session. Forking is a divergence tool, for the case where two directions must be explored from one baseline that is *known good*. When the baseline itself is the problem, the operation that helps is summarizing what was learned and starting clean, which throws away the accumulation on purpose. The two look similar in a scenario and separate cleanly on one question: is the existing context worth carrying?

**Instruct the model to remember.** In a stateless request, an agent that cannot recall something said two turns ago is not forgetting. Whether prior turns are visible at all is a property of what the caller puts in the request, and the exam guide flags the discipline directly, in the form of passing the full conversation history along with each subsequent request so the exchange stays coherent.<sup>[1]</sup> If the history is not in the array, no instruction in the system prompt will retrieve it, because the system prompt cannot make the model see something that was not sent. The symptom is distinctive and worth recognising by its shape rather than its content: it appears immediately, in short sessions, in early testing, well before any window could plausibly be under pressure, and it reproduces uniformly across every user rather than clustering around particular operations. Failures that are that even are configuration failures. A genuine state bug would be lumpier.

The common thread is that three of the four treat a degradation problem as a capacity problem and the fourth treats an assembly problem as a memory problem. The diagnostic questions are small and worth asking in order: how full is the window, was the material ever sent, and is what is already loaded worth keeping?

---

## Mitigations

Three structural mitigations, each addressing a different failure mode.

**Scratchpad files.** A scratchpad is a file the agent writes to during exploration, recording key findings as it discovers them. Before a verbose analysis phase, the agent creates a scratchpad. After each major finding, it updates the scratchpad. If context fills and compaction occurs, the agent re-reads the scratchpad to restore the specific findings that might otherwise have been lost to summarization. The exam guide places scratchpads under exactly this heading, as the way key findings persist across a context boundary, and it pairs them with the instruction to refer back to the file on later questions.<sup>[1]</sup> The scratchpad is not context. It is external state, and it persists across context resets because it was never subject to them.

That last clause is the whole mechanism and it is worth being blunt about it. A scratchpad works for the same reason the case facts block works and for the same reason the project-root CLAUDE.md works: the information is somewhere the compaction pass has no jurisdiction over. Three patterns, three placements, one principle. What has to outlive a context boundary has to live outside the context.

The scratchpad pattern requires the agent to know it should use it, and to know it should go back to it rather than answer from what it can still recall. That instruction belongs in CLAUDE.md or in the system prompt, where it survives compaction. An instruction to keep a scratchpad, given only in conversation, is an instruction that will be summarized away at roughly the moment the scratchpad starts to matter.

**/compact on demand.** When the agent can detect that context is filling with verbose output that served its purpose and is no longer needed, triggering /compact before the automatic threshold clears that material while preserving a summary. Manual compaction gives more control over when compression happens and what the summary covers. Used with summarization instructions in CLAUDE.md, or with a focus stated in the command itself, it produces more targeted compaction than the automatic behavior.<sup>[3,6]</sup> The exam guide lists this plainly as a skill for extended exploration sessions that have filled with discovery output.<sup>[1]</sup>

**Subagent delegation.** This is the structural mitigation for verbose exploration phases. A subagent starts with a fresh conversation: no parent message history, no accumulated tool outputs from the parent session. It does its work, and only its final message returns to the parent as a tool result.<sup>[3]</sup> The parent’s context grows by that finding, not by the full exploration transcript.

“Fresh” is not the same as “empty”, and the difference is operationally important. A subagent does receive its own system prompt, the project CLAUDE.md loaded through setting sources, and its tool definitions. What it does not receive is the parent's conversation history, the parent's tool results, or the parent's system prompt.<sup>[8]</sup> The consequence is a design obligation that scenarios test: the only thing crossing from parent to subagent is the prompt string in the delegation itself, so every file path, error message and prior decision the subagent needs has to be written into that prompt.<sup>[8]</sup> Delegation with an underspecified prompt does not fail loudly. It produces a competent answer to a question that was missing its context, which is a considerably worse outcome than an error.

For a codebase analysis that requires reading dozens of files and following import chains across multiple passes, the verbose output of that exploration should stay inside a subagent. The coordinator gets the findings. The coordinator’s context stays lean, and the coordinator’s recall stays sharp across the full session.<sup>[1]</sup>

Subagent delegation does not mean the coordinator loses visibility. It means the coordinator’s context contains summaries and findings rather than exploration transcripts. That is the right shape.

### Summarize, then reset

The three mitigations combine, and the order in which they combine is itself examinable.

Consider a session that has explored one subsystem thoroughly, is showing the generic-language symptom, and now has to move on to an unrelated subsystem. Every individual instinct here is defensible and only one sequence is right.

Continuing in the current context is wrong because the symptom has already been observed: the context is degrading, and adding a second subsystem's exploration to it makes the degradation worse rather than diluting it. More precisely targeted prompting does not help, because the constraint is what the model can still reach, not what it was asked.

Clearing the context is right and clearing it *first* is wrong. The exploration findings exist nowhere except in the degrading context. Discard the context and they are gone, and rebuilding them costs exactly what building them cost. The exam guide states the ordering as a skill in its own right: summarize the findings from one phase before spawning the agents for the next, and inject those summaries into the initial context of what comes next.<sup>[1]</sup>

So the sequence is fixed. Capture the findings, in a summary or in a file. Then reset, either by clearing or by delegating the next phase to an agent seeded with the summary. The two halves are not interchangeable and neither is optional. Capturing without resetting leaves the degradation in place. Resetting without capturing is the expensive mistake, and it is expensive precisely in proportion to how much work the session had already done.

The same ordering governs the pivot case, where nothing has failed and the work has simply moved on. The findings from the finished phase are still irreplaceable, and the fact that the transition was planned rather than forced changes nothing about what has to happen before the reset.

### Read for the question, not for coverage

One more 5.4 pattern, and it sits earlier in the sequence than everything above: the cheapest context to manage is the context that was never loaded.

When the goal is to understand how something is structured across a set of files, reading every file in the set is the thorough-looking approach and the wrong one. Its cost scales with the number of files, which has nothing to do with the question, and it fills the window with implementation detail before any of it has been shown to be relevant. The structural approach starts from the anchor that defines the shape, the base class or the interface, and follows only the implementations that the question actually turns on. Cost then scales with the question instead of with the directory.<sup>[1]</sup>

The distinction that selects between them is what is being asked for. Architectural understanding has a structural entry point and rewards using it. Retrieval of a specific fact, where the fact could be anywhere, is the case where a broad search is appropriate. Applying the broad sweep to an architectural question is the common error, and it is the error that produces a full window and a shallow answer at the same time.

---

## Crash Recovery Manifests

Long-running agent sessions introduce a failure mode that shorter sessions don’t have: the session can crash after significant work has been done, and none of that work is automatically recoverable.

The architectural response is crash recovery through structured state exports. The exam guide gives the shape in two halves, and both halves are load-bearing: each agent writes its state out to a location that is known in advance, and the coordinator reads a manifest on resume and injects what it finds there into the agents' prompts.<sup>[1]</sup> An agent's export records what it completed, what it produced, and where in the task it had reached. The manifest is the coordinator's index of that: which agents have run, and how far.

On session resume, the coordinator loads the manifest and injects the relevant state into each agent’s initial context. The agents do not start from scratch; they start from the checkpoint. The coordinator knows which phases are complete and which need to continue.<sup>[1]</sup>

The manifest is persistent external state, not conversation history. It survives crashes because it lives on disk. It survives context compaction for the same reason. A session that resumes with a well-designed manifest can continue as if the interruption were minor, because the critical state is not inside the context window at all.

### Why the coordinator's transcript is not a substitute

The obvious cheaper design is to persist the coordinator's own conversation log and hand it back on resume. Everything that happened is in there. It is the wrong artifact, and the reason is precise enough to be worth stating rather than asserting.

A coordinator's log records the interface between agents: what was delegated, and what came back. That is a complete record of the coordination and a lossy record of the work. An agent that was halfway through a task when the process died reported nothing about that halfway point, because reporting happens at the end. The log therefore has no entry for the state that resumption actually needs. Worse, resuming an agent from a transcript means the agent reconstructs its own prior position by reading a conversation about itself, which is inference where determinism was available.

Per-agent exports invert both properties. Each agent's state is written by the agent that owns it, in a format it defined, at checkpoints it chose, and on resume it is handed back its own record rather than a narrative account of it.

The same argument disposes of a more sophisticated wrong answer: storing prior state in a vector store and retrieving it semantically on resume. Retrieval returns what matches a query, ranked by similarity, and it is very good at that. Resumption does not want the passages most similar to a query. It wants a specific set of state fields, all of them, with no possibility that one was ranked below the cut. Semantic search over state is a probabilistic answer to a question that has a deterministic one available, and its failure mode is silence: the field that did not surface produces no error, only an agent that resumes with an incomplete picture and proceeds confidently.

The general rule underneath both: for state, address the record directly. Search is for the case where the location is unknown.

Designing for crash recovery is designing against the assumption that sessions are atomic. For any agent session that runs for more than a few minutes and does meaningful work, that assumption will eventually be wrong. When the crash originates from a subagent failure, Chapter 12’s error propagation patterns govern how that failure is communicated before the session terminates.

---

## Context Window Sizes

The size of the ceiling matters when the work is large.

There are two tiers rather than a continuum, and the durable statement is about the tiers rather than about which models occupy them. The wider tier is a 1M-token window and the narrower is 200K.<sup>[2]</sup> Membership changes with every model release and any list printed here will be wrong within a year of printing: as of writing, the 1M tier holds a growing set of recent Opus and Sonnet releases along with several models outside those two lines, and Claude Sonnet 4.5 is among the models on the 200K tier.<sup>[2]</sup> The architectural facts that outlast the roster are these: there are two tiers, the ratio between them is five to one, on models offering the wider window it is the default rather than something to opt into with a header, and long-context requests on those models are billed at standard rates.<sup>[2]</sup> Check the current table before committing a design to a specific model. Do not carry a model list in your head as though it were a rule.

A 1M-token window does not eliminate the need for context discipline. The position effect operates at 1M tokens. Tool output accumulation happens at 1M tokens. Recall still falls as the token count climbs, because that is a function of how much is loaded and not of how much room is left. Compaction will still fire if the session runs long enough. What the ceiling being five times higher changes is scope: what is feasible in one session, and where the line falls between work that can proceed in place and work that needs delegation or cross-session state.

### Context awareness

Some models track their own remaining budget, which turns a blind constraint into a visible one.

Where the feature is active, the API injects the total window size into the system prompt of every request and follows each tool call with an update stating how much has been used and how much remains.<sup>[2]</sup> The mechanism is automatic. There is nothing to enable, and the tags are injected by the API rather than sent by the caller, which is worth knowing because a scenario that has an application constructing those markers itself has described something that does not happen.<sup>[2]</sup>

Coverage is uneven and splits along model lines rather than by recency. Several Sonnet releases and Claude Haiku 4.5 receive the injected tags; the recent Opus line and some models outside these families do not, and for those the equivalent capability is an explicitly supplied task budget, currently in beta.<sup>[2]</sup> As with window sizes, the membership is worth checking rather than memorising. The architectural consequence is what matters: a model that can see its remaining budget can decide how deep to go while there is still room to act on the decision, where a model without that signal works at a constant depth until something else intervenes.

Two things context awareness is not. It is not a relevance judgment, so it cannot tell the model which of the facts it holds still matter. And it is not a mitigation, so it does not slow accumulation or improve recall. It reports a quantity. What the agent does with the report is still an architecture question.

The model selection decision is therefore also a context architecture decision, along two axes rather than one: whether the window accommodates the work, and whether the model can see how much of it is left.

---

## MCP Server Context Cost and Tool Search

MCP server context cost deserves separate treatment because it catches architects who understand conversation history accumulation but underestimate the static overhead. Chapter 5 introduced tool search as a fact about the MCP surface and deferred the cost treatment here. This is it.

The cost, when it is paid, is the whole schema set. Every tool a server exposes, with every description and parameter, in every request, before the conversation begins. Not just the tools this turn will call. A server with a dozen richly documented tools can spend thousands of tokens per request on its own, and the documented figure for fifty tools is ten to twenty thousand.<sup>[4]</sup> Three such servers and a session starts each turn with a meaningful fraction of the window already committed to describing capabilities it will mostly not use.

### Deferral is the default, and that is the correction

The behavior here has moved and the old framing is now backwards. Tool search is not an optimization to reach for once the cost becomes a problem. It is on by default, and it withholds MCP tool definitions from the window until the agent needs them.<sup>[3,4]</sup> Upfront loading is now the *fallback*, not the baseline.

The mechanism: with tool search active, the agent gets a summary of what exists rather than the schemas. When a task calls for a capability it has not loaded, it searches, and up to five of the most relevant tools come into context and stay available for later turns.<sup>[4]</sup> The catalogue can run to thousands of tools without the window seeing them.

Because it is the default, the exam-relevant question inverts. It is not whether to enable it. It is when it is *not* in effect, and there are several such cases: certain hosting platforms reject the underlying mechanism server-side, some older model generations do not support it, a base URL pointing at a non-first-party host disables it because most proxies do not forward the necessary blocks, and a configuration can switch it off outright.<sup>[4]</sup> There is also a threshold mode, which measures the deferrable definitions against the window and only activates deferral once they reach a configured share of it, on the reasoning that a small tool set is cheaper to carry than to search for.<sup>[4]</sup> An individual server can be exempted so its tools are present at full schema from the first turn, which is the right setting for a server whose tools are used on essentially every turn.<sup>[9]</sup>

The tradeoff is one extra round trip the first time a tool is discovered. For a large catalogue that is repaid immediately by a smaller context on every turn. Below roughly ten tools, whose definitions fit comfortably anyway, loading everything upfront is typically faster.<sup>[4]</sup> Deferral is for the case where the catalogue is large and the usage is selective, which describes most MCP deployments and not all of them.

One consequence connects back to compaction. Tools discovered earlier live in the message history like anything else, so a long enough session can compact them away, after which the agent searches again when it next needs one.<sup>[4]</sup> Nothing is lost and the cost is another round trip, but it means tool availability is a property of the current context rather than of the session.

### The other MCP cost: output size

Schemas are the fixed cost. Tool results are the variable one, and the SDK enforces a ceiling on them that Chapter 5 also deferred here.

A tool result larger than 25,000 tokens is not truncated and not silently dropped. The full output is written to a file, and the tool result the agent receives is replaced with an error naming that file path, so the agent can read the output back in portions.<sup>[9]</sup> A warning fires earlier, at 10,000 tokens, and that threshold is fixed.<sup>[9]</sup> The limit itself is adjustable through an environment variable, and a server author can raise the threshold for one specific tool by annotating it in the tool listing, up to a ceiling of 500,000 characters, which is the right move for a tool whose output is inherently large and inherently necessary, such as a full database schema.<sup>[9]</sup>

Two things about this are worth carrying. The first is that the fallback is *persist to disk*, not discard: the data still exists and the agent has been handed a pointer to it, which converts an unbounded context cost into a bounded read the agent can make selectively. That is the same trade as tool search, applied to results instead of schemas, and it is the same trade as a scratchpad, applied in the other direction. The second is that the annotation and the environment variable govern different scopes, and both apply only to text. A tool returning image data stays subject to the token limit regardless.<sup>[9]</sup>

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

**Question 3.** **A coordinator has three MCP servers configured, roughly 15 tools each, and uses two or three of them in a typical session. Telemetry shows the full schema set present in every request. Which investigation is most likely to explain the overhead?**

A. Determine whether the deployment is one of the cases where schema deferral has fallen back to upfront loading.  
B. Reduce the number of MCP servers from three to one.  
C. Set tool_choice to "none" so that tool calls do not occur unnecessarily.  
D. Move to a model with a 1M-token window.

***Correct answer: A.** Withholding MCP schemas until they are needed is the default behavior, so schemas appearing in every request is a symptom that the default is not in effect. Several documented conditions produce that fallback, including unsupported models, certain hosting platforms, a non-first-party base URL, and an explicit override. B removes servers the agent legitimately needs. C prevents tool use altogether, which defeats the coordinator's purpose. D raises the ceiling without reducing the per-request cost, and the per-request cost is what was measured.*

---

## Key Takeaways

- Two effects, not one. Position: the middle of a long input is processed less reliably than either end, and reordering is the fix. Volume: accuracy and recall fall as the token count grows, and removal is the fix. Reordering a bloated context does not shrink it; trimming a small one does not relocate anything.
- Progressive summarization destroys specific facts quietly. The case facts block is the counter: transactional facts extracted out of the history, carried in every request, positioned at the front, and outside anything a compression pass can reach.
- Choosing the carry-forward mechanism is the actual skill. Precision facts get extracted and protected, because a higher-fidelity summary lowers the odds of loss without ever reaching zero. Narrative conclusions get progressively summarized, because their value is in the conclusion rather than the wording. Changing state gets an authoritative object supplied with each request, so the model is handed the current answer instead of inferring it from a history that contains every earlier answer too. Known required fields get extracted at arrival; genuinely unknown future questions are what earns a retrieval layer.
- Drift within a token budget that is nowhere near full is not a window problem. The instruction was outweighed, not removed, and restating it near where it applies costs less than discarding a working session.
- Durability is a property of the loading mechanism, not of the content. System prompt and output style are outside message history entirely. Project-root CLAUDE.md, unscoped rules and auto memory are re-read from disk. Nested CLAUDE.md files and path-scoped rules are not: they entered as messages when a matching file was read, and they return only when one is read again.
- Tool output accumulation is the dominant growth cost and tool schemas the dominant fixed one. Trim outputs at arrival. MCP schema deferral is now the default rather than an optimization, so the question in a scenario is which documented condition has caused a fallback to upfront loading. An oversized MCP result persists to disk and hands back a file path rather than vanishing.
- **Three structural mitigations, in a fixed order**: capture the findings first, in a ***scratchpad file*** or a summary, then reset with ***/compact*** or a ***clear***, or hand the next phase to a ***subagent*** seeded with what was captured. Resetting before capturing is the expensive mistake. A subagent starts fresh but not empty: only the delegation prompt crosses from the parent, so it has to carry everything the subagent needs.
- Crash recovery rests on two artifacts. Each agent writes a structured state export to a location known in advance, and the coordinator reads a manifest on resume and injects what it finds into the agents' prompts. The coordinator's own transcript is the tempting substitute and records the interface rather than the work; semantic retrieval over stored state is the sophisticated one and can silently omit a field.
- Four reflexes that do not work: raising the ceiling, upgrading the model, forking a degraded session, and instructing the model to remember what was never sent.
- Context window size is a model selection variable with two tiers, 1M and 200K, and a membership list that changes with each release. Check the current table rather than memorising it. Context awareness, where a model has it, reports the remaining budget automatically; it reports a quantity and judges nothing.
