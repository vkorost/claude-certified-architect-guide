# About the Certification

## What the CCAR-F Is

The Claude Certified Architect Foundations certification is Anthropic's foundational credential for engineers and architects working with Claude at a systems level. It validates the ability to design, configure, and reason about Claude-based systems across the full range of architectural concerns: agentic loops, tool ecosystems, configuration hierarchies, prompting strategies, and context management.

The CCAR-F is not a general AI literacy credential. It is specifically about Claude's architecture and the trade-offs that appear when you build production systems with it.

The exam code is CCAR-F. Material circulating under the code CCA-F predates the current program and should be treated with suspicion on that basis alone.

## The Exam at a Glance

| | |
|---|---|
| Items | 60 |
| Time | 120 minutes |
| Structure | 4 scenarios drawn from a bank of 6 |
| Item formats | Multiple-choice and multiple-response; every item states how many responses to select |
| Delivery | Proctored, online or at a test center per program policy, administered through Pearson VUE |
| Scoring | Scaled score from 100 to 1,000 |
| Passing score | 720 |
| Fee | $125 USD per attempt |
| Validity | 12 months from the date the credential is awarded |
| Score report | Pass or fail with the scaled score, plus percent correct by domain |

Two consequences of that table are worth drawing out.

**The score is scaled, so you cannot work backwards from it.** A scaled score is not a percentage and does not convert into one. The per-domain percentages on the score report are diagnostic: they tell you where you were weak, and they do not compose into the scaled score. Use them to direct study, not to reconstruct how many items you missed.

**Sixty items in a hundred and twenty minutes is two minutes each**, and the scenario-based items take longer to read than they take to answer. The time pressure in this exam falls on comprehension rather than recall, which is why the discriminating rules matter more than the fact lists. If you have to derive a distinction during the exam that you could have memorised beforehand, you have spent your margin.

If you do not pass, the waiting period is 14 days after a first failure, 30 after a second, and 90 after a third, with up to four attempts in a rolling twelve-month period. Those limits apply per exam.

## The Multiple-Response Format

Some items ask for more than one response, and each such item states how many. This is worth a paragraph because it is the one format issue that costs points for reasons unrelated to knowledge.

Read the response count before you read the options. An item asking for two correct answers is a different question from one asking for the single best answer, and the option set is built differently: a two-response item generally contains two defensible answers and two that are wrong, rather than one clearly best answer among three weaker ones. Reading the options first and looking for the best one will make a two-response item feel ambiguous, because you are looking for a distinction the item is not making.

The published guide does not state whether partial credit applies to a multiple-response item. Treat the stated count as binding: select exactly that many, no more and no fewer.

## The Five Domains

| Domain | Weight |
|---|---|
| 1. Agentic Architecture and Orchestration | 27% |
| 2. Tool Design and MCP Integration | 18% |
| 3. Claude Code Configuration and Workflows | 20% |
| 4. Prompt Engineering and Structured Output | 20% |
| 5. Context Management and Reliability | 15% |

Those five domains contain thirty task statements between them, and items are written against the task statements rather than against the domain titles. The coverage map in the back matter lists all thirty with the chapter that owns each.

Weights describe how many items a domain contributes, not how difficult those items are. Domain 1 is the heaviest at 27 percent. Domain 5 is the lightest at 15 percent and contains several of the distinctions most likely to be answered incorrectly under time pressure, because its failure modes disguise themselves as other domains' failure modes.

## What the Exam Tests

The exam tests architectural reasoning, not recall. Questions are scenario-based. A scenario describes a system with specific constraints, then asks you to identify the correct architectural pattern, diagnose a failure mode, or choose between two plausible configurations and explain which trade-off is acceptable. Memorizing parameter names is not sufficient. Understanding why those parameters exist and what happens when you choose wrong is what the exam is after.

The characteristic wrong answer is not nonsense. It is the right intervention aimed at the wrong failure, which is why several chapters in this book name the tempting fix before naming the correct one.

## The Six Recurring Scenarios

The exam draws four of these six scenarios for any given sitting, with six items each:

1. **Customer Support Resolution Agent**
2. **Code Generation with Claude Code**
3. **Multi-Agent Research System**
4. **Developer Productivity with Claude**
5. **Claude Code for Continuous Integration**
6. **Structured Data Extraction**

Because only four appear on a form, and because forms vary between sittings, the same underlying content shows up in different scenario clothing. Tool routing and workflow enforcement can sit inside a support scenario on one sitting and inside an extraction scenario on the next. Prepare against the domain objectives rather than against the scenarios, and use the scenarios to recognise the context quickly rather than to predict the content.

The published guide gives each scenario a short framing paragraph and its primary domains. It does not publish scenario walkthroughs. The back matter includes a reference to all six with the guide's framing and a pointer to the chapters covering each one's trade-offs.

## Coverage in This Book

The chapter structure follows the domains, with three chapters on Domain 1, two on Domain 2 plus the MCP error contract in Chapter 12, three on Domain 3, two on Domain 4 plus batch and review architectures in Chapter 8, and two on Domain 5. The coverage map in the back matter cross-references every task statement to the chapter that owns it.
