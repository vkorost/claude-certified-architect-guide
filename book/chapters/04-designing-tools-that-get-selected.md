# Chapter 4: Designing Tools That Get Selected

**Summary:** *A tool’s description is not documentation for humans. It is a routing key for the model. When you write "Retrieves customer information" as your description, you are not explaining the tool to a developer who might read the code. You are giving the model the only signal it has to decide whether this is the right tool for the current request. If that signal is vague, the model guesses. When the model guesses wrong at scale, you have a reliability problem dressed up as an architecture problem.*

*This chapter introduces the concept of **tool descriptions as routing keys** and explains why getting tool descriptions right is the highest-leverage act in tool design. It covers the full structure of a tool definition, how the API assembles that structure into a system prompt the model reads, and how tool_choice lets you control selection when the model’s autonomy is not what the situation requires. It also explains what happens when agents have too many tools, and why the solution to that problem is not better descriptions: it is better distribution.*

---

## The Bug That Isn’t a Bug

Production logs show that the agent frequently calls the wrong tool. A user types “check my order #12345” and the agent calls get_customer instead of lookup_order. Both tools accept similar identifier formats. Both have minimal descriptions: "Retrieves customer information" and "Retrieves order details". The agent is not broken. It is doing exactly what you told it to do, which is to choose between two tools with descriptions so similar that the choice is essentially random noise.<sup>[1]</sup>

This is Sample Question 2 from the exam guide, and it is worth dwelling on before any technical discussion. The question asks for the most effective first step to improve selection reliability. Option A proposes adding few-shot examples to the system prompt, showing order-related queries routing to lookup_order. Option C proposes a routing layer that parses user input before each turn and pre-selects a tool based on detected keywords. Option D proposes consolidating both tools into a single lookup_entity tool that handles both cases internally.

The correct answer is B: expand each tool’s description to include input formats, example queries, edge cases, and boundaries explaining when to use it versus similar tools.<sup>[1]</sup>

Option A adds token overhead without fixing the root cause. Option C is over-engineered and bypasses the model’s natural language understanding with a brittle keyword-matching layer that needs to be maintained in parallel with the tools themselves. Option D has merit in some situations, but consolidating tools is a structural decision with tradeoffs, not a first step when the actual problem is a description that is three words long. Option B directly addresses why the routing failed. The descriptions gave the model nothing to work with.

That is the thesis of this chapter. And it is also the named concept: **tool descriptions as routing keys**. The model selects tools based on description semantics, not on the tool’s actual implementation. A tool that does something sophisticated but has a vague description will be misrouted. A tool that does something simple but has a precise, boundary-rich description will be selected correctly almost every time. Tool design is prompt engineering applied to the tool layer, and the exam tests whether you understand that.

---

## What a Tool Definition Actually Is

When you pass a tools array to the Claude API, the API does not hand those definitions to the model as raw JSON for it to interpret. It constructs a special system prompt that includes the tool definitions, formatting instructions, and any user-specified system prompt.<sup>[2]</sup>

The constructed prompt starts with something like this:

```
In this environment you have access to a set of tools you can use to answer the user's question.
{{ FORMATTING INSTRUCTIONS }}
Here are the functions available in JSONSchema format:
{{ TOOL DEFINITIONS IN JSON SCHEMA }}
{{ USER SYSTEM PROMPT }}
{{ TOOL CONFIGURATION }}
```

The model reads this before it reads anything you typed into the user message. Every tool definition you wrote is sitting in that constructed prompt in JSON Schema format, and the model’s entire decision about which tool to call flows from reading that. Your description field is the prose inside that JSON block. It is the only natural language the model has to reason about why this tool exists and when it should be called.

Each tool definition includes four top-level parameters.<sup>[2]</sup>

**name** must match the regex ^[a-zA-Z0-9_-]{1,64}$. No spaces, no dots, no unicode. Maximum 64 characters. The name is also a signal. lookup_order and get_customer are already doing more routing work than search and fetch would do. Names that encode intent reduce the load the description has to carry.

**description** is plaintext explaining what the tool does, when it should be used, and how it behaves. This is the routing key. Everything else about the definition exists to constrain or clarify the mechanics; the description is the semantic trigger that determines whether the model reaches for this tool or a different one.

**input_schema** is a JSON Schema object defining the tool’s parameters. The schema tells the model what to send when it calls the tool. Well-written property descriptions inside the schema are a secondary routing signal. If a parameter description says “Customer email address (must contain @)” or “Phone in E.164 format, e.g., +15551234567,” the model gets format constraints at the point of use.

**input_examples** is an optional field that accepts an array of example input objects. Each example must be valid according to the tool’s input_schema: invalid examples return a 400 error. Examples are included in the prompt alongside your schema, showing the model concrete patterns for well-formed tool calls. This is especially useful for tools with nested objects, optional parameters, or format-sensitive inputs where the schema alone does not make the expected structure obvious.<sup>[2]</sup>

Here is what the contrast looks like in practice. A weak description:

```json
{
  "name": "lookup_order",
  "description": "Retrieves order details",
  "input_schema": {
    "type": "object",
    "properties": {
      "id": { "type": "string" }
    }
  }
}
```

A routing key:

```json
{
  "name": "lookup_order",
  "description": "Look up a specific order by order ID, order number, or tracking number. Use this tool when the user references an order, shipment, or package — not when they are asking about their account, profile, or billing. Returns order status, line items, shipping address, and estimated delivery date. Input: exactly one of order_id (format ORD-XXXXXXXX), order_number (any numeric string), or tracking_number. Returns an order object or empty array if not found. Empty result means the order was not found, not a system error. Do NOT use this tool to look up customer account information — use get_customer for that.",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": {
        "type": "string",
        "description": "Order ID in format ORD-XXXXXXXX"
      },
      "order_number": {
        "type": "string",
        "description": "Numeric order number as a string"
      },
      "tracking_number": {
        "type": "string",
        "description": "Shipping carrier tracking number"
      }
    }
  }
}
```

The second version specifies what the tool returns, what input formats it accepts, when to use it, and explicitly when not to use it. That last part matters. The phrase “Do NOT use this tool to look up customer account information” is not redundant with having a separate get_customer tool. The model needs to see the boundary stated from both sides: what lookup_order does, and what it explicitly does not cover. If only get_customer says “use this for customer information,” the model may still be uncertain whether an order lookup is also a kind of customer information retrieval. State the boundary twice, once in each tool.<sup>[1,3]</sup>

---

## The Four Modes of Tool Selection

Once you have tools defined, you control how the model uses them with the tool_choice parameter. There are four values.<sup>[2]</sup>

**"auto"** tells the model it may call any provided tool or may respond in natural language. This is the default when tools are provided. Use it when the user’s request might not require a tool call, and a conversational response is acceptable.

**"any"** tells the model it must use one of the provided tools but does not force a particular one. The model cannot return conversational text. The API prefills the assistant message to force a tool call, which means the model will not emit natural language before the tool_use block, even if explicitly asked to do so.<sup>[2]</sup> Use "any" when the task is inherently a tool-calling task and a text response would be a failure mode. Structured extraction workflows are the canonical use case: the system needs JSON, not a paragraph.

**{"type": "tool", "name": "..."} (forced tool)** tells the model it must call one specific named tool. Like "any", this prefills the assistant message, so natural language preamble is suppressed. Use forced tool selection when you need a specific operation to happen first. The exam guide gives the pattern of forcing extract_metadata before enrichment tools run: the metadata extraction step is a prerequisite, so you force it with tool_choice in the first turn, then process enrichment in follow-up turns with the metadata result in hand.<sup>[1]</sup>

**"none"** prevents tool use entirely. This is the default when no tools are provided. You can explicitly set it when you want the model to reason and respond without any tool calls, even though tools are available in the definition list. Useful in multi-turn flows where you want a reasoning step before a tool call step.

There is one important constraint to know for the exam. When using extended thinking, "any" and forced tool selection are not supported. Extended thinking is compatible only with "auto" and "none".<sup>[2]</sup> This is a gotcha that trips candidates who design thinking-enabled agents and try to guarantee tool calls at the same time.

---

## When Descriptions Are Not Enough: The Overlap Problem

Good descriptions address the scenario where each tool has a distinct purpose but the description fails to communicate it. A different failure mode arises when two tools have descriptions that are genuinely close because the tools themselves do similar things. analyze_content and analyze_document with near-identical descriptions are the exam guide’s example.<sup>[1]</sup>

In this situation, improving descriptions helps but cannot fully solve the problem. If both tools legitimately operate on overlapping input types and produce overlapping outputs, the model is going to have low confidence in the distinction no matter how carefully you word each description. The tools themselves are the problem.

There are two fixes.

**Rename and eliminate functional overlap.** If analyze_content and analyze_document differ only in what they are called but do the same thing, merge them into one well-named tool. The exam guide frames this as renaming tools and updating descriptions to eliminate functional overlap.<sup>[1]</sup> Fewer tools with clearer functions is almost always better than more tools with carefully managed overlap.

**Split a generic tool into purpose-specific tools with defined contracts.** If a single analyze_document tool is doing three different things (extracting data points, summarizing content, and verifying claims), split it into extract_data_points, summarize_content, and verify_claim_against_source. Each now has a description that is unambiguous because it describes only one thing. The exam guide gives this pattern explicitly as a skill.<sup>[1]</sup>

The split approach is the more general solution for overloaded tools. A tool that does several things has a description that has to describe several things, which produces a description that is long and hedged and still not precise. A tool that does one thing can have a description that is short, specific, and unambiguous.

---

## System Prompt Keyword Sensitivity

This is the failure mode that surprises architects who have done everything else right. You write careful, boundary-rich descriptions for every tool. You test routing on a set of representative inputs. Routing looks clean. Then a category of request starts misrouting and you cannot figure out why, because the descriptions are correct.

The problem is in the system prompt, not the tool descriptions. The API constructs the system prompt from tool definitions plus any user-specified system prompt. If your system prompt contains a phrase like “always prioritize account verification” or “when in doubt, check the customer record first,” those instructions create keyword associations that can override tool description logic. The word “customer” in a customer-related query matches the keyword association in the system prompt before the model even reads the tool description carefully.<sup>[1]</sup>

This is system prompt keyword sensitivity. The fix is to review the full constructed prompt, not just the tool definitions, looking for instructions that might create unintended tool associations. Phrases that read as reasonable business logic to a human (“verify the customer before proceeding”) can act as hidden routing overrides to the model. Rephrase them in terms of workflow sequencing rather than entity-type associations: “call get_customer before calling process_refund” is much safer than “always check customer information first” if you have three different tools that touch customer data.

There is a secondary version of this problem: a system prompt instruction intended to shape tone or behavior can accidentally pattern-match to a tool use trigger. An instruction like “when users ask about account status, be comprehensive” might associate “account status” with get_customer in a way that pulls order-related queries toward the customer tool. Auditing system prompts for accidental keyword-to-tool associations is not optional in production systems; it is part of the tool design process.<sup>[1]</sup>

---

## The 18-Tool Problem

The exam includes a scenario where a developer productivity agent has 18 tools and is selecting the wrong ones. The instinct is to write better descriptions. The correct solution is different.

The documented optimal number of tools per agent is 4-5. At 18 tools, the decision complexity in the constructed system prompt is high enough that selection reliability degrades.<sup>[3]</sup> This is not a framing problem or a description quality problem. It is a structural problem. The model is being asked to pick from 18 options for every request, and the cognitive load of navigating 18 tool descriptions is simply too high to maintain reliable selection across diverse input types.

The solution: distribute tools across specialized subagents.<sup>[1,3]</sup>

Instead of one agent with 18 tools, a coordinator agent with 3-4 delegation tools spawns subagents with 4-5 focused tools each. The coordinator’s job is to route to the right subagent. Each subagent’s job is to pick from a small, coherent set of tools. The coordinator does not need to know which individual tool to call; it needs to know which subagent covers this class of request. The subagent does not need to worry about the full tool surface; it sees only the tools relevant to its domain.

This is the coordinator-plus-subagents pattern covered in Chapter 2. The relevant point here is that 4-5 tools per agent is a tool design constraint, not just an architecture preference. It determines whether tool selection is reliable. If you are debugging routing errors and the agent has more than 5 tools, tool count is your primary suspect, and description quality is secondary.

```
// Overloaded: one agent, 18 tools, selection degrades
{
  "tools": [
    "lookup_customer", "update_account", "verify_identity",
    "find_order", "process_refund", "update_shipping",
    "track_package", "send_email", "send_sms",
    "create_ticket", "escalate", "search_kb",
    "check_inventory", "apply_coupon", "schedule_callback",
    "log_interaction", "generate_report", "update_preferences"
  ]
}

// Correct: coordinator routes to specialized subagents
// coordinator: ["Task", "summarize_results", "format_response"]
// customer_agent: ["lookup_customer", "update_account", "verify_identity", "check_status"]
// order_agent: ["find_order", "process_refund", "update_shipping", "track_package"]
// comms_agent: ["send_email", "send_sms", "create_ticket", "escalate_human"]
```

Note that the coordinator’s tool list is small and semantic: spawn a subagent, summarize results, format a response. The coordinator never has to reason about whether to call track_package or update_shipping. That distinction lives in the order_agent, which has four tools and can reliably distinguish between them.<sup>[3]</sup>

---

## Writing a Routing Key: The Mechanics

Given everything above, here is what a complete routing-key description includes.

**What the tool does in one sentence.** This is the disambiguation anchor. It needs to be different in substance, not just word choice, from every other tool in the agent’s tool set. If two tools have descriptions whose first sentence could apply to either, the descriptions are not working.

**What the tool returns.** Output type shapes selection. “Returns an order object with line items, shipping address, and delivery estimate” tells the model something about what using this tool produces. If the next step in the workflow needs an order ID, the tool that returns one gets routed to.

**What input formats are expected.** Format constraints in the description reduce invalid parameter generation and also help routing when the user’s request contains an identifier. “Order ID in format ORD-XXXXXXXX” and “phone in E.164 format” let the model match the user’s stated identifier to the tool that handles it.

**Explicit when-not-to-use boundaries.** State the boundary conditions from this tool’s perspective. What does this tool not handle that a similar tool does? If you have lookup_order and get_customer in the same agent, each description should explicitly disclaim the other tool’s domain.

**What a successful empty result means.** This is subtle but consistently tested. A tool that returns an empty array when no matches are found should say so: “Returns empty array if no match found. Empty result means not found, not a system error.” Without this, the model may treat a not-found result as ambiguous and retry the wrong tool.<sup>[3]</sup>

The input_examples field is an additional channel for this information. For tools with complex nested inputs or optional parameters with unclear interplay, examples show the model what a well-formed call looks like. An example with only the required parameters shows that optional ones are genuinely optional. An example with a specific format for a phone number shows the format more concretely than a prose description can.<sup>[2]</sup>

---

## What the Exam Tests

The exam’s Sample Question 2 is the clearest signal about what this domain tests.<sup>[1]</sup> The correct answer is expanding descriptions, and the explanation is explicit: descriptions are the primary mechanism, and minimal descriptions leave the model without context to differentiate similar tools. The distractors are all recognizable patterns (few-shot prompting, routing layers, tool consolidation) that have legitimate uses elsewhere but are over-engineered or misdirected as first responses to a routing failure caused by bad descriptions.

The exam also tests tool_choice selection reasoning. The pattern to internalize: "auto" when the model needs discretion; "any" when a tool must be called but the specific tool is the model’s choice; forced tool when a specific operation must happen in a specific order; "none" when tool calls need to be suppressed even though tools are defined.

The 18-tool scenario from Scenario 4 (Developer Productivity) tests distribution over description. The exam’s correct answer is reducing to 4-5 tools per agent and distributing the rest to specialized subagents. Making descriptions longer, fine-tuning the model, or switching to a larger model are the distractors. They are wrong because the problem is structural, not descriptive.<sup>[3]</sup>

System prompt keyword sensitivity is a less obvious test point but it appears in Task Statement 2.1. Candidates who understand tool design only as “write good descriptions” will miss questions that require reviewing the system prompt for keyword-sensitive instructions that override those descriptions. The full audit scope is the constructed prompt, not just the tool definitions.

Model selection is part of the tool count equation. The Claude Opus model handles complex tool selection better and will seek clarification when a request is ambiguous across multiple tools. Claude Haiku models work for straightforward tool sets but may infer missing parameters rather than asking for them.<sup>[2]</sup> Know which lever to reach for first.

---

## Key Takeaways

- Every tool definition the model sees is translated into a fragment of the system prompt. The description field is the sentence that tells the model when to reach for this tool. Treat it accordingly.
- A minimal description like "Retrieves order details" is not a valid description. A valid description specifies what the tool returns, what input it expects, what makes this tool distinct from similar tools, and when not to use it.
- tool_choice has four values: "auto" (model decides; default when tools are provided), "any" (model must call a tool but picks which one), {"type": "tool", "name": "..."} (forces a specific tool), and "none" (prevents tool use; default when no tools are provided). Knowing which value to reach for in which situation is a tested exam skill.
- The input_examples field on a tool definition gives the model concrete patterns for valid inputs alongside the schema. Use it for tools with complex, nested, or format-sensitive parameters.
- 4-5 tools per agent is the documented optimal. Giving a single agent 18 tools degrades selection reliability. The solution is not a more elaborate description strategy; it is distributing tools across specialized subagents.
- System prompt keyword sensitivity is a real failure mode. A well-written tool description can still be overridden by a keyword-sensitive instruction elsewhere in the system prompt. Reviewing the full prompt for unintended tool associations is part of the audit.
