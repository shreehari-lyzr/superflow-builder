---
name: superflow
description: Write superflow JSON workflows — DAGs of code/HTTP/LLM/HITL nodes that execute durably under Restate. Use this skill when you need to build, debug, or extend a superflow workflow file.
license: MIT
metadata:
  author: lyzr
  version: "1.0.0"
  category: workflow-engineering
---

# Writing Superflows

Superflows are workflow JSON files executed by a DAG engine with Restate-backed durability. **Every workflow triggered from the frontend runs durably through Restate** — design accordingly.

## When to use this skill

- Building a new workflow JSON for the platform
- Diagnosing why a workflow doesn't produce expected output
- Adding nodes to an existing flow
- Writing demo flows for stakeholders

## File shape

```json
{
  "name": "Display Name",
  "nodes": [ { "id": "...", "name": "...", "type": "...", "typeVersion": N, "parameters": { ... }, "position": [x, y] }, ... ],
  "connections": { "Source Name": { "main": [ [ {"node": "Target Name", "type": "main", "index": 0} ] ] }, ... },
  "settings": {}
}
```

Hard rules:
- **Exactly one trigger node.** Parser rejects 0 or ≥2. Type `lyzr-nodes-base.trigger`.
- **Reference nodes by `name`, never `id`.** The `connections` map keys on `name`. `id` is local-only.
- **`position` is `[x, y]`** — required for UI layout. Pick non-overlapping coordinates.
- **Settings can be `{}`.**

## The core rule for data flow

A Code node sees only what is delivered to it through its **input edges**. Always read inputs via:

- `$input.first()` — the first input item (use when there's one direct parent)
- `$input.all()` — all input items concatenated across all parents (use when multiple parents converge)
- `$json` — shorthand for `$input.first()`

To pass data to a downstream Code node, **wire an edge from the source to the consumer**. If a node needs context from multiple upstream points, wire each as a parent and identify them in `$input.all()` by a distinguishing field.

```js
// Single parent
const packet = $input.first();
[{ ...packet, processed: true }]

// Multiple parents converging — identify each by a unique key
const items   = $input.all();
const ctx     = items.find(i => i && i.run_id)        || {};
const erp     = items.find(i => i && i.vendor_master) || {};
const bank    = items.find(i => i && i.transactions)  || {};
[{ run_id: ctx.run_id, vendor_master: erp.vendor_master, bank_debits: bank.transactions }]
```

For template strings in non-Code parameters (LLM `prompt`, HTTP `jsonBody`, approval `message`, etc.), use `{{ ... }}` expressions — see the Expression engine section below.

## Node-type registry

All registered node types are listed below. The parameter contract for each is what its executor actually reads — anything else in `parameters` is ignored.

### `lyzr-nodes-base.trigger`
No params used. Passes input items through or emits one empty `{}` if none.

### `lyzr-nodes-base.code`
```json
{
  "type": "lyzr-nodes-base.code",
  "typeVersion": 2,
  "parameters": { "jsCode": "..." }
}
```

**Key contract:**
- Param name is **`jsCode`**.
- **No top-level `return`.** The script's last expression is the completion value. `return` inside callbacks (`.map(p => p+1)`, `function(x) { return ... }`) is fine.
- Must return an **array of plain objects**: `[{key: val}, ...]`. Returning an array of primitives errors with `"code node must return an array of objects"`.
- 10-second execution timeout.
- Read inputs via `$input.first()`, `$input.all()`, or `$json`.
- Available built-ins: `Date`, `Math`, `JSON`, `Object`, `Array.prototype.*`, template literals, arrow functions, spread, `find`/`filter`/`flatMap`, `Object.assign`, `console.log`/`console.warn`.

**UUID-style IDs:**
```js
`run-${Date.now()}-${Math.random().toString(36).slice(2, 10)}`
```

**Conditional last-expression (use ternary or IIFE):**
```js
const result = cond
  ? { status: 'skip' }
  : (() => { /* logic */; return { status: 'done', detail }; })();
[result]
```

### `lyzr-nodes-base.set` (v3)
```json
"parameters": {
  "assignments": {
    "assignments": [
      {"name": "field", "value": "literal or =expression", "type": "string"}
    ]
  }
}
```
- Adds fields to **each input item** (preserves other fields).
- `type` is metadata-only — values pass through as JS values.
- Expression resolver runs on `value` strings — use `"={{ $json.x }}"` to interpolate.

### `lyzr-nodes-base.httpRequest` (v4)
```json
{
  "method": "GET|POST|PUT|PATCH|DELETE",
  "url": "https://...",
  "sendQuery": true,
  "queryParameters": { "parameters": [ {"name": "...", "value": "..."} ] },
  "sendHeaders": true,
  "headerParameters": { "parameters": [ {"name": "...", "value": "..."} ] },
  "sendBody": true,
  "contentType": "json | multipart-form-data",
  "jsonBody": "<stringified JSON>",
  "responseFormat": "json | text | blob"
}
```
- `body` does **not** accept a map. Use `jsonBody` (a string — build it in a preceding Code node) or `bodyParameters.parameters: [{name, value}]`.
- `sendBody` defaults to true for POST/PUT/PATCH.
- Output item: `{statusCode, headers, body}`. `body` is JSON-decoded unless `responseFormat` is `text` or `blob`.
- The upstream input item is **not** included in the output — only the response shape. To preserve data across an HTTP call, see the HTTP body preservation section below.
- Auth via `parameters.auth` — supported shapes: Bearer, Basic, API Key, OAuth2 client_credentials, AWS SigV4, JWT bearer, mTLS.

### `lyzr-nodes-base.if` (v2)
```json
"parameters": {
  "conditions": {
    "conditions": [
      {
        "leftValue": "={{ $json.x }}",
        "rightValue": 0,
        "operator": {"operation": "gt"},
        "leftType": "number",
        "rightType": "number"
      }
    ],
    "combineOperation": "and"
  }
}
```
- Two outputs: **0 = true**, **1 = false**.
- Operators: `equals`, `notEquals`, `gt`, `lt`, `gte`, `lte` (also `greaterThan`/`lessThan`/`greaterThanOrEqual`/`lessThanOrEqual`), `contains`, `notContains`, `startsWith`, `endsWith`, `exists`, `notExists`, `isEmpty`, `isNotEmpty`.
- `combineOperation`: `and` | `or`. Lives inside the `conditions` object.
- `leftType` / `rightType` force coercion to `"string"` | `"number"` | `"boolean"`. Omit and the executor infers from operator/values.
- AI mode: set `evaluationMode: "ai"` + `conditionPrompt` to use an LLM for the boolean.

### `lyzr-nodes-base.switch` (v3)
```json
"parameters": {
  "rules": {
    "rules": [
      { "conditions": { "conditions": [ {...same shape as If...} ], "combineOperation": "and" } },
      { "conditions": { "conditions": [ {...} ], "combineOperation": "and" } }
    ]
  }
}
```
- Output index 0..N matches rule order. Unmatched items go to output `len(rules)` (the implicit default).
- Same operators as If.

### `lyzr-nodes-base.merge`
- Mode `append`: concatenates items from multiple inputs.
- Useful when paths re-converge after If/Switch routing.

### `lyzr-nodes-base.aggregate`
Two modes — for anything non-trivial, prefer a Code node.
- `aggregate: "aggregateIndividualFields"` + `fieldsToAggregate: [{fieldToAggregate, renameField?}]` → collects per-field values across items into arrays.
- `aggregate: "aggregateAllItemData"` + `destinationFieldName: "data"` → wraps all items into one array under that field.

### `lyzr-nodes-base.splitInBatches` (Loop)
```json
"parameters": {
  "mode": "field | batches",
  "list": "={{ $json.items }}",     // field mode
  "batchSize": 10                    // batches mode
}
```
- **Field mode**: iterates the `list` expression's array. Body sees `{item, index, parent}`.
- **Batches mode**: splits inputs by `batchSize`.
- Body runs inline in the parent `ExecutionContext`.

### `lyzr-nodes-base.filter`
```json
"parameters": {
  "sourceArray": "={{ $json.items }}",
  "condition": "={{ $json.x > 5 }}"
}
```

### `lyzr-nodes-base.stopAndError`
```json
"parameters": { "errorMessage": "Halted: {{ $json.reason }}" }
```
Terminal failure — no retry. Expression-resolved.

### `lyzr-nodes-base.wait`
```json
"parameters": { "amount": 3, "unit": "minutes | hours | businessDays | ..." }
```
Durable sleep via Restate.

### `lyzr-nodes-base.waitForApproval` (HITL)
```json
"parameters": {
  "message": "Approve {{ $json.packet_id }}?",
  "subject": "Approval needed",
  "notifyEmails": ["a@b.com"],
  "formSchema": [
    {"name": "approved",      "label": "Approve?", "type": "boolean", "requiredOn": "both"},
    {"name": "reject_reason", "label": "Reason",   "type": "string",  "requiredOn": "reject"}
  ]
}
```
- Two outputs: **0 = approved (form data merged into items)**, **1 = rejected**.
- `requiredOn`: `never` (default) | `approve` | `reject` | `both`. Server-side enforced.
- `notifyEmails`: array (or comma-separated string). Sends SMTP email with a link to `{STUDIO_URL}/superflow/approvals/{id}` when SMTP env vars are configured.

### `lyzr-nodes-base.dateTime`
```json
"parameters": {
  "action": "calculate",
  "outputs": [ {"name": "now_iso", "expression": "..."} ]
}
```
For most cases a Code node with `new Date().toISOString()` is simpler.

### `lyzr-nodes-base.crypto`
```json
"parameters": {
  "action": "hash",
  "algorithm": "SHA-256 | SHA-512 | ...",
  "inputExpression": "={{ JSON.stringify($json) }}",
  "outputKey": "hash"
}
```

### `lyzr-nodes-base.renameKeys`
```json
"parameters": { "renames": [{"from": "wire_amount", "to": "transaction_amount"}] }
```

### `lyzr-nodes-base.lyzr.llm`
```json
"parameters": {
  "provider": "openai",
  "model": "gpt-4o",
  "systemPrompt": "...",
  "prompt": "Resolved at exec time: {{ JSON.stringify($json) }}",
  "responseFormat": {"type": "json_object"},
  "temperature": 0.2,
  "maxTokens": 1000
}
```
- If `prompt` is omitted, pulls the user message from input-item keys: `chatInput`, `prompt`, `query`, `input`, `message`, `text`, `output` (first non-empty).
- Output: input item merged with `{status, output, input_tokens, output_tokens}`. `output` is parsed if `responseFormat` is a JSON schema, else raw string.
- Requires either `agent_id` (loads from DB) OR `provider` + `model` + API-key (env var or credential).
- Canonical provider names: `openai`, `anthropic`, `google`, `vertex_ai`, `azure`, `aws-bedrock`, `deepseek`, `groq`, `perplexity`, `xai`, `huggingface`, `nvidia`.

### `lyzr-nodes-base.lyzr.agent`
Same shape as `lyzr.llm` but always runs ReAct. Downstream `lyzr.agent` / `lyzr.llm` / `lyzr.tool` nodes auto-become delegate tools when marked `isSubAgent: true`.

### `lyzr-nodes-base.lyzr.taskDecomposition` (AI Swarm)
LLM-driven decomposition: the LLM breaks the user message into subtasks, spawns sub-agents in parallel, then aggregates the results.

For *deterministic* parallel work (3 HTTP calls, 5 validation checks), prefer **DAG fan-out** — one upstream node with multiple downstream siblings runs them concurrently and is easier to reason about.

### `lyzr-nodes-base.lyzr.parse`
```json
"parameters": {
  "tier": "standard | advanced | agentic",
  "vlm_provider": "google",
  "vlm_model": "gemini-3-pro-preview",
  "fileUrlExpression": "={{ $json.file_url }}"
}
```
Calls RAG at `RAG_API_URL`. Output: `{status, tier, file_url, file_type, chunks[], pages[], full_text, metadata}`.

### `lyzr-nodes-base.lyzr.extraction`
```json
"parameters": {
  "tier": "standard | advanced | agentic",
  "target": "per_doc | per_page | per_table_row",
  "extraction_schema": "{\"type\":\"object\",\"properties\":{...}}"
}
```
- **`extraction_schema` must be a JSON-stringified string**, not an inline object.
- Reads `file_url` from the upstream input item. Short-circuits the download if `full_text` is already present (e.g. from an upstream Parse node).

### `lyzr-nodes-base.lyzr.label`
```json
"parameters": {
  "textExpression": "={{ $json.full_text || JSON.stringify($json) }}",
  "rules": [ {"type": "label_a", "description": "When ..."} ]
}
```

### `lyzr-nodes-base.lyzr.tool`
Direct platform tool invocation (no LLM).

## Connections

```json
"connections": {
  "Source Node": {
    "main": [
      [  // output 0
        {"node": "Target A", "type": "main", "index": 0},
        {"node": "Target B", "type": "main", "index": 0}   // fan-out from output 0
      ],
      [  // output 1
        {"node": "Target C", "type": "main", "index": 0}
      ]
    ]
  }
}
```

Mental model:
- `main: [[...], [...]]` — outer index is the **output port** (0, 1, 2…). Switch / If / approval nodes use multiple output ports.
- Each inner array is the **list of destinations from that output port**. Multiple entries = fan-out (parallel siblings).
- `index` on each destination is the **input port** on the target. Most nodes only have input port 0; Merge can have multiple.
- Empty arrays (`[]`) are valid — a missing path simply has no downstream.

**Fan-out runs concurrently.** Siblings from the same output port are dispatched in parallel.

**Convergence waits for all parents** (the engine holds a node until all incoming edges resolve). Exception: when an upstream If/Switch routes to a different output, the unused edges are "empty" and the node still fires if at least one input has data.

## Expression engine

Inside any string parameter value, `{{ ... }}` is evaluated.

```
{{ $json.field }}                       — current node's first input item, field path
{{ $json }}                              — entire first input item
{{ $('Other Node').json.field }}        — output of named node (single or double quotes)
{{ $node["Other Node"].json.field }}    — alternate syntax
{{ $input.item.json.field }}            — same as $json.field
```

For complex expressions, the engine substitutes references inline as JSON literals, then evaluates the result as JS:
```
"prompt": "Stats: {{ JSON.stringify($('Matcher').json.stats) }}"
"jsonBody": "={{ JSON.stringify({batch: $json.output, note: $json.note}) }}"
```

**Single full-expression strings preserve types.** `"={{ $json.count }}"` returns the number, not a string. Mixed strings interpolate as text.

**Helpers available inside `{{ }}`:** plain JS only — `JSON`, `Date`, `Math`, etc. There is no `$now`, `$today`, `$env`, or `$crypto`. For dynamic values, use a Code node.

## DAG patterns

### Fan-out from one node
```json
"Init": { "main": [[
  {"node": "Stream A", "type": "main", "index": 0},
  {"node": "Stream B", "type": "main", "index": 0},
  {"node": "Stream C", "type": "main", "index": 0}
]] }
```
Three siblings run in parallel.

### Convergence at one node
```json
"Stream A": { "main": [[{"node": "Aggregator", "type": "main", "index": 0}]] },
"Stream B": { "main": [[{"node": "Aggregator", "type": "main", "index": 0}]] },
"Stream C": { "main": [[{"node": "Aggregator", "type": "main", "index": 0}]] }
```
Inside Aggregator's Code node, `$input.all()` returns the concatenated items from all three. Use predicates to identify each. The engine waits for all three before running Aggregator.

### Pass context through to a convergence point
If a downstream Aggregator needs data from `Init` that didn't flow through the fan-out, wire `Init → Aggregator` as an additional edge:
```json
"Init": { "main": [[
  {"node": "Stream A",   "type": "main", "index": 0},
  {"node": "Stream B",   "type": "main", "index": 0},
  {"node": "Stream C",   "type": "main", "index": 0},
  {"node": "Aggregator", "type": "main", "index": 0}
]] }
```
Aggregator now receives four items in `$input.all()` and can identify each by a distinguishing field.

### Two If/Switch paths converging
Route multiple branches to the same downstream node — or use a Merge node.
```json
"Switch": {
  "main": [
    [{"node": "Approval", "type": "main", "index": 0}],
    [{"node": "Approval", "type": "main", "index": 0}],
    [{"node": "Stop",     "type": "main", "index": 0}]
  ]
}
```
Approval fires when either of the first two paths delivers data.

## HTTP body preservation

The HTTP node's output is only `{statusCode, headers, body}` — the upstream input item is not carried through. To preserve packet data across an HTTP call, use one of:

**Option A — Echo through the body** (works with `jsonplaceholder.typicode.com/posts`, `httpbin.org/post`, and any echo endpoint that returns your body):
```js
// Pre-HTTP Code node
const packet = $input.first();
[{ json_body: JSON.stringify(packet) }]
```
```json
// HTTP node
"jsonBody": "={{ $json.json_body }}"
```
```js
// Post-HTTP Code node — packet comes back in response.body
const resp = $input.first();
const packet = resp.body || {};
[Object.assign({}, packet, { txn_ref: '...', executed_at: new Date().toISOString() })]
```

**Option B — Wire the pre-HTTP source as an additional parent of the post-HTTP consumer.** The consumer reads both the pre-HTTP packet and the HTTP response from `$input.all()`, identifying each by a distinguishing key.

## HITL approval pattern

```
[Build summary (LLM or code)]
   → [waitForApproval]
       ├─ output 0 (approved) → [Code: build wire body] → [HTTP] → [If success] → [audit chain]
       └─ output 1 (rejected) → [stopAndError]
```

- The approval node merges form data onto items going to output 0. If the form schema is `{approved, override_note}`, the next node sees `$json.override_note` plus everything that was on the input.
- Form rendering happens in Studio; `requiredOn` is enforced server-side too.
- During replay, the approval awakeable resolves with the stored data — no re-prompt.

## Demo / mock-data strategy

For demos you usually want:
1. No external API dependencies (no real ERP, bank, etc.)
2. Deterministic data so the walk-through always lands at the same verdict
3. The HTTP node visibly executing at least once

Recipes:

**Mock data via Code nodes.** Each "fetch" becomes a Code node emitting hardcoded `[{...}]`. Wire them parallel via fan-out for the "AI Swarm" narrative.

**HTTP echo via jsonplaceholder.** Use `POST https://jsonplaceholder.typicode.com/posts` to fake any external write. It echoes the body back with an added `id: 101`, so downstream nodes can read `response.body.<field>` as if the API processed your request.

**Designed-failure variants.** Bake a config flag into the Init Code node that downstream validators read, so you can flip mock data between "happy path" / "warning path" / "fraud halt" by editing one line.

## Validation checklist

Before considering a workflow file done:

- [ ] Exactly one `lyzr-nodes-base.trigger` node.
- [ ] Every node has a unique `name`. Connections reference names, not `id`.
- [ ] Every Code node's `jsCode` ends with a bare expression — no top-level `return`.
- [ ] Code nodes return `[{...}, ...]` arrays of plain objects.
- [ ] Code nodes read inputs only via `$input.first()` / `$input.all()` / `$json`. Cross-node data flows through wired edges.
- [ ] HTTP nodes use `sendBody: true` + `jsonBody: "<string>"` (or `bodyParameters`), not a raw `body` map.
- [ ] HTTP nodes that need to preserve packet context send the packet *in the body* or have an additional parent edge wired to the consumer.
- [ ] If/Switch operators use the documented names (`gt`, `lt`, `equals`, etc.).
- [ ] `extraction_schema` is a JSON-stringified string, not an inline object.
- [ ] `notifyEmails` is an array (or comma string), not a single string.
- [ ] Approval `formSchema` uses `requiredOn`, not bare `required`.
- [ ] Dynamic values (timestamps, UUIDs, env-driven values) come from a Code node, not invented expression helpers.
- [ ] `taskDecomposition` only used when LLM-driven subtask spawning is actually wanted. For deterministic parallel work, fan out the DAG.
- [ ] File passes `python3 -c "import json; json.load(open('file.json'))"`.

## Dry-run before delivery

Whatever harness the engine exposes for offline execution, use it to walk the workflow up to the first `waitForApproval` and confirm each node's output matches expectations. This catches JS errors, missing fields, and routing mistakes before deployment.

When dry-running, truncate the workflow at `waitForApproval` to test the upstream chain. Substitute Code nodes that emit mock LLM outputs (`{output: {…}, status: 'completed', input_tokens: 0, output_tokens: 0}`) where you'd otherwise hit a model that needs credentials.
