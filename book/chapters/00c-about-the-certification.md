# About the Certification

---

## What the CCAR-F Is

The Claude Certified Architect Foundations certification is Anthropic’s foundational credential for engineers and architects working with Claude at a systems level. It validates the ability to design, configure, and reason about Claude-based systems across the full range of architectural concerns: agentic loops, tool ecosystems, configuration hierarchies, prompting strategies, and context management.

The CCAR-F is not a general AI literacy credential. It is specifically about Claude’s architecture and the trade-offs that appear when you build production systems with it.

---

## The Five Domains

The exam covers five weighted domains:

- **Domain 1, Agentic Architecture**: 27% of the exam. Covers agentic loop termination, multi-agent coordination patterns, hooks and lifecycle management.
- **Domain 2, Tool Design**: 18% of the exam. Covers tool description quality, tool selection mechanics, MCP integration, and the built-in toolkit.
- **Domain 3, Configuration and Customization**: 20% of the exam. Covers the CLAUDE.md hierarchy, slash commands, skills, plan mode, and CI/CD session isolation.
- **Domain 4, Prompting and Output**: 20% of the exam. Covers explicit prompting criteria, structured output contracts, and the validation loop.
- **Domain 5, Context Management and Reliability**: 15% of the exam. Covers context window discipline, the lost-in-the-middle effect, escalation triggers, provenance, and error propagation.

---

## What the Exam Tests

The exam tests architectural reasoning, not recall. Questions are scenario-based. A scenario describes a system with specific constraints, then asks you to identify the correct architectural pattern, diagnose a failure mode, or choose between two plausible configurations and explain which trade-off is acceptable. Memorizing parameter names is not sufficient. Understanding why those parameters exist and what happens when you choose wrong is what the exam is after.

---

## Administration

The CCAR-F is proctored through ProctorFree and administered through Skilljar. Candidates schedule and sit the exam through the Skilljar platform. ProctorFree handles the remote proctoring session.

---

## The Six Recurring Scenarios

The exam draws four of these six scenarios at random for any given sitting. Each scenario recurs across the certification because they represent the canonical architectural problems that Claude-based systems encounter:

1. **Customer Support Resolution Agent**, a multi-tool system with escalation logic
2. **Code Generation with Claude Code**, covering CI/CD integration and session isolation
3. **Multi-Agent Research System**, using hub-and-spoke coordination with provenance requirements
4. **Structured Data Extraction**, focused on structured output and validation
5. **Developer Productivity with Claude**, covering CLAUDE.md hierarchy and permission configuration
6. **Claude Code for Continuous Integration**, testing context window management and tool selection

Familiarity with all six scenarios is practical preparation even though only four appear on any given exam. They recur because they represent the canonical architectural problems that Claude-based systems solve at scale.

---

## Coverage in This Book

This book’s chapter structure maps directly to domain weighting. Three chapters cover Domain 1, two cover Domain 2, three cover Domain 3, two cover Domain 4, and two cover Domain 5. The back matter includes a full coverage map that cross-references exam task statements to specific chapters.
