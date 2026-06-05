# Chapter 10: Structured Output and the Validation Loop

**Summary:** ***tool_use** mode is the contract mechanism for machine-parseable output. When a tool definition includes a JSON schema, the model is constrained by compiled grammar to produce output that matches that schema structurally. This is a hard guarantee: no JSON.parse() errors, no missing required fields, no malformed types. But structural compliance is not semantic correctness. A line-item sum that does not match the stated total is structurally valid JSON. A date field containing a plausible-looking but wrong date is structurally valid JSON. The model filled the containers correctly and put the wrong things inside.*

*The production pattern for reliable structured extraction is a three-phase loop: generate using a schema-constrained tool call, validate semantics in application code, retry with specific error feedback if validation fails. Retry specificity is not optional. Generic “there were errors, try again” gives the model no signal about which constraint was violated. “Line items sum to 847.50 but stated_total is 900.00, discrepancy of 52.50” gives the model something it can act on.*

*The chapter also covers the two complementary structured output features (output_config.format and strict: true), schema design patterns that prevent hallucinated values, the documented complexity limits that trigger 400 errors, property ordering behavior, and the streaming incompatibility that catches developers by surprise.*

---

## Scene: The Invoice That Balanced Itself

Scenario 6 in the exam is a structured data extraction pipeline. The developer built it correctly, by most measures. The tool schema is well-designed. The call uses tool_choice to force the extraction tool every time. The response comes back as valid JSON matching the schema exactly. No parse errors. No missing fields. The validator checks the JSON structure and returns clean.

The invoice goes into the database. The line items sum to $847.50. The stated_total field in the document was $900.00. The system stored the discrepancy and nobody caught it because the validator only checked structure.<sup>[1]</sup>

This is the central fact about tool_use as a structured output mechanism: it is a JSON contract, not a correctness contract. **The schema defines what shape the output must take. It says nothing about whether the values inside that shape are accurate representations of the source document.** Those are two completely different guarantees, and conflating them is the most consistent anti-pattern the exam tests in Domain 4.

The named concept for this chapter: **tool_use as JSON contract**.

---

## Two Features, One System

Structured outputs in the Claude API are built from two complementary features that can be used independently or together.<sup>[2]</sup>

The first is output_config.format. This constrains Claude’s response itself (the final text output of the messages call) to a JSON schema. Set type: "json_schema" and provide a schema definition; the model is constrained via compiled grammar to produce output matching that schema in <sup>response.content[0].text</sup>. This is useful when the extraction result is the full response, not a tool call.

The second is strict: true on tool definitions. When a tool definition includes strict: true, the API guarantees schema validation on the tool’s inputs. The model’s tool_use block will always contain JSON matching the declared input_schema for that tool. No extra fields, no missing required fields, no type violations.

Both features use constrained decoding: the API compiles the schema into a grammar that limits which tokens the model can produce at each position. The model is not “trying harder” to comply; the token space available to it at each step is mechanically constrained by the grammar.<sup>[2]</sup>

Used together, they cover different surfaces: output_config.format governs the response-level JSON when the model finishes its tool use loop; strict: true governs each individual tool invocation during that loop. A workflow that needs guaranteed tool inputs AND a guaranteed structured final synthesis can combine them in a single request.

The incompatibility to know: output_config.format and streaming cannot be used together.<sup>[2]</sup> Streaming delivers tokens as they are generated. Constrained decoding operates on the complete grammar, so the structured output guarantee requires the full response to be assembled before delivery. Choose one or the other.

---

## Forcing the Tool Call

A tool definition with a schema does not automatically produce structured output. The model still chooses whether to call the tool or respond conversationally. Set tool_choice: "any" to guarantee the model calls a tool rather than returning text.<sup>[3]</sup>

tool_choice: "any" tells the model it must use one of the provided tools, but leaves the choice of which tool to the model. The behavioral mechanics of "any" and forced tool selection (including assistant-turn prefilling) are defined in Chapter 4; the application here is guaranteeing a tool call in extraction pipelines where a conversational response would be a failure mode. The model goes directly to the tool invocation.<sup>[3]</sup>

For extraction pipelines where the document type is known in advance, force the specific tool:

```
response = client.messages.create(
    model="claude-opus-4-7",
    tools=[extract_invoice_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[{"role": "user", "content": f"Extract: {invoice_text}"}]
)
```

When the document type is unknown and multiple extraction schemas exist, *tool_choice: "any"* lets the model select the appropriate schema while still guaranteeing a tool call happens.<sup>[4]</sup>

The tool_choice values themselves *("auto", "any", forced tool, "none")* are defined in Chapter 4. The concern here is narrower: for structured extraction, "*auto*" is the wrong default because it allows the model to decide that a conversational response is more appropriate than a tool call. If the extraction pipeline depends on always getting structured JSON back, "any" or forced tool is not optional.

---

## Schema Design: Preventing the Model From Inventing Answers

The schema is not just a shape declaration. It is an instruction to the model about what it should and should not fill in. Get the design wrong and the model fills required fields with plausible-sounding fabrications rather than admit the source document doesn’t contain the information.

Three patterns from the exam guide address this directly.<sup>[4]</sup>

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

These three patterns share a common goal: reduce the pressure on the model to produce a confident answer when the source material does not support one. The schema should make it easy to say “I don’t know” or “this doesn’t fit.”

---

## Property Ordering: The Behavior Nobody Reads About Until It Breaks Something

The structured output documentation specifies property ordering behavior that is not intuitive.<sup>[2]</sup>

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

---

## The Validation Loop

The schema guarantee eliminates an entire class of failures: JSON syntax errors, missing required fields, wrong types. What it cannot eliminate is the class of failures that live above the schema level. Values in the wrong fields. Sums that don’t add up. Dates formatted correctly but extracted from the wrong part of the document. Vendor names from the footer instead of the header.

These are semantic errors. The JSON is valid. The schema is satisfied. The data is wrong.

Application code is the only place semantic validation can happen. The validation loop is the production pattern for catching and correcting these failures.<sup>[4]</sup>

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

The right version tells the model which field failed, what the arithmetic shows, and where to look. The model can act on that. It cannot act on “there were errors.”<sup>[4]</sup>

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

---

## Self-Correction Fields: Building the Check Into the Schema

The validation loop is reactive: *extract, check externally, retry if wrong*. A complementary approach is to build the self-check into the schema itself, so the extraction produces both the answer and the material for verifying it.<sup>[4]</sup>

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

The detected_pattern field serves a different purpose: it captures metadata about the extraction itself rather than the document content.<sup>[4]</sup>

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

---

## When Retries Are Not the Answer

Retries address format mismatches and structural errors. They do not address missing information.

If the source document does not contain the data required by the schema, retrying will not produce it. The model will either fabricate a plausible value (bad), return null for optional fields (acceptable), or fill the field with content from somewhere else in the document (bad and hard to detect). More retries, more specific error messages: none of it helps when the constraint is the source material itself.<sup>[4]</sup>

Distinguish these two failure modes:

**Retryable:** The model extracted the date in MM/DD/YYYY format but the schema requires ISO 8601. The date is in the document. The extraction got the format wrong. A retry with “date must be YYYY-MM-DD format” will fix this.

**Not retryable:** The schema requires a purchase order number, but this vendor does not use PO numbers and the document has none. The model extracted something from the document that looks like a number. Retrying will produce a different wrong answer.

The diagnostic question: *is the error due to how the model extracted the information, or due to the information not being present?* The answer determines whether to retry or to route the document for human review or accept null.

The exam tests this distinction directly. The correct response to “*required field is absent from source document*” is not more retries with more specific errors. It is recognizing that the information constraint is architectural: the schema requires data the document does not have. Address it at the schema level (make the field optional) or the pipeline level (route document type to a different schema).<sup>[4]</sup>

---

## Complexity Limits: The 400 Error Nobody Expects

Constrained decoding works by compiling schemas into grammars. Complex schemas produce large grammars, and large grammars take longer to compile. The API enforces explicit limits to protect against excessive compilation times.<sup>[2]</sup>

These limits apply to all requests with output_config.format or strict: true:

| Limit | Value | Description |
| --- | --- | --- |
| Strict tools per request | 20 | Maximum tools with strict: true. Non-strict tools don’t count. |
| Optional parameters total | 24 | Total optional parameters across all strict schemas and JSON output schemas. Each parameter not in required counts. |
| Union type parameters total | 16 | Parameters using anyOf or type arrays (e.g., "type": ["string", "null"]). |

The per-request nature of the optional parameter limit is the trap. **Four tools with six optional parameters each adds up to 24: exactly at the limit, with no room.** The limits apply to the combined total across all strict schemas in a single request.<sup>[2]</sup>

Beyond the explicit limits, internal grammar size limits can trigger a 400 error even when all three explicit limits are satisfied. Schema complexity does not reduce to a single dimension: optional parameters, union types, nested objects, and the number of strict tools interact with each other. The compiled grammar can grow disproportionately large from combinations of these features. The API enforces a 180-second compilation timeout as a final stop, and schemas that pass all checks but produce very large grammars may hit it.

When hitting complexity limits, the documented strategies in order of preference:<sup>[2]</sup>

1. Mark only critical tools as strict. If the schema violation for a simpler tool is tolerable (the model naturally produces valid inputs anyway), reserve strict: true for the tools where violations cause real problems.
2. Reduce optional parameters. Each optional parameter roughly doubles a portion of the grammar’s state space. Making a parameter required and having the model supply a default explicitly is cheaper than leaving it optional.
3. Flatten nested structures. Deeply nested objects with optional fields compound complexity.
4. Split across multiple requests or subagents. If the tool set is too large for a single strict request, distribute the tools across separate calls or subagent invocations.

**The 400 error message is “Schema is too complex for compilation.” When it appears, the cause is schema complexity, not a malformed request.**

---

## Schema Compliance Without Semantic Correctness: The Core Exam Distinction

The exam scenario walkthrough for Scenario 6 states the distinction in unambiguous terms: tool_use guarantees structure, not semantics.<sup>[1]</sup>

Structure: the JSON is valid, the required fields are present, the types match, additionalProperties is respected.

Semantics: the values inside the structure accurately represent the source document.

The structural guarantee comes from constrained decoding, a mechanical property of how the API generates tokens. The semantic guarantee comes from the validation loop, a property of the application code that checks the extracted values against business rules, arithmetic constraints, and domain knowledge.

A system that trusts the structural guarantee and skips semantic validation is making an architectural error. The schema says the output is shaped correctly. It says nothing about whether the values are right.

The production architecture combines both layers. Use tool_use with strict: true to eliminate the entire class of JSON syntax failures. Use application validation to catch semantic errors the schema cannot express. Use the retry loop to correct format mismatches and structural errors in the extraction. Know when to route for human review instead of retrying.

This is not a critique of structured output as a mechanism. The structural guarantee is valuable precisely because it eliminates one entire class of failures, which makes the remaining failures (semantic) easier to reason about. The error isn’t trusting tool_use. The error is trusting it for more than it guarantees. Every tool schema in the request contributes to context window cost; Chapter 11 covers the architectural implications of that budget.

---

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

***Correct answer: B**. Six tools with 5 optional parameters each equals 30 optional parameters total**, exceeding the documented limit of 24**. The 20-tool limit is not exceeded (only 6 tools). The 16 union-type limit does not apply (no union types). This question tests whether candidates can calculate combined totals across all strict schemas, not just recall individual limit numbers.*<sup>[1]</sup>

**Question 4: An extraction pipeline uses tool_choice: "auto" with a schema-constrained extraction tool. In testing, some documents are processed with tool calls and others receive natural language responses. What configuration change guarantees a tool call for every document?**

A. Add strict: true to the tool definition 
B. Change tool_choice to "any" or force the specific tool 
C. Move the schema to output_config.format 
D. Increase the max_tokens parameter to give the model more room

***Correct answer: B.** tool_choice: "auto" allows the model to return conversational text instead of calling a tool. "any" or forced tool selection guarantees a tool call. strict: true controls schema validation, not whether the tool is called.*

---

## Key Takeaways

- tool_use with ***strict:true*** is a structural contract: the output will match the JSON schema, but it is not a semantic contract. The two structured output features are output_config.format (constrains the response-level JSON) and strict: true (constrains tool input JSON); they address different surfaces and can be combined.
- ***tool_choice: "any"*** guarantees a tool call rather than a conversational response. Without it, tool_choice: "auto" allows the model to skip the tool entirely.
- Schema design directly affects extraction quality. Optional (nullable) fields prevent fabrication. "unclear" enums give the model a truthful option. "other" + detail fields handle edge cases without misclassification.
- The validation-retry loop must include specific error feedback: which field, which constraint, what value was found. Generic “try again” gives the model no signal.
- Retries fix format mismatches and structural errors. They do not fix missing information. When the required data is not in the source document, the fix is schema design (make the field optional) or routing (send document for human review), not more retries.
- Self-correction schemas extract both *stated_total* and *calculated_total*, letting the schema itself surface arithmetic conflicts. The *detected_pattern* field enables systematic analysis of extraction failures by document structure; without it, dismissal analysis requires manual document review.
- Complexity limits are per-request and cumulative: **20 strict tools maximum, 24 optional parameters total across all schemas, 16 union-type parameters total**. Required properties appear before optional properties in structured output regardless of schema order. Streaming and output_config.format cannot be used together.
