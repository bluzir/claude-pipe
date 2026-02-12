---
name: phase-checkpoint
type: atomic
version: v1.0
description: "Create git checkpoint after phase completion"
---

# Phase Checkpoint

## Purpose

Create a git checkpoint (commit + tag) after each pipeline phase completes. Enables time-travel debugging and recovery.

## When to Invoke

Invoke after any phase completes successfully:
- After planning → tag: `{session}-phase-1-planning`
- After research → tag: `{session}-phase-2-research`
- After synthesis → tag: `{session}-phase-3-synthesis`
- After quality → tag: `{session}-phase-4-quality`
- After report → tag: `{session}-phase-5-report`

## Input

```yaml
session_id: "research_20260130_abc"
phase_id: 2
phase_name: "research"
artifacts_path: "artifacts/{session_id}"
```

## Procedure

### Step 1: Check for Changes

```bash
# Check if there are any changes to commit
git status --porcelain artifacts/{session_id}/
```

If no changes → skip (idempotent), return success.

### Step 2: Stage Artifacts

```bash
# Stage only session artifacts
git add artifacts/{session_id}/
```

### Step 3: Create Commit

```bash
git commit -m "checkpoint: {session_id} phase {phase_id} ({phase_name}) completed"
```

Commit message format:
```
checkpoint: research_20260130_abc phase 2 (research) completed

Artifacts:
- plan.yaml
- aspects/architecture.yaml
- aspects/patterns.yaml
- state.yaml

Pipeline: manager-research
```

### Step 4: Create Tag

```bash
# Tag format: {session}-phase-{N}-{name}
git tag "{session_id}-phase-{phase_id}-{phase_name}"
```

### Step 5: Update State

Add checkpoint info to state.yaml:
```yaml
checkpoints:
  - phase: 2
    name: "research"
    tag: "research_20260130_abc-phase-2-research"
    commit: "{commit_sha}"
    created_at: "2026-01-30T10:30:00Z"
```

## Output

```yaml
status: "created" | "skipped" | "failed"
checkpoint:
  tag: "research_20260130_abc-phase-2-research"
  commit: "abc123def"
  files_committed: 5
  phase: 2
  phase_name: "research"
message: "Checkpoint created successfully"
```

## Idempotency

Running checkpoint twice for same phase:
1. First run: creates commit + tag
2. Second run: detects no changes, skips

```yaml
# Second run output
status: "skipped"
reason: "No changes since last checkpoint"
existing_tag: "research_20260130_abc-phase-2-research"
```

## Error Handling

| Scenario | Action |
|----------|--------|
| No git repo | Initialize or warn |
| Tag exists | Skip (idempotent) |
| Commit fails | Report error, don't halt pipeline |
| No changes | Skip gracefully |

## Integration with manager-research

After each phase in manager-research.md:
```
Phase N completes → write artifacts → invoke phase-checkpoint
```

Example in Phase 2:
```
# After all researchers complete
Skill(skill: "phase-checkpoint", args: |
  session_id: {session}
  phase_id: 2
  phase_name: research
)
```

## Viewing Checkpoints

```bash
# List all checkpoints for a session
git tag -l "research_20260130_abc-*"

# View checkpoint details
git show research_20260130_abc-phase-2-research

# Compare phases
git diff research_20260130_abc-phase-1-planning research_20260130_abc-phase-2-research
```

## Recovery Use Case

When something goes wrong in Phase 4:
```bash
# See what changed
git diff research_20260130_abc-phase-3-synthesis HEAD

# Restore to after synthesis
git checkout research_20260130_abc-phase-3-synthesis -- artifacts/research_20260130_abc/

# Resume from Phase 4
```

See: `resume-checkpoint` skill for automated recovery.
