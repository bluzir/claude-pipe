# claude-pipe

**Operational pipelines for Claude Code**

> Readable agents. Inspectable state. Predictable costs.

Claude Code is great for tasks. But when you need multi-step pipelines — research → analyze → generate, batch processing overnight, workflows that run unattended — you hit gaps:

- No state persistence between steps
- No resume after failure
- No coordination for parallel workers
- No cost control

claude-pipe is a set of conventions that fills these gaps. File-based state you can `cat`. Flat orchestration you can debug. Agents you can `git blame`.

---

## How It Works

Claude Code has architectural constraints. claude-pipe works with them:

| Constraint | Why It Exists | Convention |
|------------|---------------|------------|
| Subagents can't spawn subagents | Forces flat hierarchy → easier debugging | Manager = Skill (ROOT reads instructions) |
| Context is finite | Prevents runaway costs | Workers write to files, not context |
| Model is "blind" until Read() | Explicit I/O = traceable | File-based state: `cat state.yaml` |
| Skills are steering, not code | Flexible, not brittle | Conventions over enforcement |

**Result**: Flat orchestration, file-based state, predictable costs.

---

## Before / After

| Problem | Without claude-pipe | With claude-pipe |
|---------|-------------------|----------------|
| Pipeline fails at step 4 | Re-run from scratch | Resume from step 4 |
| Agent loops forever | $50 bill | Circuit breaker stops it |
| "What happened?" | Dig through logs | `cat artifacts/state.yaml` |
| Share with team | Copy-paste prompts | `git push` |

---

## Core Pattern

**ROOT orchestrates, Workers execute, Files persist.**

```
┌─────────────────────────────────────────────────────┐
│                     USER                            │
│                  (slash commands)                   │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│                    ROOT                             │
│              (Claude Code CLI)                      │
│                                                     │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │  Skill  │  │  Skill  │  │  Skill  │  Skills   │
│   │(manager)│  │(atomic) │  │(domain) │  (libs)   │
│   └─────────┘  └─────────┘  └─────────┘           │
│                                                     │
│   ┌─────────────────────────────────────────────┐  │
│   │              Task tool                       │  │
│   │         (spawn subagents)                   │  │
│   └─────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌───────────┐  ┌───────────┐  ┌───────────┐
│  Worker   │  │  Worker   │  │  Worker   │
│ (subagent)│  │ (subagent)│  │ (subagent)│
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │              │              │
      ▼              ▼              ▼
┌───────────────────────────────────────────┐
│                  FILES                     │
│           (L0 → L1 → L2 → L3)             │
│         (persistent state layer)          │
└───────────────────────────────────────────┘
```

**Key insight**: Manager is a Skill loaded into ROOT (not a separate agent). ROOT reads manager instructions and executes them directly. This is how you get orchestration despite the flat hierarchy constraint.

---

## Data Layers

State flows through four layers:

```
L0 (Config)      →  L1 (Directives)  →  L2 (Operational)  →  L3 (Artifacts)
user_profile.yaml   plan.yaml           aspects/*.yaml       FINAL_REPORT.md
                    directives.yaml     synthesis.yaml
```

| Layer | What | Mutability | Example |
|-------|------|------------|---------|
| **L0** | User config | User-edited | `user_profile.yaml` |
| **L1** | Session plan | Agent-generated, user-approved | `plan.yaml` |
| **L2** | Working data | Agent-generated | `aspects/*.yaml` |
| **L3** | Final outputs | Append-only | `FINAL_REPORT.md` |

**Contract**: L1 constraints always override L2 decisions. L3 must trace to L2 sources.

---

## Examples

### Research Pipeline

Semantic decomposition → parallel research → synthesis.

```
┌─────────────┐     ┌─────────────────┐     ┌───────────┐     ┌───────────┐
│  Planning   │────▶│ Parallel Research│────▶│ Synthesis │────▶│  Report   │
│  (1 agent)  │     │   (N agents)     │     │ (1 agent) │     │           │
└─────────────┘     └─────────────────┘     └───────────┘     └───────────┘
      │                     │                     │                 │
      ▼                     ▼                     ▼                 ▼
  plan.yaml           aspects/*.yaml        synthesis.yaml    FINAL_REPORT.md
```

**State file enables resume**:
```yaml
# artifacts/{session}/state.yaml
current_phase: synthesis
phase_states:
  planning: completed
  research: completed
  synthesis: in_progress
workers:
  aspect_1: completed
  aspect_2: completed
  aspect_3: failed  # ← Resume re-runs only this
```

→ [examples/research-pipeline/](examples/research-pipeline/)

### Batch Classifier

Data partitioning → parallel classification → aggregation.

```
┌─────────────┐     ┌─────────────────┐     ┌───────────┐
│  Partition  │────▶│ Classify ×N     │────▶│ Aggregate │
│             │     │ (parallel)      │     │           │
└─────────────┘     └─────────────────┘     └───────────┘
      │                     │                     │
      ▼                     ▼                     ▼
 manifest.yaml       batches/*.yaml         taxonomy.yaml
```

**Use when**: Large datasets that can be processed independently per batch.

→ [examples/batch-classifier/](examples/batch-classifier/)

---

## Orchestration Patterns

### Fan-out (Parallel)

When N tasks are independent:

```markdown
## Manager Skill Instructions

For each aspect in plan.aspects:
  Task(
    subagent_type: "general-purpose",
    prompt: "Load aspect-researcher. Research: {aspect.name}",
    run_in_background: true
  )

Wait for all tasks to complete.
```

**Key**: All Task calls in ONE message = parallel execution.

### Pipeline (Sequential)

When each phase depends on the previous:

```
Phase 1 → Gate → Phase 2 → Gate → Phase 3
   │               │               │
   ▼               ▼               ▼
plan.yaml    aspects/*.yaml   synthesis.yaml
```

Gates = transition conditions:
```yaml
gate:
  type: file_exists
  condition: "plan.yaml exists"

gate:
  type: quality_threshold
  condition: "min 3 aspects completed"
```

---

## Skills

Skills are knowledge libraries, not agents. They inject instructions into context.

```
Agent A + grounding-protocol = Agent A that doesn't hallucinate
Agent B + grounding-protocol = Agent B that doesn't hallucinate
```

| Type | Purpose | Example |
|------|---------|---------|
| **atomic** | Single operation | `tier-weights`, `slop-check` |
| **composite** | Combined operations | `source-evaluation` |
| **domain** | Knowledge pack | `training-science` |
| **manager** | Workflow instructions | `manager-research` |

**Manager skill** = orchestration instructions that ROOT executes directly.

---

## Project Structure

```
.claude/
├── agents/          # Agent definitions (markdown)
└── skills/          # Skill definitions (markdown)
artifacts/
└── {session}/       # Runtime outputs (gitignored)
    ├── state.yaml   # Pipeline state
    └── ...          # Phase outputs
```

---

## Getting Started

1. **Copy a template**
   ```bash
   cp examples/research-pipeline/.claude/skills/manager-research.md \
      .claude/skills/manager-mypipeline.md
   ```

2. **Edit for your task**
   - Define phases
   - Specify gates
   - List workers

3. **Run**
   ```
   /manager-mypipeline "your topic"
   ```

---

## When to Use

**Good fit**:
- Multi-step research pipelines
- Data processing: gather → analyze → generate
- Workflows with phases and quality gates
- Overnight / unattended runs
- Teams sharing agent conventions via git

**Not a good fit**:
- Simple single-agent tasks
- Real-time chat applications
- Tasks without intermediate state
- Sub-millisecond latency requirements

---

## Principles (TL;DR)

1. **ROOT orchestrates, Workers execute** — no nested spawns
2. **Skills are shared libraries** — knowledge reuse
3. **Files are state** — everything persists to YAML
4. **Layers have contracts** — L1 constrains L2, L2 feeds L3
5. **Manager = Skill** — workflow instructions for ROOT

---

## Quick Reference

| Primitive | What It Does |
|-----------|--------------|
| `Task(run_in_background: true)` | Spawn parallel worker |
| `TaskOutput(block: true)` | Wait for worker |
| `Skill("manager-x")` | Load orchestration instructions |
| `state.yaml` | Pipeline state for resume |

| Convention | Why |
|------------|-----|
| Workers write to files | Context stays small, state persists |
| Manager is a Skill | ROOT must control spawning |
| Gates between phases | Quality control, cost control |
| Session directories | Isolate runs, enable replay |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
