# Rules

## Must Always
- Read Code-node inputs via `$input.first()`, `$input.all()`, or `$json` only. Cross-node data flows through wired edges.
- End every Code-node `jsCode` with a bare expression — never a top-level `return`.
- Return Code-node values as `[{...}, ...]` arrays of plain objects.
- Use documented operator names: `gt`, `lt`, `gte`, `lte`, `equals`, `notEquals`, `contains`, `startsWith`, `endsWith`, `exists`, `notExists`, `isEmpty`, `isNotEmpty`.
- Pass `extraction_schema` as a JSON-stringified string, not an inline object.
- Use `sendBody: true` + `jsonBody: "<string>"` (or `bodyParameters.parameters`) on HTTP nodes — never a raw `body` map.
- Reference nodes by `name` (not `id`) in `connections`.
- Include exactly one `lyzr-nodes-base.trigger` node per workflow.
- Validate JSON parseability before delivery (`python3 -c "import json; json.load(open('file.json'))"` or equivalent).
- When uncertain about a node-type's parameter shape, read the corresponding executor in `internal/executors/` before writing.

## Must Never
- Use invented expression helpers: `$now`, `$today`, `$env`, `$crypto.uuid()`. Dynamic values come from a Code node.
- Return arrays of primitives from Code nodes — they error.
- Configure Aggregate with `combineAll`, `outputKey`, or `namedInputs`. Its only modes are `aggregateIndividualFields` and `aggregateAllItemData`.
- Pass `taskDecomposition` a fixed `subtasks: []` array. It is LLM-driven only; for deterministic parallel work, use DAG fan-out.
- Use Set node `value` strings with assumed type coercion — the `type` field is metadata only.
- Embed schemas, prompts, or bodies as multi-line JS template literals when a Code node could build them more clearly.
- Add nodes whose types aren't registered in [internal/executors/registry.go](../../internal/executors/registry.go).
- Skip mock-data design for demo workflows — random data produces unpredictable demo paths.

## Output Constraints
- Deliver the workflow as a single JSON file at a path the user can open.
- Accompany each deliverable with a short narrative: trigger → key phases → mock-data verdict path.
- For demo flows, document the one-line edit that flips between happy/warning/halt paths.
- Keep individual Code-node `jsCode` under ~50 lines. Split into multiple nodes if the logic grows.

## Interaction Boundaries
- Do not execute the workflow against production endpoints. Use mock Code nodes or echo endpoints (`jsonplaceholder.typicode.com/posts`) for demos.
- Do not invent new node types or parameter fields. If a needed capability isn't available, name the gap and propose either a Code-node workaround or a feature request.
- Do not modify the inference-svc engine source as part of delivering a workflow — workflows must work within the engine's current contracts.
