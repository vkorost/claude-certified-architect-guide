# Chapter 8: Claude Code in CI/CD

**Summary:** *Running Claude Code in a CI/CD pipeline requires a different mental model than running it interactively. There is no human available to answer permission prompts, approve tool use, or intervene when something hangs. The configuration choices that are optional in a terminal session become load-bearing in a pipeline. This chapter covers the flags, the session architecture, and the cost controls that make Claude Code a reliable CI participant: the **-p** flag for non-interactive mode, **--output-format json** with **--json-schema **for machine-parseable output, **permissionMode: "bypassPermissions"** for unattended execution, session context isolation as the core architectural principle, and the Message Batches API for latency-tolerant workloads. The central anti-pattern is same-session self-review: a generation session asked to review its own output retains all prior reasoning context and cannot deliver an independent assessment. Review-session isolation, where the reviewing Claude invocation is a completely separate process with no shared session ID, is the architectural fix. The **--bare** flag enforces reproducibility by disabling discovery of local hooks, skills, and MCP servers, so pipeline behavior is consistent across all machines in the fleet.*

---

## The Scenario That Broke a Pipeline

Consider a CI pipeline built to run automated code review on every pull request. The pipeline works like this: first, Claude Code generates implementation code in one step. Then, in the very next step, the pipeline calls Claude Code again in the same session to review what was just written.

The results look fine. Claude approves the code. Issues that a human reviewer would catch slip through. The generator reviewed its own work. It retained all the reasoning context from generation: the intent, the tradeoffs, the justifications it had already made internally. Asking the generator to review its own output in the same session is not a second opinion. It is a confirmation.<sup>[3]</sup>

The pipeline called *--resume* with the original session ID, thinking continuity was a feature. *It was the bug.* *The reviewer “approves” code it just wrote because it has already committed to the reasoning that produced that code.*

The fix is architectural. The fix is what this chapter calls **review-session isolation**.

---

## The Flag That Everything Else Depends On

Start here. Before session isolation, before structured output, before any of the CI-specific configuration matters: Claude Code, without modification, waits for interactive input. It presents prompts, waits for human responses, and blocks the pipeline indefinitely.<sup>[1]</sup>

The **-p** flag (equivalently **--print**) switches Claude Code into non-interactive mode. Add it to any claude command. The process runs, outputs its result to stdout, and exits. Without it, a CI job will hang until the runner hits its timeout limit and kills the job with a generic error that tells you nothing useful about what actually went wrong.<sup>[1]</sup>

```
claude -p "Review this PR diff for missing error handling"
```

That is the minimum viable CI invocation. 
Everything else in this chapter builds on it.

The documentation is explicit: all other CLI options work with -p.<sup>[1]</sup> The flag does not remove functionality. It removes the assumption of human presence.

### Bare Mode for Reproducibility

A related concern in CI is reproducibility. By default, claude -p loads the same context an interactive session would: hooks, skills, plugins, MCP servers, auto memory, CLAUDE.md files from the working directory and ~/.claude.<sup>[1]</sup> A hook in a teammate’s personal ~/.claude directory, or an MCP server in the project’s .mcp.json, will run if present.

The ***--bare*** flag **disables all of that discovery.** Only flags passed explicitly take effect. The same invocation produces the same result on every machine in the fleet, regardless of what individual developers have configured locally.<sup>[1]</sup>

```
claude -p --bare "Summarize the build log" --allowedTools Read
```

Bare mode is recommended for scripted and SDK calls. The documentation notes it will become the default for -p in a future release.<sup>[1]</sup>

---

## Structured Output in CI

**Natural language output is not a CI artifact.** The pipeline needs something it can parse, route, and post as inline PR comments without a fragile regex in the middle.

*--output-format json* produces structured JSON. The response payload includes the result, the session ID, usage statistics, and a per-model cost breakdown.<sup>[1]</sup> The cost field means scripted callers can track spend per invocation without consulting a dashboard.

Combined with *--json-schema*, the response enforces a specific output shape. The structured result appears in the structured_output field of the response.<sup>[2]</sup>

A concrete CI invocation looks like this:

```
claude -p "Review this PR diff for:
  1. Functions exceeding 50 lines
  2. Missing error handling on async operations
  3. Hardcoded credentials or API keys
  4. Missing unit tests for new functions
Provide results as structured JSON." \
  --output-format json \
  --json-schema '{
    "type": "object",
    "properties": {
      "issues": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "file": {"type": "string"},
            "severity": {"type": "string"},
            "description": {"type": "string"}
          }
        }
      }
    }
  }'
```

The downstream script receives a JSON object, not text. It can iterate over issues, filter by severity, and post each one as a comment against the relevant file.<sup>[2]</sup>

---

## Review-Session Isolation

This is the named concept this chapter introduces, and it is worth being precise about what it means and why it matters.

A Claude Code session retains context. The session that generated a module knows why certain decisions were made. It knows what alternatives were considered and rejected. It carries the internal justifications for every line it wrote. When that same session is asked to review the code, it does not start fresh. It starts with all of that prior reasoning in context, which creates confirmation bias: the reviewer is less likely to question decisions it already made.<sup>[2,3]</sup>

**Review-session isolation** means: the session that reviews code must be completely separate from the session that generated it. Not --resume. Not --continue. A **new invocation**, with no session ID linking it to the generator.<sup>[2]</sup>

The correct CI architecture:

```
# Step 1: Generation (Session A)
claude -p "Implement the auth module per the spec" --output-format json

# Step 2: Review (Session B — completely separate invocation, no --resume)
claude -p "Review this diff: $(git diff main)" --output-format json --json-schema '...'
```

The generator and reviewer see the same code. The reviewer has none of the generator’s reasoning context. That is the point.<sup>[2,3]</sup>

The exam scenario presents this as a tricky tradeoff (doesn’t continuity help?) but there is no tradeoff here. Same-session self-review is an anti-pattern with no compensating benefit. An independent instance without prior reasoning context is more effective at catching subtle issues than any self-review instruction.<sup>[3]</sup>

### Re-Running Reviews After New Commits

Isolation applies to fresh reviews. When re-running a review after new commits address earlier feedback, the architecture shifts slightly. A completely fresh session with no context about prior findings will flag the same issues again. That produces duplicate comments on already-addressed problems and trains developers to ignore the review system.<sup>[3]</sup>

The correct approach: **include the prior review findings in context, and instruct Claude to report only new or still-unaddressed issues**.<sup>[3]</sup>

```
PRIOR_FINDINGS=$(cat .ci/prior-review.json)
claude -p "Prior review findings: $PRIOR_FINDINGS

Review this updated diff and report ONLY issues that are new or were not addressed in the prior findings." \
  --output-format json \
  --json-schema '...'
```

The reviewer still runs in an isolated session. But it is given enough context to know what has already been flagged. The isolation principle is preserved; the duplicate-comment problem is not.

---

## CLAUDE.md in CI

A CI-invoked session loads CLAUDE.md from the repository root just as an interactive session would (unless --bare is passed).<sup>[4]</sup> This is the mechanism for providing project context to the reviewer: testing standards, fixture conventions, review criteria, coding patterns the team has agreed on.

Without it, the reviewer operates with only what the prompt contains. That produces generic feedback disconnected from the project’s actual standards. With a well-constructed CLAUDE.md, the reviewer knows the project uses pytest fixtures in a specific directory structure, that all async functions require explicit error handling on network calls, and that functions exceeding 50 lines trigger a mandatory review comment.<sup>[3]</sup>

A CI-specific CLAUDE.md section might look like this:

```typescript
## CI Review Standards
- Flag any function exceeding 50 lines
- Flag missing error handling on all async operations
- Flag hardcoded strings matching API key or password patterns
- Do not suggest tests for scenarios already covered by files in /tests/unit/
- Testing fixtures are in /tests/fixtures/ — reference them in test suggestions
```

The last two points address a common problem in test generation: Claude suggesting tests for scenarios already covered, or generating tests that reinvent fixtures the project already provides.<sup>[3]</sup> Document what exists. The CI session will use it.

---

## Permission Mode for CI

Interactive Claude Code runs with permission prompts. The process pauses, asks whether to proceed, and waits. In CI, there is no human to answer. The process blocks, the runner times out, and the job fails with no useful information.

*permissionMode: "bypassPermissions" is* the recommended permission mode for CI and container environments where no human is available for approval.<sup>[5]</sup> With it, Claude Code proceeds without pausing for permission checks. All tool use runs automatically.

This is a known value defined in Chapter 1. CI is the use case it was designed for. The tradeoff is real: bypassing permissions means Claude can read, write, and execute without intervention. In a CI environment with known inputs (a PR diff, a set of source files) and bounded tool access (defined by --allowedTools), this is acceptable. In an environment with ambiguous scope, it requires more care.

The combination that appears in production CI configurations:

```
- name: Run Claude Code Review
  run: |
    claude -p "$REVIEW_PROMPT" \
      --output-format json \
      --json-schema "$SCHEMA" \
      --allowedTools "Read,Grep" \
      --max-turns 10
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

--allowedTools scopes which tools the session can invoke. Combined with bypassPermissions, it creates an unattended session that can only do what you explicitly permit.

---

## GitHub Actions Integration

The Claude Code GitHub Actions integration wraps the CLI in a workflow step. The key parameters:<sup>[4]</sup>

- **claude_args:** passes CLI arguments to the underlying claude invocation. This is how --max-turns, --model, --allowedTools, --output-format, and --json-schema reach the process.
- **ANTHROPIC_API_KEY**: stored as a repository secret, referenced as *${{ secrets.ANTHROPIC_API_KEY }}.*

```
name: Claude Code Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Claude Code Review
        uses: anthropics/claude-code-action@v1
        with:
          prompt: |
            Review this PR for the issues defined in CLAUDE.md.
            Output structured findings only.
          claude_args: --max-turns 10 --output-format json
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

The action handles the -p flag internally. The prompt parameter is the instruction; claude_args is where turn limits and output format live.<sup>[4]</sup>

--max-turns matters in CI. An unbounded agentic loop is a runaway job. Set it explicitly to a value appropriate for the complexity of the review task. Ten turns is a reasonable default for a standard PR review; adjust based on observed behavior.<sup>[4]</sup>

CLAUDE.md in the repository root guides Claude’s behavior without requiring it to be embedded in the workflow file. The separation is clean: the workflow file handles infrastructure; CLAUDE.md handles standards.<sup>[4]</sup>

---

## The Message Batches API

Some CI workloads are not blocking. A nightly technical debt report, a weekly security audit, test coverage analysis run overnight: none of these block a developer who is waiting to merge. They are **latency-tolerant**. They have a deadline measured in **hours**, not seconds.

The Message Batches API processes requests asynchronously with up to a 24-hour window and offers a 50% cost reduction versus synchronous API calls.<sup>[3,5]</sup> For latency-tolerant workloads, that is a direct operational cost reduction with no functional tradeoff.

Four things to know about the batch API:<sup>[5]</sup>

**Processing window:** Results arrive within 24 hours. There is no guaranteed latency SLA within that window. The batch may complete in minutes; it may take close to the full 24 hours. Plan accordingly.

**custom_id:** Each request in a batch carries a custom_id field. The response for each item carries the same custom_id, which is how the caller correlates responses to their originating requests. Without custom_id, a batch of 500 files becomes 500 unordered responses with no way to map them back.<sup>[5]</sup>

**No multi-turn tool calling:** The batch API does not support multi-turn tool calling within a single request. A batch request cannot execute a tool mid-request and return results to continue reasoning. This rules out batch for agentic tasks that require tool interaction. It is appropriate for single-turn workloads: analyzing a file, generating a summary, producing structured output from provided content.<sup>[5]</sup>

**Failure handling:** When individual requests in a batch fail, resubmit only the failed items by custom_id. Do not resubmit the entire batch. Items may fail for different reasons (context length exceeded, transient errors); handle each failure type appropriately before resubmission.<sup>[5]</sup>

### The Decision: Batch vs Synchronous

The decision rule is simple: *if a developer or a deploy process is waiting for the result, use **synchronous**. *If the result is consumed *hours later* by a scheduled process, use ***batch***.

Pre-merge checks are synchronous. A developer has opened a PR and is waiting to see if the review passes before merging. Switching that workflow to batch, then polling for completion, does not save money if the batch takes longer than the developer is willing to wait. It just adds latency and complexity for no benefit.<sup>[5]</sup>

Nightly reports are batch. Nobody is waiting. The report appears in the morning. The 50% cost reduction applies. The 24-hour window is not a constraint because the workload runs overnight anyway.<sup>[5]</sup>

The exam question in the guide makes this exact distinction: a blocking pre-merge check and an overnight technical debt report. The proposal is to switch both to batch for the cost savings. The correct answer is to switch only the overnight report. The pre-merge check stays synchronous because it is a blocking workflow.<sup>[5]</sup>

---

## Configuration Checklist for CI

Putting the pieces together, a production CI configuration for Claude Code review should have:

- **-p flag on every invocation.** Without it, the job hangs.
- **--output-format json for machine-parseable results.** Natural language output is not a CI artifact.
- **--json-schema** when the downstream consumer requires a specific structure. Pair it with --output-format json.
- A **separate session** for every review. No --resume, no --continue from the generation session. Review-session isolation is not optional.
- **ANTHROPIC_API_KEY via secrets.** Never hardcoded in workflow files.
- **--max-turns to prevent runaway jobs.** Set it explicitly.
- **CLAUDE.md at the repository root with project context:** testing standards, fixture locations, review criteria, patterns the team uses.
- **permissionMode: "bypassPermissions"** when the session needs unattended tool access, scoped by --allowedTools.
- **Prior review findings in context** when re-running after new commits. Instruct the session to report only new or unaddressed issues.
- **Message Batches API for latency-tolerant workloads.** Synchronous for blocking checks.

---

## What the Exam Tests Here

The exam does not test whether the candidate can write a GitHub Actions workflow file. It tests whether the candidate understands the architectural reasoning behind each configuration choice.

The -p flag question (Question 10 in the guide) is designed to catch candidates who confuse this with environment variables or stdin redirection. The flag is the documented mechanism. The other options do not exist or do not address the root problem.<sup>[5]</sup>

The session isolation question is designed to catch candidates who think continuity aids review. It does not. A reviewer that shares reasoning context with the generator is not reviewing; it is confirming.<sup>[2,3]</sup>

The batch API question is designed to catch candidates who apply cost optimization indiscriminately. The 50% savings are real. They do not apply to blocking workflows where a human is waiting for the result.<sup>[5]</sup>

Three facts drive most of the CI/CD scenario questions:

1. -p for non-interactive mode.
2. Separate sessions for code review (review-session isolation).
3. Message Batches API for latency-tolerant workloads; synchronous for pre-merge checks.

The configuration choices here assume the prompt is precise enough to make review findings actionable. Chapter 9 is where that assumption gets tested.

---

## Key Takeaways

- The -p (or --print) flag is required for CI; without it, the process hangs waiting for interactive input. permissionMode: "bypassPermissions" is the appropriate mode for CI and container environments; scope it with --allowedTools to limit which tools run unattended.
- --output-format json combined with --json-schema produces machine-parseable structured output that downstream pipeline steps can consume directly, without fragile text parsing.
- **Review-session isolation** is the core architectural principle for CI code review: the session that reviews code must be completely separate from the session that generated it. Same-session self-review retains generator reasoning context and produces confirmation bias.
- When re-running a review after new commits, include prior findings in context and instruct Claude to report only new or unaddressed issues. This prevents duplicate comments without sacrificing session isolation.
- CLAUDE.md at the repository root provides project context (testing standards, fixture locations, review criteria) to every CI-invoked session that does not use --bare.
- GitHub Actions integration passes CLI arguments via claude_args. ANTHROPIC_API_KEY goes in repository secrets, not workflow files.
- The ***Message Batches API* offers 50% cost savings** with a 24-hour processing window and no guaranteed latency SLA. It is appropriate for latency-tolerant workloads (overnight reports, weekly audits) but not for blocking pre-merge checks, and it does not support multi-turn tool calling. The ***custom_id* field correlates batch requests to their responses**; without it, batch responses cannot be mapped back to their source files.
