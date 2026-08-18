# Coverage Map

The exam blueprint has five weighted domains containing thirty task statements between them. Items are written against those task statements, so the blueprint is the only reliable map of what can be asked.

This map lists all thirty and points each at the chapter that owns it. The task statements are paraphrased here rather than reproduced, and the paraphrase is deliberately close enough to be recognisable and loose enough not to reprint the guide. Read the guide's own wording for the authoritative version; it is the document the items are written against, and it is worth reading end to end at least once.

One structural note before the table. The chapter order follows the reading argument of the book rather than the blueprint order, so two domains are not contiguous. Task statement 2.2, the MCP error contract, sits in Chapter 12 alongside the rest of the error propagation material, because the contract and its consequences are one subject. Task statements 4.5 and 4.6, batch processing and review architectures, sit in Chapter 8 with the CI material for the same reason. Follow the pointers rather than the chapter numbers when drilling a domain.

---

## Domain 1: Agentic Architecture and Orchestration, 27 percent

The heaviest domain. Chapters 1, 2 and 3.

| Task statement | In short | Chapter |
|---|---|---|
| 1.1 | Building agentic loops that run autonomously, and terminating them on the right signal | 1 |
| 1.2 | Coordinating multi-agent systems with a coordinator and subagents | 2 |
| 1.3 | Invoking subagents, passing context to them, and controlling how they spawn | 2 |
| 1.4 | Multi-step workflows with enforced ordering and handoff between stages | 3, with the handoff half in 12 |
| 1.5 | Using SDK hooks to intercept tool calls and normalize data | 3 |
| 1.6 | Choosing a decomposition strategy for a complex piece of work | 2 |
| 1.7 | Managing session state: resuming, continuing and forking | 2 |

---

## Domain 2: Tool Design and MCP Integration, 18 percent

Chapters 4 and 5, plus the error contract in 12.

| Task statement | In short | Chapter |
|---|---|---|
| 2.1 | Writing tool interfaces whose descriptions and boundaries route correctly | 4 |
| 2.2 | Returning structured errors from MCP tools | 12 |
| 2.3 | Distributing tools across agents, and configuring tool choice | 4 |
| 2.4 | Integrating MCP servers into Claude Code and agent workflows | 5 |
| 2.5 | Choosing among the built-in tools for a given job | 5 |

---

## Domain 3: Claude Code Configuration and Workflows, 20 percent

Chapters 6, 7 and 8. The record identifies this as the domain most worth drilling.

| Task statement | In short | Chapter |
|---|---|---|
| 3.1 | Placing and organising CLAUDE.md files across the scopes | 6 |
| 3.2 | Creating custom slash commands and skills | 7 |
| 3.3 | Scoping rules to file paths for conditional loading | 6 |
| 3.4 | Deciding between plan mode and direct execution | 7 |
| 3.5 | Refining iteratively toward a working result | 7 |
| 3.6 | Running Claude Code inside a CI/CD pipeline | 8 |

---

## Domain 4: Prompt Engineering and Structured Output, 20 percent

Chapters 9 and 10, plus batch and review architectures in 8.

| Task statement | In short | Chapter |
|---|---|---|
| 4.1 | Explicit criteria that raise precision and cut false positives | 9 |
| 4.2 | Few-shot examples that make output consistent | 9 |
| 4.3 | Enforcing structured output with tool use and JSON schemas | 10 |
| 4.4 | Validation, retry and feedback loops for extraction quality | 10 |
| 4.5 | Batch processing efficiently | 8 |
| 4.6 | Multi-instance and multi-pass review architectures | 8 |

---

## Domain 5: Context Management and Reliability, 15 percent

Chapters 11 and 12. The lightest domain by weight, and the one whose failures are hardest to see in a scenario.

| Task statement | In short | Chapter |
|---|---|---|
| 5.1 | Keeping critical information alive across a long conversation | 11 |
| 5.2 | Escalation and ambiguity resolution patterns | 12 |
| 5.3 | Propagating errors across a multi-agent system | 12 |
| 5.4 | Managing context while exploring a large codebase | 11 |
| 5.5 | Human review workflows and confidence calibration | 12 |
| 5.6 | Preserving provenance and handling uncertainty when synthesising sources | 12 |

---

## Using this map

Weights describe how many items a domain contributes, not how hard those items are. Domain 5 is the lightest at 15 percent and carries the discriminations that are easiest to get wrong under time pressure, because its failure modes look like other domains' failure modes: a context problem presents as a prompting problem, and an error propagation problem presents as a tool problem.

If you are drilling by weakness rather than by weight, the productive order is to find the task statements you cannot state the discriminating rule for, then read the chapter that owns them. A task statement you can paraphrase but not apply is not yet covered.
