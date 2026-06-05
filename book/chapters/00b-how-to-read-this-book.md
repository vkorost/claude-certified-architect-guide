# How to Read This Book

---

## Structure Follows Domain Weighting

The CCA-F exam has five domains. They are not equally weighted. Agentic Architecture is 27% of the exam. That means three chapters cover it. Context Management and Reliability is 15%. That means two chapters cover it. The chapter order reflects those priorities, not a logical tour of the API.

The domain-to-chapter mapping:

- Domain 1, Agentic Architecture (27%): Chapters 1, 2, 3
- Domain 2, Tool Design (18%): Chapters 4, 5
- Domain 3, Configuration and Customization (20%): Chapters 6, 7, 8
- Domain 4, Prompting and Output (20%): Chapters 9, 10
- Domain 5, Context Management and Reliability (15%): Chapters 11, 12

---

## Each Chapter Is Self-Contained

Each chapter opens with an executive summary. Read the summary, decide if you need the detail, proceed accordingly. The summaries are not teasers. They are compressed statements of the chapter’s architectural claims. A reader who only reads the summaries will have a weaker mental model but will not have been misled.

Chapters are designed to stand alone as reference material after a first linear read. Cross-references appear where a concept depends on something introduced elsewhere, but the chapter does not require you to have the other one open.

---

## Navigation Aids

Every chapter has Key Takeaways at the bottom. These are the points most likely to surface in exam questions, stated plainly. They are not summaries of what was just covered. They are the architectural facts you need to carry forward.

Most chapters include sample questions. These are not full exam simulations. They are structured to produce the kind of reasoning the exam rewards: scenario plus constraint plus trade-off, not “*which API parameter does X.*”

Endnotes are collected in Appendix G, not inline. The text flags them by number. If you are reading for exam prep, you can ignore the endnotes entirely on first pass. If you want the source reference for a specific claim, Appendix G has it.

---

## Code Examples

Code examples appear in Python and TypeScript. Both languages are represented across the chapters, roughly alternating. The examples are architectural illustrations, not production-ready snippets. They prioritize clarity about the pattern over completeness of error handling.

---

## Suggested Reading Order

Read it linearly the first time. The concepts compound. A coordinator pattern in Chapter 2 makes more sense after the agentic loop in Chapter 1. The CLAUDE.md hierarchy in Chapter 6 is easier to reason about after you have seen hooks in Chapter 3.

After the first pass, use it as a reference. The executive summaries and Key Takeaways support that mode. So does the coverage map in the back matter, which maps exam task statements to specific chapters.

The glossary is at the back. If a term appears in the text and you are not sure of its precise meaning, that is where to look first.
