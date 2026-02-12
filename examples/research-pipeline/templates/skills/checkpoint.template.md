---
name: {skill-name}-checkpoint
type: atomic
version: v1.0
description: "Create git checkpoint after {phase/stage} completion"
---

# {Skill Name} Checkpoint

## Purpose

Create a git checkpoint (commit + tag) after {describe what completes}. Enables time-travel debugging and recovery.

## When to Invoke

Invoke after {completion condition}:
- {list conditions when this should be invoked}

## Input

```yaml
session_id: "{session_format}"
{phase_or_stage}_id: {number_or_string}
{phase_or_stage}_name: "{name}"
artifacts_path: "{path_to_artifacts}"
```

## Procedure

### Step 1: Check for Changes

```bash
git status --porcelain {artifacts_path}/
```

If no changes → skip (idempotent), return success.

### Step 2: Stage Artifacts

```bash
git add {artifacts_path}/
```

### Step 3: Create Commit

```bash
git commit -m "checkpoint: {session_id} {phase/stage} {id} ({name}) completed"
```

### Step 4: Create Tag

```bash
git tag "{session_id}-{phase/stage}-{id}-{name}"
```

### Step 5: Update State

Add checkpoint info to state file:
```yaml
checkpoints:
  - {phase/stage}: {id}
    name: "{name}"
    tag: "{tag}"
    commit: "{sha}"
    created_at: "{timestamp}"
```

## Output

```yaml
status: "created" | "skipped" | "failed"
checkpoint:
  tag: "{tag}"
  commit: "{sha}"
  files_committed: {count}
message: "{result message}"
```

## Idempotency

Running checkpoint twice for same {phase/stage}:
1. First run: creates commit + tag
2. Second run: detects no changes, skips

## Error Handling

| Scenario | Action |
|----------|--------|
| No git repo | Initialize or warn |
| Tag exists | Skip (idempotent) |
| Commit fails | Report error, don't halt |
| No changes | Skip gracefully |

## Integration

After each {phase/stage} in {orchestrator}:
```
{Phase/Stage} N completes → write artifacts → invoke {skill-name}-checkpoint
```
