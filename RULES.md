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
- Build the workflow as a native object and serialize it with a real JSON encoder (`json.dump` / `JSON.stringify`) when a tool is available — never hand-type the final JSON. Author every `jsCode`/`prompt`/`jsonBody`/`extraction_schema` as a normal multi-line string and let the encoder escape it.
- Run the parse gate before delivery and treat it as blocking: `python3 -c "import json,sys; json.load(open(sys.argv[1])); print('PARSE OK')" superflow.json` must print `PARSE OK`. On any error, fix at the reported line:column per the repair table in the skill's "Parse-and-repair gate" section and re-run — loop until clean. No execution environment: char-scan every string value for raw control characters first, and say you could not run the gate.
- When uncertain about a node-type's parameter shape, consult the engine's executor source before writing.

## Must Never
- Use invented expression helpers: `$now`, `$today`, `$env`, `$crypto.uuid()`. Dynamic values come from a Code node.
- Return arrays of primitives from Code nodes — they error.
- Configure Aggregate with `combineAll`, `outputKey`, or `namedInputs`. Its only modes are `aggregateIndividualFields` and `aggregateAllItemData`.
- Pass `taskDecomposition` a fixed `subtasks: []` array. It is LLM-driven only; for deterministic parallel work, use DAG fan-out.
- Use Set node `value` strings with assumed type coercion — the `type` field is metadata only.
- Embed schemas, prompts, or bodies as multi-line JS template literals when a Code node could build them more clearly.
- Add nodes whose types aren't registered in the engine's executor registry.
- Skip mock-data design for demo workflows — random data produces unpredictable demo paths.
- Leave a raw line break, tab, or unescaped `"`/`\` inside any JSON string value (`jsCode`, `systemPrompt`, `prompt`, `jsonBody`, `extraction_schema`, `message`). Inside JSON strings newlines are `\n` — a raw one triggers `"Bad control character in string literal"`.
- Offload the parse gate onto the user. Never close with "save it and run the parse gate," "share the `line:column` and I'll fix it," or any variant — the Studio importer validates with `JSON.parse` and surfaces only a generic "Invalid JSON" with no position, so the user cannot relay a useful error. You run the gate yourself and deliver only JSON that printed `PARSE OK`.
- Dump the full workflow JSON into the terminal as the user's copy source. The deliverable is the clipboard + the `superflow.json` file; the terminal gets a summary + absolute path. Copying a soft-wrapped blob out of the terminal can re-create the `Bad control character` bug at paste. Print the full JSON only on explicit request, and warn that terminal copy may corrupt it.

## Output Constraints
- Default delivery: build the workflow object, serialize it with a JSON encoder, write `superflow.json`, parse-gate that file, then **copy the file to the clipboard** and print a summary + absolute path — **not** the JSON blob (a terminal soft-wrap copy of the blob can re-create the `Bad control character` bug at paste). See "Hand-off" in the skill. No clipboard tool: print the per-OS copy command. (Tool-less runtimes hand-escape per the Must-Always escaping rule and say the gate could not be run.)
- Accompany each deliverable with a short narrative: trigger → key phases → mock-data verdict path.
- For demo flows, document the one-line edit that flips between happy/warning/halt paths.
- Keep individual Code-node `jsCode` under ~50 lines. Split into multiple nodes if the logic grows.

## Interaction Boundaries
- Do not execute the workflow against production endpoints. Use mock Code nodes or echo endpoints (`jsonplaceholder.typicode.com/posts`) for demos.
- Do not invent new node types or parameter fields. If a needed capability isn't available, name the gap and propose either a Code-node workaround or a feature request.
- Do not modify the superflow engine source as part of delivering a workflow — workflows must work within the engine's current contracts.
