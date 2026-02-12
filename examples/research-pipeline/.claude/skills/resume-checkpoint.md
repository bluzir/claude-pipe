---
name: resume-checkpoint
type: atomic
version: v1.0
description: "Restore pipeline state from git checkpoint"
---

# Resume Checkpoint

## Purpose

Restore pipeline artifacts to a previous checkpoint and resume execution from that point. Enables recovery from failures and iterative debugging.

## When to Invoke

- Pipeline failed mid-phase and needs rollback
- User wants to re-run from specific phase
- Debugging requires comparing different runs
- Experimenting with alternative approaches

## Input

```yaml
session_id: "research_20260130_abc"
target_phase: 2  # Resume AFTER this phase (start phase 3)
# OR
target_tag: "research_20260130_abc-phase-2-research"  # Explicit tag
```

## Procedure

### Step 1: Validate Target

```bash
# List available checkpoints
git tag -l "{session_id}-phase-*"
```

Output example:
```
research_20260130_abc-phase-1-planning
research_20260130_abc-phase-2-research
research_20260130_abc-phase-3-synthesis
```

If target_phase specified, derive tag:
```
tag = "{session_id}-phase-{target_phase}-{phase_name}"
```

### Step 2: Check Current State

```bash
# Save current state info for logging
git rev-parse HEAD
git status --porcelain artifacts/{session_id}/
```

### Step 3: Restore Artifacts

```bash
# Restore artifacts directory from checkpoint
git checkout {target_tag} -- artifacts/{session_id}/
```

This overwrites current artifacts with checkpoint state.

### Step 4: Update state.yaml

Update state file to reflect restored position:

```yaml
session_id: "research_20260130_abc"
current_phase: "{next_phase_name}"  # Phase AFTER target
phase_states:
  planning: completed
  research: completed      # If target_phase >= 2
  synthesis: pending       # Reset phases after target
  quality_gate: pending
  report: pending
last_updated: "{now}"
restored_from:
  tag: "research_20260130_abc-phase-2-research"
  restored_at: "{now}"
  reason: "manual_resume"
```

### Step 5: Report Restoration

```yaml
status: "restored"
restored_to:
  tag: "research_20260130_abc-phase-2-research"
  phase: 2
  phase_name: "research"
artifacts_restored:
  - plan.yaml
  - aspects/architecture.yaml
  - aspects/patterns.yaml
  - state.yaml
next_phase: 3
next_phase_name: "synthesis"
message: "Restored to phase 2. Ready to resume from phase 3 (synthesis)."
```

## Output

```yaml
status: "restored" | "failed"
restored_to:
  tag: string
  phase: number
  phase_name: string
  commit: string
artifacts_restored: string[]
next_phase: number
next_phase_name: string
previous_state:
  commit: string
  had_uncommitted_changes: boolean
```

## Usage Scenarios

### Scenario 1: Retry Failed Phase

Phase 4 (quality) failed due to threshold issue:
```
1. resume-checkpoint with target_phase: 3
2. Artifacts restored to post-synthesis state
3. Modify thresholds or synthesis
4. Re-run from phase 4
```

### Scenario 2: Re-research Single Aspect

Want to re-do research for one aspect:
```
1. resume-checkpoint with target_phase: 1
2. Edit plan.yaml to modify aspect queries
3. Re-run phase 2 with updated queries
```

### Scenario 3: Compare Approaches

Testing different synthesis strategies:
```
1. Run full pipeline → checkpoint at each phase
2. resume-checkpoint to phase 2
3. Modify synthesis parameters
4. Run phases 3-5
5. Compare results with original
```

## Integration

Called by manager-research for recovery:
```
On phase failure:
  1. Log error
  2. Ask user: "Resume from phase N-1?"
  3. If yes → invoke resume-checkpoint
  4. Continue pipeline from restored state
```

Manual invocation:
```
Skill(skill: "resume-checkpoint", args: |
  session_id: research_20260130_abc
  target_phase: 2
)
```

## Safety

### Uncommitted Changes Warning

If there are uncommitted changes:
```yaml
warning: "Uncommitted changes will be overwritten"
uncommitted_files:
  - artifacts/session/synthesis.yaml
  - artifacts/session/aspects/new_aspect.yaml
action: "Confirm restoration or commit changes first"
```

### No Destructive Operations

- Does NOT delete git history
- Does NOT force push
- Only modifies working directory artifacts
- Original state recoverable via `git reflog`

## Error Cases

| Error | Handling |
|-------|----------|
| Tag not found | List available tags, suggest closest |
| Not a git repo | Error, cannot resume |
| Conflicts | Report conflicts, suggest resolution |
| Invalid session | Error with available sessions list |

## Phase Name Mapping

| Phase ID | Name | Description |
|----------|------|-------------|
| 1 | planning | Topic decomposition |
| 2 | research | Parallel aspect research |
| 3 | synthesis | Cross-aspect synthesis |
| 4 | quality_gate | Quality evaluation |
| 5 | report | Final report generation |
