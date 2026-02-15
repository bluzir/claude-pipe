# Building Your Own Pipeline

This guide walks you through designing and building a claude-pipe pipeline from scratch.

**If you're Claude Code**: Read [CLAUDE.md](CLAUDE.md) instead — it's written for you.

---

## What Pipeline Do You Want?

Before touching any files, answer these questions:

1. **What goes in?** (a topic, a dataset, a list of URLs, etc.)
2. **What comes out?** (a report, a classified dataset, a summary, etc.)
3. **What are the phases?** Name them. Draw the flow:
   ```
   Phase 1 → Phase 2 → Phase 3 → Output
      │          │          │         │
      ▼          ▼          ▼         ▼
   file.yaml  files/*.yaml  agg.yaml  REPORT.md
   ```
4. **Which phases can run in parallel?** If a phase processes N independent items, it's a fan-out.

---

## Decision Tree: What Pattern?

```
Does your pipeline process N independent items in any phase?
├── YES → Does each phase depend on the previous?
│         ├── YES → HYBRID (sequential phases, fan-out within a phase)
│         └── NO  → FAN-OUT (all items processed independently)
└── NO  → PIPELINE (strictly sequential phases)
```

**Fan-out** (batch-classifier): partition 100 items → classify 2 batches in parallel → aggregate.
**Pipeline** (sequential): plan → execute → review. Each step needs the previous output.
**Hybrid** (research-pipeline): plan → research N aspects in parallel → synthesize → report.

Most real pipelines are hybrid.

---

## Design Your Pipeline

Map your phases to the file structure:

| You Need | Create | Where |
|----------|--------|-------|
| Orchestration logic | Manager skill | `.claude/skills/manager-{name}.md` |
| Worker that runs N times | Agent definition | `.claude/agents/{role}.md` |
| Shared behavior for workers | Skills (atomic/domain) | `.claude/skills/{name}.md` |
| Classification schema | Schema file | `.claude/schemas/{entity}.yaml` |
| Runtime outputs | Session directory | `artifacts/{session}/` (gitignored) |

---

## Annotated Walkthrough: Batch Classifier

Every file in `examples/batch-classifier/` tagged by what you'd reuse.

### Directory Structure

```
examples/batch-classifier/
├── .claude/
│   ├── agents/
│   │   └── batch-worker.md          [DOMAIN-SPECIFIC] — your worker logic
│   ├── schemas/
│   │   └── classified-item.yaml     [DOMAIN-SPECIFIC] — your data schema
│   └── skills/
│       ├── manager-classify.md      [STRUCTURAL]      — adapt phases/gates for your pipeline
│       ├── partition.md             [STRUCTURAL]       — reuse for any batch pipeline
│       ├── taxonomy-builder.md      [DOMAIN-SPECIFIC]  — your aggregation logic
│       └── io-yaml-safe.md          [REUSABLE]         — copy as-is
├── data/sample/                     [DOMAIN-SPECIFIC]  — your input data
└── artifacts/                       [STRUCTURAL]       — gitignored runtime outputs
```

### `[STRUCTURAL]` — Every pipeline needs this pattern

**`manager-classify.md`** — The orchestration blueprint.

Key structural patterns to copy:
- Phase definitions with gates between them
- `state.yaml` initialization with all phases as `pending`
- Fan-out: spawn ALL workers in a single message with `run_in_background: true`
- Wait: `TaskOutput(block: true)` for each spawned task
- State updates: mark phases `in_progress` → `completed`
- Recovery: retry failed workers, resume from state

**`partition.md`** — Data partitioning for batch processing.

Reuse when: you have N items to process in parallel batches.
Replace when: your parallelism is semantic (research aspects), not data-based.

### `[REUSABLE]` — Copy as-is to any pipeline

**`io-yaml-safe.md`** — Safe YAML writing with validation and auto-repair.

Every worker that writes YAML should list this in its skills. Handles:
- Structure validation before write
- Atomic writes (temp file → validate → move)
- Auto-repair loop (up to 2 attempts via yaml-repair skill)

Also available in research-pipeline: `silence-protocol.md` (workers write files only, no chat output).

### `[DOMAIN-SPECIFIC]` — Unique to your task

**`batch-worker.md`** — Classification logic per item.

This is where YOUR domain knowledge lives. The batch-classifier version classifies by category/sentiment/tags. You'd replace this entirely with your worker's logic, keeping the structure:
- Frontmatter with name, type, model, skills, output format
- Input section (what the worker receives)
- Procedure section (step-by-step)
- Output format (exact YAML schema)
- Error handling table

**`taxonomy-builder.md`** — Aggregation of classified outputs.

Your aggregation will be different. The pattern to keep:
- Collect all output files from workers
- Build summary statistics
- Write single aggregated output file

---

## Minimal Viable Pipeline

A complete 2-phase pipeline: **plan** then **execute**.

### File: `.claude/skills/manager-example.md`

```markdown
---
name: manager-example
type: manager
version: v1.0
description: "Minimal 2-phase pipeline"
---

# Example Pipeline

## Session Setup
- Generate session ID: `example-{YYYYMMDD}-{HHMMSS}`
- Create: `artifacts/{session}/`
- Initialize state.yaml:
  ```yaml
  session_id: "{session}"
  current_phase: planning
  phase_states:
    planning: in_progress
    execution: pending
  ```

## Phase 1: Planning
1. Read user input
2. Break task into 3-5 work items
3. Write plan:
   ```yaml
   # artifacts/{session}/plan.yaml
   topic: "{user_input}"
   items:
     - id: item_01
       description: "..."
     - id: item_02
       description: "..."
   ```
4. Update state: planning = completed

## Gate: Planning → Execution
- Condition: `plan.yaml` exists with >= 1 item
- On fail: Ask user to clarify input

## Phase 2: Execution (parallel)
For each item in plan.items:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load agent from .claude/agents/worker.md
      Process item: {item.description}
      Write output to: artifacts/{session}/results/{item.id}.yaml
    description: "Process {item.id}",
    run_in_background: true
  )

Wait for all tasks with TaskOutput(block: true).
Update state: execution = completed

## Completion
Read all results from artifacts/{session}/results/*.yaml.
Write summary to artifacts/{session}/SUMMARY.md.
```

### File: `.claude/agents/worker.md`

```markdown
---
name: worker
type: worker
model: sonnet
tools: [Read, Write]
skills:
  required: [silence-protocol, io-yaml-safe]
output:
  format: yaml
  path: artifacts/{session}/results/{item_id}.yaml
---

# Worker

## Input
- `item_id`: Which item to process
- `description`: What to do
- Output path provided in prompt

## Procedure
1. Read the item description
2. Do the work (your domain logic here)
3. Write structured output to the given path

## Output Format
```yaml
item_id: "{item_id}"
result: "..."
status: completed
```
```

### File: `.claude/skills/silence-protocol.md`

Copy from: `examples/research-pipeline/.claude/skills/silence-protocol.md`

### File: `.claude/skills/io-yaml-safe.md`

Copy from: `examples/research-pipeline/.claude/skills/io-yaml-safe.md`

### Run it

```
/manager-example "Analyze the top 5 risks of our deployment strategy"
```

---

## What to Reuse vs. Create

| File | Reuse or Create? | Notes |
|------|-------------------|-------|
| Manager skill | **Create** (adapt from example) | Your phases, gates, and data flow |
| Worker agents | **Create** | Your domain logic |
| `silence-protocol.md` | **Reuse** (copy as-is) | Keeps workers quiet |
| `io-yaml-safe.md` | **Reuse** (copy as-is) | Safe YAML output |
| `grounding-protocol.md` | **Reuse** if research | No-hallucination rules |
| `search-safeguard.md` | **Reuse** if web search | Exa API retry/jitter |
| `quality-gate.md` | **Adapt** | Change thresholds for your domain |
| `anti-cringe.md` | **Reuse** if generating text | Suppress AI-typical phrasing |
| Domain skills | **Create** | Your taxonomy, rules, knowledge |
| Schemas | **Create** | Your data shapes |
| `state.yaml` pattern | **Adapt** | Change phase names, keep the structure |

---

## Next Steps

- Read [CLAUDE.md](CLAUDE.md) for the full convention reference
- Read [README.md](README.md) for the conceptual overview
- Browse `examples/research-pipeline/` for a complex hybrid pipeline
- Browse `examples/batch-classifier/` for a data-parallel pipeline
- Browse `examples/repo-to-docs/` for structural partitioning (one subagent per module)
- Browse `examples/security-audit/` for domain expertise partitioning (one subagent per vuln category)
