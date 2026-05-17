# Chapter 9: Prompting for Precision

> **Executive Summary:** The exam tests prompt engineering as specification writing. A precise prompt eliminates ambiguity by stating explicit acceptance criteria, providing format constraints, and decomposing complex requests into enumerated steps. Vague instructions produce vague outputs; explicit categorical criteria produce consistent, auditable outputs. Two techniques anchor this chapter: the shift from vague directives to explicit categorical criteria (the named concept), and few-shot prompting as the tool for communicating reasoning about ambiguous cases. The alert-fatigue pattern shows how noisy, low-precision review tools destroy developer trust across all rule categories, not just the noisy ones. The sequencing insight is that high false-positive categories should be disabled while their prompts are repaired, rather than running them during revision; this protects the credibility of accurate categories while improvements are made. Output format is not stylistic preference when review output feeds downstream automated tooling: it is a system contract, and the format constraint belongs in the prompt with the same precision as the content criteria.

---

## The Tool That Cried Wolf

The CI pipeline runs code review on every pull request. The prompt says: "Review this code for quality issues. Be thorough and flag anything suspicious."

The reports come back. Hundreds of findings. Duplicate variable names in test files, functions that could theoretically be more efficient, missing JSDoc on internal utility functions that the team deliberately chose not to document. Developers scan the first three findings, realize they're noise, and close the tab.

Two weeks later, someone merges a PR with a hardcoded API key. The review flagged it. Nobody read far enough to see it.

This is the scenario that anchors Task Statement 4.1 in the exam guide.<sup>[1]</sup> And it is not a story about the model performing badly. The model did exactly what the prompt asked. It was thorough. It flagged suspicious things. The problem is that "thorough" and "suspicious" are not specifications. They are intentions. The model fills in the blanks with its own judgment, and its judgment does not match the team's priorities.

The fix is not to ask the model to "be more selective." That is still not a specification.

The fix is to write a specification.

## The Noise Problem Has a Name

When a code review tool flags too many non-issues, developers stop reading the reports. Including the real findings. The exam guide calls this alert fatigue, and it is directly tested.<sup>[1,2]</sup>

Alert fatigue has a compounding property that makes it worse than it first appears. It is not just that developers ignore the noisy category. High false-positive categories undermine developer trust in accurate categories.<sup>[1]</sup> If the style section of the report is reliable garbage, the instinct is to distrust the security section too. The tool's overall credibility collapses, and it stops functioning as a safety mechanism even in the areas where it actually works.

The natural first response to a noisy tool is to add a confidence qualifier: "only report high-confidence findings," or "be conservative." This feels like a fix. It is not a fix.<sup>[1]</sup> Confidence qualifiers are still vague. The model has no calibrated ground truth for what "high confidence" means relative to this codebase, this team's conventions, or this specific review context. The instruction adds words without adding precision.

The correct response is **Explicit Criteria Over Vague Instructions**: replace intentional directives with categorical, measurable conditions.

That is the named concept for this chapter. Say it again: explicit criteria over vague instructions. Not "be thorough." Not "be conservative." A list of things to flag, with conditions precise enough that a second model (or a human) could verify whether each finding is correctly categorized.

## What Explicit Looks Like

The exam guide provides the contrast directly.<sup>[1,2]</sup> Compare these two prompts for code review:

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

**Rule 1** ("Functions exceeding 50 lines") is checkable. The model either finds a function longer than 50 lines or it does not. There is no ambiguity about what counts. A human reviewer can verify any flagged finding in seconds.

**Rule 3** ("Hardcoded strings matching API key patterns") specifies the patterns explicitly. The model is not guessing what an API key looks like. It is matching against `sk-`, `pk-`, `key-`. Edge cases that used to require judgment ("is this a test key?") can be addressed by extending the pattern list, not by hoping the model's intuition holds.

**Severity** is categorical. Not "rate severity 1-10" (which the model will interpret differently on different runs), but a fixed mapping: rules 3 and 5 are critical, rules 1 and 2 are warnings, rule 4 is info. The classification is auditable. Downstream tooling can filter on severity reliably.

The output format is also part of the specification. Four required fields per finding, in a consistent structure. This is not stylistic preference. When the review runs in CI and the output feeds a PR comment bot (as in Scenario 5), the format is a contract that downstream code depends on.<sup>[3]</sup>

## The Alert Fatigue Mitigation Pattern

A development team ships a code review tool with six rule categories. Three of them work well: the security rules, the null-check rules, and the async error-handling rules. Three categories are noisy: style conventions that don't match the team's actual patterns, documentation checks that flag intentionally undocumented internal utilities, and complexity metrics calibrated for a different codebase.

The noisy categories do not stay in their lane. They erode trust in the good categories, and after a few weeks, developers have stopped reading the reports. The fix seems obvious: fix the noisy prompts.

Fixing prompts takes iteration. While the iteration is in progress, the noisy categories are still running and still burning credibility. The exam guide specifies the mitigation: temporarily disable high false-positive categories while improving prompts for those categories, rather than attempting all categories simultaneously.<sup>[1]</sup>

This is a sequencing insight, not just a prompt engineering insight. Production systems depend on trust. If the noisy categories are running, they are spending trust budget that the good categories need. Disabling them is not an admission of failure; it is protecting the signal while the noise is eliminated.

The pattern: identify which categories have high false-positive rates, disable those categories, restore developer trust in the remaining output, then bring each disabled category back after the prompt has been revised and tested.

*Do not attempt to fix all categories simultaneously.* Simultaneous repair is slower overall because the whole tool's credibility stays low during the repair period. Sequential repair, category by category, preserves credibility for the working categories throughout.

## Severity Criteria Are Specifications Too

One of the skills tested in Task Statement 4.1 is defining explicit severity criteria with concrete code examples for each severity level.<sup>[1]</sup>

Severity labels without definitions are vague instructions. "Critical," "warning," and "info" sound precise, but they are not. What makes a finding critical versus a warning? If the answer is "judgment," then severity will vary across runs and across different versions of the same model. Downstream tooling that auto-closes warnings but pages on-call for criticals will misfeed.

The specification pattern is the same as for rule criteria: replace intentional labels with conditions. Not "critical means serious security issues" but "critical: findings from rules 3 and 5 (hardcoded credentials and SQL injection vectors)." The severity is determined by which rule fired, not by a separate judgment call. A finding cannot be "warning severity" if it matched rule 3, regardless of how the model might have otherwise assessed its impact.

When severity must involve a contextual judgment, the specification should include examples. Not definitions, examples. "This is a critical finding" paired with a code snippet that exhibits the pattern. "This is a warning" paired with a different snippet. Examples communicate the decision boundary more precisely than prose descriptions because they anchor the label to a concrete case. This is the same mechanism as few-shot prompting, applied to the severity axis of the output.

## Few-Shot Prompting: When and How Many

Few-shot examples are the most effective technique for achieving consistently formatted, actionable output when detailed instructions alone produce inconsistent results.<sup>[1]</sup>

The exam guide is specific about count: 2-4 targeted examples is the documented optimal range for ambiguous scenarios.<sup>[1,2]</sup> Not 5-8, not 10. Providing too many examples (more than 6) bloats the prompt without adding value.<sup>[2]</sup> Providing too few for genuinely ambiguous cases leaves the model without enough signal to generalize.

Where few-shot examples earn their cost is in tasks with ambiguous boundaries. When the boundary between "flag this" and "skip this" requires contextual judgment that prose descriptions cannot fully specify, examples show the reasoning. Not just the outcome, the reasoning.<sup>[1]</sup>

This is a critical distinction. An example that shows input and correct output teaches the model the answer for that specific case. An example that shows input, correct output, and the reasoning for why that output was chosen ("this is a genuine issue, not a local pattern, because the same function exists in 14 other files with the same missing handler") teaches the model how to approach similar cases it has not seen.

The exam guide surfaces three specific things few-shot examples should demonstrate:<sup>[1]</sup>

**Consistent output format.** Every example should use the same output structure. If the desired output is a JSON object with four fields, every example should show a JSON object with those four fields. The model generalizes the structure from the examples; inconsistent examples produce inconsistent output.

**Acceptable patterns vs genuine issues.** A finding that looks like a problem but is actually a deliberate local convention is one of the hardest judgment calls for a code review tool. An example that shows this case explicitly, with the reasoning that identifies it as acceptable, teaches the model to distinguish the two rather than defaulting to one or the other.

**Ambiguous boundary cases.** Provide at least one example where the correct action is not obvious from the rule text alone. Show the example, show the correct classification, and show why. The model uses this to calibrate its judgment on novel patterns that fall near the same boundary.

Consider the code review context. The rule is "flag functions exceeding 50 lines." But a file has a 55-line function that is a configuration constant block, not logic. Should it be flagged? A rule that says "functions exceeding 50 lines" technically says yes. But the intent was to catch complex functions, not initialization blocks. An example that shows this case and explains the decision teaches the model to apply the rule's intent, not just its literal text.

```
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

That example does three things: shows the input, shows the correct output, and explains why this case falls outside the rule's intent even though it technically matches the rule's text. A model that has seen this example can generalize to other initialization blocks, documentation generators, and similar patterns without requiring a new rule for each.

## When Few-Shot Is Unnecessary

Few-shot examples are not universally valuable. They are expensive in tokens, and for tasks with unambiguous boundaries, they add overhead without improving output quality.<sup>[2]</sup>

The test: is the task boundary clear from the rule text alone? If the answer is yes, explicit criteria are sufficient and examples add cost without benefit. "Flag SQL queries constructed with string concatenation" is largely unambiguous. String concatenation in a SQL context is a recognizable pattern with few edge cases. A clear rule without examples will produce consistent output.

The test for few-shot: would a careful human reader, given only the rule text, arrive at the same classification for genuinely ambiguous cases? If yes: skip the examples. If no: the examples are doing real work.

Few-shot is valuable in extraction tasks too, particularly when documents have varied structures.<sup>[1]</sup> Consider a system extracting citations from research documents. Some documents use inline citations (author, year, in parentheses mid-sentence). Others use footnotes. Others use numbered bibliography entries that appear at the end. Instructions alone cannot reliably handle the variety. Examples showing correct extraction from each structure teach the model the invariant (what to extract) across varying surface presentations.

The principle is the same as the code review case: examples communicate the decision boundary in contexts where prose descriptions generate inconsistent behavior.

## Decomposition as Specification

Explicit criteria work at two levels: the criteria themselves, and the structure of the request.

A prompt that says "review this file for security issues, quality problems, and documentation gaps" asks the model to simultaneously manage three different classification tasks, each with its own criteria, while also producing output in a consistent format. This is not impossible, but it creates unnecessary competition between the sub-tasks.

The alternative: decompose the request. Task Statement 4.1's skill list includes writing specific review criteria that define which issues to report (bugs, security) versus which to skip (minor style, local patterns) rather than relying on confidence-based filtering.<sup>[1]</sup> That decomposition is already implicit in the numbered rule list format: each rule is a discrete, independently evaluable criterion.

Explicit decomposition also makes prompt iteration easier. When a category is noisy, you can revise rule 4's criteria without touching rules 1-3. When a category is missing findings it should catch, you can extend the definition of rule 3 without affecting the severity logic. The structure isolates the knobs.

This connects to how the exam frames iterative refinement for interacting versus independent issues: when fixes interact, provide them together; when they are independent, handle them sequentially.<sup>[4]</sup> The same logic applies to prompt revision. Independent criteria can be tuned independently. Criteria that share ambiguous boundary regions should be revised together.

## The Format Constraint Is Not Optional

One pattern in exam questions about prompt precision: separating the criteria from the output format.<sup>[1]</sup> It is easy to write detailed criteria and then leave the output format vague. The model will produce plausible-looking output. But "plausible-looking" is not "consistently structured," and downstream tooling requires the latter.

The output format should specify:
- What fields are present in each finding
- What values are acceptable for categorical fields (severity, rule number)
- What the finding omits (when "no issues found" is itself a valid output, specify what that looks like)

The code review example above includes all three. The finding has four fields. Severity values are enumerated. By implication, if no issues match rules 1-5, the correct output is an empty findings list, not a paragraph explaining that the code looks generally good.

Structured output via `tool_use` (covered in Chapter 10) enforces the format constraint at the schema level. For prompts that do not use `tool_use`, the format constraint belongs explicitly in the prompt, specified with the same precision as the content criteria.

## What the Exam Tests

The exam tests prompt engineering as specification writing. The scenarios it uses to frame these questions are overwhelmingly from the code review context, which is the canonical case for false positives, developer trust, and the contrast between vague and explicit.<sup>[1,3]</sup>

The patterns the exam tests are narrow and specific:

**Explicit over vague.** "Flag functions exceeding 50 lines" beats "flag long functions." "Flag comments only when claimed behavior contradicts actual code behavior" beats "check that comments are accurate."<sup>[1]</sup> Every exam question that offers a vague vs explicit option: pick explicit.

**Confidence qualifiers fail.** "Be conservative" and "only report high-confidence findings" do not improve precision compared to specific categorical criteria.<sup>[1]</sup> Every exam option that proposes confidence-based filtering as the fix for false positives: discard it.

**Disable, then fix.** When a category has high false-positive rates, disable it temporarily while revising the prompt, rather than trying to reduce false positives while the category keeps running.<sup>[1]</sup> This is the sequencing principle.

**2-4 few-shot examples.** The documented optimal range. For ambiguous boundary cases, include the reasoning. For output format, make the examples consistent. Not 5-8, not 1.<sup>[1,2]</sup>

**Examples show reasoning.** A few-shot example that shows input and output without reasoning teaches pattern matching. An example that shows input, output, and why teaches generalization.<sup>[1]</sup> The exam distinguishes these.

## Practice Questions

**Q1.** A CI pipeline runs code review with the prompt "identify any code quality issues and flag them with appropriate severity." The team reports the tool flags too many minor style issues and developers have stopped reading the reports, causing critical security findings to be missed. Which change most directly addresses the root cause?

A) Add "be conservative and only flag high-confidence issues" to the prompt.
B) Replace the vague instruction with a numbered list of specific rule categories and explicit severity mappings for each.
C) Reduce the number of files sent to the review per run.
D) Add a confidence score field to the output and filter on scores above 0.8.

*Correct: B. A and D add confidence-based filtering, which the exam guide explicitly identifies as failing to improve precision. C reduces scope but does not fix the underlying vague instruction. B replaces intentional directives with categorical specifications.*

**Q2.** A code review tool has six rule categories. Three produce accurate, trusted findings. Three generate consistent false positives. The team wants to restore developer trust in the tool overall while improving the noisy categories. What is the correct sequencing?

A) Improve all six categories simultaneously before re-enabling any of them.
B) Disable the three noisy categories, restore trust in the three accurate categories, then revise and re-enable the noisy categories one at a time.
C) Reduce the number of examples in the few-shot set for the noisy categories.
D) Add "skip minor style issues" to the prompt for all six categories.

*Correct: B. The exam guide specifies temporarily disabling high false-positive categories while improving prompts for those categories. A delays the trust restoration. C addresses few-shot quantity, not the underlying criteria. D adds a vague modifier that does not produce categorical precision.*

**Q3.** A team is implementing few-shot prompting for a code review tool that must distinguish between "genuine issues" and "local patterns the team has intentionally adopted." They are considering 4, 6, or 10 examples. The examples for the ambiguous boundary cases should include what, in addition to input and output?

A) A severity score for each example, to calibrate severity classification.
B) Reasoning that explains why the output was chosen over plausible alternatives.
C) Additional examples from adjacent rule categories to improve generalization.
D) The file path and line number of the example to improve format consistency.

*Correct: B. The exam guide specifies that examples for ambiguous cases should show reasoning for why one action was chosen over plausible alternatives. A adds severity calibration, which is a different concern. C increases the example count toward the 6+ range that bloats prompts. D addresses output format, not the ambiguous-case decision boundary.*

---

## Key Takeaways

- **Explicit criteria over vague instructions** is the named concept: categorical, measurable conditions ("flag functions exceeding 50 lines") replace intentional directives ("flag long functions" or "be thorough"). The exam tests this contrast in every prompt engineering question.

- **Alert fatigue** is the documented mechanism by which false positives destroy tool credibility. High false-positive categories undermine trust in accurate categories, not just in themselves.

- **Confidence qualifiers** ("be conservative," "only report high-confidence findings") fail to improve precision. They are vague instructions. The exam treats them as wrong answers.

- **Mitigation sequencing**: temporarily disable high false-positive categories while improving their prompts, rather than running noisy categories while attempting repairs. Restore trust in working categories first.

- **Severity criteria** require the same explicit specification as rule criteria. Categorical mappings ("rules 3 and 5 are critical") are auditable and consistent; contextual severity judgments are not.

- **Few-shot examples**: 2-4 targeted examples is the documented optimal range. Most valuable for tasks with ambiguous boundaries. Examples should show reasoning for ambiguous-case choices, not just input and output. Include examples of acceptable patterns vs genuine issues to teach the distinction rather than just the outcome.

- **Format constraints and output format as contract**: format specifications belong in the prompt with the same precision as content criteria. When review output feeds automated tooling (PR comment bots, severity-based routing), format consistency is a system requirement, not a stylistic preference; inconsistent output format is a downstream contract violation in CI contexts.

---

