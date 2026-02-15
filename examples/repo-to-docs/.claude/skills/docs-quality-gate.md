---
name: docs-quality-gate
type: atomic
version: v1.0
description: "Evaluate documentation quality, return PASS/WARN/FAIL verdict"
input:
  required:
    - session_path
output:
  type: verdict
  schema: quality.yaml
---

# Docs Quality Gate

## Purpose

Evaluate generated documentation against quality thresholds. Return a verdict that determines whether to proceed to final markdown emission.

## Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_path` | string | Path to `artifacts/{session}/` |

## Procedure

### Step 1: Load Data

```
plan = Read("{session_path}/plan.yaml")
assembly = Read("{session_path}/assembly.yaml")
module_files = Glob("{session_path}/modules/*.yaml")
modules = [Read(f) for f in module_files]
```

### Step 2: Evaluate Metrics

| Metric | Threshold | Weight | Description |
|--------|-----------|--------|-------------|
| coverage | 0.8 | 0.35 | Modules documented / modules planned |
| api_completeness | 0.7 | 0.25 | Exports documented / total exports found |
| section_completeness | 0.8 | 0.20 | Required sections present (architecture, getting-started, api-reference) |
| example_coverage | 0.5 | 0.10 | Modules with at least one usage example / total modules |
| diagram_coverage | 0.5 | 0.10 | Has architecture diagram + module diagrams for multi-file modules |

For each metric:
```
score = metric_value >= threshold ? 1.0 : metric_value / threshold
weighted_score = score * weight
```

### Step 3: Check Critical Failures

**Automatic FAIL:**
- coverage < 0.5 (fewer than half the modules documented)
- Architecture diagram missing from assembly
- No modules documented at all (modules_count = 0)

### Step 4: Calculate Verdict

```
total_score = sum(weighted_scores)

PASS: total_score >= 0.8 AND no critical failures
WARN: total_score >= 0.5 AND no critical failures
FAIL: total_score < 0.5 OR any critical failure
```

### Step 5: Generate Issues and Recommendations

For each failed metric:
```yaml
issue: "Coverage below threshold (60% vs 80%)"
recommendation: "Check for failed module-documenter workers and retry"
severity: warning|critical
```

## Output Schema

```yaml
verdict: PASS|WARN|FAIL
total_score: number  # 0-1
evaluated_at: timestamp

scores:
  coverage:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  api_completeness:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  section_completeness:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  example_coverage:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  diagram_coverage:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number

critical_checks:
  min_coverage: boolean
  architecture_diagram: boolean
  modules_documented: boolean

issues:
  - metric: string
    issue: string
    severity: warning|critical

recommendations:
  - string
```

## Decision Table

| Condition | Verdict | Action |
|-----------|---------|--------|
| total >= 0.8, no criticals | PASS | Proceed to emit |
| total >= 0.5, no criticals | WARN | Proceed with caveats noted |
| total < 0.5 | FAIL | Identify gaps, suggest re-documenting |
| any critical failure | FAIL | Address critical issue first |

## Example Output

### PASS Example
```yaml
verdict: PASS
total_score: 0.92
evaluated_at: "2026-02-15T10:45:00Z"

scores:
  coverage:
    value: 1.0
    threshold: 0.8
    passed: true
    weight: 0.35
    weighted_score: 0.35
  api_completeness:
    value: 0.85
    threshold: 0.7
    passed: true
    weight: 0.25
    weighted_score: 0.25
  section_completeness:
    value: 1.0
    threshold: 0.8
    passed: true
    weight: 0.20
    weighted_score: 0.20
  example_coverage:
    value: 0.67
    threshold: 0.5
    passed: true
    weight: 0.10
    weighted_score: 0.10
  diagram_coverage:
    value: 1.0
    threshold: 0.5
    passed: true
    weight: 0.10
    weighted_score: 0.10

critical_checks:
  min_coverage: true
  architecture_diagram: true
  modules_documented: true

issues: []
recommendations: []
```

### WARN Example
```yaml
verdict: WARN
total_score: 0.63
evaluated_at: "2026-02-15T10:45:00Z"

scores:
  coverage:
    value: 0.75
    threshold: 0.8
    passed: false
    weight: 0.35
    weighted_score: 0.328
  api_completeness:
    value: 0.65
    threshold: 0.7
    passed: false
    weight: 0.25
    weighted_score: 0.232
  section_completeness:
    value: 1.0
    threshold: 0.8
    passed: true
    weight: 0.20
    weighted_score: 0.20
  example_coverage:
    value: 0.33
    threshold: 0.5
    passed: false
    weight: 0.10
    weighted_score: 0.066
  diagram_coverage:
    value: 0.5
    threshold: 0.5
    passed: true
    weight: 0.10
    weighted_score: 0.10

critical_checks:
  min_coverage: true
  architecture_diagram: true
  modules_documented: true

issues:
  - metric: coverage
    issue: "Coverage slightly below threshold (75% vs 80%)"
    severity: warning
  - metric: api_completeness
    issue: "API completeness below threshold (65% vs 70%)"
    severity: warning

recommendations:
  - "Check for failed module-documenter workers and retry"
  - "Documentation will note incomplete API coverage"
```

### FAIL Example
```yaml
verdict: FAIL
total_score: 0.31
evaluated_at: "2026-02-15T10:45:00Z"

critical_checks:
  min_coverage: false  # Critical failure
  architecture_diagram: true
  modules_documented: true

issues:
  - metric: coverage
    issue: "Critical: Coverage far below minimum (40% vs 50%)"
    severity: critical
  - metric: api_completeness
    issue: "API completeness below threshold (45% vs 70%)"
    severity: warning
  - metric: example_coverage
    issue: "No modules have usage examples"
    severity: warning

recommendations:
  - "Module documentation coverage is insufficient"
  - "Re-run Phase 2 for failed modules"
  - "Check module-documenter agent logs for errors"
```
