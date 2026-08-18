# Chapter 5: MCP and the Built-In Toolkit

**Summary:** *The Model Context Protocol gives agents a standard interface for discovering and invoking external tools across system boundaries. Claude Code’s built-in tools encode an opinionated workflow for exploring unfamiliar codebases: **Grep** to find entry points, **Read** to trace flows, **Edit** to make targeted changes, and **Bash** only when no built-in alternative exists. The exam tests both the MCP configuration model (in-code versus .mcp.json, transport types, allowedTools, environment variable expansion) and the selection rules for built-in tools. The most common failure pattern is reaching for Bash when a purpose-built tool already exists. MCP servers are configured via two paths: programmatic mcpServers options for single-agent applications, and .mcp.json at the project root for team-shared tooling committed to version control. Three installation scopes resolve local, then project, then user, and the winning scope supplies the whole server entry. Transport type follows server location: stdio for local processes, HTTP/SSE for remote servers. Credentials belong in environment variables, never hardcoded. The allowedTools contract governs which MCP tools an agent may invoke; permissionMode: "acceptEdits" does not auto-approve MCP tools, and bypassPermissions does without being absolute. The Incremental Exploration pattern for codebase navigation applies Grep first to find entry points, Read selectively to trace flows, and stops when enough context exists rather than loading every file upfront.*

---

## The 18-Tool Problem

Scenario 4 in the exam guide sets an agent to work on developer productivity: making sense of code nobody on the team wrote recently, generating scaffolding, taking over the repetitive parts. The scenario is explicit about how. It works through the built-in tools, and it also connects to MCP servers.<sup>[1]</sup> Both halves of that sentence are testable, and they fail in different ways.

Picture the agent carrying eighteen tools, asked to read a configuration file, answering with a shell command that prints the file rather than with the tool that reads it. Two things have gone wrong at once.

The first is the tool count, and the exam guide states the principle plainly under Task Statement 2.3: an agent given eighteen tools rather than four or five selects less reliably, because each additional tool widens the decision it has to make.<sup>[1]</sup> The fix is distribution, and Chapter 4 owns it.

The second is the wrong tool for the job, and that one is this chapter's. Read exists precisely to read files. **Bash exists for shell operations where no built-in tool covers the need.** Reaching past Read for a shell command that produces the same bytes is not a creative solution. It is a category error, and Task Statement 2.5 is built around not making it: every skills bullet pairs a class of work with the tool that does it.<sup>[2]</sup>

---

## The Standard: Model Context Protocol

MCP is an open standard for connecting AI agents to external tools and data sources.<sup>[3]</sup> Before MCP, every integration was bespoke: custom tool implementations, custom authentication flows, custom naming schemes, custom discovery mechanisms. MCP standardizes all of it.

The practical benefit is a growing ecosystem of community servers: GitHub, Jira, Postgres, Slack, and others. The exam guide states the preference as a selection rule rather than a suggestion: for a standard third-party integration the existing community server is the answer, and a custom server is what you write for a team-specific workflow nothing off the shelf covers.<sup>[1]</sup>

For the exam, MCP matters across four areas: what a server can expose, how you configure it, how its tools get named, and how permissions reach them.

Start with what a server exposes, because it decides how everything else surfaces. MCP offers three kinds of thing, and each arrives through a different door. Tools are model-callable: Claude decides to invoke them, and they act. Resources are fetchable content, referenced the way a file is, with an @ mention shaped like `@server:protocol://resource/path`, and pulled in as an attachment. Prompts are templates the server author wrote, and they appear in the command list as `/mcp__servername__promptname`, taking arguments after the command.<sup>[4]</sup>

The three doors do not swap. A server prompt is not prepended to the system prompt and does not join the tool registry; it becomes something you type. A resource does not become a callable function. Only tools are invoked by the model on its own initiative.

---

## Configuring MCP Servers

Two configuration paths exist: in-code via the mcpServers option on query(), and declaratively via a .mcp.json file at your project root.<sup>[3]</sup>

**In-code configuration** is appropriate when you are building a single agent application and want full programmatic control. You pass the server definition directly in the options object. It does not require any file on disk.

**.mcp.json configuration** is appropriate for team tooling. Place the file at your project root and version-control it. When query() runs with the default settingSources (which includes "project"), the file loads automatically. Any teammate who clones the repo gets the same MCP servers.<sup>[2]</sup>

The scope distinction the exam tests: .mcp.json at the project root is shared with the team. User-level MCP configuration lives in ~/.claude.json and is personal only. Use user-level config for experimental servers you are evaluating or for personal tools that do not belong in the team’s shared setup.<sup>[2]</sup>

The axis that turns on is version control rather than intent: a file at the repository root is shared by definition, because cloning brings it along, and a file in the home directory is not. The live documentation splits the personal half into two separate scopes, and the difference decides which projects a server shows up in.

Claude Code recognises three MCP installation scopes.<sup>[4]</sup> **Local** is the default: the server is written into `~/.claude.json` under the path of the project you added it from, so it loads in that project and nowhere else and stays private to you. **Project** scope is `.mcp.json` at the repository root, one project, shared with everyone who clones it. **User** scope is also `~/.claude.json`, but at the top level rather than nested under a project path, so the server loads in every project on the machine while staying private.

Two of the three therefore live in the same file, and what separates them is not privacy, which they share, but reach. A server you are trying out against one repository is local; a utility you want everywhere is user.

**The precedence order is local, then project, then user.**<sup>[4]</sup> When the same server name appears at more than one scope, Claude Code connects once and takes the definition from the highest-precedence scope that has it. The whole entry wins: fields are not merged, so a local definition that omits an environment variable the project definition sets does not quietly inherit it. Below user scope sit plugin-provided servers and then claude.ai connectors, matched as duplicates by endpoint rather than by name.

There is a vocabulary trap here worth slowing down for. MCP local scope is not the settings system's local scope. An MCP local-scoped server lives in `~/.claude.json` in your home directory; a local settings file is `.claude/settings.local.json` inside the project.<sup>[4]</sup> Same word, different file, different directory. Keep this order sealed off from the others in the book, too: settings and skills each resolve their own way, and the orders do not transfer.

### Transport Types

Three transport mechanisms are available, and selection depends on where the server runs.<sup>[3]</sup>

**stdio** is for local processes. If the server documentation gives you a command to run (something like npx @modelcontextprotocol/server-github), use stdio. The agent and the server communicate via stdin/stdout on the same machine.<sup>[3]</sup>

**HTTP/SSE** is for cloud-hosted servers and remote APIs. If the documentation gives you a URL, this is the family you want. It is two type values rather than one: `"sse"` for a server-sent-events endpoint, and `"http"` for the streamable HTTP transport.<sup>[3]</sup>

**SDK MCP servers** let you define custom tools directly inside your application code, without running a separate server process at all.<sup>[3]</sup>

The alias runs one way, and that is the part to memorize. In `.mcp.json` and other JSON configuration files, `"streamable-http"` is accepted as an alias for `"http"`, because that is the name the protocol specification uses, so a block copied out of a server's documentation loads without editing. The programmatic `mcpServers` option accepts only `"http"`.<sup>[4,3]</sup> So `"http"` is valid in both places and `"streamable-http"` in one. Configuration that works in a file does not automatically survive being pasted into code.

### Authentication Without Committing Secrets

MCP servers that connect to external services need credentials. **The only acceptable pattern is environment variable expansion.**<sup>[2,3]</sup>

In .mcp.json, use ${ENV_VAR} syntax in the env field:

```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["@company/jira-mcp-server"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    }
  }
}
```

The ${JIRA_TOKEN} syntax expands from the runtime environment. The secret stays out of the file. The file stays safe to commit.<sup>[3]</sup>

Hardcoding a literal API key in .mcp.json is the anti-pattern here, and the exam guide names expansion as the mechanism that avoids it: credentials managed without the secret being committed.<sup>[2]</sup> That file is going to version control, because being shared is the whole reason it sits at the repository root. Anything literal inside it ships with the repository.

---

## Tool Naming and the allowedTools Contract

MCP tools follow a deterministic naming pattern: mcp__<server-name>__<tool-name>.<sup>[3]</sup>

A GitHub server named "github" with a list_issues tool becomes mcp__github__list_issues. The pattern is predictable. You can enumerate exact names or use wildcards.

By default, Claude can see that MCP tools are available but cannot call them without explicit permission. Granting access requires the tool name to appear in allowedTools:<sup>[3]</sup>

```
allowedTools: ["mcp__github__list_issues", "mcp__github__create_pr"]
```

Wildcards work for granting access to an entire server:

```
allowedTools: ["mcp__github__*"]
```

This is the preferred approach when you intend the agent to use all tools from a server. The wildcard grants exactly what you specified and nothing more.<sup>[3]</sup>

The wildcard carries an anchoring requirement that is easy to miss and easy to test. In an allow rule a glob is honoured only after a literal `mcp__<server>__` prefix, and the server segment itself has to be glob-free, so the rule always names a server you actually configured. `mcp__github__get_*` is valid. A bare `mcp__*` is not: it is ignored with a startup warning and grants nothing. Deny rules do not share the restriction, and `mcp__*` on the deny side matches every MCP tool on every server.<sup>[5]</sup> A deny that reaches too far fails safe; an allow that reaches too far does not, so the allow side refuses to be written loosely.

### The Permission Mode Trap

This distinction is on the exam. permissionMode: "acceptEdits" does **not** auto-approve MCP tools. What it approves is file edits and the filesystem commands that accompany them, and only for paths inside the working directory or an additional directory you named. An MCP tool still needs its own allowedTools entry.<sup>[3]</sup>

permissionMode: "bypassPermissions" does auto-approve MCP tools, and the documentation's own recommendation is still to prefer allowedTools, because bypass is broader than the problem. A wildcard opens one server. Bypass opens everything.<sup>[3]</sup>

It is also not the blanket its name suggests, and squaring that with the rest of the book matters here. Chapter 1 recorded that bypass leaves standing exceptions rather than overriding everything, and two of them land on MCP. Deny rules are evaluated before the mode is consulted, so a deny that matches an MCP tool blocks it under bypass exactly as under any other mode. And a server can mark an individual tool as requiring a person, at which point that tool prompts even under bypass and even when an allow rule matches it; the purpose of such a tool is usually that a human agreed to something, and auto-approval would mean nobody ever did.<sup>[5]</sup>

That is the same direction Chapter 3 established for hooks: the layers that take permission away run ahead of the layers that hand it out. A hook deny survives bypass. A deny rule survives bypass. A hook allow does not defeat a deny rule underneath it. Loosening is the weak direction all the way down.

One further asymmetry belongs with these: allowedTools is a list of grants, not a ceiling. Naming three tools there does not confine bypass mode to those three, since everything left off matches no allow rule and is approved by the mode instead. Under bypass, the list that blocks is disallowedTools.<sup>[5]</sup> The permission mode values themselves are defined in Chapter 1. The application here is which mode reaches MCP tools, and what still stands in the way once it does.

---

## MCP Tool Search

When you configure many MCP servers, their tool definitions accumulate in every request’s context. A few servers with many tools can consume significant context before the agent does any actual work.<sup>[6]</sup>

MCP tool search addresses this. When it is active, tool definitions are withheld from the context window. The agent gets a summary of what exists, searches when a task calls for a capability it has not loaded, and pulls in up to five of the most relevant tools, which then stay available for subsequent turns. Tool search is enabled by default.<sup>[3,7]</sup>

Context is only half the reason, and the other half loops back to the chapter's opening. Tool selection accuracy degrades once more than thirty to fifty tools are loaded at once, and on-demand loading is what keeps accuracy up across catalogues far larger than that.<sup>[7]</sup> The chapter now carries two numbers about tool counts, and they answer different questions. Thirty to fifty describes the model's whole visible surface at one moment. Four or five is a design rule for how many tools an agent should own, which is Chapter 4's subject. A well-distributed system satisfies both, and neither figure restates the other.

The core built-in tools are exempt: Bash, Read and Edit load upfront and are never deferred, so deferral is about the MCP surface.<sup>[7]</sup>

The full treatment of context cost and ToolSearch as an on-demand loading mechanism belongs to Chapter 11. For this chapter: tool search exists, it is on by default, and it exists because MCP server schemas are expensive to carry at all times and because a large loaded tool set is harder to choose from.

---

## MCP Resources

Beyond tools, MCP servers can expose resources: content catalogs that give the agent visibility into available data without requiring exploratory tool calls.<sup>[2]</sup>

Consider an agent that needs to understand which Jira issues exist before deciding which to fetch. Without resources, the agent calls a search tool speculatively, reads results, calls again with refined parameters. With an MCP resource exposing an issue summary catalog, the agent can inspect the available content and make an informed tool call on the first attempt.<sup>[2]</sup>

Resources reduce the exploratory overhead. For the exam, know that they exist and that their purpose is reducing unnecessary tool calls by giving the agent catalog-level visibility.

Why this beats the alternatives is worth stating as a rule, because the competing fixes attack a different problem than the one causing the symptom. The symptom is that the agent cannot see what exists before it starts calling, so it probes. Folding several servers into one changes how many connections there are; putting a routing layer in front changes which server a call is sent to. Neither adds advance visibility, so neither shortens the probing. A resource does, because it puts the catalogue in front of the agent before the first call rather than reorganizing the calls afterwards.<sup>[2]</sup>

---

## Built-In Tools: The Opinionated Toolkit

Six tools carry this task statement: Read, Write, Edit, Bash, Grep and Glob.<sup>[1]</sup> They are not the entire built-in surface, which also covers web access, subagent and skill invocation, task tracking and tool search itself.<sup>[6]</sup> They are the set the exam asks you to choose among.

The mental model: built-in tools are purpose-built. Each one does exactly one thing well. Bash is the general-purpose fallback for when no purpose-built tool covers the need. Reaching for Bash when a purpose-built tool exists is always the wrong answer on the exam.

There is a faster way to hold the selection rule than six descriptions, and it is to read the symptom instead of the task. One question separates most of the field: do you have a name, or a string that lives inside a file? A name is a path question and goes to Glob; a string appearing somewhere in the code is a content question and goes to Grep.

It helps just as much to know which wrong tool each situation attracts, because the pull is consistent. Asked to locate where a string appears across a repository, the temptation is Glob, because searching feels like finding files; the answer is Grep, because what is being searched is content. Asked to change a file, the temptation is Write; the answer is Edit, because the file is already there and most of it is staying. Asked to change a file where Edit cannot anchor, the temptation is a longer anchor; the answer is Read followed by Write. The wrong option in each pair matches the verb in the request rather than the shape of the data.

### Grep: Content Search

Grep searches file contents for patterns. Use it when you need to find a function name across a codebase, locate error messages, discover all callers of an API, or find import statements.<sup>[2]</sup>

```
Task: "Find all usages of getUserById"
Correct: Grep("getUserById", "src/")
Wrong:   Bash("grep -r 'getUserById' src/")
```

The wrong answer is not a failed attempt. It is a category violation. The built-in Grep tool exists precisely for this, and the exam guide's skills bullet leaves no room: searching code content across a codebase selects Grep.<sup>[2]</sup> Running the shell's own grep through Bash returns the same lines and arrives through the wrong door.

### Glob: Path Pattern Matching

Glob finds files by name or extension patterns. Use it when you need to enumerate all TypeScript test files, find all configuration files in a subtree, or identify files matching a naming convention.<sup>[2]</sup>

**Glob operates on file paths. Grep operates on file contents.** That is the complete decision rule: reach for Glob when the question is which files exist, reach for Grep when the question is what those files contain. Bash("find . -name '*.test.ts'") is the shell equivalent of Glob, and it is the wrong tool to reach for when a purpose-built alternative exists.

### Read: Full File Contents

Read loads the complete contents of a file. Use it when you need to understand an implementation, review a configuration, trace an import, or view any file in its entirety.<sup>[2]</sup>

```
Task: "Read the configuration file"
Correct: Read("config.json")
Wrong:   Bash("cat config.json")
```

This is the failure the chapter opened with, and there is a concrete reason it is worse than a matter of taste. The permission system treats the two calls as different kinds of request. Rules for Read are scoped by file path, so a rule can say which parts of the filesystem the agent may look at; rules for Bash are scoped by command pattern instead.<sup>[5]</sup> A file read routed through Bash reaches the permission layer dressed as a shell command, and the path rules written to govern reading never get a chance to match it. The category error has a security shape, not only an aesthetic one. Using Bash for a job Read was designed to do is the canonical anti-pattern of this chapter.

### Write: New Files from Scratch

Write creates new files. The exam guide divides whole-file operations from targeted ones: Read and Write handle the entire file, Edit handles a span of it.<sup>[2]</sup> Whole-file is the operative phrase, and it is what makes Write dangerous on a file that already exists. Content you left out of the new version is not left alone; it is replaced along with everything else.

Creating a new test file is the canonical case: Write("tests/new-test.ts", content). That is correct. The equivalent shell redirect via Bash is not. The exam distinction to carry: Write is for creation, Edit is for modification. Using Write on an existing file without including all the original content destroys the content you omitted.

### Edit: Targeted Modification

Edit makes targeted changes to existing files using unique text matching as an anchor. You provide the text to find and the replacement text. Edit leaves everything else in the file untouched.<sup>[2]</sup>

Edit and Write address opposite cases. Edit modifies a known span of an existing file; Write replaces the whole thing. Fixing a bug in server.ts calls for Edit("server.ts", old_text, new_text), not Write with the entire file content. Write on an existing file is destructive: anything not included in the new content disappears.

The failure mode for Edit: when the anchor text is not unique in the file, Edit cannot determine which occurrence to replace. The fallback in that case is Read the file into context, then Write the entire corrected content. Read plus Write is the documented recovery path when Edit cannot find unique anchor text.<sup>[2]</sup>

The instinct at that moment is to lengthen the anchor, on the theory that enough surrounding text will eventually be unique. In a genuinely repetitive file the surrounding text repeats too, so a longer string reproduces the same ambiguity at greater length. This is why the guide names a different tool rather than a bigger argument: Read and Write abandon matching and state the finished file explicitly, and an operation that matches nothing cannot match the wrong thing.<sup>[2]</sup>

### Bash: Shell Operations

Bash runs shell commands, scripts, git operations, test suites, and build processes.<sup>[6]</sup> It is named alongside the other five in the task statement but occupies a different slot in the decision: it is what remains once content search, path search, whole-file operations and targeted modification have each claimed their own work.<sup>[1]</sup> Use it when no built-in tool covers the task.

Running tests, executing build scripts, git commits, arbitrary system operations: these belong to Bash because no purpose-built tool handles them. npm test is Bash. git commit is Bash. Neither Read nor Grep nor Edit has anything to say about running a test suite. The Bash selection rule is not “Bash is for things that are hard.” It is “Bash is for things that have no built-in tool.” The moment a built-in tool exists, that tool wins.

---

## Incremental Exploration

This is the named concept for this chapter: **Incremental Exploration**.

A new engineer inherits a large codebase. The naive move is to open every file and read the whole thing until something familiar appears. That is expensive, slow, and usually counterproductive. Experienced engineers do the opposite: *start at the surface, follow the signal, stop when they have enough.*<sup>[2]</sup>

Incremental Exploration is that discipline applied to agentic codebase navigation. The pattern has three moves:

1. **Grep to find entry points.** Search for the function name, the error message, the config key, the class name. Grep surfaces the files and line numbers where the thing you care about actually lives. You do not need to know the file names in advance.
2. **Read to trace the flow.** Load the specific file Grep identified. Understand the implementation. Follow the import chain to adjacent files. Read only what the evidence points to.
3. **Stop when you have enough.** The goal is not to read the whole codebase. It is to answer the question in front of you. Incremental Exploration is a discipline of restraint as much as it is a discovery method.

The exam operationalizes this pattern directly. Task Statement 2.5 describes the skill as “building codebase understanding incrementally: starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront.”<sup>[2]</sup>

Reading all files upfront is not wrong in the way that Bash("cat") is wrong. It is inefficient. It consumes context tokens that do not contribute to the answer. In a large codebase, it fills the context window with noise before the agent has oriented itself.

The anti-pattern has a name: *premature full-read*. You call Read on every file in the directory before you have established which files matter. Incremental Exploration replaces premature full-read with evidence-driven navigation.

### The Trace Pattern

One specific application of Incremental Exploration that the exam tests: tracing function usage across wrapper modules.<sup>[2]</sup>

The scenario: a codebase exports functions through multiple wrapper layers. You need to find every caller of a specific function across the whole repo. The approach:
First, identify all the exported names. Use Grep to find the export statement for the function. Read the module that exports it to understand the interface.

Then, use Grep again to search for each exported name across the codebase. Each hit is a caller. Each caller is a file worth reading in full.

This is the Grep-then-Read trace. It scales. You do not need to know the codebase structure in advance. You follow the evidence.

---

## How MCP and Built-In Tools Interact

A real configuration will have both MCP tools and built-in tools available simultaneously. The model makes selection decisions based on tool descriptions. This creates a specific tension the exam tests: an MCP tool that does the same thing as a built-in tool but with better context for the specific use case.

Simultaneously is meant literally, and it describes a property of connection rather than of routing. Every configured MCP server has its tools discovered when it connects, and from that moment all of them are available at once.<sup>[1]</sup> Nothing selects a server per turn. Configuration determines what exists; the model's read of the descriptions determines what gets called. A design that routes a request to the right server ahead of the model describes a layer the protocol does not have.

The exam guide notes that enhancing MCP tool descriptions to explain capabilities and outputs in detail prevents the agent from preferring built-in tools over more capable MCP tools.<sup>[2]</sup> That applies in reverse too: a built-in tool with a clear mandate (Grep for content search) should win over an MCP tool that also happens to support search, if the MCP tool description does not make its advantage explicit.

Tool selection by the model follows description quality. Vague MCP tool descriptions cede ground to well-specified built-in tools. This is the same principle Chapter 4 establishes for tool descriptions as routing keys. The application here is about cross-type competition: built-in versus MCP, decided by whoever wrote the better description.

---

## Parallel Execution: Read-Only Versus Stateful

One operational detail worth knowing for the exam: built-in tools that are read-only (Read, Glob, Grep) can run concurrently when Claude requests multiple tool calls in a single turn. Tools that modify state (Edit, Write, Bash) run sequentially to avoid conflicts.<sup>[6]</sup>

This matters for Incremental Exploration. When tracing a large codebase, the agent can issue multiple Grep or Read calls in a single turn and receive results for all of them in parallel. Sequential reading is not forced on read-only operations.

The rule reaches MCP tools too: a server that marks a tool read-only puts it in the concurrent group.<sup>[6]</sup> The dividing line is not built-in against external. It is whether the call changes anything.

---

## Configuration Errors and Recovery

MCP servers can fail to connect for several reasons: the server process is not installed, credentials are missing or invalid, a remote server is unreachable.<sup>[3]</sup>

The SDK emits a system message with subtype init at the start of each query. That message includes the connection status for each configured server. Check the status field to detect failures before the agent starts working. A server in “failed” status means its tools are unavailable. If the agent then attempts to call one of those tools, it will receive a rejection rather than a result.<sup>[3]</sup>

Reading that field takes more care than it looks like it should, because only some of its values are bad news. Needs-auth means the server answered and wants credentials, which is a failure for your purposes even though nothing broke. Pending is the one that misleads: a server configured through a settings file, or one whose tool list came from a cache, commonly reports pending at init and connects afterwards. Check for failed and needs-auth, not for the absence of pending, and ask for the statuses again later if you need a settled answer.<sup>[3]</sup>

Common failure causes and their fixes:<sup>[3]</sup>

- Missing environment variables: the env field specifies what the server expects; verify that all referenced variables are set in the runtime environment.
- Server not installed: for npx-based stdio servers, verify the package exists and Node.js is in PATH.
- Invalid connection string: for database servers, verify format and accessibility.
- Network issues: for remote HTTP/SSE servers, verify the URL is reachable and firewalls allow the connection.

Tools that are configured but not called are also worth diagnosing. If Claude can see MCP tools but does not use them, the likely cause is missing allowedTools permission. The tool is visible; the call is blocked. Adding the tool name (or a wildcard) to allowedTools resolves it.<sup>[3]</sup>

---

## Exam Patterns: What Gets Tested

The exam draws heavily on this chapter’s material. Here is how the testable concepts map to question patterns, using original constructions:

**Question type: Built-in tool selection.** An agent receives a task. Which tool should it use? The answer almost always eliminates Bash when a dedicated built-in exists. The decision tree is: does a purpose-built tool cover this? If yes, use it. If no, use Bash.

**Question type: Exploration strategy.** An agent needs to understand an unfamiliar codebase before modifying it. The correct approach starts with Grep (finding entry points) and uses Read selectively (following imports), not with reading every file upfront.

**Question type: MCP configuration.** A team needs to share an MCP server configuration. Which file? .mcp.json at the project root, version-controlled. Personal tools go in ~/.claude.json. Secrets use ${ENV_VAR} expansion, never hardcoded values.

**Question type: MCP scope precedence.** The same server name is configured in more than one place. Which definition connects? Local, then project, then user. One definition is used in full; scopes do not merge fields. Local scope means the project-keyed section of ~/.claude.json, not .claude/settings.local.json.

**Question type: Permission mode.** An agent is configured with permissionMode: "acceptEdits". Can it call MCP tools without additional configuration? No. acceptEdits approves file edits and filesystem Bash commands only. MCP tools require explicit allowedTools entries. The follow-on distinction is that bypassPermissions does reach MCP tools and still is not absolute: deny rules, hooks, and any tool a server has flagged as needing a person all survive it.

**Question type: Write versus Edit.** A task modifies an existing file. Which tool? Edit, because it preserves unchanged content. Write is for new files. Using Write on an existing file without including all the original content destroys the content you omitted.

**Question type: Edit fallback.** Edit cannot find unique anchor text. What is the recovery path? Read the file into context, then Write the complete corrected content.

---

## Key Takeaways

- MCP tools follow the naming pattern mcp__<server-name>__<tool-name>. Include the exact name or a wildcard mcp__<server>__* in allowedTools to grant access. Without this, Claude sees the tool but cannot call it. A glob in an allow rule is honoured only after a literal mcp__<server>__ prefix, so a bare mcp__* grants nothing, though on the deny side it matches everything. permissionMode: "acceptEdits" does not auto-approve MCP tools. bypassPermissions does, and deny rules, hooks, and tools flagged as requiring a person still hold under it.
- Transport selection depends on where the server runs: stdio for local processes (command in documentation), HTTP/SSE for remote servers (URL in documentation), SDK MCP servers for in-process custom tools. The alias runs one way: "streamable-http" stands in for "http" in .mcp.json and other JSON config files, while the programmatic mcpServers option takes only "http".
- Never hardcode credentials in .mcp.json. Use ${ENV_VAR} syntax. The file goes to version control; secrets must not.
- .mcp.json at the project root is team-shared config. ~/.claude.json is personal and holds two of the three MCP scopes: local, keyed by project path, and user, which loads everywhere. Precedence is local, then project, then user, and the winning scope supplies the whole entry rather than merging fields. MCP tool search withholds tool definitions until they are needed and is on by default; selection accuracy falls off past thirty to fifty loaded tools, a different measurement from Chapter 4's four-to-five per agent. The full context-cost treatment belongs to Chapter 11.
- Built-in tool selection rule: use the purpose-built tool; fall back to Bash only when no built-in covers the task. Bash("cat config.json") when Read("config.json") exists is the canonical anti-pattern.
- Grep is for content search (patterns inside files). Glob is for path patterns (finding files by name or extension). Read is for full file contents. Edit is for targeted modification using unique anchor text; when anchor text is not unique, the fallback is Read the file, then Write the corrected complete content. Write is for new files; it replaces entire content. Bash is the fallback when nothing else covers the job.
- Incremental Exploration: Grep to find entry points, Read to trace flows, stop when you have enough. Do not read the full codebase upfront. Follow the evidence.
