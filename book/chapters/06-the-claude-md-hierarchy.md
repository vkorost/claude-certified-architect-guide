# Chapter 6: The CLAUDE.md Hierarchy

**Summary:** *Claude Code’s configuration system is a four-scope hierarchy (managed policy, user, project, local) that resolves through concatenation, not override. Broader scopes are read first and more specific ones last, but nothing is erased and nothing takes precedence. The .claude/rules/ directory and @path/to/import syntax provide modular organization within that hierarchy. Path-scoped rules extend the system by triggering on file type rather than directory, giving teams fine-grained control without fragmented configs. The exam’s most reliable trap is instructions living in the wrong scope: user-level configuration is personal and invisible to teammates.*

---

## The Diagnostic Question

Picture it: a new engineer joins the team. They fire up Claude Code on their first real task. Claude generates code that ignores the team’s established patterns. Wrong naming conventions. Wrong error handling style. 

The team lead checks in. Does the project CLAUDE.md exist? Yes. Is it committed? Yes. Did the new engineer clone the repo? Yes.

*The file is there. Claude just isn’t following it.*

That scene is this book’s own illustration, not a transcript of anything the exam ships. The failure it describes, though, is named directly in the objectives: diagnosing a new team member who is not receiving instructions because those instructions sit in user-level rather than project-level configuration.<sup>[1]</sup> The exam’s Claude Code scenario, which frames a team using the tool for generation, refactoring, debugging and documentation, lists CLAUDE.md configuration among the things that team has to get right.<sup>[2]</sup>

Because the answer is not “your CLAUDE.md is broken.” The answer is: “those instructions were in ~/.claude/CLAUDE.md, the user-level file on the team lead’s machine, and the new engineer’s machine has no such file.”

That gap, that single config placement decision, is the named concept this chapter introduces: **Three-layer config inheritance**.

The name is slightly imprecise, and the imprecision is worth pinning down. The exam guide’s three layers are user-level, project-level, and directory-level, the last being CLAUDE.md files sitting in subdirectories of the project.<sup>[1]</sup> The documentation instead lists four locations a CLAUDE.md file can occupy, adding managed policy above them and local beneath, and treats subdirectory files as a loading behavior rather than a scope.<sup>[3]</sup> This chapter uses the documentation’s four and handles subdirectory files under loading, which is where their distinctive behavior lives.

The point is that these layers do not override each other. They concatenate. Instructions from a broader scope do not disappear when a narrower scope adds instructions. Everything loads. Everything compounds.

Knowing what each layer carries, and what makes it load, is an exam competency.

---

## The Four Scopes

Claude Code recognizes four locations where CLAUDE.md files can live, each with a different reach.<sup>[3]</sup>

**Managed policy** lives at an OS-specific path that your IT or DevOps team controls:

- *macOS: /Library/Application Support/ClaudeCode/CLAUDE.md*
- *Linux and WSL: /etc/claude-code/CLAUDE.md*
- *Windows: C:\Program Files\ClaudeCode\CLAUDE.md*

This file cannot be excluded by individual users or per-project settings. It is the organization’s floor. Whatever is in it applies to every session on every machine where it is deployed. It is delivered by configuration management tooling (MDM, Group Policy, Ansible). Individual developers cannot exclude it, and cannot opt out of it.<sup>[3]</sup>

**Project** scope is ./CLAUDE.md or ./.claude/CLAUDE.md at the root of your repository. This is the team config. It travels with the repository through version control. Every developer who clones the repo gets it. Every CI job that checks out the repo has access to it. This is where team coding standards, architectural conventions, build commands, and shared workflows belong.<sup>[3]</sup>

**User** scope is ~/.claude/CLAUDE.md. Personal preferences. All projects, all sessions, only your machine. This is the right place for things like editor keybindings, output format preferences, and personal tooling shortcuts. It is the wrong place for anything the team needs to share.<sup>[3]</sup>

**Local** scope is ./CLAUDE.local.md at the project root. Gitignored by convention, which means adding it to .gitignore explicitly. Running /init with the interactive flow enabled, which is to say with CLAUDE_CODE_NEW_INIT=1 set, and choosing the personal option does that for you. This is personal and project-specific: sandbox URLs, specific test data you use during development, credentials for a local environment. It does not travel with the repo. It is invisible to teammates and CI.<sup>[3]</sup>

The table structure is, listed in load order from broadest scope to most specific:<sup>[3]</sup>

| Scope | File location | Shared with | In version control |
| --- | --- | --- | --- |
| Managed policy | OS-specific system path | All users on the machine | No (deployed via MDM/etc.) |
| User | ~/.claude/CLAUDE.md | Nobody | No |
| Project | ./CLAUDE.md or ./.claude/CLAUDE.md | Team | Yes |
| Local | ./CLAUDE.local.md | Nobody | No (gitignored) |

The exam tests whether you can correctly assign instruction types to scopes. Personal preferences do not belong in project config. Team standards do not belong in user config. The new engineer scenario is the canonical illustration: team standards in the wrong scope means teammates start fresh, without context.

---

## How the Files Load

Understanding what “concatenation” means in practice requires knowing the load order.<sup>[3]</sup>

Two orderings are at work. Across scopes, content is assembled broadest to most specific: managed policy, then user, then project, then local, so a project instruction appears in context after a user instruction.<sup>[3]</sup> Within the project, a second ordering applies across the directory tree.

When Claude Code starts, it walks the directory tree from your current working directory upward. At each level, it checks for CLAUDE.md and CLAUDE.local.md. All discovered files are loaded. Content is ordered from the filesystem root down to the working directory: files at the top of the tree appear first in the assembled context; files closer to where you launched Claude appear last.

Consider a session launched from *projects/myapp/src/:*

1. CLAUDE.md at / (if it exists)
2. CLAUDE.md at /projects/
3. CLAUDE.md at /projects/myapp/
4. CLAUDE.md at /projects/myapp/src/
5. CLAUDE.local.md files appended after CLAUDE.md at each respective level

Order matters.

Within any given directory, CLAUDE.local.md is appended after CLAUDE.md. So **personal notes are always the last thing Claude reads at that level.**<sup>[3]</sup>

One clarification on what “more specific wins” means in a concatenation model. It does not mean anything, because in this system nothing wins. Nothing is erased and nothing takes precedence. Concatenation places more-specific-scope content later in the assembled context, and that is the entire mechanism. Position is a reading order, not a conflict-resolution rule.

This matters because the obvious inference is wrong. **When two CLAUDE.md files give different guidance for the same behavior, Claude may pick one arbitrarily.**<sup>[3]</sup> Not the later one. Not the more specific one. Arbitrarily. The documented remedy is not to arrange your files so the instruction you prefer lands last; it is to go and find the contradiction and delete one side of it. Reviewing CLAUDE.md files, nested ones in subdirectories, and .claude/rules/ for stale or conflicting instructions is maintenance the system expects, because it will not adjudicate on your behalf.<sup>[3]</sup> For deterministic enforcement, hooks exist (that’s Chapter 3’s territory).

It is worth saying the negative form out loud, because it is where candidates lose points: **CLAUDE.md has no precedence order.** Three neighboring mechanisms do have one. Settings files resolve by precedence, skill name collisions resolve by precedence, and MCP servers defined in more than one scope resolve by precedence with the winning definition used whole. Those orders are covered where each mechanism is treated, and none of them transfers here. A reader who has memorized them arrives at a CLAUDE.md question already asking which layer wins, and the question has no such answer. The files concatenate. Both are present, and neither cancels the other.

**Subdirectory files load differently.** Claude Code discovers CLAUDE.md and CLAUDE.local.md files in subdirectories under the working directory, but it does not load them at launch. They are included when Claude reads files in those subdirectories during a session.<sup>[3]</sup> This is relevant behavior to know for the exam: *a CLAUDE.md in src/api/ is not in context when the session starts; it enters context when Claude opens a file from that directory.*

An important corollary: after /compact, the project-root CLAUDE.md is re-read from disk and re-injected into the session. Nested subdirectory CLAUDE.md files are not re-injected automatically. They reload the next time Claude reads a file in that subdirectory.<sup>[3]</sup> Path-scoped rules behave the same way and for the same reason: they are not re-injected either, and reload the next time Claude reads a file matching their patterns.<sup>[3]</sup> Instructions you gave only in conversation do not survive compaction. Add them to CLAUDE.md if they need to persist.

---

## The @import Syntax

CLAUDE.md files can pull in other files using @path/to/import syntax anywhere in the file.<sup>[3]</sup>

Both relative and absolute paths are allowed. Relative paths resolve relative to the file containing the import, not the working directory. Imported files are expanded and loaded at launch alongside the CLAUDE.md that references them. Recursive imports are supported: an imported file can import further files, to a maximum depth of four hops.<sup>[3]</sup>

The practical use case is modular configuration. A monorepo with packages serving different domains can have a project-level CLAUDE.md that imports only the standards relevant to each package, rather than loading every rule in every context:

```
# .claude/CLAUDE.md
Use TypeScript with strict mode enabled.
@./rules/api-design.md
@./rules/testing.md
```

One thing the exam will test: imports do not reduce context. Splitting a 400-line CLAUDE.md into four 100-line files and importing all four produces the same total context as the monolith did. The organization benefit is real (teams can own individual files; review is easier; diffs are cleaner), but the context budget does not shrink.<sup>[3]</sup> For reducing context, path-scoped rules are the correct mechanism.

Importing also works for team-shared references. A team can maintain a standards/ directory with agreed-upon conventions and reference those files from CLAUDE.md with @ syntax. Works with AGENTS.md too: if a repository already uses AGENTS.md for another coding assistant, a CLAUDE.md that starts with @AGENTS.md lets both tools read the same instructions without duplication.<sup>[3]</sup>

---

## The .claude/rules/ Directory

The ***.claude/rules/*** directory provides an *organized alternative* to a monolithic CLAUDE.md.<sup>[3]</sup>

Place markdown files in .claude/rules/. Each file should cover one topic, with a descriptive filename: testing.md, api-design.md, deployment.md. All .md files are discovered recursively, so rules can be organized into subdirectories like frontend/ or backend/. The rules directory supports symlinks, including links to shared rule sets maintained outside the repository.<sup>[3]</sup>

There are two kinds of rules, distinguished by YAML frontmatter.

**Rules without paths frontmatter** load at launch, with the same priority as .claude/CLAUDE.md. They are always in context. Use them for conventions that apply to all work in the project.

**Rules with paths frontmatter** are path-scoped. They load only when Claude reads files matching the specified glob patterns. They do not consume context tokens unless they are relevant to the current task.<sup>[3]</sup>

```typescript
---
paths:
  - "**/*.test.tsx"
---
# Testing conventions for React components

Every test file must have a corresponding factory function for test fixtures.
```

This rule enters context when Claude opens any *.test.tsx* file, anywhere in the project. It does not enter context when Claude is working on infrastructure configuration files or backend services.

### Path-Scoped Rules vs. Subdirectory CLAUDE.md

The exam tests when to use path-scoped rules versus a subdirectory CLAUDE.md, and the distinction is meaningful. The objectives name the choice explicitly, in both directions: the advantage of glob-pattern rules over directory-level CLAUDE.md files for conventions that span multiple directories, and the skill of picking path-specific rules when conventions have to apply to files spread across the codebase.<sup>[1]</sup>

A subdirectory CLAUDE.md at src/api/CLAUDE.md covers everything Claude reads in that directory. A path-scoped rule with **/*.ts covers all TypeScript files regardless of which directory they are in.

If your conventions are directory-bound (this specific API module has unique auth requirements), a subdirectory CLAUDE.md is correct. If your conventions are type-bound (all TypeScript files must follow these patterns, regardless of whether they live in src/, lib/, test/, or scripts/), a path-scoped rule in .claude/rules/ is correct.

The case that tests this: a team wants to enforce TypeScript conventions on test files that are distributed throughout a monorepo. A subdirectory CLAUDE.md would require one file per directory containing tests. A path-scoped rule with **/*.test.tsx covers the entire codebase with one file, and the objectives use that exact pattern as their worked example of applying conventions by file type regardless of directory location.<sup>[1]</sup>

---

## What Makes Guidance Apply

Everything so far has been about where a file lives. The harder question, and the one Domain 3 keeps returning to, is what makes its content show up at all. There are four triggers, and each selects a different mechanism. Getting the trigger right is most of the work; after that the mechanism is forced.

| Trigger | Mechanism | Cost |
| --- | --- | --- |
| Always, every session | CLAUDE.md at any scope, plus rules with no paths frontmatter, plus anything reached by @import | Context, every session, whether relevant or not |
| Only at a matching path | .claude/rules/ with a paths glob | Context only when a matching file is read |
| Only during a particular task | A skill | Context only when invoked or judged relevant |
| Regardless of what Claude decides | A hook | No model discretion involved |

The first row is larger than people expect. A root CLAUDE.md section, a user CLAUDE.md, a CLAUDE.local.md, a rule with no frontmatter, and any file pulled in by @import all land in the same bucket: loaded at launch, present for the whole session, paid for whether or not the session ever touches the subject. Splitting a monolith into imports moves content between files in that bucket, never out of it.

The second row is the only one that reduces what loads. A rule with a paths field enters context when Claude reads a matching file and not before.<sup>[3]</sup> It is the only such mechanism among the memory surfaces, which is why questions about token pressure in a large repository converge on it.

The third row belongs to skills, which Chapter 7 owns. The boundary is worth stating here because it is a boundary and not a preference: always-on universal standards belong in CLAUDE.md, and a task-specific workflow invoked on demand belongs in a skill.<sup>[2]</sup> The documentation draws the same line from the memory side: an entry that is a multi-step procedure, or that matters only in one part of the codebase, is a skill or a path-scoped rule rather than a CLAUDE.md entry.<sup>[3]</sup>

The fourth row leaves the memory system entirely. CLAUDE.md and rules are context, not enforced configuration, and to block an action regardless of what Claude decides you need a PreToolUse hook.<sup>[3]</sup> No amount of emphasis in a markdown file converts guidance into a guarantee.

### The wrong fixes, named first

Each row of that table has a plausible neighbor that has cost people points. The shape is the same every time: the wrong answer reorganizes content, and the right answer changes what triggers it.

*A monolithic CLAUDE.md loads rules most sessions never need.* The reflex is to break it into labeled sections, or into several files pulled back in by @import. Both improve maintainability and neither touches the trigger; every line still loads at launch. Path scoping under .claude/rules/ is the move that stops irrelevant content from loading at all.

*A convention governs test files scattered through the tree.* The reflex is one CLAUDE.md per directory holding tests, which means writing the convention many times over and maintaining every copy, and which still misses any test file in a directory nobody remembered. An @import of a testing standards file is the other reflex, and it loads unconditionally. One glob covers the lot.

*Packages in a monorepo need different subsets of a shared standards library.* Here the temptation runs the other way, toward the mechanism that just won twice. A rule matches on where a file sits, not on which package owns it, so a paths field enumerating every package directory is a rule working against its own design. Selection by package belongs in that package’s CLAUDE.md, importing only the standards it needs. One shared-standards.md imported everywhere is the same mistake in a tidier coat.

*An instruction is followed inconsistently across sessions, with no conflict in sight.* The reflex is to restate it more forcefully, or pile on examples. That is content work applied to what may well be a loading failure. Establish loading first. Effort spent on a file that never loaded is spent twice.

That last case is a diagnostic problem rather than a design problem, and it has its own command, which this chapter reaches shortly.

---

## Size Guidance and What It Tells You

The documented target is **under 200 lines per CLAUDE.md file**.<sup>[3]</sup>

Longer files consume more context and reduce adherence. The adherence point is not just about tokens: CLAUDE.md content is delivered to Claude as a user message after the system prompt, not as part of the system prompt itself. Claude reads it and tries to follow it, but there is no guarantee of strict compliance, especially for vague or conflicting instructions.<sup>[3]</sup>

This is the architectural reality of the system. It is context, not enforced configuration. The more specific and concise the instructions, the more consistently Claude follows them. “Use 2-space indentation” works better than “format code properly.” “Run npm test before committing” works better than “test your changes.” The measurable, verifiable instruction has a better compliance rate than the directive that requires judgment to apply.

Block-level HTML comments in CLAUDE.md files (<!-- maintainer notes -->) are stripped before the content is injected into Claude’s context. Use them to leave notes for human maintainers without spending context tokens on those notes. Comments inside code blocks are preserved.<sup>[3]</sup>

For instructions where compliance is genuinely required, the right mechanism is hooks, not CLAUDE.md. Hooks execute as shell commands at fixed lifecycle events and apply regardless of what Claude decides to do. Hooks are Chapter 3’s territory, but the handoff point is important: CLAUDE.md is behavioral guidance; hooks are programmatic enforcement.

---

## User-Level Rules

The user scope applies to more than just ~/.claude/CLAUDE.md. A ~/.claude/rules/ directory works the same way as the project-level .claude/rules/ directory, but applies to every project on your machine.<sup>[3]</sup> Use it for personal preferences that are not project-specific: output verbosity, preferred code review style, debugging approaches.

User-level rules follow the same two-tier behavior: rules without paths frontmatter load universally; rules with paths frontmatter are path-scoped. They are also loaded before project rules, which is the same broad-to-specific ordering the CLAUDE.md scopes use.<sup>[3]</sup>

The sharing constraint applies here identically to user CLAUDE.md: personal rules in ~/.claude/rules/ are yours. Teammates do not see them.

---

## The /memory Command

When instructions are not being followed, the first diagnostic step named by the objectives is /memory, used to verify which memory files are loaded and to diagnose inconsistent behavior across sessions.<sup>[1]</sup>

The command lists your CLAUDE.md, CLAUDE.local.md and other memory file locations across user and project scope, including entries for user and project files that do not exist yet. It lets you toggle auto memory on or off and offers a way to open the auto memory folder. Selecting any file opens it in your editor, and selecting one that does not exist yet creates it first.<sup>[3]</sup>

A version note, because this surface has moved since the objectives were written. The exam’s model is that /memory answers the loading question, and that is the answer to give. Current documentation splits the job: /memory shows where memory files live and lets you edit them, while /context reports which files actually loaded into this session, under a memory files heading. If a file is missing from that list, Claude cannot see it.<sup>[3]</sup> The diagnostic logic is untouched. Only the command that reports loading has changed name.

Common diagnostic flow: a team member reports that Claude is not following the project style guide. Check what loaded. If the project CLAUDE.md is not there, either the file does not exist at a location that gets loaded for that session, or a claudeMdExcludes configuration is blocking it. If the file is there, check whether the instruction is specific enough. “Format code properly” will not produce consistent results; “use 4-space indentation for Python files” will.<sup>[3]</sup>

The *claudeMdExcludes* setting is relevant in large monorepos: ancestor CLAUDE.md files may contain instructions that are irrelevant to your current work. The *claudeMdExcludes* setting (available at any settings layer*: user, project, local, or managed policy*) lets you skip specific files by path or glob pattern, matched against absolute paths. Arrays from different layers merge rather than replace one another. Managed policy CLAUDE.md files cannot be excluded.<sup>[3]</sup>

---

## The Key Trap

Return to the opening scene: the new engineer whose Claude session ignores team standards.

The diagnostic is now clear. Check the loaded memory files on the engineer’s machine. If the project CLAUDE.md does not appear, start there. If it does appear, check whether the instruction that is being ignored lives in a subdirectory CLAUDE.md that has not been triggered yet (the engineer has not opened files from that directory), or in a path-scoped rule that has not matched a file yet.

But the most common cause, and the one the objectives name in as many words, is this: the instruction that was supposed to apply to the team was written in ~/.claude/CLAUDE.md on the team lead’s machine.<sup>[1]</sup>

The team lead sees correct behavior. Every other team member sees none. The file is not committed. It is not in version control. It never left the team lead’s home directory.

This is the canonical user-level-versus-project-level mistake, and it has a shape that is easy to miss because the author of the instruction experiences no symptoms. Their Claude session follows the rule. They test it. It works. They move on.

The fix is mechanical: move the instruction from ~/.claude/CLAUDE.md to .claude/CLAUDE.md in the repository. Commit it. Done.

Everything the team needs to share goes in the project scope. Everything personal stays in user scope or local scope. That is the complete decision rule.

---

## Putting It Together: A Config Anatomy

Here is the structure of a well-organized project configuration:

```
~/.claude/
  CLAUDE.md               # Personal universal prefs: vim keybindings, verbosity
  rules/
    output-format.md      # Personal rule: always summarize findings

project/
  CLAUDE.md               # Team entry point: brief, references @imports
  CLAUDE.local.md         # Gitignored: sandbox URL, local test creds
  .claude/
    CLAUDE.md             # Team config: build commands, standards, @imports
    rules/
      testing.md          # Topic-specific, no paths frontmatter: loads always
      api-design.md       # Topic-specific, no paths frontmatter: loads always
      react-components.md # paths: ["**/*.tsx"]: loads only for .tsx files
      terraform.md        # paths: ["terraform/**/*"]: loads only for tf files
    commands/
      review.md           # /review slash command (Ch7 territory)
    skills/
      refactor/
        SKILL.md          # Refactoring skill with context: fork (Ch7 territory)

  src/
    api/
      CLAUDE.md           # Directory-level: auth validation rules for this module
```

The CLAUDE.md at project root (or .claude/CLAUDE.md) is short. It covers broad team conventions and uses @import to pull in topic-specific files. The .claude/rules/ directory carries the detail. Path-scoped rules handle file-type-specific conventions. CLAUDE.local.md handles personal project-specific context. User-level files handle personal universal preferences.

Nobody wrote 800 lines in one file. Nobody put personal preferences in the committed config. Nobody wrote team standards in their home directory.

---

## Key Takeaways

**Four scopes, one direction.** Managed policy loads unconditionally for all users. User is personal and applies to all projects. Project loads from version control and applies to the team. Local is personal, gitignored, and project-specific. *More-specific scope content is read later in the assembled context.*

**Concatenation, not override.** All discovered CLAUDE.md files combine into a single assembled context. Nothing is erased and nothing takes precedence: CLAUDE.md has no precedence order, unlike settings, skills, and MCP scopes. When two files conflict, Claude may pick either one arbitrarily, so the remedy is to remove the contradiction rather than to position it. For guaranteed compliance, use hooks.

**Load order and @import behavior.** Scopes assemble broadest to narrowest: managed policy, user, project, local. Within the tree, files above the working directory load at launch; subdirectory files load when Claude reads files from those directories. @import adds context without reducing it, resolves relative to the importing file, and recurses to a maximum of four hops. For reducing context, use path-scoped rules in .claude/rules/.

**The trigger determines the mechanism.** Always-on means CLAUDE.md, a rule with no frontmatter, or an @import. At a matching path means .claude/rules/ with a paths glob, the only memory surface that reduces what loads. During a specific task means a skill. Regardless of what Claude decides means a hook. A rule with paths: ["**/*.test.tsx"] reaches matching files anywhere in the tree, where a subdirectory CLAUDE.md reaches everything in one directory and cannot span directories. A rule matches on where a file sits, though, not on which package owns it, so per-package standards belong in that package’s own CLAUDE.md via @import.

**Size guidance exists for a reason.** Target under 200 lines per file. CLAUDE.md is delivered as a user message, not enforced configuration. Longer files with vague instructions produce less reliable compliance than shorter files with specific, verifiable ones.

**/memory is the diagnostic command.** If an instruction is not being followed, establish that the file loaded before editing what it says. If the file did not load, Claude cannot see it. If it did, the instruction is too vague or conflicts with another. Current documentation moves the did-it-load report to /context, while /memory locates and opens the files.

**The exam’s primary trap.** Team instructions in user-level config (~/.claude/CLAUDE.md) are invisible to teammates. Project instructions belong in .claude/CLAUDE.md, committed to version control. This is the most commonly tested configuration mistake in Domain 3.

---

## Sample Questions

**Q1.** **A team lead configured Claude Code to follow company-specific error handling conventions. After onboarding three new engineers, none of them observe Claude following these conventions. The team lead’s environment works correctly. Which configuration change resolves this?**

A. Move the error handling instructions from ~/.claude/CLAUDE.md to .claude/CLAUDE.md in the repository and commit the file.  
B. Ask each engineer to copy the team lead’s ~/.claude/CLAUDE.md to their local machine.  
C. Add the instructions to CLAUDE.local.md at the project root and commit that file.  
D. Configure the managed policy file to load user-level settings from a shared network location.

***Correct: A.** User-level config is personal and not shared via version control. Project-level config in .claude/CLAUDE.md is shared through the repository. CLAUDE.local.md is gitignored (C is wrong). Manually syncing user config (B) is not a durable team solution and does not address the root cause.*

**Q2.** **A monorepo contains TypeScript test files distributed across twelve subdirectories. The team wants Claude Code to apply consistent testing conventions whenever it works with any test file, regardless of directory. Which approach is most appropriate?**

A. Create a .claude/rules/testing.md file with paths: ["**/*.test.ts"] frontmatter.  
B. Create a CLAUDE.md in each of the twelve subdirectories containing test files.  
C. Add the testing conventions to the root CLAUDE.md using @import syntax.  
D. Create a .claude/rules/testing.md file without any frontmatter.

***Correct: A.** Path-scoped rules with glob patterns apply by file type regardless of directory. B requires twelve files and maintenance. C loads the testing conventions for all sessions regardless of whether tests are being worked on (correct for always-applicable rules, but wasteful here). D loads at launch unconditionally, not path-scoped.*

**Q3.** **A developer imports three large reference documents into CLAUDE.md using @ syntax to keep the file itself under 200 lines. What effect does this have on the session context?**

A. The imported files are lazy-loaded only when Claude references them during the session.  
B. The imported files load at launch alongside CLAUDE.md, increasing total context proportionally.  
C. The imported files replace the content of CLAUDE.md in context, keeping total size the same.  
D. The imported files are summarized before loading to reduce context impact.

***Correct: B.** Imported files are expanded and loaded at launch. The 200-line guidance applies per file, but importing multiple files still loads all of them. @import aids organization, not context reduction.*

**Q4.** **After a */compact* operation, a developer notices that Claude no longer follows conventions that were specified in src/api/CLAUDE.md. The project-root CLAUDE.md instructions are still being followed. What is the most likely explanation?**

A. Compact operations delete CLAUDE.md files from subdirectories.  
B. The project-root CLAUDE.md survives compaction and is re-injected; nested subdirectory CLAUDE.md files are not re-injected automatically and reload only when Claude next reads files from those subdirectories.  
C. The src/api/CLAUDE.md instructions conflict with the project-root CLAUDE.md and were suppressed.  
D. Compact operations only preserve the managed policy CLAUDE.md.

***Correct: B**. After /compact, the project-root CLAUDE.md is re-read from disk and re-injected. Nested subdirectory files are not re-injected; they reload the next time Claude reads a file from that directory. Path-scoped rules behave identically and reload on the next matching file. C describes a suppression that this system does not perform: CLAUDE.md files concatenate and no scope suppresses another, so a conflict would produce an arbitrary choice rather than a silent removal.*
