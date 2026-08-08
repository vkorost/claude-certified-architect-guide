# Chapter 7: Slash Commands, Skills, and Plan Mode

**Summary*:** Claude Code exposes three distinct mechanisms for expressing intent: **slash commands** (quick, repeated actions run in the main conversation context), **skills** (reusable capability packages that can fork into isolated sub-agent contexts), and **plan mode** (a read-only exploration phase that produces a plan before a single file is touched). Each mechanism maps to a different point on the spectrum from “I know exactly what to do, just do it” to “I have no idea what this will break, let me think first.” The exam tests when to reach for each, how the SKILL.md frontmatter fields (context: fork, allowed-tools, argument-hint) shape skill behavior, and how to combine plan mode with direct execution for complex multi-phase tasks. This chapter also covers the three iterative refinement techniques that appear in Scenario 2: TDD iteration, the interview pattern, and concrete input/output examples.*

---

## The Intent Spectrum

Every request you send to Claude Code sits somewhere on a spectrum. At one end: a task so well-understood that the correct implementation is obvious and the scope is a single file. At the other end: a task with architectural implications, multiple valid approaches, and the kind of side effects that are impossible to enumerate in advance. Between those poles: work that is worth doing repeatedly, on different targets, in different projects, by different team members, where the procedure is known but you do not want to retype it every session.

These three regions of the spectrum correspond to three mechanisms.

Commands live at the quick-and-repeatable end. A slash command is a markdown file that becomes a shortcut. Type /review and the content of .claude/commands/review.md is injected into the prompt. The session does not fork, the context is not isolated, and the tool set is not restricted. **Commands are the right tool when the action is simple, fast, and trusted to run in the main conversation** without producing exploratory noise that you will have to scroll past for the rest of the session.

Skills live in the middle. A skill is also a markdown file, but packaged differently, and the packaging matters. With the right frontmatter, a skill runs in an isolated sub-agent context. The exploration it does, the intermediate reads and greps and wrong turns, stays inside the fork. The main session receives a summary. This is the mechanism for complex, verbose workflows where the procedure is known but the execution is messy.

Plan mode lives at the deliberate end. It is not a file on disk. It is a permissionMode value: "plan". With plan mode active, Claude explores the codebase, reads files, traces call graphs, and produces a written plan, but does not edit a single file. No changes until the plan is approved. The appropriate mechanism for tasks where committing to an approach before understanding the full implications is how you create technical debt or break things at 11 PM.

The named concept for this chapter is **explicit-intent execution modes**. Each mode requires the developer to be explicit about what kind of engagement they want: fast action, reusable workflow, or deliberate planning.

---

## Custom Slash Commands

A custom slash command is a markdown file stored in a specific directory. The filename, minus the .md extension, becomes the command name. Drop review.md into .claude/commands/ and /review becomes available in that project.<sup>[1]</sup>

There are two scopes.<sup>[1]</sup>

**Project-scoped commands** live in .claude/commands/. They are committed to version control and available to everyone on the team. Use this scope for commands that encode team standards: a code review format, a commit message template, a migration checklist.

**User-scoped commands** live in ~/.claude/commands/. These are personal, cross-project shortcuts. They do not show up in the repo, so a teammate does not see them unless they set up their own.

The content of the file defines what the command does. Commands support dynamic arguments via placeholders, can execute bash commands and embed the output, and can include file contents using the @ prefix. The SDK dispatches a slash command the same way it sends any prompt string: the command is included in the prompt text, and the system/init message lists the commands available in the current session.<sup>[2]</sup>

What commands do not do: they do not fork the context. The command executes in the main conversation. If the command triggers exploration that produces several hundred lines of output, that output stays in the main session. For commands handling bounded, trusted tasks, this is fine. For complex exploration, it is a problem that skills exist to solve.

The current recommended format for new work is .claude/skills/<name>/SKILL.md, which supports the same slash-command invocation plus autonomous invocation by the model.<sup>[2]</sup> The .claude/commands/ directory remains valid and supported; the two formats coexist in the CLI.

---

## Skills: The Reusable Capability Layer

A skill is a directory inside .claude/skills/ (project) or ~/.claude/skills/ (user) that contains a SKILL.md file.<sup>[3]</sup> The file has YAML frontmatter and markdown content. The frontmatter is where the interesting decisions happen.

The two scopes are not equal when they collide. A project skill in .claude/skills/ and a personal skill in ~/.claude/skills/ that share the same name do not merge and do not both activate; the project skill takes precedence and the personal one is shadowed. This has a direct operational consequence the exam tests. When a developer wants to customize a team skill for their own workflow, copying it to ~/.claude/skills/ under the same name does not work: the project version still wins, and the personal customization never runs. The fix is to give the personal version a different name, for example a personal /my-commit alongside the team's /commit. Both are then discoverable, and the model selects between them on description. This mirrors the precedence rule for agent definitions covered in Chapter 2, where a programmatically defined agent overrides a filesystem agent of the same name: same-name collisions resolve by scope precedence, not by merge.

### Discovery and invocation

When Claude Code starts, it reads skill metadata from the filesystem. The model sees each skill’s description and decides autonomously when to invoke it based on that description. A well-written description is a routing key: specific, keyword-rich, honest about what the skill does and under what conditions. A vague description produces unpredictable invocation.<sup>[3]</sup>

Skills can also be invoked explicitly by name in a prompt. Both paths work. Autonomous invocation is powerful for skills that should activate naturally during normal work. Explicit invocation is appropriate when you know exactly which capability you need.

In the SDK, the skills option on query() controls which skills are available. Pass "all" to enable every discovered skill, a list of names to enable only those, or [] to disable all.<sup>[3]</sup>

### The three frontmatter fields

Three frontmatter fields determine skill behavior in CLI usage. Understanding all three is exam-critical.

**context: fork**

This is the most important field. When a skill has context: fork in its frontmatter, the skill runs in an isolated sub-agent context. The main conversation does not see the intermediate steps: the greps, the reads, the exploratory dead ends.<sup>[4]</sup> The main session receives the summary output from the skill, not the full transcript of everything the skill did to produce it.

The alternative is a skill without context: fork, which runs in the main context just like a command. For a skill that reads one file and produces a compact answer, this is fine. For a skill that explores a large codebase, brainstorms multiple approaches, and discards three of them before settling on one, running in the main context pollutes the session with noise the developer has to mentally filter for the rest of the conversation.

(Do not confuse this with session forking. The context: fork frontmatter field isolates a skill's exploration in a sub-agent context within a single invocation. The --fork-session CLI flag, covered in Chapter 2, branches a whole session transcript so two investigation paths can diverge from a shared prior state. Same word, unrelated mechanisms.)

Consider a refactoring skill that needs to understand the full dependency graph before making any changes. Without context: fork, every grep result, every file read, every intermediate analysis lives in the main context. The developer ends up in a session where finding the actual changes requires scrolling through hundreds of lines of exploration. With context: fork, the skill does all of that inside the fork, and the main session sees only the final refactored output and a brief summary of what changed.<sup>[4]</sup>

**allowed-tools**

The allowed-tools frontmatter field restricts which tools the skill can use during execution.<sup>[4]</sup> A refactoring skill that only needs to read and edit files should list only Read, Edit, and Grep. Leaving tool access unrestricted when running a skill in a forked context means the skill could, in principle, execute arbitrary bash commands or write to unexpected locations. Restricting the tool set is defense-in-depth: the scope of what the skill can do is made explicit in the file that defines it.

One critical constraint: allowed-tools in SKILL.md frontmatter applies to CLI usage only. It does not apply when skills are used through the SDK.<sup>[3]</sup> In SDK usage, tool access is controlled through the main allowedTools option in the query configuration. This is an exam-tested distinction.

**argument-hint**

When a skill requires a parameter to operate correctly, argument-hint tells Claude Code what to prompt for if the skill is invoked without arguments.<sup>[4]</sup> A refactoring skill needs to know which file or directory to refactor. An argument-hint like "file or directory to refactor" surfaces this requirement at invocation time instead of letting the skill proceed with missing input and produce an error partway through.

### A concrete SKILL.md structure

Here is the frontmatter shape for a refactoring skill, drawn directly from the Domain 3 documentation:<sup>[4]</sup>

```
---
context: fork
allowed-tools:
  - Read
  - Edit
  - Grep
argument-hint: "file or directory to refactor"
---
```

The body of the file follows as markdown: the skill’s instructions, rules, and any patterns the model should follow. This content becomes the system prompt for the forked sub-agent.

---

## Skills vs Commands: The Decision

The distinction the exam tests is sharper than “use skills for complex things.” The decision criterion is whether execution produces output or exploratory context that would degrade the main session.<sup>[4]</sup>

Use a **command** when: - The action is quick and bounded. - The output is compact and directly useful in the main conversation. - No exploration phase is required. - The task is simple enough that running it in the main session does not add noise.

Use a **skill with context: fork** when: - The task involves exploration before action: reading many files, tracing dependencies, brainstorming and discarding approaches. - The execution produces verbose output that would pollute the main conversation context. - Tool access should be restricted to only what the task requires. - The same capability will be reused across sessions or by multiple team members with different arguments.

The anti-pattern in Scenario 2 is using a command for complex codebase exploration. The command runs in the main context, fills it with intermediate analysis, and leaves the developer debugging their conversation to find the actual output.<sup>[5]</sup>

---

## Skills vs CLAUDE.md: The Other Decision

Skills and CLAUDE.md both hold instructions. The distinction is load timing.

CLAUDE.md files load every session, regardless of what task is being done. They are the right location for universal standards: coding conventions, testing requirements, commit message formats, things that apply to every Claude Code session in the project.<sup>[4]</sup>

Skills load on demand, when invoked. They are the right location for task-specific workflows that are needed sometimes but not always: a refactoring procedure, a migration checklist, a security audit protocol. Putting these in CLAUDE.md means every session loads instructions for tasks that session may never perform. That is wasted context budget and, for large instruction sets, a noise problem.

The practical rule: if an instruction should govern every Claude Code interaction in the project, put it in CLAUDE.md. If it is a procedure for a specific recurring task, make it a skill. The two mechanisms compose: a skill can reference project standards from CLAUDE.md and apply task-specific logic from its own content.

---

## Plan Mode

Plan mode is the deliberate end of the intent spectrum. It corresponds to permissionMode: "plan" in the SDK. (The semantics of all permissionMode values are established in Chapter 1; this chapter addresses when and why to use "plan" specifically.)

In plan mode, Claude explores the codebase using read-only tools. It reads files, searches with Grep, traces imports, builds a mental model of the system. Then it produces a plan describing what changes it would make and why. It does not write a single file. No edits, no deletions, no shell commands that modify state.<sup>[6]</sup>

The developer reads the plan, asks questions, requests adjustments, and approves the approach before any execution begins. This is the safe exploration and design phase that prevents costly rework on complex changes.

### When to use plan mode

The exam provides concrete criteria for plan mode selection. Use plan mode for tasks with:<sup>[6]</sup>

- **Architectural implications**: restructuring a microservice’s internal boundaries, redesigning how modules communicate, changing a fundamental data flow.
- **Multi-file modifications**: changes that touch many files across the codebase, where missing a callsite or an interface definition causes runtime failures.
- **Multiple valid approaches with different infrastructure requirements**: choosing between two integration strategies, each requiring different dependencies and deployment configurations.

The rationale is that tasks matching these criteria have too many unknowns to execute directly. Direct execution commits to an approach before the full scope is understood. Plan mode surfaces the unknowns during the read-only phase, when course-correcting is free.

### When not to use plan mode

Direct execution is appropriate for well-understood changes with clear scope.<sup>[6]</sup> A single-file bug fix with a clear stack trace is the canonical example from the exam guide: the problem is identified, the location is known, the fix is obvious. Using plan mode here is overhead with no benefit. The developer waits for an exploration-and-planning phase on a task that needs no exploration.

The anti-patterns are symmetric. Always using plan mode is wasteful overhead on simple tasks. Never using it is reckless on complex ones.<sup>[5]</sup>

### The Explore subagent

For multi-phase tasks that involve a verbose discovery phase followed by implementation, the Explore subagent prevents context window exhaustion.<sup>[6]</sup> The pattern: delegate the discovery phase to an Explore subagent. The subagent reads files, traces dependencies, and builds a complete picture of the current state. The main agent receives a summary of what was found, not the full exploration transcript. The main agent’s context stays lean for the implementation phase.

This is the subagent-for-context-management pattern applied specifically to codebase exploration. The main agent never needs to see every intermediate grep result; it needs the conclusions. The Explore subagent produces the conclusions; the exploration transcript stays inside the subagent context and is never returned to the parent.<sup>[6]</sup>

### Combining plan mode and direct execution

The practical pattern for large tasks: use plan mode to investigate, then switch to direct execution to implement the approved plan.<sup>[6]</sup> Consider a library migration affecting dozens of files. Plan mode surfaces all the callsites, identifies the compatibility shims needed, and outlines the migration sequence. The developer approves the sequence. Direct execution then implements the plan file-by-file with a known target state. The investigation is thorough; the implementation is bounded.

---

## Iterative Refinement Techniques

Three refinement techniques are tested in Scenario 2 and assigned to this chapter. They address the challenge of communicating expected behavior precisely enough that Claude produces consistent, verifiable output across iterations.

### The TDD iteration pattern

Test-driven development as an iterative refinement strategy means writing the test before the implementation. The test is the specification.<sup>[7]</sup>

The cycle:

1. Write a test that specifies the desired behavior, including edge cases and error conditions. The test should fail, because the implementation does not exist yet.
2. Run the test. It fails. This confirms the test is actually checking something.
3. Ask Claude to implement the function to pass the tests.
4. Run the tests again. They should pass.
5. Refine the implementation (performance, error handling, additional edge cases) while keeping all tests green.

The power of this pattern is verification. “Implement this feature” produces an implementation; the developer then needs to evaluate whether the implementation is correct. “Make these tests pass” provides a concrete, runnable definition of correct. Each iteration has a binary outcome: passing or failing. There is no ambiguity about whether progress was made.

This dramatically outperforms vague instructions like “make it better” because the goal is measurable at each step.<sup>[5]</sup>

### The interview pattern

Before implementing a solution in an unfamiliar domain, have Claude ask questions. Specifically: have Claude surface the design considerations the developer may not have anticipated before committing to any implementation approach.<sup>[7]</sup>

Cache invalidation strategies. Failure modes under partial network failure. The behavior when a downstream service returns an unexpected status code. Race conditions in concurrent update scenarios. A developer new to a domain does not know which of these questions are load-bearing until they have already built the wrong thing.

The interview pattern makes these questions explicit before any code is written. Claude asks; the developer answers; the implementation that follows is grounded in answered questions rather than assumptions.<sup>[7]</sup>

This is most valuable in unfamiliar domains. In well-understood domains where the developer has mental models of all the edge cases, the interview phase adds overhead without proportional value. The technique is calibrated to uncertainty.

### Concrete input/output examples

When a prose description of expected behavior produces inconsistent results, the most effective technique is to provide concrete examples.<sup>[7]</sup> Not “transform the input into a normalized format,” but three pairs of input and expected output, covering the typical case, an edge case, and a failure case.

The exam guide specifies two to three examples as the effective range.<sup>[7]</sup> Too few examples leave ambiguous cases unresolved. Too many examples shift the problem from specification to enumeration.

The technique is most valuable when the transformation has boundary cases that prose descriptions handle poorly. Natural language descriptions of format normalizations, data extraction rules, and classification criteria tend to underspecify edge case behavior. Examples make the edge cases explicit in the most direct way possible: here is the input, here is the expected output.

### Interacting vs independent issues

One more refinement principle: when multiple issues exist in an implementation, whether to address them all in a single message or sequentially depends on whether the fixes interact.<sup>[7]</sup>

If fixing issue A changes the code in a way that affects the correct fix for issue B, they are interacting issues. They must be addressed together in a single, detailed message that describes both issues and the interaction. Sequential iteration, where the developer asks Claude to fix issue A and then separately asks it to fix issue B, risks producing a second fix that is correct in isolation but incorrect given the first fix.

If fixing issue A and fixing issue B are completely independent, they can be fixed sequentially. Sequential iteration on independent issues is preferable because each iteration is scoped, verifiable, and easier to review.

The practical question before sending a message about multiple issues: if Claude fixes one of these, does the correct fix for the others change? If yes, combine them. If no, sequence them.

---

## Composing the Mechanisms

Commands, skills, and plan mode compose because they occupy different layers: commands trigger action, skills package reusable capability, and plan mode gates execution behind explicit approval. The exam tests whether you can identify which layer a given requirement belongs to. CI/CD is where these mechanisms face their hardest test: no human available, session isolated, output machine-parsed. That is Chapter 8.

---

## Sample Questions

**Q1.** **A developer defines a codebase analysis procedure in .claude/commands/analyze.md. Running /analyze fills the main session with hundreds of lines of intermediate grep results and file reads. The actual analysis summary is hard to find. What is the most appropriate fix?**

A. Move the analysis instructions to CLAUDE.md so they load with lower priority. 
B. Migrate the command to a skill in .claude/skills/analyze/SKILL.md and add context: fork to the frontmatter. 
C. Add --compact to the command file to limit output length. 
D. Break the command into multiple smaller commands that each run separately.

**Correct answer: B.** The context: fork field runs the skill in an isolated sub-agent context. Intermediate exploration stays in the fork; the main session receives only the summary.<sup>[4,5]</sup>

**Q2.** A team needs to migrate a library used in 45+ files across the codebase. Multiple migration strategies are possible, each requiring different dependency changes. What is the appropriate first step?

A. Direct execution, asking Claude to migrate files one at a time. 
B. Plan mode (permissionMode: "plan"), allowing Claude to explore all callsites and produce a migration plan before any files are modified. 
C. A slash command that iterates through files in alphabetical order. 
D. The TDD iteration pattern, writing migration tests first.

**Correct answer: B.** The task has multi-file scope and multiple valid approaches with different infrastructure implications. These are the exact criteria for plan mode. Direct execution commits to an approach before the full scope is understood.<sup>[6]</sup>

**Q3.** A developer invokes a refactoring skill but does not provide the target file. The skill produces an error partway through execution. Which frontmatter field would have prevented this?

A. context: fork 
B. allowed-tools 
C. argument-hint 
D. description

**Correct answer: C.** argument-hint prompts the developer for required parameters when the skill is invoked without arguments.<sup>[4]</sup>

**Q4.** A team’s CLAUDE.md contains a 40-step security audit procedure that applies only when auditing authentication modules. The rest of the time, the procedure is irrelevant but adds to the context budget every session. What is the correct fix?

A. Move the procedure to .claude/rules/ with a paths glob pattern matching authentication modules. 
B. Create a skill in .claude/skills/security-audit/SKILL.md containing the procedure. 
C. Split the CLAUDE.md into two files and import the relevant one manually. 
D. Reduce the procedure to a summary to stay under the 200-line CLAUDE.md limit.

**Correct answer: B.** Skills load on demand. CLAUDE.md loads every session. Task-specific procedures belong in skills, not CLAUDE.md.<sup>[4]</sup>

**Q5.** A developer finds that Claude produces inconsistent output for a data normalization function. The natural language description of the normalization rule is technically accurate but produces different interpretations across attempts. Which technique is most effective?

A. The interview pattern: have Claude ask questions about the normalization requirements. 
B. Two to three concrete input/output examples demonstrating the expected transformation, including edge cases. 
C. Plan mode, to allow Claude to design the normalization approach before implementing. 
D. Increasing specificity of the prose description until all edge cases are covered.

**Correct answer: B.** Concrete input/output examples are the documented most-effective technique when prose descriptions produce inconsistent results. The exam guide specifies two to three examples as the effective range.<sup>[7]</sup>

---

## Key Takeaways

- Custom slash commands are markdown files in .claude/commands/ (project-scoped) or ~/.claude/commands/ (user-scoped). They run in the main conversation context and are best for quick, bounded, repeatable actions. Skills are directories containing SKILL.md files in .claude/skills/ (project) or ~/.claude/skills/ (user); the model invokes them autonomously based on description, or they can be invoked explicitly by name.
- The three SKILL.md frontmatter fields: context: fork isolates skill execution in a sub-agent context; allowed-tools restricts tool access during skill execution (CLI only, not SDK); argument-hint prompts for required parameters when the skill is invoked without arguments.
- Use skills with context: fork for complex exploration or analysis that produces verbose output. Use commands for simple, fast repeated actions. Skills load on demand; CLAUDE.md loads every session. Task-specific procedures belong in skills; universal project standards belong in CLAUDE.md.
- Plan mode (permissionMode: "plan") is for tasks with architectural implications, multi-file scope, or multiple valid approaches. Direct execution is for well-understood changes with clear scope. The Explore subagent handles verbose discovery phases so the main agent receives a summary, not the full exploration transcript.
- The TDD iteration pattern: write the test first (defines the goal), run it (confirms it fails), implement to pass, run again, refine while keeping tests green.
- The interview pattern: have Claude ask questions before implementing in unfamiliar domains, to surface design considerations the developer has not anticipated.
- Concrete input/output examples (two to three) are the most effective technique when prose descriptions produce inconsistent results. For multiple issues: combine interacting fixes in one message; sequence independent fixes.
