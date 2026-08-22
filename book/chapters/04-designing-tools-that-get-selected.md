# Chapter 4: Designing Tools That Get Selected

**Summary:** *A tool’s description is not documentation for humans. It is a routing key for the model. When you write "Retrieves customer information" as your description, you are not explaining the tool to a developer who might read the code. You are giving the model the only signal it has to decide whether this is the right tool for the current request. If that signal is vague, the model guesses. When the model guesses wrong at scale, you have a reliability problem dressed up as an architecture problem.*

*This chapter introduces the concept of **tool descriptions as routing keys** and explains why getting tool descriptions right is the highest-leverage act in tool design. It covers the full structure of a tool definition, how the API assembles that structure into a system prompt the model reads, and how tool_choice lets you control selection when the model’s autonomy is not what the situation requires. It also explains what happens when agents have too many tools, and why the solution to that problem is not better descriptions: it is better distribution.*

## The Bug That Isn’t a Bug

Production logs show that the agent frequently calls the wrong tool. A user types “check my order #12345” and the agent calls get_customer instead of lookup_order. Both tools accept similar identifier formats. Both have minimal descriptions: "Retrieves customer information" and "Retrieves order details". The agent is not broken. It is doing exactly what you told it to do, which is to choose between two tools with descriptions so similar that the choice is essentially random noise.<sup>[1]</sup>

This is Sample Question 2 from the exam guide, and it is worth dwelling on before any technical discussion. The question asks for the most effective first step to improve selection reliability. Option A proposes adding few-shot examples to the system prompt, showing order-related queries routing to lookup_order. Option C proposes a routing layer that parses user input before each turn and pre-selects a tool based on detected keywords. Option D proposes consolidating both tools into a single lookup_entity tool that handles both cases internally.

The correct answer is B: expand each tool’s description to include input formats, example queries, edge cases, and boundaries explaining when to use it versus similar tools.<sup>[1]</sup>

Option A adds token overhead without fixing the root cause. Option C is over-engineered and bypasses the model’s natural language understanding with a brittle keyword-matching layer that needs to be maintained in parallel with the tools themselves. Option D has merit in some situations, but consolidating tools is a structural decision with tradeoffs, not a first step when the actual problem is a description that is three words long. Option B directly addresses why the routing failed. The descriptions gave the model nothing to work with.

That is the thesis of this chapter. And it is also the named concept: **tool descriptions as routing keys**. The model selects tools based on description semantics, not on the tool’s actual implementation. A tool that does something sophisticated but has a vague description will be misrouted. A tool that does something simple but has a precise, boundary-rich description will be selected correctly almost every time. Tool design is prompt engineering applied to the tool layer, and the exam tests whether you understand that.

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

Notice the order of the blocks. The tool definitions sit above your own system prompt, and the tool configuration below it. To the model that assembly is one continuous piece of text, so a sentence you wrote for some unrelated purpose sits in the same block as the schemas and is read alongside them. That has consequences, and they get their own section later in this chapter.

Each tool definition carries four core fields.<sup>[2]</sup>

**name** must match the regex ^[a-zA-Z0-9_-]{1,64}$. No spaces, no dots, no unicode. Maximum 64 characters. The name is also a signal. lookup_order and get_customer are already doing more routing work than search and fetch would do. Names that encode intent reduce the load the description has to carry. Once a tool surface spans several services, the documented practice is to namespace names by service, so that github_list_prs and slack_send_message are separable by prefix before any description is read.<sup>[2]</sup>

**description** is plaintext explaining what the tool does, when it should be used, and how it behaves. This is the routing key. Everything else about the definition exists to constrain or clarify the mechanics; the description is the semantic trigger that determines whether the model reaches for this tool or a different one.

**input_schema** is a JSON Schema object defining the tool’s parameters. The schema tells the model what to send when it calls the tool. Well-written property descriptions inside the schema are a secondary routing signal. If a parameter description says “Customer email address (must contain @)” or “Phone in E.164 format, e.g., +15551234567,” the model gets format constraints at the point of use.

**input_examples** is an optional field that accepts an array of example input objects. Each example must be valid according to the tool’s input_schema: invalid examples return a 400 error. Examples are included in the prompt alongside your schema, showing the model concrete patterns for well-formed tool calls. This is especially useful for tools with nested objects, optional parameters, or format-sensitive inputs where the schema alone does not make the expected structure obvious.<sup>[2]</sup>

Three limits on the field are worth carrying. It works on tools you define and on tools using an Anthropic-published schema, but not on server tools such as web search or code execution. Each example costs prompt tokens on every request. And the documented ordering puts the description first: examples are the supplement for a tool whose inputs are complicated, not a substitute for describing the tool properly.<sup>[2]</sup>

Those four fields carry the routing argument, but they are not the whole definition. A tool definition also accepts optional properties that change how it is handled rather than what it means, among them cache_control, strict, defer_loading and allowed_callers.<sup>[2]</sup> Two matter later in this chapter. strict constrains the model’s generated arguments to conform to your schema. defer_loading keeps a definition out of the context window until it is needed, which is the mechanism behind tool search and the reason a large tool catalogue does not have to be a large system prompt.

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
  "description": "Look up a specific order by order ID, order number, or tracking number. Use this tool when the user references an order, shipment, or package, not when they are asking about their account, profile, or billing. Returns order status, line items, shipping address, and estimated delivery date. Input: exactly one of order_id (format ORD-XXXXXXXX), order_number (any numeric string), or tracking_number. Returns an order object or empty array if not found. Empty result means the order was not found, not a system error. Do NOT use this tool to look up customer account information; use get_customer for that.",
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

The second version specifies what the tool returns, what input formats it accepts, when to use it, and explicitly when not to use it. That last part matters. The phrase “Do NOT use this tool to look up customer account information” is not redundant with having a separate get_customer tool. The model needs to see the boundary stated from both sides: what lookup_order does, and what it explicitly does not cover. If only get_customer says “use this for customer information,” the model may still be uncertain whether an order lookup is also a kind of customer information retrieval. State the boundary twice, once in each tool. Both sources point the same way. The exam guide asks that a description say when to use the tool against its similar alternatives, a statement each tool has to make for itself; the API documentation asks that it cover when the tool should not be used and name what it does not return.<sup>[1,2]</sup>

## The Four Modes of Tool Selection

Once you have tools defined, you control how the model uses them with the tool_choice parameter. There are four values.<sup>[2]</sup>

**"auto"** tells the model it may call any provided tool or may respond in natural language. This is the default when tools are provided. Use it when the user’s request might not require a tool call, and a conversational response is acceptable.

**"any"** tells the model it must use one of the provided tools but does not force a particular one. The model cannot return conversational text. The API prefills the assistant message to force a tool call, which means the model will not emit natural language before the tool_use block, even if explicitly asked to do so.<sup>[2]</sup> Use "any" when the task is inherently a tool-calling task and a text response would be a failure mode. Structured extraction workflows are the canonical use case: the system needs JSON, not a paragraph.

**{"type": "tool", "name": "..."} (forced tool)** tells the model it must call one specific named tool. Like "any", this prefills the assistant message, so natural language preamble is suppressed. Use forced tool selection when you need a specific operation to happen first. The exam guide gives the pattern of forcing extract_metadata before enrichment tools run: the metadata extraction step is a prerequisite, so you force it with tool_choice in the first turn, then process enrichment in follow-up turns with the metadata result in hand.<sup>[1]</sup>

**"none"** prevents tool use entirely. This is the default when no tools are provided. You can explicitly set it when you want the model to reason and respond without any tool calls, even though tools are available in the definition list. Useful in multi-turn flows where you want a reasoning step before a tool call step.

Two of those four guarantee that a tool call happens, and only one guarantees which. That is the whole of the "any" against named-force distinction, and it resolves by asking what would count as a failure. A conversational reply arriving where a parser expects structure is prevented by any tool call, so "any" suffices. The wrong tool running, or running in the wrong order, is not: "any" constrains the model to the set it was already choosing from. Only the named force narrows the set to one.

A note on the interaction with thinking, because it has moved. Under manual extended thinking, switched on explicitly in the request, "any" and named forcing are not supported and return an error; only "auto" and "none" are compatible. Adaptive thinking is the different case, and it includes models where thinking is on by default: forced tool use is supported there.<sup>[2]</sup> The older rule, that thinking and guaranteed tool calls are simply incompatible, is the one candidates tend to carry, and it is now too broad. The constraint attaches to how thinking was enabled, not to its presence.

One more mechanism closes a gap the four modes leave open. Forcing a tool call is not the same as forcing a well-formed one: "any" guarantees the model calls something and says nothing about whether the arguments match your schema. Setting strict on the tool definitions constrains generated arguments to conform, and pairing that with "any" is the documented way to guarantee both halves at once.<sup>[2]</sup> A routing guarantee and a payload guarantee are separate requests, and a design that needs both has to make both.

All of that describes what tool_choice does. What it does not do is the more useful half, and it is where this part of the domain's most natural wrong reach lives. An agent is calling the wrong tool. tool_choice is right there, it takes a tool name, and naming the correct tool looks like it solves the problem.

tool_choice operates on a single request and constrains whether a tool is called and which one. It teaches the model nothing: it does not alter how a description is read, does not change which tool fits which request, and does not carry forward to the next turn. Applied to a routing failure, a named force does not correct the routing, it replaces the routing with a constant. On the request where the forced tool is right, the agent looks fixed. On every request where a different tool was correct, the agent now calls the wrong tool with certainty rather than with probability, and the logs stop telling you which requests those were.

The distinction that survives contact with a real system is between a selection problem and an ordering problem. A selection problem is the model choosing badly among tools that are all legitimately available, and it is answered in the definitions. An ordering problem is different in kind: one tool's output is an input the next tool cannot function without, and the sequence has to hold regardless of what the model infers about it. Nothing in a description makes a sequence hold, because a description is read at selection time and a sequence is a property of the conversation. That is the case tool_choice exists for.

The pattern is narrow enough to state exactly. Force the prerequisite on the turn where nothing has satisfied it yet, then release to "auto" for the turns that follow.<sup>[1]</sup> The release is the part that gets dropped. Forcing the prerequisite on every turn is a coherent-looking configuration that guarantees the prerequisite runs and then guarantees nothing else ever does: the dependent tools are permanently excluded by the same setting that satisfied their dependency. The constraint belongs to the turn where the dependency is unmet, not to the pipeline as a standing rule.

If instead a step must never precede its prerequisite anywhere in a session, a per-request parameter is the wrong altitude and the durable answer is to make the second tool unavailable until the first has run. Chapter 3 covers that enforcement layer.

## When Descriptions Are Not Enough: The Overlap Problem

Good descriptions address the scenario where each tool has a distinct purpose but the description fails to communicate it. A different failure mode arises when two tools have descriptions that are genuinely close because the tools themselves do similar things. analyze_content and analyze_document with near-identical descriptions are the exam guide’s example.<sup>[1]</sup>

In this situation, improving descriptions helps but cannot fully solve the problem. If both tools legitimately operate on overlapping input types and produce overlapping outputs, the model is going to have low confidence in the distinction no matter how carefully you word each description. The tools themselves are the problem.

There are two fixes.

**Rename and eliminate functional overlap.** If analyze_content and analyze_document differ only in what they are called but do the same thing, merge them into one well-named tool. The exam guide frames this as renaming tools and updating descriptions to eliminate functional overlap.<sup>[1]</sup> Fewer tools with clearer functions is almost always better than more tools with carefully managed overlap.

**Split a generic tool into purpose-specific tools with defined contracts.** If a single analyze_document tool is doing three different things (extracting data points, summarizing content, and verifying claims), split it into extract_data_points, summarize_content, and verify_claim_against_source. Each now has a description that is unambiguous because it describes only one thing. The exam guide gives this pattern explicitly as a skill.<sup>[1]</sup>

The split approach is the more general solution for overloaded tools. A tool that does several things has a description that has to describe several things, which produces a description that is long and hedged and still not precise. A tool that does one thing can have a description that is short, specific, and unambiguous.

Splitting and merging point in opposite directions, so it is worth saying which failure each answers. Split when one tool carries several contracts, because no single description states several output contracts precisely and the model has to guess which it is invoking. Merge when several tools carry one contract, because the descriptions then differ only in wording and the model is asked to detect a distinction that does not exist. The documented form of the merge is one more capable tool with an action parameter naming the operation rather than a tool per action: fewer and broader tools reduce the ambiguity of the selection itself.<sup>[2]</sup> The two moves are not in tension; they answer opposite conditions.

One wrong reach is specific to multi-agent designs, and it is worth naming before the next section makes distribution sound like a universal remedy. When two tools overlap, the reflex is to put them in different agents so that neither agent sees both. That works only if the two genuinely belong to different specializations. If they serve the same job, they land in the same specialized agent and the ambiguous pair is intact inside it. Distribution moved the tools without removing the overlap, because the overlap lives in the definitions. Only consolidation removes it.

## System Prompt Keyword Sensitivity

This is the failure mode that surprises architects who have done everything else right. You write careful, boundary-rich descriptions for every tool. You test routing on a set of representative inputs. Routing looks clean. Then a category of request starts misrouting and you cannot figure out why, because the descriptions are correct.

The problem is in the system prompt, not the tool descriptions. The API constructs the system prompt from tool definitions plus any user-specified system prompt. If your system prompt contains a phrase like “always prioritize account verification” or “when in doubt, check the customer record first,” those instructions create keyword associations that can override tool description logic. The word “customer” in a customer-related query matches the keyword association in the system prompt before the model even reads the tool description carefully.<sup>[1]</sup>

This is system prompt keyword sensitivity. The fix is to review the full constructed prompt, not just the tool definitions, looking for instructions that might create unintended tool associations. Phrases that read as reasonable business logic to a human (“verify the customer before proceeding”) can act as hidden routing overrides to the model. Rephrase them in terms of workflow sequencing rather than entity-type associations: “call get_customer before calling process_refund” is much safer than “always check customer information first” if you have three different tools that touch customer data.

There is a secondary version of this problem: a system prompt instruction intended to shape tone or behavior can accidentally pattern-match to a tool use trigger. An instruction like “when users ask about account status, be comprehensive” might associate “account status” with get_customer in a way that pulls order-related queries toward the customer tool. Auditing system prompts for accidental keyword-to-tool associations is not optional in production systems; it is part of the tool design process.<sup>[1]</sup>

## The 18-Tool Problem

The exam includes a scenario where a developer productivity agent has 18 tools and is selecting the wrong ones. The instinct is to write better descriptions. The correct solution is different.

The exam guide states the principle directly under the task statement on distributing tools: an agent handed far more tools than it needs, eighteen where four or five would do, selects less reliably, because every additional tool enlarges the decision the model has to make on every single request.<sup>[3]</sup> This is not a framing problem or a description quality problem. It is a structural problem. The model is being asked to pick from 18 options for every request, and the cognitive load of navigating 18 tool descriptions is simply too high to maintain reliable selection across diverse input types.

That is worth separating from the failure the first half of this chapter described, because the two look similar in a log and have opposite fixes. A description problem clusters. The same pair of tools keeps trading places, the pair is one you can name, and each tool in it has a real and distinct purpose the description is failing to carry. That is answered in the definitions, and expanding them works.

A structural problem scatters. The wrong calls do not concentrate on a pair, and some are not near-misses at all: the agent reaches for tools belonging to a part of the system that is not its job, which the guide records as its own symptom, that an agent holding tools outside its specialization tends to misuse them.<sup>[3]</sup> No description rewrite reaches that, and the reason is worth stating plainly. Every description in the set is read against every other. Sharpening one boundary does not reduce the number of boundaries, and eighteen precise descriptions still present eighteen candidates.

Two documented thresholds sit near each other here, measuring different things. Four or five is a design rule for a single specialized agent, and it comes from the exam guide.<sup>[3]</sup> The API documentation gives a second and much higher number: the model's ability to pick the right tool degrades once the available set exceeds roughly thirty to fifty tools, and tool search holds selection accuracy up across far larger catalogues by loading only a focused set on demand.<sup>[4]</sup> One is a property of the model across the whole tool surface; the other is a rule about how much any single agent should be reasoning over. Both hold at once, and the gap between them is the useful part.

What the pair rules out is the fix that would otherwise be obvious. If overload were only a ceiling problem, the answer would be to raise the ceiling, and mechanisms for raising it exist. The exam's answer runs the other way: it does not enlarge what one agent can handle, it reduces what any one agent is asked to handle, by giving each agent only the tools its role needs.<sup>[3]</sup> Loading tools on demand solves context cost and discovery, which is Chapter 11's subject. It does not turn an eighteen-tool decision into a four-tool one.

The solution: distribute tools across specialized subagents.<sup>[1,3]</sup>

Instead of one agent with 18 tools, a coordinator agent with 3-4 delegation tools spawns subagents with 4-5 focused tools each. The coordinator’s job is to route to the right subagent. Each subagent’s job is to pick from a small, coherent set of tools. The coordinator does not need to know which individual tool to call; it needs to know which subagent covers this class of request. The subagent does not need to worry about the full tool surface; it sees only the tools relevant to its domain.

This is the coordinator-plus-subagents pattern covered in Chapter 2. The relevant point here is that 4-5 tools per agent is a tool design constraint, not just an architecture preference. It determines whether tool selection is reliable. If you are debugging routing errors and the agent has more than 5 tools, tool count is your primary suspect, and description quality is secondary.

Scoping is not quite as absolute as that makes it sound, and the guide names the exception. A specialized agent may hold a narrow cross-role tool for a need it hits constantly, so that a high-frequency question does not become a round trip through the coordinator every time. Both words bound the exception: narrow, and high-frequency. The tool covers the common case, and anything outside it still routes through the coordinator.<sup>[3]</sup> The same section gives a related move: replacing a broad tool with a constrained one, a general fetch-a-URL tool giving way to a load-a-document tool that validates what it is handed. The constrained tool is easier to select correctly because there is less it could be for, and harder to misuse because the constraint sits in the tool rather than in a sentence asking the model to be careful.

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

Note that the coordinator’s tool list is small and semantic: spawn a subagent, summarize results, format a response. The coordinator never has to reason about whether to call track_package or update_shipping. That distinction has been pushed down into the order_agent, where it is a choice among four related tools rather than one among eighteen unrelated ones, which is the scoping the guide asks for: each agent holds the tools its role needs and no others.<sup>[3]</sup>

## Writing a Routing Key: The Mechanics

Given everything above, here is what a complete routing-key description includes.

**What the tool does in one sentence.** This is the disambiguation anchor. It needs to be different in substance, not just word choice, from every other tool in the agent’s tool set. If two tools have descriptions whose first sentence could apply to either, the descriptions are not working.

**What the tool returns.** Output type shapes selection. “Returns an order object with line items, shipping address, and delivery estimate” tells the model something about what using this tool produces. If the next step in the workflow needs an order ID, the tool that returns one gets routed to.

That last sentence reaches past the description and into the response itself. A tool used mid-workflow is feeding the next tool call, and what the next call needs is something it can pass as an argument. The documented guidance is to return stable, meaningful identifiers, a slug or a UUID rather than an opaque internal reference, and only the fields the model needs to decide what to do next.<sup>[2]</sup> A human-readable label is no substitute: a title can change, two records can share one, and nothing about it is guaranteed to resolve back to the record it came from. The corresponding failure is a tool that returns a paragraph of prose to an agent that needed a key.

**What input formats are expected.** Format constraints in the description reduce invalid parameter generation and also help routing when the user’s request contains an identifier. “Order ID in format ORD-XXXXXXXX” and “phone in E.164 format” let the model match the user’s stated identifier to the tool that handles it.

**Explicit when-not-to-use boundaries.** State the boundary conditions from this tool’s perspective. What does this tool not handle that a similar tool does? If you have lookup_order and get_customer in the same agent, each description should explicitly disclaim the other tool’s domain.

**What a successful empty result means.** This is subtle but consistently tested. A tool that returns an empty array when no matches are found should say so: “Returns empty array if no match found. Empty result means not found, not a system error.” This falls under two documented headings at once: the exam guide's requirement that a description cover edge cases and boundaries, and the API documentation's that it name the tool's caveats and limitations.<sup>[1,2]</sup> The distinction being drawn is between a query that ran and found nothing and a query that failed to run, two different events that can look identical in a response body. Chapter 5 covers the other side of that boundary, the error contract for a call that genuinely did fail.

**Where a parameter's value comes from.** When a required argument is only obtainable from an earlier call, an order identifier the model has no way to know unaided, saying so is the difference between a model that asks for it and one that supplies something plausible. This is the general shape of the description's job. It is read before the call is generated, so it is the last place a bad call can be prevented rather than detected; server-side validation catches an invented value only after the model has invented it. Both are worth having, but they are not alternatives, and the one that addresses the cause is the one that runs first.

The input_examples field is an additional channel for this information. For tools with complex nested inputs or optional parameters with unclear interplay, examples show the model what a well-formed call looks like. An example with only the required parameters shows that optional ones are genuinely optional. An example with a specific format for a phone number shows the format more concretely than a prose description can.<sup>[2]</sup>

## What the Exam Tests

The exam’s Sample Question 2 is the clearest signal about what this domain tests.<sup>[1]</sup> The correct answer is expanding descriptions, and the explanation is explicit: descriptions are the primary mechanism, and minimal descriptions leave the model without context to differentiate similar tools. The distractors are all recognizable patterns (few-shot prompting, routing layers, tool consolidation) that have legitimate uses elsewhere but are over-engineered or misdirected as first responses to a routing failure caused by bad descriptions.

The exam also tests tool_choice selection reasoning. The pattern to internalize: "auto" when the model needs discretion; "any" when a tool must be called but the specific tool is the model’s choice; forced tool when a specific operation must happen in a specific order; "none" when tool calls need to be suppressed even though tools are defined.

The 18-tool scenario from Scenario 4 (Developer Productivity) tests distribution over description. The exam’s correct answer is reducing to 4-5 tools per agent and distributing the rest to specialized subagents. Making descriptions longer, fine-tuning the model, or switching to a larger model are the plausible alternatives, and each of them leaves the number of candidates unchanged. The guide locates the cause in the size of the decision the agent is being asked to make, so a fix that does not shrink that decision does not address it.<sup>[3]</sup>

System prompt keyword sensitivity is a less obvious test point but it appears in Task Statement 2.1. Candidates who understand tool design only as “write good descriptions” will miss questions that require reviewing the system prompt for keyword-sensitive instructions that override those descriptions. The full audit scope is the constructed prompt, not just the tool definitions.

Model selection is part of the tool count equation. The documented guidance is the latest Claude Opus model for complex tools and ambiguous queries, because it handles multiple tools better and will seek clarification when it needs to, and Claude Haiku models for straightforward tools, with the caveat that they may infer a missing parameter rather than ask for it.<sup>[2]</sup> Know which lever to reach for first: a model choice does not repair a tool surface.

## Key Takeaways

- Every tool definition the model sees is translated into a fragment of the system prompt. The description field is the sentence that tells the model when to reach for this tool. Treat it accordingly.
- A minimal description like "Retrieves order details" is not a valid description. A valid description specifies what the tool returns, what input it expects, what makes this tool distinct from similar tools, and when not to use it.
- tool_choice has four values: "auto" (model decides; default when tools are provided), "any" (model must call a tool but picks which one), {"type": "tool", "name": "..."} (forces a specific tool), and "none" (prevents tool use; default when no tools are provided). It is a forcing mechanism for a single request, not a repair for bad routing: force a prerequisite on the turn where it is unmet, then release to "auto", because forcing it on every turn blocks the tools that depended on it.
- The input_examples field on a tool definition gives the model concrete patterns for valid inputs alongside the schema. Use it for tools with complex, nested, or format-sensitive parameters, after the description is right rather than instead of getting it right.
- Two thresholds, two levels. 4-5 tools per agent is a design rule for a specialized agent; selection degrades across the whole tool surface somewhere past thirty to fifty tools. Giving a single agent 18 tools degrades selection reliability, and the answer is not a more elaborate description strategy or a higher ceiling; it is distributing tools across specialized subagents.
- System prompt keyword sensitivity is a real failure mode. A well-written tool description can still be overridden by a keyword-sensitive instruction elsewhere in the system prompt. Reviewing the full prompt for unintended tool associations is part of the audit.
