# CLAUDE.md — Instructions for Claude Code

This repo is **claude-pipe**: conventions for building multi-step pipelines with Claude Code.
Agents are markdown files. State is YAML files. Orchestration is flat (ROOT spawns workers, no nesting).

Read this file before building anything. Then use the examples as reference.

---

## Core Rules

1. **Flat orchestration** — ROOT (Claude Code CLI) orchestrates everything. Subagents cannot spawn subagents.
2. **Manager = Skill** — The manager is a Skill loaded into ROOT, not a separate agent. ROOT reads the manager's instructions and executes them (including spawning workers via Task tool).
3. **Files are state** — Workers write results to YAML/markdown files. State lives in `artifacts/{session}/`. You can always `cat state.yaml` to see where you are.
4. **Data layers** — L0 (user config) → L1 (plan, agent-generated) → L2 (working data) → L3 (final output). L1 constraints override L2 decisions. L3 must trace to L2 sources.
5. **L1 > L2** — If the plan says "3 aspects", workers don't get to decide there should be 5. The plan is authoritative.

---

## How to Build a New Pipeline

### Step 1: Define Phases and Data Flow

Before writing any files, answer:
- What are the phases? (e.g., plan → research → synthesize → report)
- What data flows between phases? (e.g., plan.yaml → aspects/*.yaml → synthesis.yaml → REPORT.md)
- Which phases can run in parallel? (independent tasks = fan-out)
- What are the gate conditions? (e.g., "all aspects completed" before synthesis)

### Step 2: Create Directory Structure

```bash
mkdir -p .claude/agents
mkdir -p .claude/skills
mkdir -p artifacts
echo "artifacts/" >> .gitignore
```

### Step 3: Create the Manager Skill First

This is your orchestration entry point. The user runs it as `/manager-yourpipeline`.

File: `.claude/skills/manager-yourpipeline.md`

```markdown
---
name: manager-yourpipeline
type: manager
version: v1.0
description: Short description of your pipeline
---

# Your Pipeline Name

## Session Setup
- Generate session ID: `{pipeline-name}-{YYYYMMDD}-{HHMMSS}`
- Create directory: `artifacts/{session}/`
- Initialize state.yaml with all phases as `pending`

## Phase 1: [Name]
1. [Instructions for this phase]
2. Write output to `artifacts/{session}/[output].yaml`
3. Update state.yaml: phase_states.[name] = completed

## Gate: [Name] → [Next]
- Condition: [what must be true]
- On fail: [what to do]

## Phase 2: [Name] (parallel)
For each item in [previous output]:
  Spawn worker:
    Task(
      subagent_type: "general-purpose",
      prompt: "You are a [role]. Load skill [skill-name]... Process: {item}. Write to: artifacts/{session}/[path]",
      run_in_background: true
    )

IMPORTANT: All Task calls in ONE message = parallel execution.
Wait for all with TaskOutput(block: true).

## Phase 3: [Name]
...
```

### Step 4: Create Worker Agents

File: `.claude/agents/your-worker.md`

Workers must include `silence-protocol` and `io-yaml-safe` in their skills:

```markdown
---
name: your-worker
type: worker
functional_role: [researcher|classifier|generator|...]
model: [sonnet|haiku]
tools: [Read, Write, ...]
skills:
  required: [silence-protocol, io-yaml-safe]
output:
  format: yaml
  path: artifacts/{session}/[specific-path]
---

# Your Worker

## Input
- Read from: [specific path or prompt parameters]

## Procedure
1. [Step-by-step instructions]
2. [Be specific — agents follow instructions literally]

## Output Format
Write YAML to `{output_path}`:
```yaml
field: value
```
```

### Step 5: Copy Reusable Skills

These skills work across any pipeline — copy them from an example:

```bash
# Required for workers
cp examples/research-pipeline/.claude/skills/silence-protocol.md .claude/skills/
cp examples/research-pipeline/.claude/skills/io-yaml-safe.md .claude/skills/
```

### Step 6: Create Domain-Specific Skills

Add skills unique to your pipeline (quality gates, domain knowledge, output formatting).

---

## Reusable Skills Reference

Copy these from `examples/research-pipeline/.claude/skills/`:

| Skill | Type | Purpose | Copy for |
|-------|------|---------|----------|
| `silence-protocol` | atomic | Workers write files only, no chat output | Every pipeline |
| `io-yaml-safe` | atomic | Safe YAML writing with validation + repair | Every pipeline |
| `grounding-protocol` | domain | No-hallucination rules, source traceability | Research pipelines |
| `search-safeguard` | atomic | Exa API retry/jitter/error handling | Pipelines using web search |
| `quality-gate` | atomic | Evaluate synthesis quality, PASS/WARN/FAIL | Pipelines with quality requirements |
| `anti-cringe` | domain | Suppress AI-typical phrases and hedging | Pipelines producing human-readable text |

Create new skills for: domain taxonomy, classification logic, aggregation rules, output formatting.

---

## Orchestration Patterns

### Fan-out (Parallel Workers)

Use when N tasks are independent (e.g., research N aspects, classify N batches):

```
For each item:
  Task(
    subagent_type: "general-purpose",
    prompt: "...",
    run_in_background: true
  )
# All Task calls in ONE message → parallel execution

Then: TaskOutput(block: true) for each task
```

### Pipeline (Sequential Phases)

Use when each phase depends on the previous:

```
Phase 1 → Gate → Phase 2 → Gate → Phase 3
```

Gates check file existence, quality thresholds, or item counts before proceeding.

### Hybrid

Most pipelines are hybrid: sequential phases with fan-out within a phase.
See `examples/research-pipeline/` (plan → fan-out research → synthesize → report).

---

## State File Schema

Every pipeline tracks state in `artifacts/{session}/state.yaml`:

```yaml
session_id: "pipeline-name-20260214-103000"
pipeline: "your-pipeline-name"
current_phase: phase_name
phase_states:
  phase_1: completed
  phase_2: in_progress
  phase_3: pending
workers:
  worker_1: completed
  worker_2: in_progress
  worker_3: failed
```

This enables resume: on re-run, skip completed phases and retry failed workers.

---

## File Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Manager skill | `manager-{pipeline}.md` | `manager-research.md` |
| Worker agent | `{role}.md` | `aspect-researcher.md` |
| Atomic skill | `{action-or-concept}.md` | `io-yaml-safe.md` |
| Domain skill | `{domain}.md` | `grounding-protocol.md` |
| Composite skill | `{combined-name}.md` | `taxonomy-builder.md` |
| State file | `state.yaml` | `artifacts/{session}/state.yaml` |
| Session directory | `{pipeline}-{YYYYMMDD}-{HHMMSS}` | `research-20260214-103000` |

---

## Key Conventions

- **Session ID format**: `{pipeline-name}-{YYYYMMDD}-{HHMMSS}`
- **Worker prompts**: Always tell workers which skills to load, what to read, and where to write
- **Parallel spawning**: All `Task()` calls in a single message = parallel. Separate messages = sequential.
- **Model selection**: Use `sonnet` for complex reasoning, `haiku` for simple classification/extraction
- **Frontmatter**: Every agent/skill file starts with YAML frontmatter (`---` delimited)

---

## Example Pipelines

| Pipeline | Pattern | Path |
|----------|---------|------|
| Research Pipeline | plan → fan-out research → synthesize → quality gate → report | `examples/research-pipeline/` |
| Batch Classifier | partition → fan-out classify → aggregate → export | `examples/batch-classifier/` |
| Repo-to-Docs | scan → fan-out document → assemble → quality gate → fan-out emit | `examples/repo-to-docs/` |
| Security Audit | recon → fan-out scan (by vuln category) → triage → quality gate → report | `examples/security-audit/` |

Start by reading the manager skill in each example — it's the orchestration blueprint.
