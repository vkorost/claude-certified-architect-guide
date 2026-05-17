# Chapter 6: The CLAUDE.md Hierarchy

> **Executive Summary:** Claude Code's configuration system is a four-scope hierarchy (managed policy, project, user, local) that resolves through concatenation, not override. Instructions from more specific scopes are read last and carry more immediate weight, but nothing gets erased. The `.claude/rules/` directory and `@path/to/import` syntax provide modular organization within that hierarchy. Path-scoped rules extend the system by triggering on file type rather than directory, giving teams fine-grained control without fragmented configs. The exam's most reliable trap is instructions living in the wrong scope: user-level configuration is personal and invisible to teammates.

---

## The Diagnostic Question

Picture it: a new engineer joins the team. They fire up Claude Code on their first real task. Claude generates code that ignores the team's established patterns. Wrong naming conventions. Wrong error handling style. Wrong.

The team lead checks in. Does the project `CLAUDE.md` exist? Yes. Is it committed? Yes. Did the new engineer clone the repo? Yes.

*The file is there. Claude just isn't following it.*

This is Scenario 2 from the certified architect exam, and it is one of the more instructive traps the exam sets.<sup>[1]</sup> Because the answer is not "your CLAUDE.md is broken." The answer is: "those instructions were in `~/.claude/CLAUDE.md`, the user-level file on the team lead's machine, and the new engineer's machine has no such file."

That gap, that single config placement decision, is the named concept this chapter introduces: **Three-layer config inheritance**.

The name is slightly imprecise because there are actually four scopes. But the exam tests three of them routinely, and the fourth (managed policy) is mostly an enterprise concern. The point is that these layers do not override each other. They concatenate. Instructions from a broader scope do not disappear when a narrower scope adds instructions. Everything loads. Everything compounds.

Knowing which layer wins, and when, is an exam competency.

---

## The Four Scopes

Claude Code recognizes four locations where `CLAUDE.md` files can live, each with a different reach.<sup>[2]</sup>

**Managed policy** lives at an OS-specific path that your IT or DevOps team controls:

- macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`
- Linux and WSL: `/etc/claude-code/CLAUDE.md`
- Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`

This file cannot be excluded by individual users or per-project settings. It is the organization's floor. Whatever is in it applies to every session on every machine where it is deployed. It is delivered by configuration management tooling (MDM, Group Policy, Ansible). Individual developers cannot override it, and cannot opt out of it.<sup>[2]</sup>

**Project** scope is `./CLAUDE.md` or `./.claude/CLAUDE.md` at the root of your repository. This is the team config. It travels with the repository through version control. Every developer who clones the repo gets it. Every CI job that checks out the repo has access to it. This is where team coding standards, architectural conventions, build commands, and shared workflows belong.<sup>[2]</sup>

**User** scope is `~/.claude/CLAUDE.md`. Personal preferences. All projects, all sessions, only your machine. This is the right place for things like editor keybindings, output format preferences, and personal tooling shortcuts. It is the wrong place for anything the team needs to share.<sup>[2]</sup>

**Local** scope is `./CLAUDE.local.md` at the project root. Gitignored by convention (add it to `.gitignore` explicitly; running `/init` and selecting the personal option does this automatically). This is personal and project-specific: sandbox URLs, specific test data you use during development, credentials for a local environment. It does not travel with the repo. It is invisible to teammates and CI.<sup>[2]</sup>

The table structure is:

| Scope | File location | Shared with | In version control |
|---|---|---|---|
| Managed policy | OS-specific system path | All users on the machine | No (deployed via MDM/etc.) |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team | Yes |
| User | `~/.claude/CLAUDE.md` | Nobody | No |
| Local | `./CLAUDE.local.md` | Nobody | No (gitignored) |

The exam tests whether you can correctly assign instruction types to scopes. Personal preferences do not belong in project config. Team standards do not belong in user config. The new engineer scenario is the canonical illustration: team standards in the wrong scope means teammates start fresh, without context.

---

## How the Files Load

Understanding what "concatenation" means in practice requires knowing the load order.<sup>[2]</sup>

When Claude Code starts, it walks the directory tree from your current working directory upward. At each level, it checks for `CLAUDE.md` and `CLAUDE.local.md`. All discovered files are loaded. Content is ordered from the filesystem root down to the working directory: files at the top of the tree appear first in the assembled context; files closer to where you launched Claude appear last.

Consider a session launched from `projects/myapp/src/`:

1. `CLAUDE.md` at `/` (if it exists)
2. `CLAUDE.md` at `/projects/`
3. `CLAUDE.md` at `/projects/myapp/`
4. `CLAUDE.md` at `/projects/myapp/src/`
5. `CLAUDE.local.md` files appended after `CLAUDE.md` at each respective level

Order matters.

Within any given directory, `CLAUDE.local.md` is appended after `CLAUDE.md`. So personal notes are always the last thing Claude reads at that level.<sup>[2]</sup>

One clarification on what "more specific wins" means in a concatenation model: nothing gets erased. Concatenation places more-specific-scope content later in the assembled context. Because Claude reads sequentially, later instructions carry more immediate weight when prior instructions conflict. But it is probabilistic, not guaranteed. Vague or contradictory instructions from different layers produce ambiguous behavior. For deterministic enforcement, hooks exist (that's Chapter 3's territory).

**Subdirectory files load differently.** Claude Code discovers `CLAUDE.md` and `CLAUDE.local.md` files in subdirectories under the working directory, but it does not load them at launch. They are included when Claude reads files in those subdirectories during a session.<sup>[2]</sup> This is relevant behavior to know for the exam: a `CLAUDE.md` in `src/api/` is not in context when the session starts; it enters context when Claude opens a file from that directory.

An important corollary: after `/compact`, the project-root `CLAUDE.md` is re-read from disk and re-injected into the session. Nested subdirectory `CLAUDE.md` files are not re-injected automatically. They reload the next time Claude reads a file in that subdirectory.<sup>[2]</sup> Instructions you gave only in conversation do not survive compaction. Add them to `CLAUDE.md` if they need to persist.

---

## The @import Syntax

CLAUDE.md files can pull in other files using `@path/to/import` syntax anywhere in the file.<sup>[2]</sup>

Both relative and absolute paths are allowed. Relative paths resolve relative to the file containing the import, not the working directory. Imported files are expanded and loaded at launch alongside the CLAUDE.md that references them. Recursive imports are supported, with a maximum depth of five hops.

The practical use case is modular configuration. A monorepo with packages serving different domains can have a project-level CLAUDE.md that imports only the standards relevant to each package, rather than loading every rule in every context:

```
# .claude/CLAUDE.md
Use TypeScript with strict mode enabled.

@./rules/api-design.md
@./rules/testing.md
```

One thing the exam will test: imports do not reduce context. Splitting a 400-line CLAUDE.md into four 100-line files and importing all four produces the same total context as the monolith did. The organization benefit is real (teams can own individual files; review is easier; diffs are cleaner), but the context budget does not shrink.<sup>[2]</sup> For reducing context, path-scoped rules are the correct mechanism.

Importing also works for team-shared references. A team can maintain a `standards/` directory with agreed-upon conventions and reference those files from CLAUDE.md with `@` syntax. Works with AGENTS.md too: if a repository already uses AGENTS.md for another coding assistant, a CLAUDE.md that starts with `@AGENTS.md` lets both tools read the same instructions without duplication.<sup>[2]</sup>

---

## The .claude/rules/ Directory

The `.claude/rules/` directory provides an organized alternative to a monolithic CLAUDE.md.<sup>[2,3]</sup>

Place markdown files in `.claude/rules/`. Each file should cover one topic, with a descriptive filename: `testing.md`, `api-design.md`, `deployment.md`. All `.md` files are discovered recursively, so rules can be organized into subdirectories like `frontend/` or `backend/`. The rules directory supports symlinks, including links to shared rule sets maintained outside the repository.<sup>[2]</sup>

There are two kinds of rules, distinguished by YAML frontmatter.

**Rules without `paths` frontmatter** load at launch, with the same priority as `.claude/CLAUDE.md`. They are always in context. Use them for conventions that apply to all work in the project.

**Rules with `paths` frontmatter** are path-scoped. They load only when Claude reads files matching the specified glob patterns. They do not consume context tokens unless they are relevant to the current task.<sup>[2]</sup>

```yaml
---
paths:
  - "**/*.test.tsx"
---
# Testing conventions for React components

Every test file must have a corresponding factory function for test fixtures.
```

This rule enters context when Claude opens any `.test.tsx` file, anywhere in the project. It does not enter context when Claude is working on infrastructure configuration files or backend services.

### Path-Scoped Rules vs. Subdirectory CLAUDE.md

The exam tests when to use path-scoped rules versus a subdirectory CLAUDE.md, and the distinction is meaningful.<sup>[3,4]</sup>

A subdirectory CLAUDE.md at `src/api/CLAUDE.md` covers everything Claude reads in that directory. A path-scoped rule with `**/*.ts` covers all TypeScript files regardless of which directory they are in.

If your conventions are directory-bound (this specific API module has unique auth requirements), a subdirectory CLAUDE.md is correct. If your conventions are type-bound (all TypeScript files must follow these patterns, regardless of whether they live in `src/`, `lib/`, `test/`, or `scripts/`), a path-scoped rule in `.claude/rules/` is correct.

The exam scenario that tests this: a team wants to enforce TypeScript conventions on test files that are distributed throughout a monorepo. A subdirectory CLAUDE.md would require one file per directory containing tests. A path-scoped rule with `**/*.test.tsx` covers the entire codebase with one file.<sup>[3]</sup>

---

## Size Guidance and What It Tells You

The documented target is under 200 lines per CLAUDE.md file.<sup>[2]</sup>

Longer files consume more context and reduce adherence. The adherence point is not just about tokens: CLAUDE.md content is delivered to Claude as a user message after the system prompt, not as part of the system prompt itself. Claude reads it and tries to follow it, but there is no guarantee of strict compliance, especially for vague or conflicting instructions.<sup>[2]</sup>

This is the architectural reality of the system. It is context, not enforced configuration. The more specific and concise the instructions, the more consistently Claude follows them. "Use 2-space indentation" works better than "format code properly." "Run `npm test` before committing" works better than "test your changes." The measurable, verifiable instruction has a better compliance rate than the directive that requires judgment to apply.

Block-level HTML comments in CLAUDE.md files (`<!-- maintainer notes -->`) are stripped before the content is injected into Claude's context. Use them to leave notes for human maintainers without spending context tokens on those notes. Comments inside code blocks are preserved.<sup>[2]</sup>

For instructions where compliance is genuinely required, the right mechanism is hooks, not CLAUDE.md. Hooks execute as shell commands at fixed lifecycle events and apply regardless of what Claude decides to do. Hooks are Chapter 3's territory, but the handoff point is important: CLAUDE.md is behavioral guidance; hooks are programmatic enforcement.

---

## User-Level Rules

The user scope applies to more than just `~/.claude/CLAUDE.md`. A `~/.claude/rules/` directory works the same way as the project-level `.claude/rules/` directory, but applies to every project on your machine.<sup>[2]</sup> Use it for personal preferences that are not project-specific: output verbosity, preferred code review style, debugging approaches.

User-level rules follow the same two-tier behavior: rules without `paths` frontmatter load universally; rules with `paths` frontmatter are path-scoped.

The sharing constraint applies here identically to user CLAUDE.md: personal rules in `~/.claude/rules/` are yours. Teammates do not see them.

---

## The /memory Command

When instructions are not being followed, the first diagnostic step is `/memory`.<sup>[2,4]</sup>

The `/memory` command lists all CLAUDE.md, CLAUDE.local.md, and rules files loaded in the current session. If a file is not in that list, Claude cannot see it. This is the fastest way to confirm that the file Claude should be reading is actually present in context.

The command also provides a link to open the auto memory folder and lets you toggle auto memory on or off. Selecting any file from the list opens it in your editor.

Common diagnostic flow: team member reports that Claude is not following the project style guide. Run `/memory`. If the project CLAUDE.md is not listed, either the file does not exist at a location that gets loaded for that session, or there is a `claudeMdExcludes` configuration blocking it. If the file is listed, check whether the instruction is specific enough. "Format code properly" will not produce consistent results; "use 4-space indentation for Python files" will.<sup>[2]</sup>

The `claudeMdExcludes` setting is relevant in large monorepos: ancestor CLAUDE.md files may contain instructions that are irrelevant to your current work. The `claudeMdExcludes` setting (available at any settings layer: user, project, local, or managed policy) lets you skip specific files by path or glob pattern. Managed policy CLAUDE.md files cannot be excluded.<sup>[2]</sup>

---

## The Key Trap

Return to the opening scene: the new engineer whose Claude session ignores team standards.

The diagnostic is now clear. Run `/memory` on the engineer's machine. If the project CLAUDE.md does not appear, start there. If it does appear, check whether the instruction that is being ignored lives in a subdirectory CLAUDE.md that has not been triggered yet (the engineer has not opened files from that directory).

But the most common cause, and the one the exam tests directly, is this: the instruction that was supposed to apply to the team was written in `~/.claude/CLAUDE.md` on the team lead's machine.<sup>[1,4]</sup>

The team lead sees correct behavior. Every other team member sees none. The file is not committed. It is not in version control. It never left the team lead's home directory.

This is the canonical user-level-versus-project-level mistake, and it has a shape that is easy to miss because the author of the instruction experiences no symptoms. Their Claude session follows the rule. They test it. It works. They move on.

The fix is mechanical: move the instruction from `~/.claude/CLAUDE.md` to `.claude/CLAUDE.md` in the repository. Commit it. Done.

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

The CLAUDE.md at project root (or `.claude/CLAUDE.md`) is short. It covers broad team conventions and uses `@import` to pull in topic-specific files. The `.claude/rules/` directory carries the detail. Path-scoped rules handle file-type-specific conventions. `CLAUDE.local.md` handles personal project-specific context. User-level files handle personal universal preferences.

Nobody wrote 800 lines in one file. Nobody put personal preferences in the committed config. Nobody wrote team standards in their home directory.

---

## Key Takeaways

**Four scopes, one direction.** Managed policy loads unconditionally for all users. Project loads from version control and applies to the team. User is personal and applies to all projects. Local is personal, gitignored, and project-specific. More-specific scope content is read later in the assembled context.

**Concatenation, not override.** All discovered CLAUDE.md files combine into a single assembled context. Nothing is erased. Conflicting instructions from different scopes produce ambiguous behavior; for guaranteed compliance, use hooks.

**Load order and `@import` behavior.** Files above the working directory load at launch; subdirectory files load when Claude reads files from those directories. `@import` adds context without reducing it: splitting a large CLAUDE.md into imported files produces identical context load. For reducing context, use path-scoped rules in `.claude/rules/`.

**Path-scoped rules are about file type, not directory.** A rule with `paths: ["**/*.test.tsx"]` applies to all matching files anywhere in the codebase. A subdirectory CLAUDE.md applies to all files in that directory. Use path-scoped rules when conventions follow file type across directory boundaries.

**Size guidance exists for a reason.** Target under 200 lines per file. CLAUDE.md is delivered as a user message, not enforced configuration. Longer files with vague instructions produce less reliable compliance than shorter files with specific, verifiable ones.

**`/memory` is the diagnostic command.** If an instruction is not being followed, run `/memory` first. If the file is not listed, Claude cannot see it. If the file is listed, the instruction is too vague or conflicts with another.

**The exam's primary trap.** Team instructions in user-level config (`~/.claude/CLAUDE.md`) are invisible to teammates. Project instructions belong in `.claude/CLAUDE.md`, committed to version control. This is the most commonly tested configuration mistake in Scenario 2.

---

## Sample Questions

**Q1.** A team lead configured Claude Code to follow company-specific error handling conventions. After onboarding three new engineers, none of them observe Claude following these conventions. The team lead's environment works correctly. Which configuration change resolves this?

A. Move the error handling instructions from `~/.claude/CLAUDE.md` to `.claude/CLAUDE.md` in the repository and commit the file.
B. Ask each engineer to copy the team lead's `~/.claude/CLAUDE.md` to their local machine.
C. Add the instructions to `CLAUDE.local.md` at the project root and commit that file.
D. Configure the managed policy file to load user-level settings from a shared network location.

*Correct: A. User-level config is personal and not shared via version control. Project-level config in `.claude/CLAUDE.md` is shared through the repository. `CLAUDE.local.md` is gitignored (C is wrong). Manually syncing user config (B) is not a durable team solution and does not address the root cause.*

---

**Q2.** A monorepo contains TypeScript test files distributed across twelve subdirectories. The team wants Claude Code to apply consistent testing conventions whenever it works with any test file, regardless of directory. Which approach is most appropriate?

A. Create a `.claude/rules/testing.md` file with `paths: ["**/*.test.ts"]` frontmatter.
B. Create a `CLAUDE.md` in each of the twelve subdirectories containing test files.
C. Add the testing conventions to the root CLAUDE.md using `@import` syntax.
D. Create a `.claude/rules/testing.md` file without any frontmatter.

*Correct: A. Path-scoped rules with glob patterns apply by file type regardless of directory. B requires twelve files and maintenance. C loads the testing conventions for all sessions regardless of whether tests are being worked on (correct for always-applicable rules, but wasteful here). D loads at launch unconditionally, not path-scoped.*

---

**Q3.** A developer imports three large reference documents into `CLAUDE.md` using `@` syntax to keep the file itself under 200 lines. What effect does this have on the session context?

A. The imported files are lazy-loaded only when Claude references them during the session.
B. The imported files load at launch alongside CLAUDE.md, increasing total context proportionally.
C. The imported files replace the content of CLAUDE.md in context, keeping total size the same.
D. The imported files are summarized before loading to reduce context impact.

*Correct: B. Imported files are expanded and loaded at launch. The 200-line guidance applies per file, but importing multiple files still loads all of them. `@import` aids organization, not context reduction.*

---

**Q4.** After a `/compact` operation, a developer notices that Claude no longer follows conventions that were specified in `src/api/CLAUDE.md`. The project-root CLAUDE.md instructions are still being followed. What is the most likely explanation?

A. Compact operations delete CLAUDE.md files from subdirectories.
B. The project-root CLAUDE.md survives compaction and is re-injected; nested subdirectory CLAUDE.md files are not re-injected automatically and reload only when Claude next reads files from those subdirectories.
C. The `src/api/CLAUDE.md` instructions conflict with the project-root CLAUDE.md and were suppressed.
D. Compact operations only preserve the managed policy CLAUDE.md.

*Correct: B. After `/compact`, the project-root CLAUDE.md is re-read from disk and re-injected. Nested subdirectory files are not re-injected; they reload the next time Claude reads a file from that directory.*

---

