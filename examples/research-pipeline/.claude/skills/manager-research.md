---
name: manager-research
type: manager
version: v1.0
description: "Research pipeline orchestration (phases 1-5)"
---

# Research Pipeline Manager

## Purpose

Orchestrate multi-phase research pipeline from topic decomposition to final report.

## Overview

```
Phase 1        Phase 2           Phase 3       Phase 4        Phase 5
Planning  ───▶ Research ×N ───▶ Synthesis ───▶ Quality ───▶ Report
   │              │                 │             │            │
   ▼              ▼                 ▼             ▼            ▼
plan.yaml    aspects/*.yaml   synthesis.yaml  quality.yaml  FINAL_REPORT.md
```

## Prerequisites

Before starting:
- Topic provided by user
- Session ID generated (format: `research_{YYYYMMDD}_{random}`)
- artifacts/{session_id}/ directory created

---

## Phase 1: Planning

**Gate:** None (entry point)

**Actions:**
1. Invoke research-planner skill with topic
2. Decompose into 3-7 aspects
3. Generate queries for each aspect

**Orchestration:**
```
Skill(skill: "research-planner", args: "{topic}")
```

**Output:** `artifacts/{session}/plan.yaml`

```yaml
# plan.yaml schema
topic: string
created_at: timestamp
aspects:
  - id: string
    name: string
    description: string
    queries: string[]
settings:
  max_sources_per_aspect: 15
  min_aspects: 3
```

**Quality Check:**
- aspects.length >= 3
- Each aspect has >= 2 queries

**Checkpoint:**
```
Skill(skill: "phase-checkpoint", args: |
  session_id: {session}
  phase_id: 1
  phase_name: planning
)
```

**Next:** Phase 2

---

## Phase 2: Parallel Research

**Gate:**
```yaml
type: file_exists
condition: "plan.yaml"
validation: "aspects.length >= 3"
```

**Actions:**
1. Read plan.yaml
2. Spawn aspect-researcher for each aspect
3. Wait for all to complete

**Orchestration:**
```
plan = Read("artifacts/{session}/plan.yaml")

# Spawn all researchers in parallel (single message, multiple Task calls)
For each aspect in plan.aspects:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load aspect-researcher agent from .claude/agents/aspect-researcher.md

      Research this aspect:
      - aspect_id: {aspect.id}
      - aspect_name: {aspect.name}
      - aspect_description: {aspect.description}
      - queries: {aspect.queries}
      - session_id: {session}
      - output_path: artifacts/{session}/aspects/{aspect.id}.yaml

      Write findings to the output path.
    description: "Research {aspect.name}",
    run_in_background: true
  )

# Wait for all
For each task_id in spawned_tasks:
  TaskOutput(task_id: task_id, block: true)
```

**Output:** `artifacts/{session}/aspects/*.yaml`

**Quality Check:**
- count(aspects/*.yaml) >= plan.min_aspects
- Each file has findings.length >= 3

**Checkpoint:**
```
Skill(skill: "phase-checkpoint", args: |
  session_id: {session}
  phase_id: 2
  phase_name: research
)
```

**Next:** Phase 3 (or retry failed aspects)

---

## Phase 3: Synthesis

**Gate:**
```yaml
type: quality_threshold
condition: "count(aspects/*.yaml) >= 3"
```

**Actions:**
1. Load all aspect files
2. Invoke synthesis skill
3. Find cross-aspect patterns
4. Generate aggregated insights

**Orchestration:**
```
Skill(skill: "synthesis")

# Synthesis skill reads from artifacts/{session}/aspects/
# Writes to artifacts/{session}/synthesis.yaml
```

**Output:** `artifacts/{session}/synthesis.yaml`

```yaml
# synthesis.yaml schema
metadata:
  session_id: string
  aspects_count: number
  total_findings: number
  total_sources: number
  created_at: timestamp

insights:
  - id: string
    title: string
    description: string
    evidence:
      - finding_id: string
        aspect_id: string
        weight: number
    confidence: high|medium|low

cross_aspect_patterns:
  - pattern: string
    aspects: string[]
    strength: number

themes:
  - name: string
    insights: string[]

quality_metrics:
  saturation: number      # 0-100, information completeness
  diversity: number       # 0-1, source variety
  tier_quality: number    # Weighted avg of source tiers
```

**Checkpoint:**
```
Skill(skill: "phase-checkpoint", args: |
  session_id: {session}
  phase_id: 3
  phase_name: synthesis
)
```

**Next:** Phase 4

---

## Phase 4: Quality Gate

**Gate:**
```yaml
type: file_exists
condition: "synthesis.yaml"
```

**Actions:**
1. Invoke quality-gate skill
2. Evaluate against thresholds
3. Route based on verdict

**Orchestration:**
```
Skill(skill: "quality-gate")

quality = Read("artifacts/{session}/quality.yaml")

# Route based on verdict
```

**Output:** `artifacts/{session}/quality.yaml`

```yaml
# quality.yaml schema
verdict: PASS|WARN|FAIL
scores:
  saturation: number
  diversity: number
  tier_quality: number
  evidence_depth: number
thresholds:
  saturation: 50
  diversity: 0.5
issues: string[]
recommendations: string[]
```

**Routing:**
| Verdict | Action |
|---------|--------|
| PASS | → Phase 5 |
| WARN | → Phase 5 (with caveats noted) |
| FAIL | → Report gaps, suggest re-research |

**Checkpoint:**
```
Skill(skill: "phase-checkpoint", args: |
  session_id: {session}
  phase_id: 4
  phase_name: quality_gate
)
```

**Next:** Phase 5 or halt

---

## Phase 5: Report Generation

**Gate:**
```yaml
type: quality_verdict
condition: "verdict in [PASS, WARN]"
```

**Actions:**
1. Spawn report-generator
2. Generate final report
3. Update state to completed

**Orchestration:**
```
Task(
  subagent_type: "general-purpose",
  prompt: |
    Load report-generator agent from .claude/agents/report-generator.md

    Generate research report:
    - synthesis_path: artifacts/{session}/synthesis.yaml
    - plan_path: artifacts/{session}/plan.yaml
    - quality_path: artifacts/{session}/quality.yaml
    - session_id: {session}
    - output_path: artifacts/{session}/FINAL_REPORT.md
  description: "Generate final report"
)
```

**Output:** `artifacts/{session}/FINAL_REPORT.md`

**Checkpoint:**
```
Skill(skill: "phase-checkpoint", args: |
  session_id: {session}
  phase_id: 5
  phase_name: report
)
```

**Next:** None (terminal)

---

## State Management

### State File

Location: `artifacts/{session}/state.yaml`

```yaml
session_id: "research_20260130_abc123"
topic: "AI agents orchestration"
workflow: "manager-research"
current_phase: "synthesis"
phase_states:
  planning: completed
  research: completed
  synthesis: in_progress
  quality_gate: pending
  report: pending
started_at: "2026-01-30T10:00:00Z"
last_updated: "2026-01-30T10:30:00Z"
error: null
```

### Update Pattern

Before phase:
```yaml
current_phase: "{phase}"
phase_states.{phase}: "in_progress"
last_updated: now()
```

After phase:
```yaml
phase_states.{phase}: "completed"
last_updated: now()
```

On error:
```yaml
phase_states.{phase}: "failed"
error: "{error_message}"
```

---

## Recovery

### On Worker Failure

```
1. Log which aspect failed
2. Continue with remaining workers
3. At phase end:
   - If completed >= min_aspects → continue
   - If completed < min_aspects → retry failed only
```

### On Phase Failure

```
1. Update state with error
2. Report to user:
   - What phase failed
   - What was completed
   - Specific error
3. Suggest action:
   - Retry command
   - Manual intervention
```

### On Resume

```
1. Read state.yaml
2. Find current_phase
3. If in_progress → resume from there
4. If failed → offer retry or rollback
5. Skip completed phases
```

### On Rollback

Use resume-checkpoint skill to restore to previous phase:
```
Skill(skill: "resume-checkpoint", args: |
  session_id: {session}
  target_phase: {phase_to_restore_to}
)
```

This restores artifacts and updates state.yaml to continue from the next phase.

---

## Example Run

```
User: /research "AI agents orchestration patterns"

Phase 1: Planning
  ✓ Decomposed into 5 aspects
  → artifacts/research_20260130_abc/plan.yaml

Phase 2: Research
  ✓ Spawning 5 researchers in parallel
  ✓ aspects/architecture.yaml (12 findings)
  ✓ aspects/patterns.yaml (8 findings)
  ✓ aspects/tools.yaml (15 findings)
  ✓ aspects/challenges.yaml (7 findings)
  ✓ aspects/future.yaml (6 findings)

Phase 3: Synthesis
  ✓ Aggregated 48 findings
  ✓ Identified 4 cross-aspect patterns
  → artifacts/research_20260130_abc/synthesis.yaml

Phase 4: Quality Gate
  ✓ Saturation: 72% (threshold: 50%)
  ✓ Diversity: 0.81 (threshold: 0.5)
  → Verdict: PASS

Phase 5: Report
  ✓ Generated 2,400 word report
  → artifacts/research_20260130_abc/FINAL_REPORT.md

Status: COMPLETED
```
