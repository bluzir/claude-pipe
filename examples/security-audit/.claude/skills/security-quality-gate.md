---
name: security-quality-gate
type: atomic
version: v1.0
description: "Evaluate audit quality — coverage, evidence, confidence, category completion"
input:
  required:
    - triage_path
    - recon_path
output:
  type: verdict
  schema: quality.yaml
---

# Security Quality Gate

## Purpose

Evaluate the quality and completeness of the security audit before generating the final report. Ensure adequate coverage, evidence quality, and scan completeness.

## Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `triage_path` | string | Path to triage.yaml |
| `recon_path` | string | Path to recon.yaml |

## Procedure

### Step 1: Load Data

```
triage = Read(triage_path)
recon = Read(recon_path)
```

### Step 2: Evaluate Metrics

| Metric | Threshold | Weight | Description |
|--------|-----------|--------|-------------|
| coverage | 0.7 | 0.3 | files examined / total source files |
| evidence_quality | 0.6 | 0.3 | findings with code snippets / total findings |
| confidence_distribution | 0.5 | 0.2 | high+medium confidence / total findings |
| category_coverage | 0.8 | 0.2 | completed scans / planned scans |

For each metric:
```
score = metric_value >= threshold ? 1.0 : metric_value / threshold
weighted_score = score * weight
```

### Step 3: Check Critical Failures

**Automatic FAIL:**
- coverage < 0.5 (missed too many files)
- Any CRITICAL finding lacks code evidence (code_snippet is empty)
- Zero findings AND project has dependencies (suspicious — likely scanner failure)
- category_coverage < 0.5 (more than half of scans failed)

### Step 4: Check False Positive Indicators

Flag if:
- All findings are in test files only → likely false positives
- All findings are in vendor/node_modules dirs → excluded dirs not filtered
- All findings are in commented-out code → pattern matching too broad

### Step 5: Calculate Verdict

```
total_score = sum(weighted_scores)

PASS: total_score >= 0.8 AND no critical failures AND no false positive flags
WARN: total_score >= 0.5 AND no critical failures
FAIL: total_score < 0.5 OR any critical failure
```

### Step 6: Generate Issues & Recommendations

For each failed metric:
```yaml
issue: "{metric} below threshold ({value} vs {threshold})"
recommendation: "Specific action to improve"
severity: warning|critical
```

## Output Schema

```yaml
verdict: PASS|WARN|FAIL
total_score: number
evaluated_at: timestamp

scores:
  coverage:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  evidence_quality:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  confidence_distribution:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  category_coverage:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number

critical_checks:
  min_coverage: boolean
  critical_evidence: boolean
  zero_findings_check: boolean
  min_category_coverage: boolean

false_positive_flags:
  all_in_test_files: boolean
  all_in_vendor_dirs: boolean
  all_in_comments: boolean

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
| total >= 0.8, no criticals, no FP flags | PASS | Proceed to report |
| total >= 0.5, no criticals | WARN | Proceed with caveats noted |
| total < 0.5 | FAIL | Identify gaps, suggest re-scan |
| any critical failure | FAIL | Address critical issue first |

## Example Output

### PASS Example
```yaml
verdict: PASS
total_score: 0.92
evaluated_at: "2026-02-15T10:45:00Z"

scores:
  coverage:
    value: 0.88
    threshold: 0.7
    passed: true
    weight: 0.3
    weighted_score: 0.3
  evidence_quality:
    value: 1.0
    threshold: 0.6
    passed: true
    weight: 0.3
    weighted_score: 0.3
  confidence_distribution:
    value: 0.75
    threshold: 0.5
    passed: true
    weight: 0.2
    weighted_score: 0.2
  category_coverage:
    value: 1.0
    threshold: 0.8
    passed: true
    weight: 0.2
    weighted_score: 0.2

critical_checks:
  min_coverage: true
  critical_evidence: true
  zero_findings_check: true
  min_category_coverage: true

false_positive_flags:
  all_in_test_files: false
  all_in_vendor_dirs: false
  all_in_comments: false

issues: []
recommendations: []
```

### WARN Example
```yaml
verdict: WARN
total_score: 0.65

scores:
  coverage:
    value: 0.55
    threshold: 0.7
    passed: false
    weight: 0.3
    weighted_score: 0.236
  evidence_quality:
    value: 0.8
    threshold: 0.6
    passed: true
    weight: 0.3
    weighted_score: 0.3
  confidence_distribution:
    value: 0.45
    threshold: 0.5
    passed: false
    weight: 0.2
    weighted_score: 0.18
  category_coverage:
    value: 0.875
    threshold: 0.8
    passed: true
    weight: 0.2
    weighted_score: 0.2

issues:
  - metric: coverage
    issue: "File coverage below threshold (55% vs 70%)"
    severity: warning
  - metric: confidence_distribution
    issue: "Low confidence ratio (45% vs 50%)"
    severity: warning

recommendations:
  - "Expand file scanning to include additional directories"
  - "Review low-confidence findings for upgrade or removal"
  - "Report will note limited coverage in methodology section"
```

### FAIL Example
```yaml
verdict: FAIL
total_score: 0.38

critical_checks:
  min_coverage: false
  critical_evidence: true
  zero_findings_check: true
  min_category_coverage: false

issues:
  - metric: coverage
    issue: "Critical: File coverage far below minimum (35% vs 50%)"
    severity: critical
  - metric: category_coverage
    issue: "Critical: Only 3 of 8 categories completed"
    severity: critical

recommendations:
  - "Scanner coverage is insufficient — re-run with broader file patterns"
  - "Multiple category scanners failed — check target path and permissions"
  - "Address scanner failures before generating report"
```
