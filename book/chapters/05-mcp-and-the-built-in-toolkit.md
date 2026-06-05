# Chapter 5: MCP and the Built-In Toolkit

**Summary:** *The Model Context Protocol gives agents a standard interface for discovering and invoking external tools across system boundaries. Claude Code’s built-in tools encode an opinionated workflow for exploring unfamiliar codebases: **Grep** to find entry points, **Read** to trace flows, **Edit** to make targeted changes, and **Bash** only when no built-in alternative exists. The exam tests both the MCP configuration model (in-code versus .mcp.json, transport types, allowedTools, environment variable expansion) and the selection rules for built-in tools. The most common failure pattern is reaching for Bash when a purpose-built tool already exists. MCP servers are configured via two paths: programmatic mcpServers options for single-agent applications, and .mcp.json at the project root for team-shared tooling committed to version control. Transport type follows server location: stdio for local processes, HTTP/SSE for remote servers. Credentials belong in environment variables, never hardcoded. The allowedTools contract governs which MCP tools an agent may invoke, and permissionMode: "acceptEdits" does not auto-approve MCP tools. The Incremental Exploration pattern for codebase navigation applies Grep first to find entry points, Read selectively to trace flows, and stops when enough context exists rather than loading every file upfront.*

---

## The 18-Tool Problem

Scenario 4 from the exam guide opens like this: an agent built for developer productivity has 18 tools and is asked to read a configuration file. The agent calls Bash("cat config.json") instead of Read("config.json"). Two things went wrong simultaneously, and both are testable.<sup>[1]</sup>

The first problem is the 18-tool count. As Chapter 4 established, selection reliability degrades when an agent carries too many tools. The fix is distribution: reduce to 4-5 tools per agent and push the rest to specialized subagents. That is a topic Chapter 4 owns.

The second problem is the wrong tool for the job. Read exists precisely to read files. **Bash exists for shell operations where no built-in tool covers the need.** Calling Bash("cat config.json") is not a creative solution. It is a category error. The exam treats it as a hard anti-pattern, and this chapter is where that principle lives.<sup>[1]</sup>

---

## The Standard: Model Context Protocol

MCP is an open standard for connecting AI agents to external tools and data sources.<sup>[2]</sup> Before MCP, every integration was bespoke: custom tool implementations, custom authentication flows, custom naming schemes, custom discovery mechanisms. MCP standardizes all of it.

The practical benefit is a growing ecosystem of community servers: GitHub, Jira, Postgres, Slack, and others. When a standard integration exists, you use the community server. Custom MCP servers are for team-specific workflows that the community has not covered.<sup>[3]</sup>

For the exam, MCP matters across three areas: how you configure servers, how tools get named, and how permissions work.

---

## Configuring MCP Servers

Two configuration paths exist: in-code via the mcpServers option on query(), and declaratively via a .mcp.json file at your project root.<sup>[2]</sup>

**In-code configuration** is appropriate when you are building a single agent application and want full programmatic control. You pass the server definition directly in the options object. It does not require any file on disk.

**.mcp.json configuration** is appropriate for team tooling. Place the file at your project root and version-control it. When query() runs with the default settingSources (which includes "project"), the file loads automatically. Any teammate who clones the repo gets the same MCP servers.<sup>[3,4]</sup>

The scope distinction the exam tests: .mcp.json at the project root is shared with the team. User-level MCP configuration lives in ~/.claude.json and is personal only. Use user-level config for experimental servers you are evaluating or for personal tools that do not belong in the team’s shared setup.<sup>[3,4]</sup>

### Transport Types

Three transport mechanisms are available, and selection depends on where the server runs.<sup>[2]</sup>

**stdio** is for local processes. If the server documentation gives you a command to run (something like npx @modelcontextprotocol/server-github), use stdio. The agent and the server communicate via stdin/stdout on the same machine.<sup>[2]</sup>

**HTTP/SSE** is for cloud-hosted servers and remote APIs. If the documentation gives you a URL, use HTTP. In programmatic code, the transport type is "http". In .mcp.json and other JSON config files, "streamable-http" is accepted as an alias.<sup>[2]</sup>

**SDK MCP servers** let you define custom tools directly inside your application code, without running a separate server process at all.<sup>[2]</sup>

The alias matters for the exam: "streamable-http" works in .mcp.json; "http" is what the programmatic mcpServers option accepts. They are not interchangeable across contexts.

### Authentication Without Committing Secrets

MCP servers that connect to external services need credentials. **The only acceptable pattern is environment variable expansion.**<sup>[</sup><sup>2,3]</sup>

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

The ${JIRA_TOKEN} syntax expands from the runtime environment. The secret stays out of the file. The file stays safe to commit.<sup>[2]</sup>

Hardcoding a literal API key in .mcp.json is the anti-pattern the exam tests. That file will be committed to version control. The key will be exposed. Do not do it.<sup>[3]</sup>

---

## Tool Naming and the allowedTools Contract

MCP tools follow a deterministic naming pattern: mcp__<server-name>__<tool-name>.<sup>[2]</sup>

A GitHub server named "github" with a list_issues tool becomes mcp__github__list_issues. The pattern is predictable. You can enumerate exact names or use wildcards.

By default, Claude can see that MCP tools are available but cannot call them without explicit permission. Granting access requires the tool name to appear in allowedTools:<sup>[2]</sup>

```
allowedTools: ["mcp__github__list_issues", "mcp__github__create_pr"]
```

Wildcards work for granting access to an entire server:

```
allowedTools: ["mcp__github__*"]
```

This is the preferred approach when you intend the agent to use all tools from a server. The wildcard grants exactly what you specified and nothing more.<sup>[2]</sup>

### The Permission Mode Trap

This distinction is on the exam. permissionMode: "acceptEdits" does **not** auto-approve MCP tools. MCP tools remain ungated unless you include them explicitly in allowedTools.<sup>[2]</sup>

permissionMode: "bypassPermissions" does auto-approve MCP tools because it disables all permission prompts system-wide. That is broader than necessary for most use cases. A wildcard in allowedTools gives you MCP access with surgical precision. The permission mode values themselves are defined in Chapter 1; the exam application here is understanding which mode handles MCP and which does not.<sup>[2]</sup>

---

## MCP Tool Search

When you configure many MCP servers, their tool definitions accumulate in every request’s context. A few servers with many tools can consume significant context before the agent does any actual work.<sup>[5]</sup>

MCP tool search addresses this. When enabled, tool definitions are withheld from context. The agent loads only the definitions it needs for each turn. Tool search is enabled by default.<sup>[2]</sup>

The full treatment of context cost and ToolSearch as an on-demand loading mechanism belongs to Chapter 11. For this chapter: tool search exists, it is on by default, and it exists specifically because MCP server schemas are expensive to carry at all times.

---

## MCP Resources

Beyond tools, MCP servers can expose resources: content catalogs that give the agent visibility into available data without requiring exploratory tool calls.<sup>[4]</sup>

Consider an agent that needs to understand which Jira issues exist before deciding which to fetch. Without resources, the agent calls a search tool speculatively, reads results, calls again with refined parameters. With an MCP resource exposing an issue summary catalog, the agent can inspect the available content and make an informed tool call on the first attempt.<sup>[4]</sup>

Resources reduce the exploratory overhead. For the exam, know that they exist and that their purpose is reducing unnecessary tool calls by giving the agent catalog-level visibility.

---

## Built-In Tools: The Opinionated Toolkit

Claude Code ships with six built-in tools. Knowing which to reach for, and when, is directly tested in Scenario 4 and across multiple Domain 2 questions.<sup>[1,3]</sup>

The mental model: built-in tools are purpose-built. Each one does exactly one thing well. Bash is the general-purpose fallback for when no purpose-built tool covers the need. Reaching for Bash when a purpose-built tool exists is always the wrong answer on the exam.

### Grep: Content Search

Grep searches file contents for patterns. Use it when you need to find a function name across a codebase, locate error messages, discover all callers of an API, or find import statements.<sup>[3,4]</sup>

```
Task: "Find all usages of getUserById"
Correct: Grep("getUserById", "src/")
Wrong:   Bash("grep -r 'getUserById' src/")
```

The wrong answer is not a failed attempt. It is a category violation. The built-in Grep tool exists precisely for this. Invoking Bash to run the shell grep command when Grep is available is the anti-pattern the exam tests repeatedly.<sup>[3]</sup>

### Glob: Path Pattern Matching

Glob finds files by name or extension patterns. Use it when you need to enumerate all TypeScript test files, find all configuration files in a subtree, or identify files matching a naming convention.<sup>[3,4]</sup>

**Glob operates on file paths. Grep operates on file contents.** That is the complete decision rule: reach for Glob when the question is which files exist, reach for Grep when the question is what those files contain. Bash("find . -name '*.test.ts'") is the shell equivalent of Glob, and it is the wrong tool to reach for when a purpose-built alternative exists.

### Read: Full File Contents

Read loads the complete contents of a file. Use it when you need to understand an implementation, review a configuration, trace an import, or view any file in its entirety.<sup>[3,4]</sup>

```
Task: "Read the configuration file"
Correct: Read("config.json")
Wrong:   Bash("cat config.json")
```

This is the specific failure from Scenario 4. cat is a shell command. Read is a purpose-built file reading tool with proper permission handling, content formatting, and context integration. Using Bash for a job Read was designed to do is the canonical anti-pattern of this chapter.<sup>[1]</sup>

### Write: New Files from Scratch

Write creates new files. It replaces entire file content, which makes it dangerous on existing files. If the file already exists and you use Write, anything you did not include in the new content disappears.<sup>[3]</sup>

Creating a new test file is the canonical case: Write("tests/new-test.ts", content). That is correct. The equivalent shell redirect via Bash is not. The exam distinction to carry: Write is for creation, Edit is for modification. Using Write on an existing file without including all the original content destroys the content you omitted.

### Edit: Targeted Modification

Edit makes targeted changes to existing files using unique text matching as an anchor. You provide the text to find and the replacement text. Edit leaves everything else in the file untouched.<sup>[3,4]</sup>

Edit and Write address opposite cases. Edit modifies a known span of an existing file; Write replaces the whole thing. Fixing a bug in server.ts calls for Edit("server.ts", old_text, new_text), not Write with the entire file content. Write on an existing file is destructive: anything not included in the new content disappears.

The failure mode for Edit: when the anchor text is not unique in the file, Edit cannot determine which occurrence to replace. The fallback in that case is Read the file into context, then Write the entire corrected content. Read plus Write is the documented recovery path when Edit cannot find unique anchor text.<sup>[4]</sup>

### Bash: Shell Operations

Bash runs shell commands, scripts, git operations, test suites, and build processes. Use it when no built-in tool covers the task.<sup>[3]</sup>

Running tests, executing build scripts, git commits, arbitrary system operations: these belong to Bash because no purpose-built tool handles them. npm test is Bash. git commit is Bash. Neither Read nor Grep nor Edit has anything to say about running a test suite. The Bash selection rule is not “Bash is for things that are hard.” It is “Bash is for things that have no built-in tool.” The moment a built-in tool exists, that tool wins.

---

## Incremental Exploration

This is the named concept for this chapter: **Incremental Exploration**.

A new engineer inherits a large codebase. The naive move is to open every file and read the whole thing until something familiar appears. That is expensive, slow, and usually counterproductive. Experienced engineers do the opposite: *start at the surface, follow the signal, stop when they have enough.*<sup>[4]</sup>

Incremental Exploration is that discipline applied to agentic codebase navigation. The pattern has three moves:

1. **Grep to find entry points.** Search for the function name, the error message, the config key, the class name. Grep surfaces the files and line numbers where the thing you care about actually lives. You do not need to know the file names in advance.
2. **Read to trace the flow.** Load the specific file Grep identified. Understand the implementation. Follow the import chain to adjacent files. Read only what the evidence points to.
3. **Stop when you have enough.** The goal is not to read the whole codebase. It is to answer the question in front of you. Incremental Exploration is a discipline of restraint as much as it is a discovery method.

The exam operationalizes this pattern directly. Task Statement 2.5 describes the skill as “building codebase understanding incrementally: starting with Grep to find entry points, then using Read to follow imports and trace flows, rather than reading all files upfront.”<sup>[4]</sup>

Reading all files upfront is not wrong in the way that Bash("cat") is wrong. It is inefficient. It consumes context tokens that do not contribute to the answer. In a large codebase, it fills the context window with noise before the agent has oriented itself.

The anti-pattern has a name: *premature full-read*. You call Read on every file in the directory before you have established which files matter. Incremental Exploration replaces premature full-read with evidence-driven navigation.

### The Trace Pattern

One specific application of Incremental Exploration that the exam tests: tracing function usage across wrapper modules.<sup>[4]</sup>

The scenario: a codebase exports functions through multiple wrapper layers. You need to find every caller of a specific function across the whole repo. The approach:
First, identify all the exported names. Use Grep to find the export statement for the function. Read the module that exports it to understand the interface.

Then, use Grep again to search for each exported name across the codebase. Each hit is a caller. Each caller is a file worth reading in full.

This is the Grep-then-Read trace. It scales. You do not need to know the codebase structure in advance. You follow the evidence.

---

## How MCP and Built-In Tools Interact

A real configuration will have both MCP tools and built-in tools available simultaneously. The model makes selection decisions based on tool descriptions. This creates a specific tension the exam tests: an MCP tool that does the same thing as a built-in tool but with better context for the specific use case.

The exam guide notes that enhancing MCP tool descriptions to explain capabilities and outputs in detail prevents the agent from preferring built-in tools over more capable MCP tools.<sup>[4]</sup> That applies in reverse too: a built-in tool with a clear mandate (Grep for content search) should win over an MCP tool that also happens to support search, if the MCP tool description does not make its advantage explicit.

Tool selection by the model follows description quality. Vague MCP tool descriptions cede ground to well-specified built-in tools. This is the same principle Chapter 4 establishes for tool descriptions as routing keys. The application here is about cross-type competition: built-in versus MCP, decided by whoever wrote the better description.

---

## Parallel Execution: Read-Only Versus Stateful

One operational detail worth knowing for the exam: built-in tools that are read-only (Read, Glob, Grep) can run concurrently when Claude requests multiple tool calls in a single turn. Tools that modify state (Edit, Write, Bash) run sequentially to avoid conflicts.<sup>[5]</sup>

This matters for Incremental Exploration. When tracing a large codebase, the agent can issue multiple Grep or Read calls in a single turn and receive results for all of them in parallel. Sequential reading is not forced on read-only operations.

---

## Configuration Errors and Recovery

MCP servers can fail to connect for several reasons: the server process is not installed, credentials are missing or invalid, a remote server is unreachable.<sup>[2]</sup>

The SDK emits a system message with subtype init at the start of each query. That message includes the connection status for each configured server. Check the status field to detect failures before the agent starts working. A server in “failed” status means its tools are unavailable. If the agent then attempts to call one of those tools, it will receive a rejection rather than a result.<sup>[2]</sup>

Common failure causes and their fixes:<sup>[2]</sup>

- Missing environment variables: the env field specifies what the server expects; verify that all referenced variables are set in the runtime environment.
- Server not installed: for npx-based stdio servers, verify the package exists and Node.js is in PATH.
- Invalid connection string: for database servers, verify format and accessibility.
- Network issues: for remote HTTP/SSE servers, verify the URL is reachable and firewalls allow the connection.

Tools that are configured but not called are also worth diagnosing. If Claude can see MCP tools but does not use them, the likely cause is missing allowedTools permission. The tool is visible; the call is blocked. Adding the tool name (or a wildcard) to allowedTools resolves it.<sup>[2]</sup>

---

## Exam Patterns: What Gets Tested

The exam draws heavily on this chapter’s material. Here is how the testable concepts map to question patterns, using original constructions:

**Question type: Built-in tool selection.** An agent receives a task. Which tool should it use? The answer almost always eliminates Bash when a dedicated built-in exists. The decision tree is: does a purpose-built tool cover this? If yes, use it. If no, use Bash.

**Question type: Exploration strategy.** An agent needs to understand an unfamiliar codebase before modifying it. The correct approach starts with Grep (finding entry points) and uses Read selectively (following imports), not with reading every file upfront.

**Question type: MCP configuration.** A team needs to share an MCP server configuration. Which file? .mcp.json at the project root, version-controlled. Personal tools go in ~/.claude.json. Secrets use ${ENV_VAR} expansion, never hardcoded values.

**Question type: Permission mode.** An agent is configured with permissionMode: "acceptEdits". Can it call MCP tools without additional configuration? No. acceptEdits approves file edits and filesystem Bash commands only. MCP tools require explicit allowedTools entries.

**Question type: Write versus Edit.** A task modifies an existing file. Which tool? Edit, because it preserves unchanged content. Write is for new files. Using Write on an existing file without including all the original content destroys the content you omitted.

**Question type: Edit fallback.** Edit cannot find unique anchor text. What is the recovery path? Read the file into context, then Write the complete corrected content.

---

## Key Takeaways

- MCP tools follow the naming pattern mcp__<server-name>__<tool-name>. Include the exact name or a wildcard mcp__<server>__* in allowedTools to grant access. Without this, Claude sees the tool but cannot call it. permissionMode: "acceptEdits" does not auto-approve MCP tools; use allowedTools for precise MCP access control.
- Transport selection depends on where the server runs: stdio for local processes (command in documentation), HTTP/SSE for remote servers (URL in documentation), SDK MCP servers for in-process custom tools. In .mcp.json, "streamable-http" is accepted as an alias for "http".
- Never hardcode credentials in .mcp.json. Use ${ENV_VAR} syntax. The file goes to version control; secrets must not.
- .mcp.json at the project root is team-shared config. ~/.claude.json is personal. MCP tool search withholds tool definitions from context until they are needed and is on by default; the full context-cost treatment belongs to Chapter 11.
- Built-in tool selection rule: use the purpose-built tool; fall back to Bash only when no built-in covers the task. Bash("cat config.json") when Read("config.json") exists is the canonical anti-pattern.
- Grep is for content search (patterns inside files). Glob is for path patterns (finding files by name or extension). Read is for full file contents. Edit is for targeted modification using unique anchor text; when anchor text is not unique, the fallback is Read the file, then Write the corrected complete content. Write is for new files; it replaces entire content. Bash is the fallback when nothing else covers the job.
- Incremental Exploration: Grep to find entry points, Read to trace flows, stop when you have enough. Do not read the full codebase upfront. Follow the evidence.
