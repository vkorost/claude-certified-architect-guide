# The Enforcement Ladder

Every domain in this book contains at least one question of the same shape. A requirement is stated, a measured failure rate against it is quoted, and four mechanisms are offered. The mechanisms are not equivalent, they are ranked, and the ranking is the answer.

Each rung is explained in the chapter that owns its mechanism, which is the right place for it. What no single chapter can give is the comparison, because comparing them means holding six chapters at once. That is what this card is for. It duplicates no argument; it indexes them.

The ordering principle is distance. How far does the rule sit from anything the model, the conversation, or the agent's own configuration can reach?

## The rungs, strongest first

**1. Structurally impossible.** The bad state cannot be expressed. Splitting one tool into two so an invalid parameter combination has nowhere to live; issuing a single-use token from a preview call that the executing call requires; marking a field optional so the model is never under pressure to invent a value for it. Nothing is being enforced here, which is the point: there is no rule to break because there is no way to state the violation. Chapters 4 and 10.

**2. Tool-internal.** The threshold check lives in the body of the tool's own implementation. Whatever the model passes, the function evaluates its own precondition and refuses. The agent has no path to this code at all. Chapter 3.

**3. Permission rule.** A declarative allow, ask or deny entry in settings.json, matched on a tool name and often a path, command or domain specifier. Outside model discretion entirely, and cheaper than anything below it, but its vocabulary is pattern matching, so it cannot reason about a value. Chapter 6.

**4. Hook.** Code that runs on a lifecycle event and returns a computed decision. It can inspect arguments, compare against state, and apply a threshold, which a permission rule cannot. It depends on a configuration layer being registered and loaded, which tool-internal enforcement does not. Chapter 3.

**5. Orchestration layer.** An annotation, a routing step, or a validation stage the orchestrator is expected to honour. Deterministic in principle, dependent in practice on a component outside the tool being configured correctly and staying that way.

**6. Validation after the call.** Server-side checks that catch the bad value once it exists. A genuine safety net, and the correct answer when the question is about containing damage. It is never the correct answer when the question is about root cause, because catching a fabricated value is not the same act as preventing one. Chapters 4 and 10.

**7. Tool description.** Advisory, and the strongest lever available for a different problem. Descriptions are what the model reads when deciding which tool to call and what to put in its parameters, so they dominate any question about selection accuracy. They guarantee nothing. Chapter 4.

**8. System prompt, few-shot examples, emphasis.** Probabilistic. The model weighs the instruction against everything else in its context and then acts or does not. Chapters 3 and 9.

## Using it

**Take the highest rung that appears in the options, not the highest rung you can imagine.** The ladder ranks mechanisms in the abstract; a question offers four. If no tool-internal option is on the page, the hook is the answer, and it is the answer without qualification. If a tool-internal option is on the page, the hook is second best. Two questions can quote the same threshold, the same failure rate and the same policy, and take different answers because the option sets differ. That is not inconsistency in the material. It is the ladder working.

**A quoted failure rate kills every rung below the fourth.** When a stem reports that an instruction is already in place and being violated in some measurable fraction of cases, it has told you that the instruction is being weighed rather than applied. Restating it more forcefully, adding examples, or capitalising it all operate on the layer that produced the failures. They move the rate; they do not reach zero.

**Ask what the question wants before you rank anything.** The ladder answers *how do I guarantee this*. It is the wrong instrument for *why did the model choose that*. Where a stem asks for a root cause, or what would make the model select correctly, or what fixes this in the first place, the winner is whichever mechanism reaches the model before it emits the call, and rung seven beats rung six. Both readings are legitimate and the stem always says which one it wants. Read the question stem for the verb, not the scenario for the drama.

**Strength is not the only axis.** A tool-internal check protects one tool. A hook holds a policy across a set of them, including servers whose implementation nobody on the team owns. A permission rule states a flat prohibition more cheaply and more legibly than either. Where a requirement spans tools, the highest rung is not automatically the right home for it.

## The failures each rung answers

- **Selection accuracy**, the model calling the wrong tool or filling a parameter with an invented value: descriptions first, then structural elimination. Enforcement mechanisms do not improve selection; they only contain its consequences.
- **Compliance**, a rule that must hold whatever the model concludes: rungs one through four, highest available.
- **Damage limitation**, a bad value that must not reach a downstream system: validation after the call, which is where it belongs and where it is not a consolation prize.
- **Consistency**, output that varies in form or judgment case to case: demonstrated examples, which is rung eight used for the job it is actually good at.
