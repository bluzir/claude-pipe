---
name: manager-docs
type: manager
version: v1.0
description: "Repo-to-docs pipeline orchestration (phases 1-5)"
---

# Repo-to-Docs Pipeline Manager

## Purpose

Orchestrate multi-phase documentation generation from codebase scanning to final markdown output.

## Overview

```
Phase 1        Phase 2              Phase 3        Phase 4        Phase 5
Scan     ───▶  Document ×N   ───▶  Assemble ───▶  Quality  ───▶  Emit ×N
  │               │                    │              │              │
  ▼               ▼                    ▼              ▼              ▼
plan.yaml    modules/*.yaml      assembly.yaml   quality.yaml   docs/*.md
```

## Prerequisites

Before starting:
- Target repository path provided by user
- Session ID generated (format: `docs-{YYYYMMDD}-{HHMMSS}`)
- `artifacts/{session_id}/` directory created
- `artifacts/{session_id}/modules/` directory created
- `artifacts/{session_id}/docs/` and `artifacts/{session_id}/docs/modules/` directories created

---

## Phase 1: Scan

**Gate:** None (entry point)

**Actions:**
1. Invoke codebase-scanner skill with target path
2. Detect language, framework, module structure
3. Generate documentation plan

**Orchestration:**
```
Skill(skill: "codebase-scanner", args: "{target_path}")
```

**Output:** `artifacts/{session}/plan.yaml`

```yaml
# plan.yaml schema
project:
  name: string
  language: string
  framework: string|null
  root: string
modules:
  - id: string
    name: string
    path: string
    files: string[]
    entry_point: string
    loc_estimate: number
    description: string
doc_sections:
  - id: string
    type: architecture|getting-started|api-reference|module
    source_module: string|null
    title: string
```

**Quality Check:**
- modules.length >= 1
- doc_sections includes architecture, getting-started, api-reference
- Each module has files.length >= 1

**Checkpoint:**
```yaml
current_phase: "scan"
phase_states.scan: "completed"
```

**Next:** Phase 2

---

## Phase 2: Document (Parallel)

**Gate:**
```yaml
type: file_exists
condition: "plan.yaml"
validation: "modules.length >= 1"
```

**Actions:**
1. Read plan.yaml
2. Spawn module-documenter for each module
3. Wait for all to complete

**Orchestration:**
```
plan = Read("artifacts/{session}/plan.yaml")

# Spawn all documenters in parallel (single message, multiple Task calls)
For each module in plan.modules:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load module-documenter agent from .claude/agents/module-documenter.md

      Document this module:
      - module_id: {module.id}
      - module_name: {module.name}
      - module_path: {module.path}
      - files: {module.files}
      - session_id: {session}
      - output_path: artifacts/{session}/modules/{module.id}.yaml

      Read the source files at {project.root}/{module.path}/ and write documentation to the output path.
    description: "Document {module.name}",
    run_in_background: true
  )

# Wait for all
For each task_id in spawned_tasks:
  TaskOutput(task_id: task_id, block: true)
```

**Output:** `artifacts/{session}/modules/*.yaml`

**Quality Check:**
- count(modules/*.yaml) >= plan.modules.length * 0.5 (at least half succeeded)
- Each file has api.length >= 1

**Checkpoint:**
```yaml
current_phase: "document"
phase_states.document: "completed"
workers:
  module_1: completed
  module_2: completed
```

**Next:** Phase 3 (or retry failed modules)

---

## Phase 3: Assemble

**Gate:**
```yaml
type: quality_threshold
condition: "count(modules/*.yaml) >= 1"
```

**Actions:**
1. Load plan.yaml and all module YAML files
2. Invoke docs-assembler skill
3. Generate architecture, getting-started, API index

**Orchestration:**
```
Skill(skill: "docs-assembler", args: |
  session_path: artifacts/{session}
)

# Assembler reads from artifacts/{session}/modules/
# Writes to artifacts/{session}/assembly.yaml
```

**Output:** `artifacts/{session}/assembly.yaml`

```yaml
# assembly.yaml schema
metadata:
  session_id: string
  modules_count: number
  total_api_items: number
  created_at: timestamp
architecture:
  overview: string
  diagram: string
getting_started:
  installation: string
  quick_example: string
  key_concepts: [{name, description}]
api_index:
  - {name, type, module, signature, file}
cross_references:
  - {from_module, to_module, relationship}
```

**Checkpoint:**
```yaml
current_phase: "assemble"
phase_states.assemble: "completed"
```

**Next:** Phase 4

---

## Phase 4: Quality Gate

**Gate:**
```yaml
type: file_exists
condition: "assembly.yaml"
```

**Actions:**
1. Invoke docs-quality-gate skill
2. Evaluate against thresholds
3. Route based on verdict

**Orchestration:**
```
Skill(skill: "docs-quality-gate", args: |
  session_path: artifacts/{session}
)

quality = Read("artifacts/{session}/quality.yaml")

# Route based on verdict
```

**Output:** `artifacts/{session}/quality.yaml`

```yaml
# quality.yaml schema
verdict: PASS|WARN|FAIL
total_score: number
scores:
  coverage: {value, threshold, passed, weight, weighted_score}
  api_completeness: {value, threshold, passed, weight, weighted_score}
  section_completeness: {value, threshold, passed, weight, weighted_score}
  example_coverage: {value, threshold, passed, weight, weighted_score}
  diagram_coverage: {value, threshold, passed, weight, weighted_score}
issues: [{metric, issue, severity}]
recommendations: [string]
```

**Routing:**
| Verdict | Action |
|---------|--------|
| PASS | Proceed to Phase 5 |
| WARN | Proceed to Phase 5 (with caveats noted) |
| FAIL | Report gaps, suggest re-documenting |

**Checkpoint:**
```yaml
current_phase: "quality_gate"
phase_states.quality_gate: "completed"
```

**Next:** Phase 5 or halt

---

## Phase 5: Emit (Parallel)

**Gate:**
```yaml
type: quality_verdict
condition: "verdict in [PASS, WARN]"
```

**Actions:**
1. Read plan.yaml for doc_sections list
2. Spawn docs-writer for each output file
3. Wait for all to complete

**Orchestration:**
```
plan = Read("artifacts/{session}/plan.yaml")

# Build task list from doc_sections
# Cross-cutting pages read from assembly.yaml
# Module pages read from modules/{id}.yaml

For each section in plan.doc_sections:
  If section.type == "module":
    source_path = "artifacts/{session}/modules/{section.source_module}.yaml"
    output_file = "docs/modules/{section.source_module}.md"
  Else if section.type == "architecture":
    source_path = "artifacts/{session}/assembly.yaml"
    output_file = "docs/README.md"
  Else if section.type == "getting-started":
    source_path = "artifacts/{session}/assembly.yaml"
    output_file = "docs/getting-started.md"
  Else if section.type == "api-reference":
    source_path = "artifacts/{session}/assembly.yaml"
    output_file = "docs/api-reference.md"

  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load docs-writer agent from .claude/agents/docs-writer.md

      Write this documentation page:
      - doc_id: {section.id}
      - doc_type: {section.type}
      - source_data_path: {source_path}
      - output_path: artifacts/{session}/{output_file}
      - session_id: {session}

      Read the source YAML and write the final markdown.
    description: "Write {section.title}",
    run_in_background: true
  )

# Wait for all
For each task_id in spawned_tasks:
  TaskOutput(task_id: task_id, block: true)
```

**Output:** `artifacts/{session}/docs/*.md` and `artifacts/{session}/docs/modules/*.md`

**Checkpoint:**
```yaml
current_phase: "emit"
phase_states.emit: "completed"
```

**Next:** None (terminal)

---

## State Management

### State File

Location: `artifacts/{session}/state.yaml`

```yaml
session_id: "docs-{YYYYMMDD}-{HHMMSS}"
target_path: "/path/to/repo"
workflow: "manager-docs"
current_phase: "scan"
phase_states:
  scan: in_progress
  document: pending
  assemble: pending
  quality_gate: pending
  emit: pending
started_at: "2026-02-15T10:30:00Z"
last_updated: "2026-02-15T10:30:00Z"
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
1. Log which module failed
2. Continue with remaining workers
3. At phase end:
   - If completed >= 50% of modules → continue
   - If completed < 50% → retry failed only
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

---

## Example Run

```
User: /docs /path/to/my-express-api

Phase 1: Scan
  ✓ Detected: TypeScript + Express
  ✓ Found 3 modules: routes, middleware, models
  → artifacts/docs-20260215-103000/plan.yaml

Phase 2: Document
  ✓ Spawning 3 documenters in parallel
  ✓ modules/routes.yaml (4 API items)
  ✓ modules/middleware.yaml (2 API items)
  ✓ modules/models.yaml (3 API items)

Phase 3: Assemble
  ✓ Built architecture diagram
  ✓ Generated getting-started guide
  ✓ Indexed 9 API items
  → artifacts/docs-20260215-103000/assembly.yaml

Phase 4: Quality Gate
  ✓ Coverage: 100% (threshold: 80%)
  ✓ API completeness: 90% (threshold: 70%)
  → Verdict: PASS

Phase 5: Emit
  ✓ Spawning 6 writers in parallel
  ✓ docs/README.md (architecture)
  ✓ docs/getting-started.md
  ✓ docs/api-reference.md
  ✓ docs/modules/routes.md
  ✓ docs/modules/middleware.md
  ✓ docs/modules/models.md

Status: COMPLETED
```
