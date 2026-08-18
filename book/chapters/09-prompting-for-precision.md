# Chapter 9: Prompting for Precision

**Summary:** *The exam tests prompt engineering as specification writing. A precise prompt eliminates ambiguity by stating explicit acceptance criteria, providing format constraints, and decomposing complex requests into enumerated steps. Vague instructions produce vague outputs; explicit categorical criteria produce consistent, auditable outputs. Two techniques anchor this chapter: the shift from vague directives to explicit categorical criteria (the named concept), and few-shot prompting as the tool for communicating reasoning about ambiguous cases. The alert-fatigue pattern shows how noisy, low-precision review tools destroy developer trust across all rule categories, not just the noisy ones. The sequencing insight is that high false-positive categories should be disabled while their prompts are repaired, rather than running them during revision; this protects the credibility of accurate categories while improvements are made. Output format is not stylistic preference when review output feeds downstream automated tooling: it is a system contract, and the format constraint belongs in the prompt with the same precision as the content criteria.*

---

## The Tool That Cried Wolf

The CI pipeline runs code review on every pull request. The prompt says: *“Review this code for quality issues. Be thorough and flag anything suspicious.”*

The reports come back. Hundreds of findings. Duplicate variable names in test files, functions that could theoretically be more efficient, missing JSDoc on internal utility functions that the team deliberately chose not to document. Developers scan the first three findings, realize they’re noise, and close the tab.

Two weeks later, someone merges a PR with a hardcoded API key. The review flagged it. Nobody read far enough to see it.

That vignette is this book's own. The exam guide does not print a case like it. What the guide supplies is the frame around it: Task Statement 4.1 is about designing prompts with explicit criteria so that precision improves and false positives fall, and Scenario 5 puts Claude Code in a CI pipeline that reviews code and returns feedback on pull requests, with the stated design problem being feedback a developer can act on without a flood of false alarms.<sup>[1,2]</sup> The illustration above is what that problem looks like unsolved.

And it is not a story about the model performing badly. The model did exactly what the prompt asked. It was thorough. It flagged suspicious things. The problem is that “thorough” and “suspicious” are not specifications. They are intentions. The model fills in the blanks with its own judgment, and its judgment does not match the team’s priorities.

The fix is not to ask the model to “be more selective.” That is still not a specification.

The fix is to write a specification.

---

## The Noise Problem Has a Name

When a code review tool flags too many non-issues, developers stop reading the reports. Including the real findings. This book calls that alert fatigue. The exam guide does not use the phrase, and no Anthropic source does either, so the term is a convenience rather than a citation. The mechanism underneath it is another matter: the guide states it directly under Task Statement 4.1, and the mechanism is what gets tested.<sup>[1]</sup>

Alert fatigue has a compounding property that makes it worse than it first appears. It is not just that developers ignore the noisy category. High false-positive categories undermine developer trust in accurate categories.<sup>[1]</sup> If the style section of the report is reliable garbage, the instinct is to distrust the security section too. The tool’s overall credibility collapses, and it stops functioning as a safety mechanism even in the areas where it actually works.

The natural first response to a noisy tool is to add a confidence qualifier: “only report high-confidence findings,” or “be conservative.” This feels like a fix. It is not a fix.<sup>[1]</sup> Confidence qualifiers are still vague. The model has no calibrated ground truth for what “high confidence” means relative to this codebase, this team’s conventions, or this specific review context. The instruction adds words without adding precision.

The correct response is **Explicit Criteria Over Vague Instructions**: replace intentional directives with categorical, measurable conditions.

That is the named concept for this chapter. Say it again: explicit criteria over vague instructions. Not “be thorough.” Not “be conservative.” A list of things to flag, with conditions precise enough that a second model (or a human) could verify whether each finding is correctly categorized.

---

## What Explicit Looks Like

The exam guide draws this contrast in a single line. It pairs a comment check stated as a condition, flag a comment only where its claim and the code's behavior conflict, against the same check stated as a goal, comment accuracy.<sup>[1]</sup> One of those can be evaluated. The other has to be interpreted.

The prompt pair below expands that one line into a working review specification. It is this book's construction, not a block the guide prints. Compare them:

```
# VAGUE
Review this code for quality issues.
Be thorough and flag anything suspicious.

# EXPLICIT
Review this code and flag ONLY the following:
1. Functions exceeding 50 lines of code
2. Async operations missing try-catch error handling
3. Hardcoded strings matching API key patterns (sk-, pk-, key-)
4. Public functions missing JSDoc documentation
5. SQL queries constructed with string concatenation

For each issue found, provide:
- File path and line number
- Which rule (1-5) was violated
- Severity: critical (3, 5) | warning (1, 2) | info (4)
- One-line fix suggestion
```

The vague prompt delegates judgment. The explicit prompt provides a contract.

Notice what the explicit version accomplishes that vague instructions cannot:

**Rule 1** (“Functions exceeding 50 lines”) is checkable. The model either finds a function longer than 50 lines or it does not. There is no ambiguity about what counts. A human reviewer can verify any flagged finding in seconds.

**Rule 3** (“Hardcoded strings matching API key patterns”) specifies the patterns explicitly. The model is not guessing what an API key looks like. It is matching against sk-, pk-, key-. Edge cases that used to require judgment (“is this a test key?”) can be addressed by extending the pattern list, not by hoping the model’s intuition holds.

**Severity** is categorical. Not “rate severity 1-10” (which the model will interpret differently on different runs), but a fixed mapping: rules 3 and 5 are critical, rules 1 and 2 are warnings, rule 4 is info. The classification is auditable. Downstream tooling can filter on severity reliably.

The output format is also part of the specification. Four required fields per finding, in a consistent structure. This is not stylistic preference. Scenario 5 puts the review in a CI pipeline that returns feedback on pull requests, and Task Statement 3.6 names the mechanism that carries feedback there: findings emitted in a machine-parseable structure so that a later step can post them inline.<sup>[2]</sup> The comment bot in this book's example illustrates that later step. The guide does not describe a bot, and nothing here depends on one. Once any downstream process parses the output, the format is a contract, which is why Task Statement 4.2 makes output format its own demonstration target rather than folding it into the content criteria.<sup>[1]</sup>

---

## The Alert Fatigue Mitigation Pattern

A development team ships a code review tool with six rule categories. Three of them work well: the security rules, the null-check rules, and the async error-handling rules. Three categories are noisy: style conventions that don’t match the team’s actual patterns, documentation checks that flag intentionally undocumented internal utilities, and complexity metrics calibrated for a different codebase.

The noisy categories do not stay in their lane. They erode trust in the good categories, and after a few weeks, developers have stopped reading the reports. The fix seems obvious: **fix the noisy prompts.**

Fixing prompts takes iteration. While the iteration is in progress, the noisy categories are still running and still burning credibility. The exam guide specifies the mitigation: temporarily disable high false-positive categories while improving prompts for those categories, rather than attempting all categories simultaneously.<sup>[1]</sup>

This is a sequencing insight, not just a prompt engineering insight. Production systems depend on trust. If the noisy categories are running, they are spending trust budget that the good categories need. Disabling them is not an admission of failure; it is protecting the signal while the noise is eliminated.

The pattern: **identify which categories have high false-positive rates, disable those categories, restore developer trust in the remaining output, then bring each disabled category back after the prompt has been revised and tested.**

*Do not attempt to fix all categories simultaneously.* Simultaneous repair is slower overall because the whole tool’s credibility stays low during the repair period. Sequential repair, category by category, preserves credibility for the working categories throughout.

---

## Severity Criteria Are Specifications Too

One of the skills tested in Task Statement 4.1 is defining explicit severity criteria with concrete code examples for each severity level.<sup>[1]</sup>

Severity labels without definitions are vague instructions. “Critical,” “warning,” and “info” sound precise, but they are not. What makes a finding critical versus a warning? If the answer is “judgment,” then severity will vary across runs and across different versions of the same model. Downstream tooling that auto-closes warnings but pages on-call for criticals will misfeed.

The specification pattern is the same as for rule criteria: replace intentional labels with conditions. Not “critical means serious security issues” but “critical: findings from rules 3 and 5 (hardcoded credentials and SQL injection vectors).” The severity is determined by which rule fired, not by a separate judgment call. A finding cannot be “warning severity” if it matched rule 3, regardless of how the model might have otherwise assessed its impact.

When severity must involve a contextual judgment, the specification should include examples. Not definitions, examples. “This is a critical finding” paired with a code snippet that exhibits the pattern. “This is a warning” paired with a different snippet. Examples communicate the decision boundary more precisely than prose descriptions because they anchor the label to a concrete case. This is the same mechanism as few-shot prompting, applied to the severity axis of the output.

---

## Few-Shot Prompting: When and How Many

Few-shot examples are the most effective technique for achieving consistently formatted, actionable output when detailed instructions alone produce inconsistent results.<sup>[1]</sup>

The exam guide is specific about count. Its skill bullet for this task statement asks for 2 to 4 targeted examples for ambiguous scenarios, each of them showing why one action was chosen rather than the plausible alternatives available at the time.<sup>[1]</sup> That is the number to carry into the exam, and the reasoning requirement is not decoration attached to it; it is half of what the bullet asks for.

No Anthropic source names a maximum. A specific ceiling quoted in third-party study material is invented until the documentation carries it, and the documentation does not. What the current prompting guidance does name is a preferred count, three to five examples for best results.<sup>[3]</sup> That is advice about examples in general rather than about the narrow ambiguous-case use Task Statement 4.2 frames, and the two ranges overlap at three and four. Where they diverge, the exam's number governs, because the exam is testing its own material. An answer of three or four satisfies both and is the safe place to stand.

The lower bound is worth as much attention as the upper one. A single example does not establish a pattern; it establishes a case, and the model has no way to tell which features of that case were the point. Too few examples for a genuinely ambiguous boundary leaves the model without enough signal to generalize, which is exactly what the guide's floor of two is guarding against.

Where few-shot examples earn their cost is in tasks with ambiguous boundaries. When the boundary between “flag this” and “skip this” requires contextual judgment that prose descriptions cannot fully specify, examples show the reasoning. Not just the outcome, the reasoning.<sup>[1]</sup>

This is a critical distinction. An example that shows input and correct output teaches the model the answer for that specific case. An example that shows input, correct output, and the reasoning for why that output was chosen (“this is a genuine issue, not a local pattern, because the same function exists in 14 other files with the same missing handler”) teaches the model how to approach similar cases it has not seen.

Three of the demonstrations the exam guide asks few-shot examples to perform bear directly on review output:<sup>[1]</sup>

**Consistent output format.** Every example should use the same output structure. If the desired output is a JSON object with four fields, every example should show a JSON object with those four fields. The model generalizes the structure from the examples; inconsistent examples produce inconsistent output.

**Acceptable patterns vs genuine issues.** A finding that looks like a problem but is actually a deliberate local convention is one of the hardest judgment calls for a code review tool. An example that shows this case explicitly, with the reasoning that identifies it as acceptable, teaches the model to distinguish the two rather than defaulting to one or the other.

**Ambiguous boundary cases.** Provide at least one example where the correct action is not obvious from the rule text alone. Show the example, show the correct classification, and show why. The model uses this to calibrate its judgment on novel patterns that fall near the same boundary.

The guide names two ambiguous cases in particular, and both are worth holding onto, because an item testing this task statement is more likely to be built on one of them than on anything else. The first is choosing the right tool when the request does not make clear which tool applies. The second is finding test coverage gaps at the level of individual branches, inside functions that already have tests.<sup>[1]</sup>

The second one is the sharper of the pair, and it repays a moment of attention. A review tool that flags a function with no tests at all is doing something an instruction can fully specify: look for functions, check whether a corresponding test exists, report the ones that do not. A review tool that notices an untested `else` inside a function whose happy path is covered is doing something else entirely. The function passes every check the rule text describes. The gap is real, and nothing in the rule names it. That is a recognition failure, and recognition is what an example teaches.

Which makes this the cleanest illustration in the domain of why the two task statements are not interchangeable. The reflex fix for missed branches is an instruction: enumerate every conditional in the function and confirm each has a corresponding assertion. That instruction is not a bad instruction. It is a Task Statement 4.1 instruction, and it sharpens a criterion the model is already applying. It does nothing about a case the model is not seeing in the first place. When the diagnosis is that the model does not recognize the case, adding precision to the rule it already follows is effort spent in the wrong place.

Consider the code review context. The rule is “flag functions exceeding 50 lines.” But a file has a 55-line function that is a configuration constant block, not logic. Should it be flagged? A rule that says “functions exceeding 50 lines” technically says yes. But the intent was to catch complex functions, not initialization blocks. An example that shows this case and explains the decision teaches the model to apply the rule’s intent, not just its literal text.

```typescript
Example 3 (Edge case: initialization block):
Input:
  File: src/config/defaults.js, line 47
  Function: DEFAULT_RETRY_CONFIG (58 lines)
  Content: Large object literal assigning default values

Output:
  {"flagged": false,
   "reasoning": "Rule 1 targets complex logic exceeding 50 lines.
   This function is a configuration constant (object literal, no
   branches, no state). Flagging it adds noise without identifying
   actual maintainability risk. Local pattern: configuration
   defaults are intentionally verbose in this codebase."}
```

That example does three things: shows the input, shows the correct output, and explains why this case falls outside the rule’s intent even though it technically matches the rule’s text. A model that has seen this example can generalize to other initialization blocks, documentation generators, and similar patterns without requiring a new rule for each.

Two mechanical points about how examples reach the model, both from the current prompting documentation rather than from the exam guide. The first is structural. Examples belong inside `<example>` tags, with a set of them wrapped in an `<examples>` tag, so that the model can tell a demonstration from an instruction.<sup>[3]</sup> A prompt that runs its examples together with its rules is inviting the model to read one as the other, and the usual symptom is a model that treats the content of an example as a standing constraint on every case.

The second is about what varies across a set. The documentation asks for examples that are diverse enough to cover edge cases and varied enough that the model does not pick up a pattern nobody intended.<sup>[3]</sup> That advice cuts in a specific direction, and it is easy to apply backwards. Vary the case each example shows. Do not vary the shape of the output. The shape is the thing the set is holding constant, and a set whose examples disagree about format teaches the model that format is a choice.

Everything above treats an example as prompt text, which is how Domain 4 frames it. There is a second channel worth knowing about. A tool definition accepts an optional `input_examples` array, and those examples are placed alongside the schema to show the model what a well-formed call looks like.<sup>[4]</sup> Chapter 4 owns that field and covers its behavior, including the fact that it is unsupported on server tools and that it is billed as prompt tokens on every request. The point here is only that the two channels are not substitutes. The `input_examples` array shapes how a tool gets called. Prompt examples shape the judgment the model applies before it decides to call anything and after the result comes back. A precision failure in review output is not repaired by adding examples to a tool definition, and a malformed tool call is not repaired by adding examples to the prompt.

---

## When Few-Shot Is Unnecessary

Few-shot examples are not universally valuable, and the reason is narrower than a general appeal to cost. The exam guide reaches for examples on a stated condition: detailed instructions alone are already producing inconsistent results.<sup>[1]</sup> Where the instructions are producing consistent results, the condition is not met and the technique has nothing to repair.

Examples do occupy the prompt, and it is fair to say so, but the only place Anthropic attaches a number to that is the tool definition, where a simple `input_examples` entry is documented at roughly twenty to fifty tokens and a complex nested one at roughly one hundred to two hundred.<sup>[4]</sup> Those are small figures. The argument against unnecessary examples is not that they are expensive. It is that they do no work, and a prompt full of demonstrations that demonstrate nothing is harder to revise later than a prompt without them.

The test: is the task boundary clear from the rule text alone? If the answer is yes, explicit criteria are sufficient and examples add cost without benefit. “*Flag SQL queries constructed with string concatenation*” is largely unambiguous. String concatenation in a SQL context is a recognizable pattern with few edge cases. A clear rule without examples will produce consistent output.

The test for few-shot: would a careful human reader, given only the rule text, arrive at the same classification for genuinely ambiguous cases? If yes: skip the examples. If no: the examples are doing real work.

Few-shot is valuable in extraction tasks too, particularly when documents have varied structures.<sup>[1]</sup> Consider a system extracting citations from research documents. Some documents use inline citations (author, year, in parentheses mid-sentence). Others use footnotes. Others use numbered bibliography entries that appear at the end. Instructions alone cannot reliably handle the variety. Examples showing correct extraction from each structure teach the model the invariant (what to extract) across varying surface presentations.

The principle is the same as the code review case: examples communicate the decision boundary in contexts where prose descriptions generate inconsistent behavior.

---

## Decomposition as Specification

Explicit criteria work at two levels: the criteria themselves, and the structure of the request.

A prompt that says “review this file for security issues, quality problems, and documentation gaps” asks the model to simultaneously manage three different classification tasks, each with its own criteria, while also producing output in a consistent format. This is not impossible, but it creates unnecessary competition between the sub-tasks.

The alternative: decompose the request. Task Statement 4.1’s skill list includes writing specific review criteria that define which issues to report (bugs, security) versus which to skip (minor style, local patterns) rather than relying on confidence-based filtering.<sup>[1]</sup> That decomposition is already implicit in the numbered rule list format: each rule is a discrete, independently evaluable criterion.

Explicit decomposition also makes prompt iteration easier. When a category is noisy, you can revise rule 4’s criteria without touching rules 1-3. When a category is missing findings it should catch, you can extend the definition of rule 3 without affecting the severity logic. The structure isolates the knobs.

This connects to how the exam frames iterative refinement for interacting versus independent issues: when fixes interact, provide them together; when they are independent, handle them sequentially.<sup>[5]</sup> The same logic applies to prompt revision. Independent criteria can be tuned independently. Criteria that share ambiguous boundary regions should be revised together.

---

## The Format Constraint Is Not Optional

The exam guide keeps criteria and output format as separate concerns, giving format its own skill bullet under Task Statement 4.2 rather than treating it as a property of the criteria.<sup>[1]</sup> The separation is worth respecting, because the two fail independently. It is easy to write detailed criteria and then leave the output format vague. The model will produce plausible-looking output. But “plausible-looking” is not “consistently structured,” and downstream tooling requires the latter.

The output format should specify: - What fields are present in each finding - What values are acceptable for categorical fields (severity, rule number) - What the finding omits (when “no issues found” is itself a valid output, specify what that looks like)

The code review example above includes all three. The finding has four fields. Severity values are enumerated. By implication, if no issues match rules 1-5, the correct output is an empty findings list, not a paragraph explaining that the code looks generally good.

Structured output via tool_use (covered in Chapter 10) enforces the format constraint at the schema level. For prompts that do not use tool_use, the format constraint belongs explicitly in the prompt, specified with the same precision as the content criteria.

---

## What the Exam Tests

The exam tests prompt engineering as specification writing. Domain 4 has two scenario homes in the exam guide, and both of them name prompt engineering as a primary domain: Scenario 5, a CI pipeline running automated reviews and returning feedback on pull requests, and Scenario 6, a structured extraction system reading unstructured documents and validating the result against a schema.<sup>[2]</sup>

That split runs through the task statements as well, and it is a useful thing to notice early. Every illustration under Task Statement 4.1 is a code review illustration, which makes code review the canonical setting for false positives, developer trust, and the contrast between vague and explicit. Task Statement 4.2 divides its illustrations between code review and extraction, the extraction half turning on informal values and on documents whose structure varies from one to the next.<sup>[1]</sup> Reading a stem for which of the two settings it is in is worth the second it costs, because in the common case it narrows which task statement is in play before a single option has been read.

The patterns the exam tests are narrow and specific:

**Explicit over vague.** “Flag functions exceeding 50 lines” beats “flag long functions.” “Flag comments only when claimed behavior contradicts actual code behavior” beats “check that comments are accurate.”<sup>[1]</sup> Every exam question that offers a vague vs explicit option: pick explicit.

**Confidence qualifiers fail.** “Be conservative” and “only report high-confidence findings” do not improve precision compared to specific categorical criteria.<sup>[1]</sup> Every exam option that proposes confidence-based filtering as the fix for false positives: discard it.

**Disable, then fix.** When a category has high false-positive rates, disable it temporarily while revising the prompt, rather than trying to reduce false positives while the category keeps running.<sup>[1]</sup> This is the sequencing principle.

**2 to 4 few-shot examples.** The count the exam guide names for ambiguous scenarios. For ambiguous boundary cases, include the reasoning. For output format, keep the examples consistent. One example does not establish a pattern. No documented maximum exists anywhere in Anthropic's material, and the general prompting guidance outside the exam prefers three to five, so three or four is the answer that satisfies both.<sup>[1,3]</sup>

**Examples show reasoning.** A few-shot example that shows input and output without reasoning teaches pattern matching. An example that shows input, output, and why teaches generalization.<sup>[1]</sup> The exam distinguishes these.

**One question separates 4.1 from 4.2.** Is the model reporting things it should not, or failing to report things it should? Over-reporting is a precision failure, it belongs to Task Statement 4.1, and the fix is a criterion that names the category to skip. Failing to see a case at all is a recognition failure, it belongs to Task Statement 4.2, and the fix is an example that shows the case being handled. The remedies are not substitutes in either direction. An option offering the right remedy for the other failure is the most common way this domain builds a wrong answer, and it is convincing precisely because the remedy on offer is a good one.

**Prefer the root cause to the workaround.** When the model fails to recognize a case, options that accept the gap and route around it are available and wrong: post-processing the bad output away, degrading gracefully, retrying, or filtering afterwards. All of them leave the recognition failure in place and pay for it on every request. The same principle applies on the 4.1 side, where the wrong reflex is to strip the noisy findings out downstream rather than to stop generating them.

**A version note on shaping output.** Older prompt engineering material controls the shape of a response by prefilling the opening of the assistant turn and letting the model continue from it. That technique is no longer available. On Claude 4.6 and later models, a request carrying a prefilled final assistant message returns a 400 error, and the documented replacement for the formatting and tone cases is an instruction in the system prompt naming what the response should and should not do.<sup>[3]</sup> An option that proposes prefilling a partial response is describing a previous generation's idiom, whatever else is right about it.

---

## Practice Questions

**Q1.** A CI pipeline runs code review with the prompt “identify any code quality issues and flag them with appropriate severity.” The team reports the tool flags too many minor style issues and developers have stopped reading the reports, causing critical security findings to be missed. Which change most directly addresses the root cause?

A. Add “be conservative and only flag high-confidence issues” to the prompt.  
B. Replace the vague instruction with a numbered list of specific rule categories and explicit severity mappings for each.  
C. Reduce the number of files sent to the review per run.  
D. Add a confidence score field to the output and filter on scores above 0.8.

*Correct: B. A and D add confidence-based filtering, which the exam guide explicitly identifies as failing to improve precision. C reduces scope but does not fix the underlying vague instruction. B replaces intentional directives with categorical specifications.*

**Q2.** A code review tool has six rule categories. Three produce accurate, trusted findings. Three generate consistent false positives. The team wants to restore developer trust in the tool overall while improving the noisy categories. What is the correct sequencing?

A. Improve all six categories simultaneously before re-enabling any of them.  
B. Disable the three noisy categories, restore trust in the three accurate categories, then revise and re-enable the noisy categories one at a time.  
C. Reduce the number of examples in the few-shot set for the noisy categories.  
D. Add “skip minor style issues” to the prompt for all six categories.

*Correct: B. The exam guide specifies temporarily disabling high false-positive categories while improving prompts for those categories. A delays the trust restoration. C addresses few-shot quantity, not the underlying criteria. D adds a vague modifier that does not produce categorical precision.*

**Q3.** A team is implementing few-shot prompting for a code review tool that must distinguish between “genuine issues” and “local patterns the team has intentionally adopted.” They are considering 4, 6, or 10 examples. The examples for the ambiguous boundary cases should include what, in addition to input and output?

A. A severity score for each example, to calibrate severity classification.  
B. Reasoning that explains why the output was chosen over plausible alternatives.  
C. Additional examples from adjacent rule categories to improve generalization.  
D. The file path and line number of the example to improve format consistency.

*Correct: B. The exam guide's skill bullet for this task statement asks that an example for an ambiguous case carry the reasoning behind the choice, not just the choice.<sup>[1]</sup> A adds severity calibration, which is a different concern. C pulls examples in from adjacent rule categories, which dilutes the set rather than sharpening the boundary the stem is about; the number of examples is not what is failing here, and no source names a maximum in any case. D addresses output format, not the ambiguous-case decision boundary.*

## Choosing the Intervention

Every Domain 4 scenario question hands you a failing system and four plausible fixes. The fixes are not interchangeable. Each one repairs a different *class* of failure, and the exam's wrong answers are almost always the right intervention applied to the wrong failure class. The skill being tested is not "*know what few-shot is*." It is *"read the failure signature and pick the matching tool*." The chapters teach these tools in isolation because the domains are weighted separately. The exam does not respect that separation, so this section collapses the boundary.

There are five interventions in play across Domains 1, 2, and 4. They map to five distinct failure signatures.

**The Five Interventions**

| Intervention | Repairs | Failure signature in the stem | Ruled out by |
| --- | --- | --- | --- |
| Explicit criteria | Undefined standard of what counts as a violation | "check that X is accurate," vague directive, both false positives and false negatives at once | A criterion that is already stated and checkable. If the rule names its own boundary and the model still misses a case, the definition is not what is missing |
| Few-shot examples | A single consistent pattern the model fails to recognize, or a mapping too complex to verbalize in prose | Clean accuracy drop between two task variants the model otherwise handles well; "interprets prose differently each time" on a fixed transformation | A defect that varies case to case; any quoted tool-call, round-trip or redundancy count; a criterion that was never defined; a behavior the model already supports and was simply never asked for |
| Decompose / parallelize / restructure | A workflow that is structurally wasteful | Inflated tool-call counts, redundant data fetching, sequential investigation of independent concerns, resolution-rate collapse with high round-trips | A stem with no efficiency signal in it. Output that is wrong or uneven at ordinary cost is a comprehension or quality failure, and reshaping an already-efficient workflow moves neither |
| Self-critique / evaluator-optimizer | Output that is correct but whose quality varies unpredictably per case | "gaps vary by case," "inconsistently explains," omissions that differ case to case, no fixed pattern to the defect | A fixed, repeatable defect, which is a criterion or an example. Also a correctness failure by the generator, which needs an independent instance, and attention diluted across many files, which needs a multi-pass split. Both belong to Chapter 8 |
| Prompt for a latent native capability | The model can already do the right thing but isn't | Sequential tool calls the model could batch; behavior the model supports natively but has not been instructed to use | Any behavior the model does not already support, and any failure of judgment or definition. An instruction cannot unlock a capability that is absent, and it cannot supply a rule nobody wrote |

The right-hand column is the half that does the work under time pressure. Finding the intervention that matches a stem is easy when the signature is clean, and the exam rarely offers a clean signature. Eliminating three interventions on the strength of what the stem does not say is both faster and more reliable, because absence is unambiguous in a way that presence is not. A stem that quotes no counts is not a restructuring problem no matter how inefficient the described system sounds. A stem whose defect repeats identically is not a self-critique problem no matter how uneven the output feels to read about.

Two of those eliminations need care, because they are the ones the neighbouring chapters constrain.

The first is the boundary between self-critique and the review architectures. Chapter 8 separates three operations that a stem can make look alike, and the separation is load-bearing here. A generator asked to review its own work for correctness cannot do it honestly, because it carries the reasoning that produced the work and is disposed to confirm it; the remedy is an independent instance with no shared context. A review spread thin across a large changeset is a different failure, dilution rather than bias, and the remedy is a multi-pass split with an integration pass. Neither of those is the evaluator-optimizer pattern, which Chapter 12 treats as a second-pass check of a single draft against a completeness rubric, in the same session, on output that is already correct. Three failures, three owners. The classifier's fourth row is the third of them only, and it is selected by variability, not by error.

The second is the boundary between few-shot and explicit criteria, which is the pair this chapter exists to separate. Few-shot loses whenever the underlying rule was never defined. Examples built on an undefined criterion teach the model a handful of instances of a rule that does not exist, and they fail on the first case that falls outside the set, which is the opposite of what examples are chosen for. Define the rule first. If the rule is defined and the model still cannot see the case, then the examples have something to attach themselves to.

The validation loop (Chapter 10) is a sixth tool, orthogonal to these: it catches *semantic* errors (valid structure, wrong values) and belongs to application code, not the prompt. It is never the answer to a prompt-engineering failure, and it is never substituted by any of the five above.

**The Classifier**

Run the stem through these questions in order. Stop at the first match.

1. Is the model failing because no clear standard of *what counts as a violation* exists? The prompt names a goal ("be accurate," "be thorough") but never defines the boundary. → **Explicit criteria.** Examples cannot substitute for a missing definition; they teach instances of an undefined rule and fail to generalize.
2. Is the workflow *structurally* wasteful? The stem quotes tool-call counts, round-trips, redundant fetching, or sequential handling of independent work. → **Decompose / parallelize / restructure.** This is a quantitative-efficiency failure. No amount of prompt content reshapes the workflow.
3. Does the model already natively support the correct behavior and simply isn't using it? → **Prompt for the capability.** Do not build infrastructure (composite tools, preprocessing model calls, speculative execution) for a behavior one prompt line unlocks.
4. Is the output *correct* but inconsistent in quality, with the defect *varying by case* in no fixed pattern? → **Self-critique / evaluator-optimizer.** A per-output check against a rubric catches whatever gap appears in that specific case; a finite example set cannot enumerate a moving target.
5. Is there a *single, consistent* pattern the model fails to recognize, or a transformation mapping too complex to express in prose, where the model otherwise performs well? → **Few-shot examples.** This is the only cell few-shot owns.

Few-shot is the default wrong answer when the failure is definitional (1), structural (2), capability-latent (3), or variable-quality (4). It is correct only in case 5.

**Few-Shot Elimination Triggers**

Specific stem language that *rules few-shot out*:

- "the specific context gaps vary by case" / "inconsistently" / "differ from case to case" → variable quality → self-critique, not few-shot.
- "averages N+ tool calls" / "redundant" / "sequentially" / "round-trips" → structural → decompose, not few-shot.
- "check that X is accurate" / "be conservative" / "be thorough" → undefined criterion → explicit criteria, not few-shot.
- "Claude already supports" / "natively" / "in separate sequential turns even when both are needed" → latent capability → prompt for batching, not few-shot or composite tools.

Specific stem language that *rules few-shot in*:

- A clean accuracy split between two variants of the same task ("94% on single-concern, 58% on multi-concern") where the model handles the easy variant well → recognition gap → few-shot.
- "interprets the requirements differently each time" on a *fixed* transformation target whose mapping is hard to state in prose → few-shot beats rewriting the prose more precisely.
- An abstract instruction that behaves inconsistently the further into a long context it sits → replace the abstraction with concrete examples of the behavior. The tempting options here move the instruction somewhere more prominent or re-inject it every few turns. Both treat position as the problem. The instruction is abstract wherever it sits, and re-stating an abstraction more often does not make it concrete. Chapter 11 covers what position genuinely governs.
- Extraction that omits a field on some documents and formats it differently on others → few-shot, and specifically not making the field required. Marking a field required removes the omissions by removing the option of silence, which converts a gap into a fabricated value; the exam guide takes this seriously enough to make optional fields a skill in its own right under Task Statement 4.3, on the grounds that a required field pressures the model into inventing something to fill it.<sup>[6]</sup> Chapter 10 owns the schema mechanics. The point for this chapter is that required fields address presence, examples address the transformation, and only one of those is the failure being described.

**Minimal Pairs**

The pairs below are near-identical in surface topic and opposite in correct answer. The discriminator is printed in the last column. Memorize the discriminators, not the scenarios.

**Pair 1: Multi-concern customer request**

|  | Scenario | Correct | Discriminator |
| --- | --- | --- | --- |
| A | Agent handles single-concern at 94%, multi-concern at 58%; mixes up parameters, addresses only one concern | Few-shot | The agent already executes well; it fails to recognize the multi-concern pattern. Recognition gap, single consistent pattern. |
| B | Complex cases average 12+ tool calls, 54% resolution; investigates concerns sequentially, re-fetches customer data per concern | Decompose + parallelize + shared context | Quantitative-efficiency collapse. The workflow shape is wasteful. Few-shot cannot restructure it. |

Same domain, same "multi-concern" framing. A is a recognition gap (few-shot); B is a structural gap (restructure). The numbers tell you which: B quotes tool-call counts and redundancy, A quotes a clean accuracy split.

**Pair 2: Few-shot vs self-critique**

|  | Scenario | Correct | Discriminator |
| --- | --- | --- | --- |
| A | Resolutions technically correct but explanations inconsistent; sometimes omits policy, sometimes timeline, sometimes next steps; gaps vary by case | Self-critique (evaluator-optimizer) | Defect is variable. No finite example set covers a moving target. A per-output rubric check does. |
| B | Agent fails a single consistent pattern it doesn't recognize (e.g., multi-concern routing) | Few-shot | Defect is a fixed pattern. Examples teach it directly. |

The trigger word is "vary." When the omission varies case to case, few-shot is eliminated and self-critique wins.

**Pair 3: Schema vs few-shot**

|  | Scenario | Correct | Discriminator |
| --- | --- | --- | --- |
| A | Output structure varies; fields nested differently; model interprets prose transformation differently each iteration | Few-shot | The mapping is wrong, not the container. A schema validates form, not transformation logic. Comprehension gap, not verification gap. |
| B | Downstream parser breaks on malformed output; missing fields, wrong types | Schema (strict: true) | The form is wrong. Schema enforces form at the grammar level. |

Schema fixes syntax; few-shot fixes the transformation. They are not substitutes. A schema will happily validate a wrongly-transformed-but-type-correct output.

**Pair 4: Explicit criteria vs few-shot**

|  | Scenario | Correct | Discriminator |
| --- | --- | --- | --- |
| A | Prompt says "check comments are accurate"; flags acceptable patterns, misses genuinely stale comments | Explicit criteria | The criterion is undefined. "Accurate" has no boundary. Examples paper over a missing definition and fail to generalize to novel contradictions. |
| B | Criterion is clear; the mapping or boundary is hard to verbalize | Few-shot | The definition exists; examples calibrate the fuzzy edge. |

When the underlying rule is undefined, define it first. Few-shot built on a vague instruction is building on sand.

**The One-Sentence Rule**

Few-shot owns exactly one failure class: a consistent, recognizable pattern (or hard-to-verbalize mapping) the model fails to apply despite performing well otherwise. Every other failure class, definitional, structural, capability-latent, or variable-quality, has a different owner, and selecting few-shot for those is the exam's most common engineered wrong answer.

(For the multi-pass review architecture that decomposes a large changeset into per-file passes plus an integration pass, see Chapter 8. Multi-pass review corrects for attention dilution, not for any of the five intervention failures above; it is a decomposition of the review task itself.)

---

## Key Takeaways

- **Explicit criteria over vague instructions** is the named concept: categorical, measurable conditions (“flag functions exceeding 50 lines”) replace intentional directives (“flag long functions” or “be thorough”). The exam tests this contrast in every prompt engineering question.
- **Alert fatigue** is this book's name for a documented mechanism, not a term the exam guide uses. The mechanism is what matters and it is stated plainly under Task Statement 4.1: high false-positive categories undermine trust in accurate categories, not just in themselves.
- **Confidence qualifiers** (“be conservative,” “only report high-confidence findings”) fail to improve precision. They are vague instructions. The exam treats them as wrong answers.
- **Mitigation sequencing**: temporarily disable high false-positive categories while improving their prompts, rather than running noisy categories while attempting repairs. Restore trust in working categories first.
- **Severity criteria** require the same explicit specification as rule criteria. Categorical mappings (“rules 3 and 5 are critical”) are auditable and consistent; contextual severity judgments are not.
- **Few-shot examples**: 2 to 4 targeted examples is the count the exam guide names for ambiguous scenarios, and no Anthropic source states a maximum; the general prompting guidance prefers three to five, so three or four sits inside both. Most valuable for tasks with ambiguous boundaries. Examples should show reasoning for ambiguous-case choices, not just input and output. Include examples of acceptable patterns vs genuine issues to teach the distinction rather than just the outcome. The guide's two named ambiguous cases are tool selection under an unclear request and branch-level coverage gaps inside functions that already have tests.
- **Few-shot owns recognition, explicit criteria owns precision.** One question separates Task Statement 4.1 from 4.2: does the output contain findings that should not be there, or is it missing a case the model never saw? Applying either remedy to the other's failure is the domain's most common engineered wrong answer.
- **Format constraints and output format as contract**: format specifications belong in the prompt with the same precision as content criteria. When review output feeds automated tooling (PR comment bots, severity-based routing), format consistency is a system requirement, not a stylistic preference; inconsistent output format is a downstream contract violation in CI contexts.
