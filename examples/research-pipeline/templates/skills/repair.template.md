---
name: {format}-repair
type: atomic
version: v1.0
description: "Self-correction loop for invalid {FORMAT}"
---

# {FORMAT} Repair

## Purpose

Automatically fix malformed {FORMAT} by analyzing parse errors and applying corrections. Prevents pipeline failures due to formatting issues.

## When to Invoke

This skill is invoked by {io-skill} when {FORMAT} validation fails. Do NOT invoke directly unless debugging.

## Input

```yaml
broken_{format}: |
  {raw string that failed to parse}
parse_error: "{error message from parser}"
attempt: 1  # Current repair attempt (max: 2)
```

## Procedure

### Step 1: Analyze Error

Parse the error message to identify the issue category:

| Error Pattern | Category | Fix Strategy |
|--------------|----------|--------------|
| {pattern1} | {category1} | {fix1} |
| {pattern2} | {category2} | {fix2} |
| {pattern3} | {category3} | {fix3} |

### Step 2: Locate Problem

1. Extract line/position from error (if present)
2. Identify the problematic section
3. Look at context (surrounding content)

### Step 3: Apply Fix

{Show before/after examples for common fixes}

### Step 4: Validate Fix

After applying fix:
1. Parse the repaired {FORMAT}
2. If valid → return fixed {FORMAT}
3. If still invalid → increment attempt counter

### Step 5: Recursion Guard

```
if attempt >= 2:
  HALT with error:
    "{FORMAT} repair failed after 2 attempts"
    Include: original error, repair attempts log
```

## Output

```yaml
status: "repaired" | "failed"
repaired_{format}: |
  {fixed string}
repairs_applied:
  - location: {line/position}
    issue: "{issue type}"
    fix: "{what was changed}"
attempt: {number}
original_error: "{error message}"
```

## Common Patterns

{List common error patterns and their fixes}

## Integration

Called by {io-skill}:
```
1. {io-skill} attempts to validate {FORMAT}
2. Validation fails with parse error
3. {io-skill} invokes {format}-repair
4. {format}-repair returns fixed {FORMAT}
5. {io-skill} re-validates
6. If still invalid → {format}-repair again (attempt 2)
7. If still invalid → halt pipeline
```

## Error Escalation

When repair fails after max attempts:
```yaml
error:
  type: "{format}_repair_failed"
  message: "Could not repair {FORMAT} after 2 attempts"
  original_{format}: "{truncated}"
  parse_errors:
    - attempt: 1
      error: "{first error}"
    - attempt: 2
      error: "{second error}"
  recommendation: "Manual intervention required"
```
