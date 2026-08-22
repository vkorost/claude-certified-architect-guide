# Chapter 7: Slash Commands, Skills, and Plan Mode

**Summary*:** Claude Code exposes three distinct mechanisms for expressing intent: **slash commands** (quick, repeated actions run in the main conversation context), **skills** (reusable capability packages that can fork into isolated sub-agent contexts), and **plan mode** (a read-only exploration phase that produces a plan before a single file is touched). Each mechanism maps to a different point on the spectrum from “I know exactly what to do, just do it” to “I have no idea what this will break, let me think first.” The exam tests when to reach for each, how the SKILL.md frontmatter fields (context: fork, allowed-tools, argument-hint) shape skill behavior, and how to combine plan mode with direct execution for complex multi-phase tasks. This chapter also covers the iterative refinement techniques named under Task Statement 3.5: TDD iteration, the interview pattern, concrete input/output examples, and the choice between batching fixes and sequencing them.*

## The Intent Spectrum

Every request you send to Claude Code sits somewhere on a spectrum. At one end: a task so well-understood that the correct implementation is obvious and the scope is a single file. At the other end: a task with architectural implications, multiple valid approaches, and the kind of side effects that are impossible to enumerate in advance. Between those poles: work that is worth doing repeatedly, on different targets, in different projects, by different team members, where the procedure is known but you do not want to retype it every session.

These three regions of the spectrum correspond to three mechanisms.

Commands live at the quick-and-repeatable end. A slash command is a markdown file that becomes a shortcut. Type /review and the content of .claude/commands/review.md is injected into the prompt. The session does not fork, the context is not isolated, and the tool set is not restricted. **Commands are the right tool when the action is simple, fast, and trusted to run in the main conversation** without producing exploratory noise that you will have to scroll past for the rest of the session.

Skills live in the middle. A skill is also a markdown file, but packaged differently, and the packaging matters. With the right frontmatter, a skill runs in an isolated sub-agent context. The exploration it does, the intermediate reads and greps and wrong turns, stays inside the fork. The main session receives a summary. This is the mechanism for complex, verbose workflows where the procedure is known but the execution is messy.

Plan mode lives at the deliberate end. It is not a file on disk. It is a permissionMode value: "plan". With plan mode active, Claude explores the codebase, reads files, traces call graphs, and produces a written plan rather than editing anything. No changes to source until the plan is approved, with one exception the Plan Mode section below sets out. The appropriate mechanism for tasks where committing to an approach before understanding the full implications is how you create technical debt or break things at 11 PM.

The named concept for this chapter is **explicit-intent execution modes**. Each mode requires the developer to be explicit about what kind of engagement they want: fast action, reusable workflow, or deliberate planning.

## Custom Slash Commands

A custom slash command is a markdown file stored in a specific directory. The filename, minus the .md extension, becomes the command name: put review.md in .claude/commands/ and the project gains /review, invoked by that file name.<sup>[1]</sup>

There are two scopes.<sup>[2]</sup>

**Project-scoped commands** live in .claude/commands/. They are committed to version control and available to everyone on the team. Use this scope for commands that encode team standards: a code review format, a commit message template, a migration checklist.

**User-scoped commands** live in ~/.claude/commands/. These are personal, cross-project shortcuts. They do not show up in the repo, so a teammate does not see them unless they set up their own.

The content of the file defines what the command does. Commands support dynamic arguments via placeholders, can execute bash commands and embed the output, and can include file contents using the @ prefix. The SDK dispatches a slash command the same way it sends any prompt string: the command is included in the prompt text, and the system/init message lists the commands available in the current session.<sup>[3]</sup>

What commands do not do: they do not fork the context. The command executes in the main conversation. If the command triggers exploration that produces several hundred lines of output, that output stays in the main session. For commands handling bounded, trusted tasks, this is fine. For complex exploration, it is a problem that skills exist to solve.

The current recommended format for new work is .claude/skills/<name>/SKILL.md, which supports the same slash-command invocation plus autonomous invocation by the model.<sup>[3]</sup> The .claude/commands/ directory remains valid and supported; the two formats coexist in the CLI.

**Version note.** The exam guide treats commands and skills as two distinct surfaces, and that is the model this chapter follows and the model the exam scores. The product has since moved further than that. The current documentation describes custom commands as having been merged into skills: a file at .claude/commands/deploy.md and a skill at .claude/skills/deploy/SKILL.md both produce /deploy and behave the same way, existing command files keep working, and where a skill and a command share a name the skill is the one that runs. What the skill form adds is a directory for supporting files, frontmatter controlling who may invoke it, and the ability for the model to load it on its own.<sup>[1]</sup> A reader who meets that behavior in a live session is not seeing something the chapter contradicts; they are seeing a later version of the same system. Answer the exam on the two-surface model, and expect the merged one at the terminal.

## Skills: The Reusable Capability Layer

A skill is a directory inside .claude/skills/ (project) or ~/.claude/skills/ (user) that contains a SKILL.md file.<sup>[4]</sup> The file has YAML frontmatter and markdown content. The frontmatter is where the interesting decisions happen. Two further locations exist and are worth knowing by name even though the exam works with the first two: an enterprise location deployed through managed settings, and a plugin location, where a plugin ships skills alongside whatever else it carries.<sup>[1]</sup>

The scopes are not equal when they collide, and the direction of the inequality is the fact most reliably carried backwards into this exam. Skills resolve by precedence: an enterprise skill wins over a personal one, and a personal skill in ~/.claude/skills/ wins over a project skill of the same name in .claude/skills/. With a deploy skill in both places, typing /deploy runs the personal one. A skill at any of those levels also displaces a bundled skill sharing its name, though not that bundled skill's aliases, so typing the alias still reaches the built-in. Plugin skills sit outside the question entirely: they carry a plugin-name:skill-name namespace and therefore cannot collide with anything.<sup>[1]</sup>

Now read that against the order settings files use. Settings resolve project over user: a value in the repository's .claude/settings.json overrides the same value in the developer's home directory. Skills resolve the other way, personal over project. Two neighboring subsystems, two opposite directions, both configured through directories that look alike and are named alike. That inversion is the whole trap, and it is the single most common defect in third-party study material for this certification, which states the skills order as though it followed the settings order. It does not. For skills, personal wins.

The operational consequence is not the one the inverted version predicts, and it is worth stating carefully because the practical advice is the same either way. The exam guide, under Task Statement 3.2, tells candidates to build personal variants in ~/.claude/skills/ under names of their own.<sup>[2]</sup> That advice is correct. The reason is not that a same-name personal copy fails to run. It runs, and it wins. A developer who copies the team's /commit skill into their home directory, edits it, and keeps the name has quietly stopped using the team's version. Nothing announces this. When the team improves the shared skill, that developer keeps invoking their own stale fork, and the only symptom is that their output drifts away from everyone else's over weeks. Give the personal variant a name of its own, a /my-commit standing beside the team's /commit, and both stay in the menu and both stay discoverable, with the model selecting between them on their descriptions.

One case does not collide at all. Skills in nested .claude/skills/ directories below the working directory coexist with a root skill of the same name: the nested one appears under a directory-qualified name, its description records which directory it applies to, and Claude picks the variant matching the files in front of it.<sup>[1]</sup> A monorepo package can therefore ship a package-specific variant of a shared skill without displacing anything.

Same-name collisions in Claude Code resolve by precedence rather than by merge: one definition is used whole, the other is shadowed. Chapter 2 makes that structural point for subagents, where a programmatically defined agent shadows a filesystem agent of the same name. What does not carry across is the direction. Each subsystem publishes its own order, and there is no general principle that the more specific scope wins.

### Discovery and invocation

When Claude Code starts, it reads skill metadata from the filesystem. The model sees each skill’s description and decides autonomously when to invoke it based on that description. A well-written description is a routing key: specific, keyword-rich, honest about what the skill does and under what conditions. A vague description produces unpredictable invocation.<sup>[4]</sup>

Skills can also be invoked explicitly by name in a prompt. Both paths work. Autonomous invocation is powerful for skills that should activate naturally during normal work. Explicit invocation is appropriate when you know exactly which capability you need.

Discovery reaches further than the two directories suggest. Project skills load from .claude/skills/ in the directory the session started in and in every parent up to the repository root, so starting in a subdirectory still picks up what the root defines. Skills nested below the starting directory are not loaded at launch; they become available the first time Claude reads or edits a file inside that subdirectory.<sup>[1]</sup> The consequence a team question turns on is a distribution one: a project skill is committed to version control and arrives with the repository, so an update reaches everyone on their next pull, while a personal skill exists on one machine and has to be redistributed by hand every time it changes.<sup>[1]</sup> A skill a team is still iterating on belongs at project scope for that reason alone.

In the SDK, the skills option on query() controls which skills are available. Pass "all" to enable every discovered skill, a list of names to enable only those, or [] to disable all.<sup>[4]</sup>

### The three frontmatter fields

Three frontmatter fields determine skill behavior in CLI usage. Understanding all three is exam-critical.

**context: fork**

This is the most important field. When a skill has context: fork in its frontmatter, the skill runs in an isolated sub-agent context. The main conversation does not see the intermediate steps: the greps, the reads, the exploratory dead ends.<sup>[2]</sup> The main session receives the summary output from the skill, not the full transcript of everything the skill did to produce it.

The alternative is a skill without context: fork, which runs in the main context just like a command. For a skill that reads one file and produces a compact answer, this is fine. For a skill that explores a large codebase, brainstorms multiple approaches, and discards three of them before settling on one, running in the main context pollutes the session with noise the developer has to mentally filter for the rest of the conversation.

The isolation runs in both directions, which is the part that surprises people. The skill's content becomes the prompt that drives the subagent, and the subagent does not receive the conversation history it was launched from.<sup>[1]</sup> A forked skill has to carry its own instructions in full.

(Do not confuse this with session forking. The context: fork frontmatter field isolates a skill's exploration in a sub-agent context within a single invocation. The --fork-session CLI flag, covered in Chapter 2, branches a whole session transcript so two investigation paths can diverge from a shared prior state. Same word, unrelated mechanisms.)

Consider a refactoring skill that needs to understand the full dependency graph before making any changes. Without context: fork, every grep result, every file read, every intermediate analysis lives in the main context. The developer ends up in a session where finding the actual changes requires scrolling through hundreds of lines of exploration. With context: fork, the skill does all of that inside the fork, and the main session sees only the final refactored output and a brief summary of what changed.<sup>[2]</sup>

**allowed-tools**

The allowed-tools frontmatter field restricts which tools the skill can use during execution.<sup>[2]</sup> A refactoring skill that only needs to read and edit files should list only Read, Edit, and Grep. Leaving tool access unrestricted when running a skill in a forked context means the skill could, in principle, execute arbitrary bash commands or write to unexpected locations. Restricting the tool set is defense-in-depth: the scope of what the skill can do is made explicit in the file that defines it.

One critical constraint: allowed-tools in SKILL.md frontmatter applies to CLI usage only. It does not apply when skills are used through the SDK.<sup>[4]</sup> In SDK usage, tool access is controlled through the main allowedTools option in the query configuration. This is an exam-tested distinction.

**argument-hint**

When a skill requires a parameter to operate correctly, argument-hint tells Claude Code what to prompt for if the skill is invoked without arguments.<sup>[2]</sup> A refactoring skill needs to know which file or directory to refactor. An argument-hint like "file or directory to refactor" surfaces this requirement at invocation time instead of letting the skill proceed with missing input and produce an error partway through.

**Version note on the last two fields.** Both allowed-tools and argument-hint have moved since the exam guide described them, and the guide's descriptions are the ones the exam scores. The current documentation describes allowed-tools as a permission grant rather than a restriction: the listed tools run during the turn that invokes the skill without prompting, the grant expires when the next message is sent, and every other tool stays callable under the session's normal permission settings. The field that removes tools from the pool while a skill is active is disallowed-tools. The same reference describes argument-hint more narrowly than the guide does, as the hint shown during autocomplete to indicate which arguments a skill expects.<sup>[1]</sup> Neither shift changes the exam-facing point, which is that the three fields exist to isolate context, bound tool surface, and surface a missing parameter at invocation time. It does change what a reader will observe at a terminal, which is why it is recorded here rather than folded silently into the text above.

### A concrete SKILL.md structure

Here is a frontmatter shape for a refactoring skill, assembled from the three fields Task Statement 3.2 names:<sup>[2]</sup>

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

## Skills vs Commands: The Decision

The distinction the exam tests is sharper than “use skills for complex things.” The decision criterion is whether execution produces output or exploratory context that would degrade the main session.<sup>[2]</sup>

Use a **command** when: - The action is quick and bounded. - The output is compact and directly useful in the main conversation. - No exploration phase is required. - The task is simple enough that running it in the main session does not add noise.

Use a **skill with context: fork** when: - The task involves exploration before action: reading many files, tracing dependencies, brainstorming and discarding approaches. - The execution produces verbose output that would pollute the main conversation context. - Tool access should be restricted to only what the task requires. - The same capability will be reused across sessions or by multiple team members with different arguments.

The failure mode worth naming, and it is this book's illustration rather than a case the guide works through, is reaching for a command when the work is codebase exploration. The command runs in the main conversation. Several hundred lines of intermediate greps and file reads land there, the answer sits somewhere in the middle of them, and every subsequent turn carries the whole pile forward. The developer ends up debugging their own transcript to find the output they asked for. Task Statement 3.2 names the remedy in the abstract: the fork exists for skills whose output is bulky, with codebase analysis given as the case, and for skills whose work is exploratory, such as weighing several approaches before settling on one.<sup>[2]</sup>

## Skills vs CLAUDE.md: The Other Decision

Skills and CLAUDE.md both hold instructions. The distinction is load timing.

CLAUDE.md files load every session, regardless of what task is being done. They are the right location for universal standards: coding conventions, testing requirements, commit message formats, things that apply to every Claude Code session in the project.<sup>[2]</sup>

Skills load on demand, when invoked. They are the right location for task-specific workflows that are needed sometimes but not always: a refactoring procedure, a migration checklist, a security audit protocol. Putting these in CLAUDE.md means every session loads instructions for tasks that session may never perform. That is wasted context budget and, for large instruction sets, a noise problem.

The practical rule: if an instruction should govern every Claude Code interaction in the project, put it in CLAUDE.md. If it is a procedure for a specific recurring task, make it a skill. The two mechanisms compose: a skill can reference project standards from CLAUDE.md and apply task-specific logic from its own content.

### The two neighbors on either side

The choice is wider than two options, and a scenario offering four of them is testing whether the candidate selects on trigger or on content. Chapter 6 lays the full set out as a table keyed on what causes content to load; that table is not repeated here. What belongs here is the pair of boundaries a skill sits between.

On one side, the path-scoped rule. A rule under .claude/rules/ with a paths glob fires when Claude reads a file matching the pattern. Nobody invokes it. It supplies conventions that hold while working in a region of the codebase and stops loading when the work moves elsewhere. A skill fires when it is invoked, by the developer typing its name or by the model matching its description, and it supplies a procedure with steps in it. The two have converged somewhat: skill frontmatter now accepts a paths field limiting automatic invocation to matching files.<sup>[1]</sup> The trigger still differs, and the trigger is what the question turns on. Conventions that apply while editing certain files are a rule. A sequence of steps somebody decides to run is a skill.

On the other side, the hook. Neither a skill nor a CLAUDE.md entry is enforcement. Both are instructions the model reads and may or may not act on. A requirement that must hold regardless of what the model decides is a hook, which is Chapter 3's territory. The distinction survives every rephrasing of the scenario: writing an instruction more emphatically, or moving it to a more specific file, changes when it loads and never changes whether it binds.

## Plan Mode

Plan mode is the deliberate end of the intent spectrum. It corresponds to permissionMode: "plan" in the SDK. (The semantics of all permissionMode values are established in Chapter 1; this chapter addresses when and why to use "plan" specifically.)

In plan mode, Claude researches and proposes instead of acting. It reads files, searches, traces imports, builds a picture of the system, and writes a plan describing what it would change and why. What it does not do is edit your source.<sup>[5]</sup>

The mechanism behind that sentence is worth stating precisely, because the loose version of it is wrong in a way the exam can reach. Plan mode is not a read-only sandbox in which an edit is impossible. File edits are never auto-approved in this mode; they route through the approval callback, and they stay blocked until the plan itself is approved.<sup>[6]</sup> Chapter 1 makes the same point from the permissionMode side. The difference between "cannot happen" and "cannot happen without a human saying so" is the difference between a guarantee and a gate, and only one of those is on offer here. There is a documented exception that confirms the shape of it: a session that already has bypass permissions available does not enforce plan mode's blocks at all. Claude is still instructed to plan rather than edit, but an edit it attempts during planning simply runs.<sup>[6]</sup>

Exploratory shell commands follow the session's own rules rather than a blanket prohibition. Where auto mode is available, a classifier reviews them instead of the developer; otherwise anything outside the built-in read-only set prompts for approval.<sup>[6]</sup> Plan mode restrains what gets written, not what gets looked at.

The developer reads the plan, asks questions, requests adjustments, and approves the approach before any execution begins. This is the safe exploration and design phase that prevents costly rework on complex changes. Approving the plan is also a mode change: it exits plan mode and moves the session into whichever permission mode the chosen approval option describes, so implementation starts immediately rather than waiting for a further instruction. Plan mode is entered by cycling modes at the keyboard, by prefixing a single prompt, or by starting the session in it from the command line, and leaving it without approving anything is a matter of cycling back out.<sup>[6]</sup>

### When to use plan mode

The exam provides concrete criteria for plan mode selection. Use plan mode for tasks with:<sup>[5]</sup>

- **Architectural implications**: restructuring a microservice’s internal boundaries, redesigning how modules communicate, changing a fundamental data flow.
- **Multi-file modifications**: changes that touch many files across the codebase, where missing a callsite or an interface definition causes runtime failures.
- **Multiple valid approaches with different infrastructure requirements**: choosing between two integration strategies, each requiring different dependencies and deployment configurations.

The rationale is that tasks matching these criteria have too many unknowns to execute directly. Direct execution commits to an approach before the full scope is understood. Plan mode surfaces the unknowns during the read-only phase, when course-correcting is free.

### When not to use plan mode

Direct execution is appropriate for well-understood changes with clear scope.<sup>[5]</sup> A single-file bug fix with a clear stack trace is the canonical example from the exam guide: the problem is identified, the location is known, the fix is obvious. Using plan mode here is overhead with no benefit. The developer waits for an exploration-and-planning phase on a task that needs no exploration.

The anti-patterns are symmetric, and the guide states both sides rather than one. Plan mode on a change that is already understood is a wait the developer pays for and gets nothing back from. Skipping it on a large-scale change with several viable approaches gives up the thing the guide says it is there for, which is exploring and designing before committing, so that rework does not have to be paid for later.<sup>[5]</sup>

### Reading the scenario, not the vocabulary

Those criteria are reliable when a scenario states them plainly, and scenarios do not always state them plainly. The recurring way candidates lose these items is selecting on surface vocabulary: a file count appears, and the count is read as though it were the criterion.

File count is not the criterion. Judgment is. A rename applied uniformly across a large set of files has a wide footprint and almost no judgment in it. The target state is known before the work starts, every occurrence gets the same treatment, and a planning phase has nothing to discover. Direct execution handles it, and handles it faster. The same set of files carrying several different kinds of incompatible change, where each occurrence has to be classified before it can be fixed and some admit more than one defensible fix, is a planning problem, and would still be one at a third the size.

Two questions settle most of these scenarios. First: is the target state known before the work begins, or does it have to be worked out? Second: does any part of the work require choosing between approaches whose consequences differ? A clear stack trace answers the first in the affirmative. The location is already established, so exploration has nothing left to find and a planning phase buys the developer a delay and no information. A task with two viable integration strategies that pull in different dependencies answers the second, and that choice is cheaper made on paper than discovered halfway through an implementation.

Importance is not one of the two questions, and this is where the trap is set. A one-line change on a payment path is high-stakes and still completely understood. Plan mode does not make it safer, because there is no unknown to surface; review and tests do. Plan mode scales with what is unknown, not with what is at risk.

### The Explore subagent

For multi-phase tasks that involve a verbose discovery phase followed by implementation, the Explore subagent prevents context window exhaustion.<sup>[5]</sup> The pattern: delegate the discovery phase to an Explore subagent. The subagent reads files, traces dependencies, and builds a complete picture of the current state. The main agent receives a summary of what was found, not the full exploration transcript. The main agent’s context stays lean for the implementation phase.

This is the subagent-for-context-management pattern applied specifically to codebase exploration. The main agent never needs to see every intermediate grep result; it needs the conclusions. The Explore subagent produces the conclusions; the exploration transcript stays inside the subagent context and is never returned to the parent.<sup>[5]</sup>

Two built-in subagents serve this shape and are close relatives. Explore is a read-only research agent, with Write and Edit denied to it, that Claude delegates to when it needs to search or understand a codebase without changing anything. Plan is the research agent plan mode delegates to when it needs to understand the codebase before presenting a plan, which keeps exploration output in a separate context window while the main conversation stays read-only. Both skip the project's CLAUDE.md files and the parent session's git status, deliberately, to keep research fast and cheap.<sup>[7]</sup> That last property is a trade and not a defect: their findings are not shaped by project conventions, because they never read them.

### Combining plan mode and direct execution

The practical pattern for large tasks: use plan mode to investigate, then switch to direct execution to implement the approved plan.<sup>[5]</sup> Consider a library migration affecting dozens of files. Plan mode surfaces all the callsites, identifies the compatibility shims needed, and outlines the migration sequence. The developer approves the sequence. Direct execution then implements the plan file-by-file with a known target state. The investigation is thorough; the implementation is bounded.

## Iterative Refinement Techniques

Task Statement 3.5 names a set of refinement techniques, and this chapter owns them.<sup>[8]</sup> They address the same challenge from different directions: communicating expected behavior precisely enough that Claude produces consistent, verifiable output across iterations.

### The TDD iteration pattern

Test-driven development as an iterative refinement strategy means writing the test before the implementation. The test is the specification.<sup>[8]</sup>

The cycle:

1. Write a test that specifies the desired behavior, including edge cases and error conditions. The test should fail, because the implementation does not exist yet.
2. Run the test. It fails. This confirms the test is actually checking something.
3. Ask Claude to implement the function to pass the tests.
4. Run the tests again. They should pass.
5. Refine the implementation (performance, error handling, additional edge cases) while keeping all tests green.

The power of this pattern is verification. “Implement this feature” produces an implementation; the developer then needs to evaluate whether the implementation is correct. “Make these tests pass” provides a concrete, runnable definition of correct. Each iteration has a binary outcome: passing or failing. There is no ambiguity about whether progress was made.

The guide frames the technique the same way: write the suite first, then iterate by handing the failures back.<sup>[8]</sup> An instruction to improve something has no completion condition, so the developer supplies one by inspection every round. A failing test supplies its own, and nobody has to adjudicate it.

### The interview pattern

Before implementing a solution in an unfamiliar domain, have Claude ask questions. Specifically: have Claude surface the design considerations the developer may not have anticipated before committing to any implementation approach.<sup>[8]</sup>

Cache invalidation strategies. Failure modes under partial network failure. The behavior when a downstream service returns an unexpected status code. Race conditions in concurrent update scenarios. A developer new to a domain does not know which of these questions are load-bearing until they have already built the wrong thing.

The interview pattern makes these questions explicit before any code is written. Claude asks; the developer answers; the implementation that follows is grounded in answered questions rather than assumptions.<sup>[8]</sup>

This is most valuable in unfamiliar domains. In well-understood domains where the developer has mental models of all the edge cases, the interview phase adds overhead without proportional value. The technique is calibrated to uncertainty.

One boundary is worth marking, because it is where this chapter's two halves get confused for each other. Plan mode and the interview pattern both sit in front of implementation and both look like diligence. They answer different questions. Plan mode answers questions about the codebase: what is there, what calls what, what breaks when this moves. The interview answers questions about the problem: what the system has to do under conditions the developer has not enumerated. A developer who knows the domain but not the codebase wants plan mode. A developer who knows the codebase but not the domain wants the interview. Reaching for plan mode in the second case is the more tempting error, because it produces something that looks like the answer: a confident, well-organized plan built on the developer's own unexamined assumptions. Exploring a codebase cannot surface a requirement that is not in it.

### Concrete input/output examples

When a prose description of expected behavior produces inconsistent results, the most effective technique is to provide concrete examples.<sup>[8]</sup> Not “transform the input into a normalized format,” but three pairs of input and expected output, covering the typical case, an edge case, and a failure case.

The exam guide specifies two to three examples as the effective range.<sup>[8]</sup> Too few examples leave ambiguous cases unresolved. Too many examples shift the problem from specification to enumeration.

The technique is most valuable when the transformation has boundary cases that prose descriptions handle poorly. Natural language descriptions of format normalizations, data extraction rules, and classification criteria tend to underspecify edge case behavior. Examples make the edge cases explicit in the most direct way possible: here is the input, here is the expected output.

### Interacting vs independent issues

One more refinement principle: when multiple issues exist in an implementation, whether to address them all in a single message or sequentially depends on whether the fixes interact.<sup>[8]</sup>

If fixing issue A changes the code in a way that affects the correct fix for issue B, they are interacting issues. They must be addressed together in a single, detailed message that describes both issues and the interaction. Sequential iteration, where the developer asks Claude to fix issue A and then separately asks it to fix issue B, risks producing a second fix that is correct in isolation but incorrect given the first fix.

If fixing issue A and fixing issue B are completely independent, they can be fixed sequentially. Sequential iteration on independent issues is preferable because each iteration is scoped, verifiable, and easier to review.

The practical question before sending a message about multiple issues: if Claude fixes one of these, does the correct fix for the others change? If yes, combine them. If no, sequence them.

### Matching the technique to the failure

None of these techniques outranks the others. Each answers a different kind of failure, and the selection is made on the shape of the failure rather than on the difficulty of the work.

- **The output varies between attempts and the written specification is already accurate.** The ambiguity is inside the prose, and more prose will not remove it, because the next description is a description too. Two or three worked pairs of input and expected output will.<sup>[8]</sup>
- **One input is handled wrongly and everything else is correct.** Supply that input and the output it should have produced, as a case.<sup>[8]</sup> Describing the defect and asking for a rewrite puts working logic back on the table for no reason, which is how a narrow bug becomes a regression.
- **The work is algorithmic, the edge cases can be listed, and performance is part of the requirement.** Write the suite first and iterate on the failures.<sup>[8]</sup> Correctness is machine-checkable here, and a human reading output is the slower instrument.
- **The developer does not yet know which questions the domain will ask.** This is the interview pattern's case and nothing else covers it.<sup>[8]</sup> Starting minimal and iterating surfaces only what testing can detect, which excludes every consideration nobody thought to test for.
- **Several defects are open at once.** Interaction decides the message count, not the defect count, as the previous section sets out.
- **The patterns the work should follow already exist in files the developer can name.** Include those files in the prompt with the @ prefix rather than asking for a search.<sup>[3]</sup> The exemplar arrives intact instead of paraphrased. Conventions meant to apply permanently are a different question and belong in CLAUDE.md.<sup>[2]</sup>

One wrong move recurs underneath most of these. When something is not right, the reflex is to discard the work and ask for it again from a longer description. Regeneration trades a defect whose location is known for a defect whose location is not, and it does that at full price every time.

## Composing the Mechanisms

Commands, skills, and plan mode compose because they occupy different layers: commands trigger action, skills package reusable capability, and plan mode gates execution behind explicit approval. The exam tests whether you can identify which layer a given requirement belongs to. CI/CD is where these mechanisms face their hardest test: no human available, session isolated, output machine-parsed. That is Chapter 8.

## Sample Questions

**Q1.** **A developer defines a codebase analysis procedure in .claude/commands/analyze.md. Running /analyze fills the main session with hundreds of lines of intermediate grep results and file reads. The actual analysis summary is hard to find. What is the most appropriate fix?**

A. Move the analysis instructions to CLAUDE.md so they load with lower priority.  
B. Migrate the command to a skill in .claude/skills/analyze/SKILL.md and add context: fork to the frontmatter.  
C. Add --compact to the command file to limit output length.  
D. Break the command into multiple smaller commands that each run separately.

**Correct answer: B.** The context: fork field runs the skill in an isolated sub-agent context. Intermediate exploration stays in the fork; the main session receives only the summary.<sup>[2,1]</sup>

**Q2.** A team needs to migrate a library used in 45+ files across the codebase. Multiple migration strategies are possible, each requiring different dependency changes. What is the appropriate first step?

A. Direct execution, asking Claude to migrate files one at a time.  
B. Plan mode (permissionMode: "plan"), allowing Claude to explore all callsites and produce a migration plan before any files are modified.  
C. A slash command that iterates through files in alphabetical order.  
D. The TDD iteration pattern, writing migration tests first.

**Correct answer: B.** The task has multi-file scope and multiple valid approaches with different infrastructure implications. These are the exact criteria for plan mode. Direct execution commits to an approach before the full scope is understood.<sup>[5]</sup> Note which half of the stem is load-bearing: the several viable strategies settle it. Had the same file count carried one mechanical change repeated throughout, direct execution would be the answer.

**Q3.** A developer invokes a refactoring skill but does not provide the target file. The skill produces an error partway through execution. Which frontmatter field would have prevented this?

A. context: fork  
B. allowed-tools  
C. argument-hint  
D. description

**Correct answer: C.** argument-hint prompts the developer for required parameters when the skill is invoked without arguments.<sup>[2]</sup> The version note earlier in this chapter records how the current product documentation describes that field; the guide's description is the one this item is scored against.

**Q4.** A team’s CLAUDE.md contains a 40-step security audit procedure that applies only when auditing authentication modules. The rest of the time, the procedure is irrelevant but adds to the context budget every session. What is the correct fix?

A. Move the procedure to .claude/rules/ with a paths glob pattern matching authentication modules.  
B. Create a skill in .claude/skills/security-audit/SKILL.md containing the procedure.  
C. Split the CLAUDE.md into two files and import the relevant one manually.  
D. Reduce the procedure to a summary to stay under the 200-line CLAUDE.md limit.

**Correct answer: B.** Skills load on demand. CLAUDE.md loads every session. Task-specific procedures belong in skills, not CLAUDE.md.<sup>[2]</sup> A deserves a second look and still fails: a path-scoped rule loads when Claude reads a matching file, and an audit is something a person decides to run, not something that follows from opening a file. The trigger is invocation, so the mechanism is a skill.

**Q5.** A developer finds that Claude produces inconsistent output for a data normalization function. The natural language description of the normalization rule is technically accurate but produces different interpretations across attempts. Which technique is most effective?

A. The interview pattern: have Claude ask questions about the normalization requirements.  
B. Two to three concrete input/output examples demonstrating the expected transformation, including edge cases.  
C. Plan mode, to allow Claude to design the normalization approach before implementing.  
D. Increasing specificity of the prose description until all edge cases are covered.

**Correct answer: B.** Concrete input/output examples are the documented most-effective technique when prose descriptions produce inconsistent results. The exam guide specifies two to three examples as the effective range.<sup>[8]</sup>

## Key Takeaways

- Custom slash commands are markdown files in .claude/commands/ (project-scoped) or ~/.claude/commands/ (user-scoped). They run in the main conversation context and are best for quick, bounded, repeatable actions. Skills are directories containing SKILL.md files in .claude/skills/ (project) or ~/.claude/skills/ (user); the model invokes them autonomously based on description, or they can be invoked explicitly by name. When the same skill name exists at more than one level, precedence runs enterprise, then personal, then project, then bundled, so the personal skill wins over the project one. That is the inverse of the settings order, where project beats user, and the inversion is the trap. A personal variant of a team skill needs a name of its own; keep the shared name and it silently displaces the team's version, and stops receiving updates to it, with nothing to signal that it happened.
- The three SKILL.md frontmatter fields: context: fork isolates skill execution in a sub-agent context; allowed-tools restricts tool access during skill execution (CLI only, not SDK); argument-hint prompts for required parameters when the skill is invoked without arguments.
- Use skills with context: fork for complex exploration or analysis that produces verbose output. Use commands for simple, fast repeated actions. Skills load on demand; CLAUDE.md loads every session. Task-specific procedures belong in skills; universal project standards belong in CLAUDE.md.
- Plan mode (permissionMode: "plan") is for tasks with architectural implications, multi-file scope, or multiple valid approaches. Direct execution is for well-understood changes with clear scope. It scales with what is unknown, not with file count and not with what is at stake: a uniform mechanical change across many files is direct execution, and a clear stack trace means the location is already found. Edits in plan mode are not impossible, they are blocked until the plan is approved and routed through the approval callback until then. The Explore subagent handles verbose discovery phases so the main agent receives a summary, not the full exploration transcript.
- The TDD iteration pattern: write the test first (defines the goal), run it (confirms it fails), implement to pass, run again, refine while keeping tests green.
- The interview pattern: have Claude ask questions before implementing in unfamiliar domains, to surface design considerations the developer has not anticipated.
- Concrete input/output examples (two to three) are the most effective technique when prose descriptions produce inconsistent results. For multiple issues: combine interacting fixes in one message; sequence independent fixes.
