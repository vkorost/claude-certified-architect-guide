# Six Scenario Reference

Six scenarios exist. Four of them are served on any given sitting, six items each, so a form does not cover all six and you cannot predict which four you will get. Prepare for all of them.

A scenario is a recurring architectural context rather than a self-contained vignette. Each one frames several items across more than one domain, which is why the same underlying content can appear in different scenario clothing between sittings: the tool routing and workflow enforcement material that sits inside a support block on one form can appear inside an extraction block on another. Study to the domain objectives, not to the scenario skins.

**What the exam guide actually provides for each scenario is a short framing paragraph and a list of its primary domains.** That is the whole of it. There are no published walkthroughs, no worked examples, and no per-scenario fact lists. Anything of that kind circulating elsewhere is third-party commentary and does not carry the guide's authority.

Below, the system description and the primary domains are the guide's. The chapter pointers are this book's mapping from those domains onto its own contents.

## Scenario 1: Customer Support Resolution Agent

**Primary domains:** 1 (Agentic Architecture and Orchestration), 2 (Tool Design and MCP Integration), 5 (Context Management and Reliability)

**The system:** a support resolution agent built on the Agent SDK, handling high-ambiguity work such as returns, billing disputes and account problems, reaching backend systems through custom MCP tools for customer lookup, order lookup, refunds and human escalation. The stated target is a high first-contact resolution rate combined with knowing when to escalate.

**Where its trade-offs live in this book:** loop termination and permission modes (Chapter 1), deterministic enforcement of business thresholds before a tool runs (Chapter 3), tool boundaries and routing between overlapping lookups (Chapters 4 and 5), preserving transactional facts across a long conversation (Chapter 11), and valid escalation triggers, structured handoff and the empty-result-versus-failure distinction (Chapter 12).

## Scenario 2: Code Generation with Claude Code

**Primary domains:** 3 (Claude Code Configuration and Workflows), 5 (Context Management and Reliability)

**The system:** Claude Code used across a team for generation, refactoring, debugging and documentation, integrated into the development workflow with custom slash commands and CLAUDE.md configuration, including the decision of when to use plan mode rather than direct execution.

**Where its trade-offs live in this book:** the configuration hierarchy and which layer owns which instruction (Chapter 6), commands, skills, plan mode and iterative refinement (Chapter 7), session isolation and reproducibility (Chapter 8), and context discipline across long sessions (Chapter 11).

## Scenario 3: Multi-Agent Research System

**Primary domains:** 1 (Agentic Architecture and Orchestration), 2 (Tool Design and MCP Integration), 5 (Context Management and Reliability)

**The system:** a coordinator delegating to specialized subagents, one searching the web, one analyzing documents, one synthesizing findings and one generating reports, producing comprehensive cited reports.

**Where its trade-offs live in this book:** coordinator topology, context passing, decomposition and session state (Chapter 2), distributing tools across specialized agents (Chapter 4), context isolation as a budget mechanism (Chapter 11), and error propagation and provenance through synthesis (Chapter 12). The requirement that reports be cited is the provenance requirement stated as a product goal.

## Scenario 4: Developer Productivity with Claude

**Primary domains:** 2 (Tool Design and MCP Integration), 3 (Claude Code Configuration and Workflows), 1 (Agentic Architecture and Orchestration)

**The system:** productivity tooling on the Agent SDK aimed at engineers meeting a codebase they do not know. It navigates unfamiliar and legacy code, produces scaffolding, and takes over repetitive chores, working through the built-in tools with MCP servers attached.

**Where its trade-offs live in this book:** built-in tool selection and the reflex to reach for Bash when a purpose-built tool exists, plus MCP configuration and scoping (Chapter 5), tool set size and distribution (Chapter 4), configuration placement and path-scoped rules (Chapters 6 and 7), delegation for exploration (Chapter 2), and context management in large codebase exploration (Chapter 11).

## Scenario 5: Claude Code for Continuous Integration

**Primary domains:** 3 (Claude Code Configuration and Workflows), 4 (Prompt Engineering and Structured Output)

**The system:** Claude Code inside a CI/CD pipeline running automated reviews, generating test cases and giving feedback on pull requests, with the stated design problem being feedback that is actionable and low in false positives.

**Where its trade-offs live in this book:** non-interactive invocation, reproducibility, review-session isolation, multi-pass review and cost control (Chapter 8), explicit criteria and the false-positive problem (Chapter 9), and machine-parseable output for downstream automation (Chapter 10).

Note that this scenario's second domain is prompt engineering rather than tool design. The false-positive problem is the prompting half of the scenario, and it is where the items go when they are not about pipeline configuration.

## Scenario 6: Structured Data Extraction

**Primary domains:** 4 (Prompt Engineering and Structured Output), 5 (Context Management and Reliability)

**The system:** an extraction system pulling information out of unstructured documents, validating output against JSON schemas, maintaining high accuracy, handling edge cases gracefully and integrating with downstream systems.

**Where its trade-offs live in this book:** schema enforcement, the validation loop, retry-versus-escalate triage and self-correction fields (Chapter 10), explicit criteria and few-shot for ambiguous cases (Chapter 9), and confidence calibration, sampling, per-segment accuracy and provenance (Chapter 12). Edge cases handled gracefully is the empty-result-versus-failure distinction wearing different clothes.
