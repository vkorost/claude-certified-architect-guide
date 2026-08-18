# Chapter 8: Claude Code in CI/CD

**Summary:** *Running Claude Code in a CI/CD pipeline requires a different mental model than running it interactively. There is no human available to answer permission prompts, approve tool use, or intervene when something hangs. The configuration choices that are optional in a terminal session become load-bearing in a pipeline. This chapter covers the flags, the session architecture, and the cost controls that make Claude Code a reliable CI participant: the **-p** flag for non-interactive mode, --output-format json with --json-schema for machine-parseable output, the permission modes that let a session run unattended and what they do not switch off, session context isolation as the core architectural principle, turn and dollar caps enforced inside the process rather than by the job runner, and the Message Batches API for latency-tolerant workloads. The central anti-pattern is same-session self-review: a generation session asked to review its own output retains all prior reasoning context and cannot deliver an independent assessment. Review-session isolation, where the reviewing Claude invocation is a completely separate process with no shared session ID, is the architectural fix. The --bare flag enforces reproducibility by disabling every kind of local auto-discovery, from hooks and skills through to memory files, so pipeline behavior is consistent across every machine in the fleet.*

---

## The Scenario That Broke a Pipeline

Consider a CI pipeline built to run automated code review on every pull request. The pipeline works like this: first, Claude Code generates implementation code in one step. Then, in the very next step, the pipeline calls Claude Code again in the same session to review what was just written.

The results look fine. Claude approves the code. Issues that a human reviewer would catch slip through. The generator reviewed its own work. It retained all the reasoning context from generation: the intent, the tradeoffs, the justifications it had already made internally. A model that still carries the reasoning it used to produce something is less likely to interrogate the decisions inside it, and the exam names that retained context as the reason self-review underperforms.<sup>[1]</sup> Asking the generator to review its own output in the same session is not a second opinion. It is a confirmation.

The pipeline called --resume with the original session ID, thinking continuity was a feature. *It was the bug.* *The reviewer “approves” code it just wrote because it has already committed to the reasoning that produced that code.*

The fix is architectural. The fix is what this chapter calls **review-session isolation**.

---

## The Flag That Everything Else Depends On

Start here. Before session isolation, before structured output, before any of the CI-specific configuration matters: Claude Code, without modification, waits for interactive input. It presents prompts, waits for human responses, and blocks the pipeline indefinitely.<sup>[2]</sup>

The **-p** flag (equivalently the --print flag) switches Claude Code into non-interactive mode. Add it to any claude command. The process runs, outputs its result to stdout, and exits. Without it, a CI job will hang until the runner hits its timeout limit and kills the job with a generic error that tells you nothing useful about what actually went wrong.<sup>[2]</sup>

```
claude -p "Review this PR diff for missing error handling"
```

That is the minimum viable CI invocation. 
Everything else in this chapter builds on it.

The flag does not remove functionality. It removes the assumption of human presence. Most other CLI options combine with it, though not all of them: a background-run flag is rejected outright, and a cloud-session flag paired with a task description is rejected with an error naming the conflict.<sup>[2]</sup> The practical consequence for a pipeline is that an invalid flag combination fails before the run starts and reports to stderr, while a failure inside the run, missing authentication being the common one, is printed as the result on stdout instead. A script that only inspects stdout will read an authentication failure as a review finding. Branch on the exit code: zero on success, non-zero when the run fails.<sup>[2]</sup>

### Bare Mode for Reproducibility

A related concern in CI is reproducibility. By default, claude -p loads the same context an interactive session would: hooks, skills, plugins, MCP servers, auto memory, CLAUDE.md files from the working directory and ~/.claude.<sup>[2]</sup> A hook in a teammate’s personal ~/.claude directory, or an MCP server in the project’s .mcp.json, will run if present.

There is a sharper edge to this than reproducibility. A -p session shows no workspace trust dialog and no per-server approval prompt, because there is nobody to show them to. So without bare mode, Claude Code runs the hooks declared in a repository’s .claude/settings.json even in a checkout nobody has ever trusted, and connects the MCP servers that repository declares.<sup>[2]</sup> For a pipeline that builds pull requests from forks, that is the whole attack surface in one sentence.

The --bare flag **disables all of that discovery.** Only flags passed explicitly take effect. The same invocation produces the same result on every machine in the fleet, regardless of what individual developers have configured locally, and regardless of what the branch under review has added to the repository.<sup>[2]</sup>

```
claude -p --bare "Summarize the build log" --allowedTools Read
```

Bare mode is recommended for scripted and SDK calls. The documentation notes it will become the default for -p in a future release.<sup>[2]</sup>

One consequence surprises people the first time they hit it. Bare mode does not read OAuth credentials or the system keychain, so a subscription login that works in a terminal will not authenticate a bare run. Set ANTHROPIC_API_KEY in the environment, or supply an apiKeyHelper through the settings passed on the command line. Bedrock, Google Cloud’s Agent Platform, and Microsoft Foundry keep reading their own provider credentials as usual.<sup>[2]</sup> Bare mode also narrows the starting tool surface to Bash, file read, and file edit; anything else, including MCP servers and custom agents, has to be handed in by flag.<sup>[2,3]</sup>

---

## Structured Output in CI

**Natural language output is not a CI artifact.** The pipeline needs something it can parse, route, and post as inline PR comments without a fragile regex in the middle.

The --output-format flag takes three values: text, which is the default, json, and stream-json for newline-delimited streaming.<sup>[2]</sup> For CI, json is the one that matters. Its response payload carries the text result in a result field alongside the session ID, usage statistics, a total cost figure and a per-model cost breakdown.<sup>[2]</sup> Those cost fields let scripted callers track spend per invocation without consulting a dashboard, with one caveat worth writing into whatever consumes them: they are client-side estimates and can differ from the billed amount.<sup>[2]</sup>

Combined with --json-schema, the response also conforms to a shape you specify. The structured result appears in the structured_output field, separate from the result field that holds the prose.<sup>[2]</sup> Two behaviors of that flag are worth knowing before a pipeline depends on it. An invalid schema now fails the invocation with an explicit error and the validator’s diagnostic rather than silently degrading to unstructured text. And the format keyword is accepted but treated as an annotation, not enforced, so a schema that declares a field as an email address has documented an intention rather than imposed a constraint. Validation of field contents remains the caller’s job.<sup>[2]</sup>

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

The downstream script receives a JSON object, not text. It can iterate over issues, filter by severity, and post each one as a comment against the relevant file. Producing findings a pipeline can post as inline comments without parsing prose is the stated purpose of pairing the two flags, and it is the form the exam expects.<sup>[1]</sup>

---

## Review-Session Isolation

This is the named concept this chapter introduces, and it is worth being precise about what it means and why it matters.

A Claude Code session retains context. The session that generated a module knows why certain decisions were made. It knows what alternatives were considered and rejected. It carries the internal justifications for every line it wrote. When that same session is asked to review the code, it does not start fresh. It starts with all of that prior reasoning in context, and a model holding the reasoning that produced a decision is less likely to put that decision back on trial. The exam states the limitation in exactly those terms and treats it as a property of the session rather than a property of the prompt.<sup>[1]</sup> This book calls the resulting failure confirmation bias, which is a name for the mechanism rather than an additional claim about it.

**Review-session isolation** means: the session that reviews code must be completely separate from the session that generated it. Not --resume. Not --continue. Those two flags are the documented mechanisms for carrying a prior conversation forward, --continue picking up the most recent one and --resume attaching to a specific session ID, and neither belongs in a review step.<sup>[2]</sup> What the review step needs is a **new invocation**, with no session ID linking it to the generator: an independent instance that never saw the generator's working.<sup>[1]</sup>

(This is the review-specific exception to a general capability. For when session resumption and forking are correct, and the decision rules governing resume-versus-fresh, see the session-state section in Chapter 2.)

The correct CI architecture:

```
# Step 1: Generation (Session A)
claude -p "Implement the auth module per the spec" --output-format json

# Step 2: Review (Session B, a completely separate invocation, no --resume)
claude -p "Review this diff: $(git diff main)" --output-format json --json-schema '...'
```

The generator and reviewer see the same code. The reviewer has none of the generator’s reasoning context. That is the point.<sup>[1]</sup>

The exam presents this as a tricky tradeoff (doesn’t continuity help?) but there is no tradeoff here. Same-session self-review is an anti-pattern with no compensating benefit. A fresh instance that never held the generator's reasoning catches subtle issues that a same-session reviewer misses, and it outperforms two things candidates reach for instead: an instruction telling the model to review its own work critically, and extended thinking applied to the same session.<sup>[1]</sup> Both of those operate inside the context that is causing the problem. Neither removes it.

The diagnostic signature matters as much as the fix, because it is what a scenario gives you to reason from. A session running out of room degrades everything it touches rather than one class of finding, and a reviewer with less codebase access sees less rather than questions less. A clean catch by a separate reviewer, inside output that otherwise looked normal, points at what the first reviewer was willing to challenge.

### Re-Running Reviews After New Commits

Isolation applies to fresh reviews. When re-running a review after new commits address earlier feedback, the architecture shifts slightly. A completely fresh session with no context about prior findings will flag the same issues again, and the pipeline will post duplicate comments on problems the author has already fixed.<sup>[1]</sup>

The correct approach puts the earlier findings into the reviewer's context and narrows what counts as reportable: anything already raised and still outstanding, or anything not seen before, and nothing else.<sup>[1]</sup> Note where the fix lives. It goes on the input side, as prior findings plus a rule about what now counts as reportable. Filtering the output text discards findings the run has already paid for, and neither a session reset nor a different API surface addresses the cause. A reviewer needs to know what was already said in order to say something else.

```
PRIOR_FINDINGS=$(cat .ci/prior-review.json)
claude -p "Prior review findings: $PRIOR_FINDINGS

Review this updated diff and report ONLY issues that are new or were not addressed in the prior findings." \
  --output-format json \
  --json-schema '...'
```

The reviewer still runs in an isolated session. But it is given enough context to know what has already been flagged. The isolation principle is preserved; the duplicate-comment problem is not.

---

## Multi-Pass Review

Review-session isolation solves the bias problem: the reviewer does not share the generator's reasoning context. Multi-pass review solves a different problem: attention dilution across files.

The failure signature is distinctive. A single-pass review of a large changeset (ten or more files in one invocation) produces inconsistent depth: obvious issues caught in one file are missed in another, the same anti-pattern is flagged in file three and approved without comment in file seven, and the reviewer's thoroughness visibly decays as context fills. The root cause is that the model's attention is spread across too many files simultaneously, and the later files receive whatever capacity remains after the earlier files consumed their share.

The fix is structural decomposition of the review itself into two layers, and it is the split the exam names.<sup>[1]</sup> Per-file local passes: each file gets its own focused review invocation, examining internal logic, style violations, and local correctness in isolation. Then a separate cross-file integration pass: a final invocation that receives the per-file findings plus the dependency graph and checks for issues that span boundaries (data-flow mismatches, inconsistent error-handling conventions, API contract violations between modules, import cycles introduced by the changeset as a whole). The local passes catch depth issues; the integration pass catches breadth issues. Neither alone covers both.

The same decomposition answers a different-looking complaint: findings misfiled across categories when a single prompt asks for security, performance and style at once. That is the same dilution, spread across subject matter instead of across files, and the response is the same. Give each domain its own pass. It is not a capability shortfall, and a more capable model does not repair a focus problem.

This is orthogonal to the multi-instance pattern described above. Three review architectures exist for three distinct failure modes, and the exam expects candidates to distinguish them cleanly. Multi-instance review (independent reviewer, no shared context) corrects for generation-context bias: the generator's retained reasoning would corrupt a same-session review. Multi-pass review (per-file local passes plus cross-file integration) corrects for attention dilution: a single pass spread over many files, or many concerns, loses depth. Second-pass self-critique, the evaluator-optimizer pattern in Chapter 12, corrects for per-case quality variance: the same draft checked against a completeness rubric before delivery. Reviewer identity, reviewer scope, and output quality control are three separate axes. Different failures, different architectures. The stem tells you which failure is in play; match the architecture to it.

A further pass type belongs to this task statement and is easy to miss, because it changes nothing about how the review is split. A verification pass can have the model report a confidence level beside each finding, which turns a flat list into something a pipeline can route.<sup>[1]</sup> That is the structural answer to a reviewer that is precise but quiet: tuned to suppress anything uncertain, it loses real bugs with the noise. The fix is not a gentler suppression instruction. It is to report everything found, tagged, and decide what surfaces in a stage not also responsible for finding things.

---

## CLAUDE.md in CI

A CI-invoked session loads CLAUDE.md from the repository root just as an interactive session would (unless --bare is passed).<sup>[4]</sup> This is the mechanism for providing project context to the reviewer, including the coding patterns the team has agreed on.

Without it, the reviewer operates with only what the prompt contains. That produces generic feedback disconnected from the project’s actual standards. The exam names three things CLAUDE.md is the mechanism for supplying to a CI-invoked session: testing standards, fixture conventions, and review criteria.<sup>[1]</sup> Each of those is a class of thing the reviewer cannot infer from a diff. It can see that a function is long; it cannot know the length your team decided to argue about. It can see a test file; it cannot know which fixtures already exist and are meant to be reused.

The following is this book’s illustration of what that looks like written down, not a configuration the exam guide prints:

```typescript
## CI Review Standards
- Flag any function exceeding 50 lines
- Flag missing error handling on all async operations
- Flag hardcoded strings matching API key or password patterns
- Do not suggest tests for scenarios already covered by files in /tests/unit/
- Testing fixtures are in /tests/fixtures/. Reference them in test suggestions
```

The last two points address the failure the exam calls out by name in test generation: suggested tests that duplicate scenarios the suite already covers, and generated tests that reinvent fixtures the project already provides. Two remedies are named for it, and they operate at different times. Putting the existing test files into the session's context tells the reviewer what has already been tested on this run. Documenting the standards, the criteria for a test worth having, and the available fixtures in CLAUDE.md raises the quality of what gets generated in the first place and cuts the low-value output at its source.<sup>[1]</sup>

That timing distinction answers a scenario worth naming: developers reject the generated tests as trivial, and the instinct is a second Claude call that scores them and drops the weak ones. A scenario that also forbids added latency and pipeline changes eliminates that instinct by its own cost. CLAUDE.md acts before generation and changes what the first call produces rather than filtering what it produced.<sup>[1]</sup> Document what exists. The CI session will use it.

---

## Permission Mode for CI

Interactive Claude Code runs with permission prompts. The process pauses, asks whether to proceed, and waits. In CI, there is no human to answer. The process blocks, the runner times out, and the job fails with no useful information.

Two of the permission modes enumerated in Chapter 1 answer that, and they answer it differently. The distinction is worth holding, because the two are routinely conflated and only one of them is a bounded configuration.

**bypassPermissions** runs everything without asking, including writes to protected paths. Its documented scope is isolated containers and VMs, environments where Claude Code cannot damage a host, and it carries an explicit warning against use anywhere else.<sup>[5]</sup> A CI runner in a disposable container qualifies. A self-hosted runner with credentials mounted and a persistent workspace does not, and the difference is not cosmetic.

**dontAsk** runs only what has been pre-approved. Anything that would otherwise raise a prompt is auto-denied instead. What runs is what matches the permission allow rules, the built-in read-only command set, and anything a PreToolUse hook approves. Its documented use is CI pipelines and restricted environments where the permitted actions are defined in advance, and it never waits for input.<sup>[5]</sup> For an unattended review that should read a diff and write a JSON result, this is the mode that expresses that intent as a boundary rather than as a hope.

Now the part that gets missed, and the reason the choice between the two is not a matter of taste. **Bypassing permissions does not switch off the permission system.** Deny rules and explicit ask rules apply in every mode, bypass included. Connector tools an organization has set to ask, and MCP tools marked as requiring user interaction, still prompt under bypass.<sup>[5]</sup> In an interactive session that prompt is an inconvenience. In CI it is the hang the mode was chosen to prevent, arriving from a direction nobody instrumented. Chapter 3's rule holds here too: a hook deny survives bypass, because hooks are evaluated ahead of the rules and the mode, while a hook allow never grants what a settings-level deny forbids.

There is a second inversion in the same neighborhood. Under bypassPermissions, **allow rules have no effect**, because everything else is already approved.<sup>[5]</sup> A pipeline that passes --allowedTools "Read,Grep" alongside bypass has not scoped anything. The flag grants tools permission to run without prompting; it does not remove the others from the session. Restricting which tools exist at all is a different flag, --tools.<sup>[3]</sup> Under dontAsk the allow list does the work people expect --allowedTools to do under bypass, which is most of the argument for preferring it.

Two operational details close the section. On Linux and macOS, Claude Code refuses to start in bypass mode as root or under sudo, outside a recognized sandbox, and many runners execute as root by default; the dev container configuration exists partly to solve this by running as a non-root user.<sup>[5]</sup> And an administrator can disable the mode outright through managed settings, which means a workflow depending on it can be correct on a laptop and dead in the organization that owns the pipeline.<sup>[5]</sup>

A CI invocation with the caps and the tool surface stated explicitly:

```
- name: Run Claude Code Review
  run: |
    claude -p "$REVIEW_PROMPT" \
      --output-format json \
      --json-schema "$SCHEMA" \
      --permission-mode dontAsk \
      --allowedTools "Read,Grep" \
      --max-turns 10 \
      --max-budget-usd 2.00
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Under dontAsk, --allowedTools is the definition of what the session may do without asking, and everything outside it that would have prompted is denied instead. That is what makes the pair an unattended session with a boundary rather than an unattended session with a hope.

---

## GitHub Actions Integration

The Claude Code GitHub Actions integration wraps the CLI in a workflow step. The key parameters:<sup>[4]</sup>

- **prompt:** the instruction for Claude, as plain text or a skill invocation. Its presence is also a mode switch. Supply a prompt and the action runs automatically on whatever event fired the workflow. Omit it and the action waits for a trigger phrase, @claude by default, in a comment or a newly opened issue, and responds to that request instead.<sup>[4]</sup>
- **claude_args:** passes CLI arguments to the underlying claude invocation. This is how --max-turns, --model, --allowedTools and the rest of the CLI reach the process.<sup>[4]</sup>
- **anthropic_api_key:** the Claude API key, stored as a repository secret and referenced as *${{ secrets.ANTHROPIC_API_KEY }}*. A subscription token goes in the sibling input claude_code_oauth_token instead. Never in the workflow file.<sup>[4]</sup>
- **settings:** Claude Code settings, as a JSON string or a path to a settings file, which is where permission rules go when claude_args would be unwieldy.<sup>[4]</sup>

```
name: Claude Code Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      id-token: write
    steps:
      - uses: actions/checkout@v6
      - name: Run Claude Code Review
        uses: anthropics/claude-code-action@v1
        with:
          prompt: |
            Review this PR for the issues defined in CLAUDE.md.
            Output structured findings only.
          claude_args: --max-turns 10 --output-format json
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

You do not pass -p yourself; the action runs Claude Code for you. The prompt parameter is the instruction; claude_args is where turn limits and output format live.<sup>[4]</sup> Two of the workflow permissions are load-bearing rather than boilerplate. The action's default GitHub App authentication needs id-token: write. Granting actions: read is what lets Claude read the CI results on a pull request, which is usually the difference between a review that knows the build failed and one that does not.<sup>[4]</sup>

By default an automation-mode run writes its results to the workflow log rather than to the pull request. Claude posts to the PR when the prompt directs it to and it has a tool that can post, which in practice means naming the inline-comment tool in --allowedTools.<sup>[4]</sup> A workflow that appears to run cleanly and produces no comments is more often this than a reviewer with nothing to say.

CLAUDE.md in the repository root guides Claude’s behavior without requiring it to be embedded in the workflow file. The separation is clean: the workflow file handles infrastructure; CLAUDE.md handles standards.<sup>[4]</sup>

### Two Reviewers, Two Configuration Surfaces

There is a second GitHub reviewer that is not this one, and telling them apart is the whole of a question that otherwise looks like a CLAUDE.md question.

The workflow integration above is Claude Code in your CI, configured by what you commit to .github/workflows and by CLAUDE.md. Separately there is a managed review service an organization owner enables once, which analyzes pull requests on Anthropic's infrastructure and posts severity-tagged inline comments with no workflow file to maintain.<sup>[6]</sup> A scenario saying the review arrives through the managed GitHub App, rather than a workflow the team wrote, has selected that service and the file that configures it.

The service reads two files and they are not interchangeable.<sup>[6]</sup>

- **CLAUDE.md** is project context, the instructions every session gets, governing all Claude Code work rather than reviews. Violations the pull request newly introduces are reported at the lowest severity.
- **REVIEW.md**, at the repository root, is review-only, and it outranks the default review guidance inside the reviewing agents themselves. Severity recalibration lives here, with caps on minor comments, excluded paths and categories, and any evidence requirement placed on a finding before it is posted.

So for a reviewer producing too much noise, the tempting fix is a CLAUDE.md paragraph explaining why the pattern is intentional. That lands as project context and does not change what the reviewer treats as reportable. Skip rules and an evidence bar in REVIEW.md change the reviewer's own instructions. Because that file is pasted in literally, import syntax is not expanded and referenced files are not pulled in.<sup>[6]</sup> One asymmetry: the local /code-review command follows CLAUDE.md but ignores REVIEW.md entirely.<sup>[6]</sup>

### Cost and Startup: Two Reflexes Worth Suppressing

Both of the CI failures that produce the most heat have an obvious fix that does not work.

**When the cost per run climbs, the reflex is a cheaper model.** It is the wrong lever, and not because a smaller model cannot review code. It is wrong because what overran was iterations, and a cheaper model has exactly as few iteration limits as an expensive one. The caps that bind are enforced inside Claude Code: --max-turns limits agentic turns and exits with an error on reaching the limit, with no limit by default, and --max-budget-usd stops the run once API spend reaches a stated dollar amount, counting subagent spend toward the same cap.<sup>[3]</sup>

The other tempting answer is a timeout on the job step. It is a real control and it is not this control: it kills the runner after the fact, does not constrain what was spent before the kill, and yields no result rather than a bounded one. When a scenario insists the ceiling be held by Claude Code itself and not by whatever runs the job around it, that requirement is doing the selecting. Job timeouts, concurrency limits and usage dashboards are all sensible; none caps turns or dollars.<sup>[4,3]</sup>

**When startup is slow, the reflex is to trim CLAUDE.md.** The delay comes from discovery: before the first turn, Claude Code walks the filesystem for every kind of local configuration it might be expected to honour. Trimming those files leaves the walk in place and removes the standards the review depends on, trading a slow correct review for a fast generic one.

The fix is to skip discovery entirely and then load exactly what is needed. --bare eliminates the walk at its source; --append-system-prompt-file loads one named file into the system prompt and nothing else.<sup>[2,3]</sup> Repeating the criteria inline in the -p prompt on every invocation reaches the same place by a worse road: they drift between workflows and nobody updates all of them.

This is where the system prompt flags earn their distinction, because the wrong one silently deletes behavior a CI reviewer depends on. The flag --system-prompt **replaces the default system prompt entirely**, while --append-system-prompt **adds to it**, and --append-system-prompt-file does the same from a file.<sup>[3]</sup> A review that has stopped honoring documented standards, or stopped reading files it obviously should have read, is very often a --system-prompt where an append belonged: the replacement discarded the default prompt along with the file-navigation guidance built into it. Granting more tools misreads the symptom. The tools were never denied. The instructions for using them were.

---

## The Message Batches API

Some CI workloads are not blocking. A nightly technical debt report, a weekly security audit, test coverage analysis run overnight: none of these block a developer who is waiting to merge. They are **latency-tolerant**. They have a deadline measured in **hours**, not seconds.

The Message Batches API processes requests asynchronously and charges all usage at half the standard prices.<sup>[7]</sup> The exam carries the same figure.<sup>[1]</sup> For latency-tolerant workloads, that is a direct operational cost reduction with no functional tradeoff: the same model, the same prompts, the same output quality, at half the per-token price. Nothing else on the usual list of cost measures does that. Concurrency makes the same requests finish sooner and costs exactly as much; a smaller model changes the quality; merging documents into one request changes the task. The discount is available only to work that can wait, which is why latency tolerance, and not volume, selects this path.

Four things to know about the batch API:<sup>[1]</sup>

**Processing window:** The 24 hours is a ceiling, not a forecast, and this is the most misread fact in the area. Most batches finish in under an hour. Results become available once every message has completed or once 24 hours have elapsed, whichever comes first, and a batch still unfinished at that point expires.<sup>[7]</sup> What is absent anywhere in that range is a latency guarantee. So the question is not whether this will take a day; usually it will not. The question is whether anything breaks if it does. That is why a blocking check cannot move onto this path no matter how fast batches happen to be running this week.

**custom_id:** Each request in a batch carries a custom_id field. The response for each item carries the same custom_id, which is how the caller correlates responses to their originating requests.<sup>[1,7]</sup> Without custom_id, a batch of 500 files becomes 500 unordered responses with no way to map them back. Requests that expire unprocessed are not billed.<sup>[7]</sup>

**No multi-turn tool calling:** The batch API does not support multi-turn tool calling within a single request. A batch request cannot execute a tool mid-request and return results to continue reasoning. This rules out batch for agentic tasks that require tool interaction. It is appropriate for single-turn workloads: analyzing a file, generating a summary, producing structured output from provided content.<sup>[1]</sup>

**Failure handling:** When individual requests in a batch fail, resubmit only the failed items by custom_id. Do not resubmit the entire batch. Items may fail for different reasons, and the modification belongs to the failure type rather than to the batch: a document that exceeded the context limit needs chunking before it goes back, which is an input-size repair on that document and not a reason to reprocess the items that succeeded.<sup>[1]</sup> The related practice, and the one that reduces how often this happens, is to refine the prompt against a sample before committing a large volume, so that the first pass succeeds on most of the set instead of funding an iteration.<sup>[1]</sup>

### Sizing the Submission Interval

One arithmetic result belongs to this task statement, and it is the only place in the chapter where a number has to be worked rather than remembered.

When batched work carries a commitment measured from when an item arrives, two intervals stack: the item can arrive just after a submission goes out, so it waits for the next one, and then it waits for the batch. Worst case is the interval plus the window, and that sum must stay inside the commitment. So the interval can be no larger than the commitment minus the window.

With a 24-hour window and a 30-hour commitment the ceiling is six hours, and six-hour submissions are still wrong: they land on the boundary exactly, with nothing left for a retry. The exam's worked case submits every four hours, worst-casing at 28 hours with two hours of margin.<sup>[1]</sup> The trap is an interval picked for operational tidiness that ignores the window entirely.

### The Decision: Batch vs Synchronous

The decision rule is simple: *if a developer or a deploy process is waiting for the result, use **synchronous**. *If the result is consumed *hours later* by a scheduled process, use ***batch***.

Pre-merge checks are synchronous. A developer has opened a PR and is waiting to see if the review passes before merging. Switching that workflow to batch, then polling for completion, does not save money if the batch takes longer than the developer is willing to wait. It just adds latency and complexity for no benefit.<sup>[1]</sup>

Nightly reports are batch. Nobody is waiting. The report appears in the morning. The 50% cost reduction applies. The 24-hour window is not a constraint because the workload runs overnight anyway.<sup>[1]</sup>

The exam question in the guide makes this exact distinction: a blocking pre-merge check and an overnight technical debt report. The proposal is to switch both to batch for the cost savings. The correct answer is to switch only the overnight report. The pre-merge check stays synchronous because it is a blocking workflow.<sup>[1]</sup>

---

## Configuration Checklist for CI

Putting the pieces together, a production CI configuration for Claude Code review should have:

- **-p flag on every invocation.** Without it, the job hangs.
- **Structured results:** --output-format json, because natural language output is not a CI artifact. Natural language output is not a CI artifact.
- **A schema:** --json-schema when the downstream consumer requires a specific structure. Pair it with --output-format json, and read the payload from structured_output rather than from result.
- A **separate session** for every review. No --resume, no --continue from the generation session. Review-session isolation is not optional.
- **The API key via secrets**, and an environment variable rather than a login when the invocation is bare. Never hardcoded in workflow files.
- **Runaway guards:** --max-turns and --max-budget-usd. Set both explicitly; neither has a default, and a job timeout is not a substitute for either.
- **An appending system-prompt flag**, --append-system-prompt or --append-system-prompt-file rather than --system-prompt, whenever the default behavior should survive the addition.
- **CLAUDE.md at the repository root with project context:** testing standards, fixture conventions, review criteria, patterns the team uses. **REVIEW.md** instead when the reviewer is the managed GitHub App rather than a workflow you wrote.
- **A permission mode chosen deliberately:** dontAsk with an explicit allow list for a locked-down run, bypassPermissions only in a genuinely disposable container, and in either case an expectation that deny rules, ask rules and hook denials still apply.
- **Prior review findings in context** when re-running after new commits. Instruct the session to report only new or unaddressed issues.
- **Message Batches API for latency-tolerant workloads.** Synchronous for blocking checks.

---

## What the Exam Tests Here

The exam does not test whether the candidate can write a GitHub Actions workflow file. It tests whether the candidate understands the architectural reasoning behind each configuration choice.

The -p flag question (Question 10 in the guide) is designed to catch candidates who confuse this with environment variables or stdin redirection. The flag is the documented mechanism. The other options do not exist or do not address the root problem.<sup>[1]</sup>

The session isolation question is designed to catch candidates who think continuity aids review. It does not. A reviewer that shares reasoning context with the generator is not reviewing; it is confirming.<sup>[1]</sup>

The batch API question is designed to catch candidates who apply cost optimization indiscriminately. The 50% savings are real. They do not apply to blocking workflows where a human is waiting for the result.<sup>[1]</sup>

The scenario the exam attaches to this material is a short framing rather than a walkthrough: a team putting Claude Code into a CI/CD pipeline that runs automated reviews, generates test cases, and comments on pull requests, with the stated design problem being prompts that give actionable feedback and produce few false positives. Its second named domain is prompt engineering, not configuration, which is a fair warning about where the questions actually land. Everything in this chapter is the substrate beneath that design problem rather than the solution to it.

Three facts carry most of the weight here, and they are worth holding as a set because a scenario usually turns on exactly one of them:

1. -p for non-interactive mode.
2. Separate sessions for code review (review-session isolation).
3. Message Batches API for latency-tolerant workloads; synchronous for pre-merge checks.

The configuration choices here assume the prompt is precise enough to make review findings actionable. Chapter 9 is where that assumption gets tested.

---

## Key Takeaways

- The -p (or --print) flag is required for CI; without it, the process hangs waiting for interactive input. For the permission baseline, dontAsk runs only pre-approved tools and denies the rest without waiting, which is the documented shape for a locked-down pipeline; bypassPermissions is reserved for genuinely isolated containers and VMs, refuses to start as root outside a sandbox, and can be disabled by managed settings.
- **Bypassing permissions does not switch off the permission system.** Deny rules and explicit ask rules apply in every mode, org-restricted connector tools and interaction-requiring MCP tools still prompt, which in CI means still hang, and a hook deny still holds. Allow rules, including --allowedTools, have no effect under bypass because everything is already approved; --tools is the flag that narrows which tools exist.
- --output-format json combined with --json-schema produces machine-parseable structured output that downstream pipeline steps can consume directly, without fragile text parsing. The schema-conforming payload arrives in structured_output; the prose result arrives in result. The cost figures in the payload are client-side estimates.
- **Runaway cost is capped inside Claude Code, not around it:** --max-turns bounds agentic turns and --max-budget-usd bounds dollars, both with no default and both counting subagent work. A job timeout kills the runner without capping the spend, and a cheaper model changes the price of an unbounded loop rather than bounding it.
- **Replace against append.** The flag --system-prompt replaces the default prompt; --append-system-prompt and --append-system-prompt-file add to it. A CI reviewer that ignores documented standards or stops reading files is usually a replace where an append belonged, not a tool-access problem. For slow startup, --bare skips discovery and --append-system-prompt-file loads exactly what is required; trimming CLAUDE.md addresses neither.
- **Review-session isolation** is the core architectural principle for CI code review: the session that reviews code must be completely separate from the session that generated it. Same-session self-review retains generator reasoning context and produces confirmation bias, and an independent instance beats both a self-review instruction and extended thinking applied inside the same session. When re-running after new commits, include prior findings in context and instruct Claude to report only new or unaddressed issues.
- CLAUDE.md at the repository root provides project context (testing standards, fixture conventions, review criteria) to every CI-invoked session that does not use --bare, and it acts before generation, which is why it fixes low-value generated tests without adding a filtering call. **REVIEW.md is the separate, review-only file read by the managed Code Review service on GitHub**, outranking that service's default guidance, and it is where skip rules, severity recalibration and evidence requirements belong.
- GitHub Actions integration passes CLI arguments via claude_args; the presence of a prompt input is what switches the action from responding to a trigger phrase to running automatically. The API key goes in repository secrets, not workflow files.
- The ***Message Batches API* charges half the standard price**, with no latency guarantee. Most batches finish in under an hour, and the 24 hours is the expiry ceiling rather than the expected wait, so the choice turns on latency tolerance rather than duration. It suits overnight reports and weekly audits, never blocking pre-merge checks, and it does not support multi-turn tool calling. The ***custom_id* field correlates batch requests to their responses**, and it is also how failed items are resubmitted alone. Where an SLA applies, submission interval plus processing window must fit inside it.
