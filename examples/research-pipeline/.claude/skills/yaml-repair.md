---
name: yaml-repair
type: atomic
version: v1.0
description: "Self-correction loop for invalid YAML"
---

# YAML Repair

## Purpose

Automatically fix malformed YAML by analyzing parse errors and applying corrections. Prevents pipeline failures due to formatting issues.

## When to Invoke

This skill is invoked by io-yaml-safe when YAML validation fails. Do NOT invoke directly unless debugging.

## Input

```yaml
broken_yaml: |
  {raw YAML string that failed to parse}
parse_error: "line 5: mapping values are not allowed here"
attempt: 1  # Current repair attempt (max: 2)
```

## Procedure

### Step 1: Analyze Error

Parse the error message to identify the issue category:

| Error Pattern | Category | Fix Strategy |
|--------------|----------|--------------|
| `mapping values not allowed` | Indentation | Fix indent at line N |
| `could not find expected ':'` | Missing colon | Add colon after key |
| `found unexpected ':'` | Extra colon in value | Quote the value |
| `did not find expected key` | Structure error | Re-indent block |
| `found character that cannot start` | Special char | Quote string |
| `while scanning a quoted scalar` | Unclosed quote | Close quote |
| `while parsing a block mapping` | Block structure | Fix parent indent |

### Step 2: Locate Problem

1. Extract line number from error (if present)
2. Identify the problematic section
3. Look at context (2 lines before/after)

### Step 3: Apply Fix

**Indentation Issues:**
```yaml
# Before (broken)
items:
- name: foo
  value: bar

# After (fixed)
items:
  - name: foo
    value: bar
```

**Unquoted Special Values:**
```yaml
# Before (broken)
title: Yes: A Story
version: 1.0

# After (fixed)
title: "Yes: A Story"
version: "1.0"
```

**Multi-line Strings:**
```yaml
# Before (broken)
description: This is a long
description that spans lines

# After (fixed)
description: |
  This is a long
  description that spans lines
```

**Missing Colons:**
```yaml
# Before (broken)
name foo
type bar

# After (fixed)
name: foo
type: bar
```

### Step 4: Validate Fix

After applying fix:
1. Parse the repaired YAML
2. If valid → return fixed YAML
3. If still invalid → increment attempt counter

### Step 5: Recursion Guard

```
if attempt >= 2:
  HALT with error:
    "YAML repair failed after 2 attempts"
    Include: original error, repair attempts log
```

## Output

```yaml
status: "repaired" | "failed"
repaired_yaml: |
  {fixed YAML string}
repairs_applied:
  - line: 5
    issue: "indentation"
    fix: "added 2 spaces"
  - line: 12
    issue: "unquoted special char"
    fix: "quoted value containing ':'"
attempt: 1
original_error: "mapping values are not allowed here"
```

## Common Patterns

### Pattern 1: List Indent Under Key

```yaml
# WRONG
tags:
- one
- two

# CORRECT
tags:
  - one
  - two
```

### Pattern 2: Colon in Value

```yaml
# WRONG
title: My Title: Subtitle

# CORRECT
title: "My Title: Subtitle"
```

### Pattern 3: Boolean/Number as String

```yaml
# WRONG (becomes boolean true)
answer: yes

# CORRECT (stays string)
answer: "yes"
```

### Pattern 4: Multi-line Without Indicator

```yaml
# WRONG
desc: line one
line two

# CORRECT
desc: |
  line one
  line two
```

## Integration

Called by io-yaml-safe:
```
1. io-yaml-safe attempts to validate YAML
2. Validation fails with parse error
3. io-yaml-safe invokes yaml-repair
4. yaml-repair returns fixed YAML
5. io-yaml-safe re-validates
6. If still invalid → yaml-repair again (attempt 2)
7. If still invalid → halt pipeline
```

## Error Escalation

When repair fails after max attempts:
```yaml
error:
  type: "yaml_repair_failed"
  message: "Could not repair YAML after 2 attempts"
  original_yaml: "{truncated}"
  parse_errors:
    - attempt: 1
      error: "{first error}"
    - attempt: 2
      error: "{second error}"
  recommendation: "Manual intervention required"
```
