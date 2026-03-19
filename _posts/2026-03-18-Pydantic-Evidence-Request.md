# Building an Evidence Request Ingestion Flow with Pydantic Validation

## Intro
I built a new evidence request page that accepts freeform request text, parses it into structured fields, and saves validated records into a database table. The main goal was to learn how to use a Python parser powered by Pydantic, which gives us a strict, predictable schema boundary between unstructured user text and database writes.

## The Problem
During audits evidence requests come in as text, which needs to be parsed by a human to turn them into formal requests that can be tracked.

Pain points:
- Free text is messy and inconsistent.
- Parsing logic can produce partial or malformed output.
- DB writes fail late if required fields are missing/invalid.
- Semantic matching can fail silently without clear error shape.

![Evidence Request](/images/evidence_request.png)

## Implementation Notes
- **What is Pydantic?**
  - Pydantic is a Python data-validation and settings library built around type hints.
  - You define models (`BaseModel`) with typed fields and constraints.
  - At runtime, Pydantic validates/coerces input and raises structured `ValidationError` if data is invalid.
- In this implementation:
  - `ParserInput` enforces non-empty bounded text.
  - `LlmExtraction` and `ParserOutput` enforce field types and confidence bounds (`0..1`).
  - Validation errors are returned as field-level JSON errors for the UI.
- Guardrails used:
  - Description truncation before insert.
  - System fallback to `"unknown"`.
  - Multi-stage control resolution to avoid hard failures.

![Request Submitted](/images/request_submitted.png)

## Pydantic Use Cases
**1. Validating LLM responses**
- Enforce a strict schema on model outputs (e.g., JSON with required fields) to prevent malformed or hallucinated structures from entering your pipeline  
- Automatically coerce and validate types (strings → ints, enums, dates), raising clear errors or enabling retries when outputs don’t match expectations  

**2. Passing a model in an API call**
- Use Pydantic models as request/response contracts so your API always receives and returns well-structured, type-safe data  
- Enable automatic serialization (`.model_dump()`) and validation, reducing boilerplate and preventing invalid payloads from reaching business logic  

**3. Tool calling**
- Define tool input schemas with Pydantic to guarantee the LLM generates correctly structured arguments for each tool  
- Validate and parse tool outputs into typed objects, making downstream processing (DB writes, chaining tools) reliable and predictable

## The Solution
I added a dedicated request intake path with two layers:
- A **Python parsing script** that validates and normalizes request data with Pydantic.
- A **Next.js API route** that resolves control mappings and inserts into Supabase.

The parser extracts structural data about the evidence request:
- `reference_code`
- `description`
- `system_name`
- `confidence`
- `method` (`explicit`, `semantic`, `fallback`)

Then the API resolves control UUIDs and writes:
- `audit_id`
- `control_id`
- `description`
- `collected = false`
- `system_name`

## How It Works
### 1) UI Capture
A dashboard page collects:
- Editable `audit_id` (defaulted)
- Freeform evidence-request text

### 2) Python Parse + Pydantic Validation
The API calls `python-scripts/parse_evidence_request.py`, which:
- Validates input payload shape using `ParserInput`.
- Extracts explicit control code when present.
- Extracts/normalizes `system_name` from labeled and natural phrases.
- Uses semantic inference when explicit code is absent.
- Returns strict JSON validated by `ParserOutput`.

### 3) Control Resolution + Semantic Matching
The API route attempts control resolution in this order:
1. Explicit reference-code mapping (`mapping_references -> mappings.control_id`)
2. Semantic matching using stored vectors in `controls.specification_embedding`
3. Semantic fallback with on-the-fly control embeddings (if stored vectors unavailable)
4. `AC-1` fallback mapping as last resort

### 4) Persist + Return Debug Signals
On success, it inserts into `evidence_request` and returns metadata, including:
- `semantic_source` (`stored` or `on_the_fly`)
- `semantic_similarity_score`

### Simple Flow Diagram
```text
UI Text Input
   |
   v
API (/api/evidence/request)
   |
   +--> Python parser (Pydantic)
   |        -> {reference_code, description, system_name, confidence, method}
   |
   +--> Control resolver
   |      explicit map -> semantic(stored vectors) -> semantic(on-the-fly) -> AC-1
   |
   v
Database insert (evidence_request)
   |
   v
UI success + debug metadata
```

## Example / Walkthrough
Input:
- `Request for password policy data related to the Outlook system.`

Expected behavior:
1. Parser normalizes description and system hints.
2. If explicit code is absent, API computes semantic nearest control from control specifications.
3. API inserts request with matched `control_id` UUID.
4. UI shows reference code and the matched control specification in “Last Created Request”.
