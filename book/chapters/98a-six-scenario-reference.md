# Six Scenario Reference

The six exam scenarios are not isolated vignettes. They are recurring architectural contexts that appear across multiple questions and multiple domains. Each one tests a different cluster of trade-offs. Knowing the scenarios before you sit the exam means you are not spending cognitive load on “what kind of system is this” when the question is actually “which configuration choice is correct here.”

---

## Scenario 1: Customer Support Agent

**Primary domains:** Domain 1 (Agentic Architecture), Domain 5 (Context Management and Reliability)

**What it tests:** Multi-tool orchestration, escalation logic, and the decision boundary between autonomous action and human handoff. This scenario focuses on when an agent should stop and route rather than continue acting. Tool selection under ambiguous user intent. Managing session state across a support conversation.

**Chapters:** 1, 11, 12

---

## Scenario 2: Code Review Pipeline

**Primary domains:** Domain 3 (Configuration and Customization), Domain 1 (Agentic Architecture)

**What it tests:** CI/CD integration patterns, session isolation using the –bare flag and -p flag, and the risks of persistent state across review sessions. How CLAUDE.md configuration interacts with automated pipeline runs. Preventing context bleed between independent review jobs.

**Chapters:** 3, 6, 8

---

## Scenario 3: Multi-Agent Research System

**Primary domains:** Domain 1 (Agentic Architecture), Domain 5 (Context Management and Reliability)

**What it tests:** Hub-and-spoke coordination, coordinator-to-subagent delegation, and the provenance problem: knowing which subagent produced which output and whether that output can be trusted. Handling subagent failure without corrupting the coordinator’s result. Context partitioning across a multi-agent graph.

**Chapters:** 2, 11, 12

---

## Scenario 4: Document Processing Pipeline

**Primary domains:** Domain 4 (Prompting and Output), Domain 2 (Tool Design)

**What it tests:** Structured output contracts using tool_use as a JSON schema enforcement mechanism. The validation loop pattern: generating output, validating against schema, retrying on failure. Designing tool descriptions that produce consistent extraction results across varied document formats.

**Chapters:** 4, 9, 10

---

## Scenario 5: Enterprise Deployment

**Primary domains:** Domain 3 (Configuration and Customization), Domain 2 (Tool Design)

**What it tests:** The three-layer CLAUDE.md hierarchy: global, project, and folder-level configuration. Permission inheritance and override semantics. How slash commands and skills interact with organizational policy. The permissionMode settings and when bypassPermissions is appropriate versus dangerous. MCP server scoping in an enterprise context.

**Chapters:** 5, 6, 7

---

## Scenario 6: Monitoring Dashboard Agent

**Primary domains:** Domain 5 (Context Management and Reliability), Domain 2 (Tool Design)

**What it tests:** Context window discipline in a long-running agent that continuously ingests telemetry. The lost-in-the-middle effect and its practical consequences for alert detection. Tool selection strategy when many tools are available and the agent must not hallucinate tool calls. Knowing when to summarize, when to truncate, and when to escalate because the context is too degraded to act reliably.

**Chapters:** 4, 11, 12
