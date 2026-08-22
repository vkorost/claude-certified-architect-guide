# Chapter 10: Structured Output and the Validation Loop

**Summary:** ***tool_use** mode is the contract mechanism for machine-parseable output, and **strict: true** is what turns the contract into a guarantee. A tool definition carrying a JSON schema tells the model what shape to aim for. Setting strict on that definition constrains the sampler itself, so the tool input the model emits is schema-valid by construction rather than by effort: no JSON.parse() errors, no absent required fields, no type surprises. But structural compliance is not semantic correctness. A line-item sum that does not match the stated total is structurally valid JSON. A date field containing a plausible-looking but wrong date is structurally valid JSON. The model filled the containers correctly and put the wrong things inside.*

*The production pattern for reliable structured extraction is a three-phase loop: generate using a schema-constrained tool call, validate semantics in application code, retry with specific error feedback if validation fails. Retry specificity is not optional. Generic “there were errors, try again” gives the model no signal about which constraint was violated. “Line items sum to 847.50 but stated_total is 900.00, discrepancy of 52.50” gives the model something it can act on.*

*The chapter also covers the two complementary structured output features (output_config.format and strict: true), schema design patterns that prevent hallucinated values, format normalization applied at extraction time rather than after it, the documented complexity limits that trigger 400 errors, property ordering behavior, the three documented cases in which output does not match the schema at all, and the triage that separates an error worth retrying from one worth escalating.*

## Scene: The Invoice That Balanced Itself

Scenario 6 in the exam is structured data extraction. The guide gives it four lines: a system that pulls information out of unstructured documents, checks what it pulled against JSON schemas, keeps accuracy high, copes with the awkward documents rather than falling over on them, and hands its results to systems downstream. That is the whole of it. There is no walkthrough attached, no worked example, no annotated failure. Six scenarios exist in the bank and four are drawn for any given sitting, so a candidate may or may not see this one at all.<sup>[1]</sup>

What follows is therefore this book's illustration of that framing going wrong. The guide does not print it, and nothing in the argument depends on it.

A developer builds the pipeline, and builds it well by most measures. The tool schema is thought through. The call forces the extraction tool every time, so a conversational reply is not among the possible outcomes. The response comes back as JSON matching the schema exactly. No parse errors. No absent fields. The validator checks the JSON structure and returns clean.

The invoice goes into the database. The line items sum to $847.50. The stated_total field in the document was $900.00. Nothing in the pipeline was looking at that, because the only thing under validation was shape.

The exam states the underlying point with no illustration at all, and states it as a limit rather than as a caution: strict JSON schemas delivered through tool use remove syntax errors and do not remove semantic ones. Totals that fail to reconcile, and values sitting in a field they do not belong to, are the guide's own examples of what survives a schema check intact.<sup>[2]</sup>

This is the central fact about tool_use as a structured output mechanism: it is a JSON contract, not a correctness contract. **The schema defines what shape the output must take. It says nothing about whether the values inside that shape are accurate representations of the source document.** Those are two completely different guarantees, and conflating them is the most consistent anti-pattern the exam tests in Domain 4.

The named concept for this chapter: **tool_use as JSON contract**.

## Two Features, One System

Structured outputs in the Claude API are built from two complementary features that can be used independently or together.<sup>[3]</sup>

The first is output_config.format. This constrains Claude's response itself, the text the messages call returns, to a JSON schema. Set type: "json_schema", supply a schema definition, and the response arrives as valid JSON matching that schema in the response's text content block. This is useful when the extraction result is the whole response rather than a tool call.

The second is strict: true on tool definitions. It is one of the optional properties a tool definition accepts, alongside cache_control, defer_loading and allowed_callers, which Chapter 4 covers. When it is set, two things are guaranteed rather than encouraged: the input in the tool_use block conforms to that tool's declared input_schema, and the tool name is always one that actually exists.<sup>[4]</sup> No extra fields, no absent required fields, no type violations.

That second feature is the one worth being precise about, because the loose version of the claim is the version most study material carries. A tool definition with a JSON schema and no strict flag does not get the guarantee. It gets a well-described target. The documentation is blunt about what that leaves on the table: without strict mode the model may hand back a value of the wrong type, a string "2" where the schema asked for the integer 2, or leave out a field the schema marked required.<sup>[4]</sup> Those are exactly the failures a validation-and-retry loop is usually written to absorb. Setting strict removes the need for that half of the loop rather than making the loop more reliable.

Both features work the same way underneath. The API compiles the schema into a grammar and constrains token sampling to what that grammar permits at each position. The model is not trying harder to comply; the token space available to it at each step is mechanically narrowed.<sup>[3,4]</sup>

Used together, they cover different surfaces: output_config.format governs the response-level JSON, strict: true governs each individual tool invocation. A workflow needing both guaranteed tool inputs and a guaranteed structured final synthesis can set both in a single request.

The scope of each grammar matters and is easy to get backwards. The grammar compiled from output_config.format applies to Claude's direct output only. It does not extend to tool use calls, to tool results, or to thinking blocks, and grammar state resets between sections so that reasoning can run unconstrained while the final response still lands in the required shape.<sup>[3]</sup> Which means a request that sets output_config.format and expects that alone to police its tool inputs has policed nothing. Tool inputs are strict: true's job.

Two documented incompatibilities are worth carrying. Citations cannot be combined with output_config.format, because citation blocks have to interleave with text and that interleaving conflicts with a schema constraint on the whole response; the request returns a 400. Message prefilling is likewise incompatible with JSON outputs. Streaming, contrary to a widely repeated belief, is not on that list: structured outputs stream like any other response, with the caveat that the full JSON has to accumulate before it can be deserialized.<sup>[3]</sup>

Two performance characteristics follow from compilation and shape how an extraction pipeline should be built. The first request against a given schema pays extra latency while the grammar compiles, and compiled grammars are then cached for twenty-four hours from last use. The cache is invalidated by a change to the schema structure or to the set of tools in the request, and specifically not by a change to a name or a description alone.<sup>[3]</sup> A pipeline that assembles a slightly different schema per document therefore pays the compilation cost on every document and gets no benefit from the cache. A pipeline that holds its schemas fixed and varies only the document pays it once. Structured output also injects an additional system prompt describing the expected format, which raises the input token count and, when output_config.format changes, invalidates the prompt cache for that thread.<sup>[3]</sup>

One constraint on schema content sits outside performance entirely. Compiled schemas are cached separately from message content and do not carry the protections that prompts and responses carry, so protected health information must not appear in schema property names, enum values, const values or pattern expressions.<sup>[3,4]</sup> For an extraction pipeline pointed at clinical documents this is a schema design rule, not an infrastructure footnote: the sensitive material belongs in the document being extracted, never in the shape being extracted into.

## Forcing the Tool Call

A tool definition with a schema does not automatically produce structured output. The model still chooses whether to call the tool or respond conversationally. Set tool_choice: "any" to guarantee the model calls a tool rather than returning text.<sup>[5]</sup>

tool_choice: "any" tells the model it must use one of the provided tools, but leaves the choice of which tool to the model. The behavioral mechanics of "any" and forced tool selection are defined in Chapter 4; the application here is guaranteeing a tool call in extraction pipelines where a conversational response would be a failure mode. Under either "any" or a named tool the API prefills the assistant turn to force the call, with the documented consequence that no natural language preamble appears before the tool_use block even if the prompt asks for one.<sup>[5]</sup> For an extraction pipeline that is a feature. For a pipeline that also wanted a human-readable note alongside the extraction, it is a design constraint: the note has to come from a schema field or from a separate turn.

One restriction carries over from Chapter 4 and matters here because extraction prompts often have thinking switched on. Under manual extended thinking, that is, thinking explicitly enabled in the request, neither "any" nor a named tool is permitted and the request errors; only "auto" and "none" are compatible. Adaptive thinking, including on models where thinking is on by default, supports forced tool use.<sup>[5]</sup> An extraction pipeline that turns thinking on manually and then reaches for tool_choice to guarantee its output has chosen two things that do not combine.

For extraction pipelines where the document type is known in advance, force the specific tool:

```
response = client.messages.create(
    model="claude-opus-4-7",
    tools=[extract_invoice_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[{"role": "user", "content": f"Extract: {invoice_text}"}]
)
```

When the document type is unknown and multiple extraction schemas exist, *tool_choice: "any"* lets the model select the appropriate schema while still guaranteeing a tool call happens.<sup>[2]</sup>

The tool_choice values themselves *("auto", "any", forced tool, "none")* are defined in Chapter 4. The concern here is narrower: for structured extraction, "*auto*" is the wrong default because it allows the model to decide that a conversational response is more appropriate than a tool call. If the extraction pipeline depends on always getting structured JSON back, "any" or forced tool is not optional.

### What Selects "any" and What Selects a Named Tool

Both settings guarantee that some tool runs. They differ in who picks it, and the property in the requirement that selects between them is whether the pipeline knows in advance which extraction should happen.

"any" is the setting for the case where it does not. Several extraction schemas are defined, the document arriving is of unknown type, and the correct schema is a function of the document rather than of the pipeline. Handing the choice to the model is the point; the guarantee that is needed is only that something structured comes back.<sup>[2]</sup>

A named tool is the setting for the case where one particular extraction has to run, usually because a later step consumes its output. The guide's example is a metadata extraction that has to complete before enrichment tools can do anything useful.<sup>[6]</sup> Here "any" is not sufficient: it guarantees a call and permits the model to make that call be the wrong one.

Then the trap underneath both, which is that neither setting can order a sequence. A single tool_choice governs one turn. It cannot compel a chain of three tools to fire in a stated order, and the shape of the wrong answer is naming the last tool in the chain and expecting the earlier ones to have been run first. The documented pattern is to force the prerequisite on the opening turn and then process the dependent steps on follow-up turns, where their inputs actually exist.<sup>[6]</sup> Sequencing is a property of the loop, not a property of a parameter.

Finally, the combination the documentation recommends explicitly and which is easy to miss because the two features live on different pages: setting tool_choice: "any" alongside strict: true on the tool definitions guarantees both halves at once, that one of the defined tools will be called, and that its input will conform to the schema.<sup>[5]</sup> The first without the second guarantees a call whose payload may still need repair. The second without the first guarantees a payload that may never arrive.

## Schema Design: Preventing the Model From Inventing Answers

The schema is not just a shape declaration. It is an instruction to the model about what it should and should not fill in. Get the design wrong and the model fills required fields with plausible-sounding fabrications rather than admit the source document doesn’t contain the information.

Three patterns from the exam guide address this directly.<sup>[2]</sup>

**Make fields optional (nullable) when the source may not contain the information.** A field marked required tells the model it must produce a value. If the source document genuinely lacks that value, the model will invent one that fits the type constraint. An invoice without a purchase order number should produce *"po_number": null*, not a guessed PO number. Mark the field optional and let null be a valid answer:

```json
{
  "type": "object",
  "properties": {
    "vendor_name": {"type": "string"},
    "invoice_number": {"type": "string"},
    "po_number": {"type": ["string", "null"]},
    "total": {"type": "number"}
  },
  "required": ["vendor_name", "invoice_number", "total"],
  "additionalProperties": false
}
```

There is a companion decision on the collection fields, and it is the same decision made one level down. A field that holds a list of things has two ways to say the list is empty, and they are not equivalent. An empty array says the document was read and contained no entries, and it stays type-consistent, so downstream code iterates over nothing and moves on. A null says the field is absent, and every consumer of that field now needs a null branch before it can iterate. When the extraction genuinely found no line items, an empty array is the honest and cheaper answer. Null is for the case where the question could not be answered at all. Collapsing the two is how a downstream system ends up unable to distinguish an invoice with no line items from an invoice whose line items were never looked for.

**Use "unclear" as an enum value for genuinely ambiguous fields.** When the model cannot determine the correct value from the source document, it needs somewhere to put that uncertainty. Without an explicit ambiguity option, it picks the most plausible enum value anyway. Adding "unclear" as a valid enum value gives the model a truthful option:

```
"payment_terms": {
  "type": "string",
  "enum": ["net_30", "net_60", "immediate", "unclear"]
}
```

**Use "other" plus a detail field for extensible categorization.** Enums with a fixed set of values will misclassify documents that don’t fit any category. The "other" plus detail pattern handles this:

```
"document_type": {
  "type": "string",
  "enum": ["standard_invoice", "credit_note", "proforma", "other"]
},
"document_type_detail": {
  "type": "string",
  "description": "Required if document_type is other"
}
```

The document_type_detail field is optional in the schema but operationally required when document_type is "other". The prompt instructions should state this explicitly, because JSON Schema cannot express conditional requirements of this form in a way that constrained decoding enforces.

A caution that belongs with the enum patterns rather than after them: one escape value is a fix, two is usually a mistake. An "unclear" option earns its place because it covers a state the extraction actually reaches, the state of having read the field and not been able to resolve it. Adding a second option beside it that describes a different thing, a "neutral" alongside an "unclear" in a sentiment field, for instance, does not cover more ground. It introduces a boundary the model now has to adjudicate on every document, and it converts a clean ambiguity signal into two categories that reviewers will find used interchangeably. Add the option that names the state the pipeline needs to see. Stop there.

One documented behavior sits underneath all of this and quietly weakens the enum guarantee. Structured outputs do not guarantee the capitalization of string enum and const values. The model may return a value differing from a schema entry only in case, typically in the first letter of a word that follows a space, and the response completes normally with no error and no distinguishing stop_reason.<sup>[3]</sup> Two consequences for extraction code. Compare enum values case-insensitively rather than by equality, and do not define two enum values that differ only in capitalization, because nothing will reliably tell them apart. This is the smallest instance of the chapter's whole argument: the structural guarantee is real, it is bounded, and the boundary is documented rather than discovered.

These three patterns share a common goal: reduce the pressure on the model to produce a confident answer when the source material does not support one. The schema should make it easy to say “I don’t know” or “this doesn’t fit.”

### Normalization Belongs at Extraction Time

There is a fourth schema-design skill under the same task statement, and it is the one candidates most often reason about backwards. Source documents are inconsistent about format. Dates arrive in three conventions, amounts carry currency symbols or do not, vendor names arrive with and without legal suffixes. The schema can declare that a date field is a date. It cannot, on its own, tell the model which of the numbers on a European invoice is the day.

There are two places to put the normalization rules, and the exam's answer is the earlier one: state them in the prompt, alongside the strict output schema, so they apply while the extraction is happening.<sup>[2]</sup> The alternative, a post-processing pass that rewrites already-extracted strings, looks equivalent and is not. The selecting property is access to the source. At extraction time the model can see that the document is addressed to a European office, that the other dates on the page are unambiguous, and that a two-digit leading field is therefore a day. A post-processing rule sees a string. It applies a deterministic transformation with no way to check the transformation against the document it came from, and when its assumption is wrong it produces a clean, confident, wrong value that no later validator will question.

This does not make post-processing useless. It makes it the right tool for transformations that need no context at all, stripping whitespace, uppercasing a country code. Anything requiring a judgment about what the source meant belongs in the extraction step, where the source is still in front of the model.

## Property Ordering: The Behavior Nobody Reads About Until It Breaks Something

The structured output documentation specifies property ordering behavior that is not intuitive.<sup>[3]</sup>

Properties appear in the output in schema order, with one modification: required properties appear first (in their schema order), then optional properties (in their schema order), regardless of how the properties are ordered in the schema definition itself.

Consider this schema:

```json
{
  "type": "object",
  "properties": {
    "notes": {"type": "string"},
    "name": {"type": "string"},
    "email": {"type": "string"},
    "age": {"type": "integer"}
  },
  "required": ["name", "email"],
  "additionalProperties": false
}
```

*notes* is defined first in the properties object, but it is optional. The output will order properties as: *name, email, notes, age.* The required fields appear first even though they are not listed first in the schema.

If property order in the output matters to downstream parsing logic, mark all fields as *required* (or account for this reordering in the parser). If the required/optional split is architecturally meaningful for the schema but the output consumer needs a specific order, the consumer must sort after parsing.

**If order matters, mark everything required.**

## The Validation Loop

The schema guarantee eliminates an entire class of failures: JSON syntax errors, missing required fields, wrong types. What it cannot eliminate is the class of failures that live above the schema level. Values in the wrong fields. Sums that don’t add up. Dates formatted correctly but extracted from the wrong part of the document. Vendor names from the footer instead of the header.

These are semantic errors. The JSON is valid. The schema is satisfied. The data is wrong.

Application code is the only place semantic validation can happen. The validation loop is the production pattern for catching and correcting these failures.<sup>[2]</sup>

The loop structure:

```python
def extract_with_validation(document, max_retries=3):
    messages = [{"role": "user", "content": f"Extract: {document}"}]

    for attempt in range(max_retries):
        response = client.messages.create(
            model="claude-opus-4-7",
            tools=[extract_tool],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages,
        )

        data = parse_tool_response(response)
        errors = validate_semantics(data)

        if not errors:
            return data

        # Append SPECIFIC errors and retry
        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": (
                f"Validation failed. Fix these specific errors and re-extract:\n"
                + "\n".join(f"- {e}" for e in errors)
            )
        })

    raise ExtractionError(f"Extraction failed after {max_retries} attempts")
```

The critical word in that structure is “*specific*.” The retry message must tell the model which field failed, what the constraint is, and what the actual vs expected value is. “*There were validation errors, please try again*” gives the model no signal. It will produce the same extraction or a random variation of it.

Compare these two retry messages:

Wrong: *“The extraction had errors. Please try again.”*

Right: *“Line items sum to 847.50 but stated_total is 900.00. Either the line items are incomplete or the stated_total was extracted incorrectly. Re-examine the document and correct the extraction.”*

The right version tells the model which field failed, what the arithmetic shows, and where to look. The model can act on that. It cannot act on “there were errors.”<sup>[2]</sup>

The semantic validation function is application-specific, but a few patterns appear in extraction pipelines reliably:

```python
def validate_semantics(data):
    errors = []

    if data["total"] <= 0:
        errors.append(f"Total must be positive, got {data['total']}")

    if "line_items" in data and data["line_items"]:
        calculated = sum(item["amount"] for item in data["line_items"])
        if abs(calculated - data["subtotal"]) > 0.01:
            errors.append(
                f"Line items sum to {calculated:.2f} but subtotal is "
                f"{data['subtotal']:.2f}"
            )

    if not re.match(r"\d{4}-\d{2}-\d{2}", data.get("date", "")):
        errors.append(f"Date must be ISO 8601 format, got: {data.get('date')}")
    return errors
```

Numeric consistency checks, date format verification, referential integrity between fields. None of these can be enforced by JSON Schema in a way that constrained decoding will catch at generation time. They are logic constraints, not type constraints.

The line between the two is further to the application's side than most developers assume, and the documentation draws it explicitly. The supported JSON Schema subset excludes numerical constraints, so minimum, maximum and multipleOf are not available. It excludes string constraints, so minLength and maxLength are not available. It excludes recursive schemas, and it excludes array constraints beyond a minItems of zero or one. Using any of them returns a 400 rather than being silently ignored.<sup>[3]</sup> So it is not only the arithmetic across fields that cannot live in the schema. "This total must be greater than zero" cannot live there either. Every constraint of that kind is application code or it does not exist.

### What the Loop Can Detect and What It Cannot

A validation loop detects exactly the errors someone wrote a check for, which sounds obvious and has a consequence that is not.

The loop is strong wherever the extraction contains redundancy. Line items and a stated total are redundant: each can be computed from the other, so a mismatch is detectable without leaving the extracted record. A date field and a date format rule are redundant in the same weak sense. An invoice number and a checksum, a set of amounts and a subtotal, a currency code and a symbol appearing in the raw text: all of these give application code two things to compare.

The loop is blind wherever the extraction contains a single unchecked value. If a vendor name is lifted from the footer rather than the header, the resulting record is structurally perfect, semantically wrong, and internally consistent with itself. There is nothing to compare it against. The same is true of a plausible date drawn from the wrong line, a plausible address, a plausible reference number. No validator catches these, because catching them would require reading the source document, which is the job the extraction was supposed to have done.

This is why the redundancy is worth designing in rather than hoping for, and it is the whole rationale for the self-correction fields below. It is also the honest limit on the pattern: a validation loop raises confidence in the fields that cross-check and says nothing at all about the fields that do not. Sampling those against the source, by document type and by field rather than in aggregate, is a different mechanism belonging to Chapter 12's territory, and it is the only thing that measures what the loop cannot see.

## Self-Correction Fields: Building the Check Into the Schema

The validation loop is reactive: *extract, check externally, retry if wrong*. A complementary approach is to build the self-check into the schema itself, so the extraction produces both the answer and the material for verifying it.<sup>[2]</sup>

The stated_total vs calculated_total pattern:

```json
{
  "type": "object",
  "properties": {
    "stated_total": {
      "type": "number",
      "description": "The total amount as stated in the document"
    },
    "calculated_total": {
      "type": "number",
      "description": "Sum of all line item amounts"
    },
    "totals_match": {
      "type": "boolean",
      "description": "True if stated_total equals calculated_total 
within 0.01"
    },
    "conflict_detected": {
      "type": "boolean",
      "description": "True if any extracted values are internally 
inconsistent"
    }
  },
  "required": ["stated_total", "calculated_total", "totals_match", 
"conflict_detected"]
}
```

By requiring the model to extract both the stated total and compute the sum of line items, the downstream validator does not need to re-sum the line items. The totals_match flag is set by the model during extraction. When conflict_detected is true, the application routes the document for human review or triggers a targeted retry.

This approach shifts some of the validation work into the extraction pass itself. The model is doing arithmetic and consistency checking as part of filling the schema, rather than requiring a separate validation pass in the application. The tradeoff is schema complexity: more fields, higher chance of hitting complexity limits.

It is worth being clear about what these fields are and are not. They are redundancy, deliberately introduced so that a later check has two things to compare. They are not a second opinion. The model that extracted the line items is the same model computing their sum, so a systematic misreading of the line items produces a calculated_total that agrees with itself perfectly. What the pattern catches is the disagreement between the document's own assertion and the document's own arithmetic, which is a real and common defect, and it catches it without the application having to re-implement the summing. What it does not catch is an error that runs through both halves.

Which is the distinction to hold on to when a schema is tempted toward a confidence field. Attaching a self-reported confidence score to each extracted value feels like the same move and is not. The self-correction fields introduce a comparison between two derivations of the same quantity. A confidence field introduces a second output channel with no external referent, which the model fills in the same pass and under the same misapprehensions as the value it is scoring; a fabricated field arrives with a confident score attached. If the problem being solved is fabrication, the fix is in the extraction constraint, an optional field and an explicit instruction to return null when the source does not state the value, not in a parallel commentary on the fabrication. Field-level confidence does have a documented job, which is routing scarce reviewer attention once thresholds have been calibrated against a labeled validation set.<sup>[7]</sup> That job is downstream of extraction quality and does not substitute for it. The two are separable and the exam separates them.

The detected_pattern field serves a different purpose: it captures metadata about the extraction itself rather than the document content.<sup>[2]</sup>

```
"detected_pattern": {
  "type": "string",
  "enum": ["standard_layout", "multi_page", "table_heavy", 
"narrative_heavy", "unclear"],
  "description": "The structural pattern of the source 
document"
}
```

When developers review extraction results and dismiss findings as incorrect, the detected_pattern field allows systematic analysis: are dismissals concentrated in documents with a particular structural pattern? If so, few-shot examples covering that pattern (as noted in Chapter 9) or schema adjustments for that pattern are the fix. Without detected_pattern, dismissal analysis requires manual review of source documents.

(For the decision framework that distinguishes schema fixes from few-shot and from other interventions when an extraction pipeline fails, see the intervention classifier in Chapter 9.)

## When Retries Are Not the Answer

Retries address format mismatches and structural errors. They do not address missing information.

If the source document does not contain the data required by the schema, retrying will not produce it. The model will either fabricate a plausible value (bad), return null for optional fields (acceptable), or fill the field with content from somewhere else in the document (bad and hard to detect). More retries, more specific error messages: none of it helps when the constraint is the source material itself.<sup>[2]</sup>

Distinguish these two failure modes:

**Retryable:** The model extracted the date in MM/DD/YYYY format but the schema requires ISO 8601. The date is in the document. The extraction got the format wrong. A retry with “date must be YYYY-MM-DD format” will fix this.

**Not retryable:** The schema requires a purchase order number, but this vendor does not use PO numbers and the document has none. The model extracted something from the document that looks like a number. Retrying will produce a different wrong answer.

The diagnostic question: *is the error due to how the model extracted the information, or due to the information not being present?* The answer determines whether to retry or to route the document for human review or accept null.

The exam tests this distinction directly. The correct response to “*required field is absent from source document*” is not more retries with more specific errors. It is recognizing that the information constraint is architectural: the schema requires data the document does not have. Address it at the schema level (make the field optional) or the pipeline level (route document type to a different schema).<sup>[2]</sup>

### The Triage: Retry, Follow Up, or Escalate

The two failure modes above are the ends of a three-way split, and the middle one is where most production extraction actually fails. The whole triage runs on one question asked in a particular order, and asking it in the wrong order is the most reliable way to burn a retry budget on a problem retries cannot touch.

**First: is the information in the input at all?** If it is not, the loop is over before it starts. No restatement of the error supplies what the context does not contain, and each additional attempt increases the chance of a fabricated value rather than a correct one. The disposition is to route the document to manual review or to a source that actually carries the field.<sup>[2]</sup> This question goes first because a negative answer makes the other two moot, and because the symptom of an absent field looks identical to the symptom of a badly extracted one.

**Second: is the data present but shaped wrongly?** A date read correctly and emitted in the wrong convention, a number carrying a currency symbol the schema did not want, a field placed one level too deep. These are what retry-with-feedback is for, and they resolve reliably because the model has everything it needs and simply produced the wrong surface form.<sup>[2]</sup> The retry has to carry the specific constraint that was violated. A retry carrying "there were errors" is a re-roll of the dice, not a correction.

**Third: is the data present, correctly shaped, and internally inconsistent?** Amounts that do not reconcile, an identifier that fails a validity check, a date outside the range the rest of the document implies. This is the middle case, and the observation that identifies it is that plain retries change nothing. If resending the same request produced a different result, the failure was sampling variance. If it produces the same failure every time, the extraction logic is reproducing itself faithfully and needs a different input, not another attempt.<sup>[2]</sup>

What that case requires is a follow-up request rather than a retry, and the difference is what the request carries. Three things have to be in it: the document the extraction came from, the extraction that failed, and the specific errors that were found in it.<sup>[2]</sup> Drop the document and the model is correcting a record it cannot check. Drop the failed extraction and it re-derives the same output from scratch. Drop the specific errors and it has no correction target. A retry is worth precisely the diagnostic it carries.

There is a fourth situation which is none of the above and which the exam treats as its own case. Two independent error sources can produce the same symptom. A total that does not reconcile may mean the model misread the line items, or it may mean the document itself was damaged in scanning and the digits it presents are not the digits that were printed. When both explanations are live, neither a retry nor a follow-up is the answer, because the pipeline cannot tell which of the two it is correcting. The disposition is to capture both figures, the one the document states and the one the line items produce, flag the inconsistency, and route it to a person to adjudicate.<sup>[2,7]</sup> The tempting alternative is a reconciliation rule that forces the two into agreement. It should be named for what it does: it converts a record that was visibly wrong into a record that is invisibly wrong, and it removes the only signal anyone had.

**Name the wrong fix first.** Where retries keep failing, the reflex is more of them, a longer backoff, or a larger model. The diagnosis that resolves it is the first question above, asked about the input rather than about the attempt. Where semantic checks keep failing, the reflex is a tighter schema, and a tighter schema cannot express the constraint being violated. The fix is a follow-up carrying the error.

## Complexity Limits: The 400 Error Nobody Expects

Constrained decoding works by compiling schemas into grammars. Complex schemas produce large grammars, and large grammars take longer to compile. The API enforces explicit limits to protect against excessive compilation times.<sup>[3]</sup>

These limits apply to all requests with output_config.format or strict: true:

| Limit | Value | Description |
| --- | --- | --- |
| Strict tools per request | 20 | Maximum tools with strict: true. Non-strict tools don’t count. |
| Optional parameters total | 24 | Total optional parameters across all strict schemas and JSON output schemas. Each parameter not in required counts. |
| Union type parameters total | 16 | Parameters using anyOf or type arrays (e.g., "type": ["string", "null"]). |

The per-request nature of the optional parameter limit is the trap. **Four tools with six optional parameters each adds up to 24: exactly at the limit, with no room.** The limits apply to the combined total across all strict schemas in a single request.<sup>[3]</sup>

Beyond the explicit limits, internal grammar size limits can trigger a 400 error even when all three explicit limits are satisfied. Schema complexity does not reduce to a single dimension: optional parameters, union types, nested objects, and the number of strict tools interact with each other. The compiled grammar can grow disproportionately large from combinations of these features. The API enforces a 180-second compilation timeout as a final stop, and schemas that pass all checks but produce very large grammars may hit it.

When hitting complexity limits, the documented strategies in order of preference:<sup>[3]</sup>

1. Mark only critical tools as strict. If the schema violation for a simpler tool is tolerable (the model naturally produces valid inputs anyway), reserve strict: true for the tools where violations cause real problems.
2. Reduce optional parameters. Each optional parameter roughly doubles a portion of the grammar’s state space. Making a parameter required and having the model supply a default explicitly is cheaper than leaving it optional.
3. Flatten nested structures. Deeply nested objects with optional fields compound complexity.
4. Split across multiple requests or subagents. If the tool set is too large for a single strict request, distribute the tools across separate calls or subagent invocations.

**The 400 error message is “Schema is too complex for compilation.” When it appears, the cause is schema complexity, not a malformed request.**

### The Failure That Looks Like Schema Complexity and Is Not

Tool definitions and injected format instructions occupy input tokens, and an extraction pipeline pointed at long documents is spending its context budget on both the schema and the document. When the two together approach the window, quality falls off at the end of the document first.

The diagnostic worth carrying is the shape of the failure rather than its content, and this one is the book's reasoning rather than a documented rule. Schema complexity and general attention effects both degrade extraction uniformly: they apply to a two-page document as much as to a sixty-page one, because neither has anything to do with length. Context pressure does not behave that way. It produces a threshold. Short documents extract cleanly, long ones extract cleanly until a certain size, and past that size the misses cluster at the end. If the accuracy curve against document length has a knee in it, and if what goes missing sits at the tail rather than being scattered, the constraint is the window and not the schema. Simplifying the schema will not move it. Chapter 11 covers what does.

## Schema Compliance Without Semantic Correctness: The Core Exam Distinction

The exam states the distinction as a property of the mechanism, in the knowledge statements for the task statement that owns structured output: schemas enforced through tool use eliminate syntax errors and leave semantic errors untouched. Totals that do not reconcile and values placed in the wrong field are given as the illustrations.<sup>[2]</sup> The documentation states the same boundary from the other side, in terms of what the guarantee covers: strict tool use guarantees that the tool input conforms to the declared schema and that the tool name is valid.<sup>[4]</sup> Conformance and validity, both of them properties of shape. Neither source claims anything about whether the values are true.

Structure: the JSON is valid, the required fields are present, the types match, additionalProperties is respected.

Semantics: the values inside the structure accurately represent the source document.

The structural guarantee comes from constrained decoding, a mechanical property of how the API generates tokens. The semantic guarantee comes from the validation loop, a property of the application code that checks the extracted values against business rules, arithmetic constraints, and domain knowledge.

A system that trusts the structural guarantee and skips semantic validation is making an architectural error. The schema says the output is shaped correctly. It says nothing about whether the values are right.

There is a smaller error available to a system that trusts the structural guarantee too literally, and the documentation names three cases where the output does not match the schema at all. The first is a refusal: the response carries stop_reason "refusal", returns a 200 rather than an error, is billed for the tokens it generated, and contains the refusal in place of the schema-shaped output. The second is truncation: a response cut off at max_tokens carries stop_reason "max_tokens" and is incomplete, which for JSON means unparseable, and the disposition is a larger token allowance rather than a retry at the same size. The third is the enum casing behavior described earlier, which completes normally and gives no signal at all.<sup>[3]</sup> So parsing code should read stop_reason before it reads content. A pipeline whose error handling assumes that a 200 implies parseable JSON will treat a refusal as a malformed extraction and retry it, which is a loop that cannot terminate on its own.

The production architecture combines both layers. Use tool_use with strict: true to eliminate the entire class of JSON syntax failures. Use application validation to catch semantic errors the schema cannot express. Use the retry loop to correct format mismatches and structural errors in the extraction. Know when to route for human review instead of retrying.

This is not a critique of structured output as a mechanism. The structural guarantee is valuable precisely because it eliminates one entire class of failures, which makes the remaining failures (semantic) easier to reason about. The error isn’t trusting tool_use. The error is trusting it for more than it guarantees. Every tool schema in the request contributes to context window cost; Chapter 11 covers the architectural implications of that budget.

## The Reflex and the Fix

Four defects in this domain have a reflex response that is plausible, proportionate, and aimed at the wrong layer. Learning the reflex is as useful as learning the fix, because the reflex is what the distractors are built from.

**JSON syntax errors keep appearing in the output.** The reflex is to describe the required format more forcefully in the prompt and to wrap the call in retry logic. Both improve the odds. Neither changes the kind of guarantee on offer, and a probabilistic guarantee against parse failure is what the pipeline already had. The fix is to define a tool whose input schema is the target shape, set strict on it, and read the extraction out of the tool_use input. That moves compliance from something the model is asked to do into something the sampler cannot do otherwise.<sup>[2,4]</sup> This is the single most repeated discrimination in the domain, and it holds because instructions and constrained decoding are different categories of thing rather than different amounts of the same thing.

**The model invents values for fields the document does not contain.** The reflex is to tighten the constraint: keep the field required, add a description insisting on accuracy, perhaps add a format. The reflex is precisely backwards. A required field is an instruction that a value must be produced, and under that instruction a model with nothing to draw on produces something type-valid and plausible. The pressure causing the fabrication is the requirement itself. Making the field optional and nullable removes the cause instead of arguing with the symptom.<sup>[2]</sup> This is a schema defect, not a prompt defect, and the layer the fix lands on is the tell.

**Fields are already nullable and fabrication continues.** The reflex is to add observability: a confidence score beside each value, so the bad ones can be filtered later. The fix is upstream of that. The schema now permits null but nothing has told the model when null is the right answer, so it continues to prefer a plausible value over an empty one. An explicit instruction to return null when the document does not state the value directly constrains the extraction step itself.<sup>[2]</sup> Filtering afterwards is a different activity with different prerequisites.

**An enum drifts out of date as new categories appear in the corpus.** The reflex is to add the new value and ship. It works, once, and it commits the team to a deployment per novel category and to misclassification in the interval before each one. The fix is structural: an "other" member plus a free-text detail field, which is open-ended by construction and needs no maintenance as the corpus changes.<sup>[2]</sup> The choice is between a schema that absorbs novelty and a schema that requires a release to absorb novelty.

The pattern under all four is the same. Ask which layer the defect lives on before choosing a fix. A structural problem does not yield to a prompt, and a prompt problem does not yield to a schema.

## Sample Questions

**Question 1: A structured extraction pipeline uses tool_use with a strict JSON schema. After extracting invoice data, the validator finds that the line-item amounts sum to a different value than the stated_total field. The JSON structure is valid and all required fields are present. What type of error has occurred?**

A. A schema compliance failure, which strict: true should have prevented  
B. A *semantic* error, which requires application-level validation to detect  
C. A token limit error causing truncated output  
D. A tool_choice configuration error preventing correct tool selection

***Correct answer: B.** The JSON structure is valid (strict: true succeeded). The discrepancy between line-item sums and stated total is a **semantic error: **the values are internally inconsistent, which is above what schema validation can enforce.*

**Question 2: A validation retry loop is appending errors to the prompt and retrying. After three retries, the extraction still fails because a required field (contract_reference) is not present anywhere in the source document. What is the correct next step?**

A. Add more specific error feedback about the contract_reference field and retry again  
B. Switch to output_config.format instead of tool_use for better extraction  
C. Make the contract_reference field optional in the schema and handle null downstream  
D. Increase max_tokens to give the model more room to find the field

***Correct answer: C.** The required information is absent from the source document. **This is not a retryable error**; more retries will produce fabricated values or the same failure. The schema should treat the field as optional (nullable), and downstream logic should handle null gracefully.*

**Question 3: A request using strict: true on all 6 tools fails with a 400 error: “Schema is too complex for compilation.” Each tool has 4 required parameters and 5 optional parameters. None of the parameters use union types. Which limit has been exceeded?**

A. The 20 strict-tools-per-request limit  
B. The 24-optional-parameters total limit  
C. The 16 union-type parameters limit  
D. None of the explicit limits; the grammar size limit was exceeded

***Correct answer: B**. Six tools with 5 optional parameters each equals 30 optional parameters total**, exceeding the documented limit of 24**. The 20-tool limit is not exceeded (only 6 tools). The 16 union-type limit does not apply (no union types). The required parameters are a distraction: only parameters absent from* required *count toward the optional total. The rule the arithmetic rests on is that the three limits are cumulative across every strict schema in one request rather than per tool, so no single tool needs to look complex for a request to exceed them.*<sup>[3]</sup>

**Question 4: An extraction pipeline uses tool_choice: "auto" with a schema-constrained extraction tool. In testing, some documents are processed with tool calls and others receive natural language responses. What configuration change guarantees a tool call for every document?**

A. Add strict: true to the tool definition  
B. Change tool_choice to "any" or force the specific tool  
C. Move the schema to output_config.format  
D. Increase the max_tokens parameter to give the model more room

***Correct answer: B.** tool_choice: "auto" allows the model to return conversational text instead of calling a tool. "any" or forced tool selection guarantees a tool call. strict: true controls schema validation, not whether the tool is called.*

## Key Takeaways

- tool_use with ***strict: true*** is a structural contract: the tool input will conform to the schema and the tool name will be valid, but neither is a semantic contract. A schema without strict is a well-described target, not a guarantee; without it the model may return the wrong type or omit a required field. The two structured output features are output_config.format (constrains the response-level JSON) and strict: true (constrains tool input JSON); they address different surfaces and can be combined, and the response-level grammar does not reach tool calls.
- ***tool_choice: "any"*** guarantees a tool call rather than a conversational response; tool_choice: "auto" allows the model to skip the tool entirely. "any" is selected when the document type is unknown and the model must pick the schema; a named tool is selected when one specific extraction must run before later steps. No tool_choice setting can order a sequence: force the prerequisite, then release to follow-up turns. Under manual extended thinking, forced tool use is unavailable.
- Schema design directly affects extraction quality. Optional (nullable) fields prevent fabrication. "unclear" enums give the model a truthful option, and one escape value is enough. "other" + detail fields handle edge cases without misclassification. An empty array and a null are different claims about the source. Normalization rules stated in the prompt run while the source is still visible; post-processing runs blind.
- The validation-retry loop must include specific error feedback: which field, which constraint, what value was found. Generic “try again” gives the model no signal. Constraints such as minimum, maximum, minLength and maxLength are outside the supported schema subset, so range and length checks are application code or they do not exist.
- Triage before retrying. Absent information is not retryable and routes to human review or another source. Format and structural errors retry successfully with the specific constraint attached. Internal inconsistency needs a follow-up request carrying three things: the source document, the failed extraction, and the specific errors. When two independent error sources could explain the same discrepancy, surface it for a person rather than reconciling it algorithmically.
- Self-correction schemas extract both *stated_total* and *calculated_total*, letting the schema itself surface arithmetic conflicts; they are designed redundancy, not a second opinion, and an error running through both derivations survives them. A confidence field is not the same move: it adds a channel with no external referent and does not constrain the extraction. The *detected_pattern* field enables systematic analysis of extraction failures by document structure; without it, dismissal analysis requires manual document review.
- Complexity limits are per-request and cumulative: **20 strict tools maximum, 24 optional parameters total across all schemas, 16 union-type parameters total**, with a 400 reading “Schema is too complex for compilation” when internal grammar limits are hit as well. Required properties appear before optional properties in structured output regardless of schema order. Streaming works with structured outputs; citations and message prefilling do not. The schema guarantee is suspended by a refusal, by truncation at max_tokens, and by enum capitalization, so read stop_reason before parsing content.
