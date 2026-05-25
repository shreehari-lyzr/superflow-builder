# superflow-architect

A git-native ([openGAP](https://github.com/open-gitagent/opengap)) agent that writes superflow JSON workflows for the inference-svc engine.

## What it does

Takes a process description — phase-by-phase blueprint, natural-language sketch, or feature request — and produces a single, executable workflow JSON. Designs against the real executor contracts (parameter shapes, expression semantics, durable-execution constraints) so the output runs end-to-end on the first try.

## What it produces

- A workflow JSON file at a clear path
- A short narrative: trigger source, phase-by-phase data flow, what verdict the mock data lands at, and the one-line edits to flip between demo paths
- Optional: smoke-test harness invocation log proving the upstream chain executes cleanly

## When to use it

- Building a new workflow from a product spec or process blueprint
- Adapting an existing workflow when the engine's executors gain new capabilities
- Debugging a workflow that "passed locally but fails in production" — the agent knows the durable-mode rules cold
- Writing demo flows for stakeholders where the verdict path must be deterministic

## Structure

```
superflow-architect/
├── agent.yaml          # Manifest (model, skills, runtime)
├── SOUL.md             # Identity and collaboration style
├── RULES.md            # Hard constraints (must always / must never)
├── PROMPT.md           # Default task framing and output format
├── AGENTS.md           # Framework-agnostic fallback instructions
├── README.md           # This file
└── skills/
    └── superflow/
        └── SKILL.md    # Node-type registry, contracts, patterns, gotchas
```

## Source dependencies

The agent reads (and references) source files in the inference-svc repo:
- `internal/executors/` — executor implementations (authoritative on parameter contracts)
- `internal/engine/` — DAG building, expression resolution, in-process runner
- `internal/orchestrator/` — durable Restate runner

The skill cites these paths so the agent (and human reviewers) can always check a contract against ground truth.

## Compatible runtimes

Any framework that supports openGAP-format agents. Confirmed: Claude Code (via the `superflow` skill loaded into `~/.claude/skills/` or invoked via the Skill tool). The standard's framework-agnostic fallback (`AGENTS.md`) supports OpenAI, LangChain, CrewAI, AutoGen, and others.
