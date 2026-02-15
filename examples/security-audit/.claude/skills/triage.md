---
name: triage
type: composite
version: v1.0
description: "Phase 3 — deduplicate, cross-reference, build attack chains, assign final severity"
input:
  required:
    - findings_dir
    - session_id
output:
  type: triage
  schema: triage.yaml
---

# Triage

## Purpose

Aggregate findings from all category scanners, deduplicate by location, detect compounding attack chains, and assign final severity scores.

## Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `findings_dir` | string | Path to `artifacts/{session}/findings/` |
| `session_id` | string | Current audit session ID |

## Procedure

### Step 1: Load All Findings

```
For each *.yaml in findings_dir:
  findings += Read(file).findings
```

Collect all findings into a single list with their source category.

### Step 2: Deduplicate by Location

Group findings by `location.file` + `location.line`:

```
For each unique (file, line):
  If multiple findings reference same location:
    Merge into single finding:
      - Keep highest severity
      - Combine category tags
      - Combine CWE IDs
      - Keep all original IDs in original_ids[]
```

### Step 3: Cross-Reference Findings

Look for compounding patterns across categories:

| Pattern | Findings Involved | Effect |
|---------|-------------------|--------|
| Unauthenticated injection | Missing auth + SQL/NoSQL injection | Raise to CRITICAL |
| Credential exposure chain | Hardcoded secret + No token expiry | Raise to HIGH |
| Data breach chain | IDOR + Sensitive data exposure | Raise to HIGH |
| Session hijacking chain | XSS + Accessible session cookie | Raise to HIGH |
| Defense-in-depth failure | 3+ MEDIUM on same endpoint | Raise to HIGH |

For each pattern:
1. Check if required findings exist
2. Verify findings are on connected code paths
3. If confirmed, create an attack chain entry

### Step 4: Build Attack Chains

For each confirmed compounding pattern:

```yaml
attack_chain:
  id: "CHAIN-001"
  name: "Descriptive chain name"
  severity: CRITICAL|HIGH
  description: "Narrative of the attack path"
  steps:
    - finding_id: "AC-001"
      role: "entry point"
    - finding_id: "INJ-001"
      role: "exploitation"
    - finding_id: "SD-001"
      role: "impact"
  combined_impact: "What attacker achieves through this chain"
```

### Step 5: Assign Final Severity

For each finding:
1. Start with scanner-assigned severity
2. Check if involved in attack chain → apply compounding rules from severity-classifier
3. Check for mitigating factors across categories (e.g., input validation found by one scanner)
4. Assign final `severity_score` (0.1-10.0)

### Step 6: Sort and Output

Sort findings by:
1. `severity_score` descending
2. `confidence` descending (HIGH > MEDIUM > LOW)
3. `location.file` alphabetical

Write to `artifacts/{session}/triage.yaml`.

## Output Schema

```yaml
metadata:
  session_id: string
  total_raw_findings: number
  deduplicated_findings: number
  attack_chains_found: number
  created_at: timestamp

findings:
  - id: string
    original_ids: string[]
    title: string
    severity: CRITICAL|HIGH|MEDIUM|LOW|INFO
    severity_score: number
    confidence: HIGH|MEDIUM|LOW
    categories: string[]
    cwe: string[]
    location:
      file: string
      line: number
    code_snippet: string
    description: string
    impact: string
    remediation:
      description: string
      code_example: string
      effort: low|medium|high
    compounding_factors: string[]|null

attack_chains:
  - id: string
    name: string
    severity: CRITICAL|HIGH
    description: string
    steps:
      - finding_id: string
        role: string
    combined_impact: string

summary:
  by_severity:
    critical: number
    high: number
    medium: number
    low: number
    info: number
  top_affected_files:
    - file: string
      finding_count: number
  categories_scanned: number
  categories_with_findings: number
```

## Quality Criteria

- [ ] All raw findings accounted for (none dropped silently)
- [ ] Deduplication preserves the highest-severity instance
- [ ] Attack chains trace through actual code paths
- [ ] Final severity scores reflect compounding
- [ ] Findings sorted by severity then confidence
