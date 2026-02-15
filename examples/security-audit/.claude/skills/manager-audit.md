---
name: manager-audit
type: manager
version: v1.0
description: "Security audit pipeline orchestration (phases 1-5)"
---

# Security Audit Pipeline Manager

## Purpose

Orchestrate multi-phase security audit from reconnaissance to final report. Scans a target codebase for vulnerabilities across 8 categories, triages findings, detects attack chains, and generates an actionable security report.

## Overview

```
Phase 1        Phase 2              Phase 3     Phase 4        Phase 5
Recon    ───▶  Scan ×N        ───▶  Triage ───▶ Quality  ───▶  Report
  │              │                    │            │              │
  ▼              ▼                    ▼            ▼              ▼
recon.yaml   findings/*.yaml    triage.yaml  quality.yaml  SECURITY_REPORT.md
```

## Prerequisites

Before starting:
- Target path provided by user (directory to audit)
- Session ID generated (format: `audit-{YYYYMMDD}-{HHMMSS}`)
- `artifacts/{session_id}/` directory created
- `artifacts/{session_id}/findings/` directory created

---

## Phase 1: Reconnaissance

**Gate:** None (entry point)

**Actions:**
1. Invoke reconnaissance skill with target path
2. Map languages, frameworks, entry points, auth boundaries
3. Determine which of 8 vulnerability categories are relevant
4. Build file inventory for scanner scope

**Orchestration:**
```
Skill(skill: "reconnaissance", args: |
  target_path: {target_path}
  session_id: {session_id}
)
```

**Output:** `artifacts/{session}/recon.yaml`

```yaml
# recon.yaml schema
project:
  name: string
  root: string
  languages: string[]
  frameworks: string[]
attack_surface:
  entry_points: [...]
  auth_boundaries: [...]
  data_flows: [...]
scan_categories:
  - id: string
    relevant: boolean
    reason: string
    priority: high|medium|low
file_inventory:
  total_files: number
  source_files: number
  vendor_dirs: string[]
```

**Quality Check:**
- scan_categories has at least 3 relevant entries
- file_inventory.source_files > 0
- attack_surface.entry_points has at least 1 entry

**Checkpoint:**
```yaml
current_phase: "recon"
phase_states.recon: "completed"
```

**Next:** Phase 2

---

## Phase 2: Category Scan (Parallel)

**Gate:**
```yaml
type: file_exists
condition: "recon.yaml"
validation: "scan_categories.filter(relevant).length >= 3"
```

**Actions:**
1. Read recon.yaml
2. Filter scan_categories to relevant ones
3. Spawn category-scanner for each relevant category
4. Wait for all to complete

**Orchestration:**
```
recon = Read("artifacts/{session}/recon.yaml")
categories = recon.scan_categories.filter(c => c.relevant)

# Spawn all scanners in parallel (single message, multiple Task calls)
For each category in categories:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load category-scanner agent from .claude/agents/category-scanner.md

      Scan for vulnerabilities:
      - category_id: {category.id}
      - category_name: {category_name}
      - target_path: {target_path}
      - file_inventory: {recon.file_inventory}
      - session_id: {session}
      - output_path: artifacts/{session}/findings/{category.id}.yaml

      Write findings to the output path.
    description: "Scan {category.id}",
    run_in_background: true
  )

# Wait for all
For each task_id in spawned_tasks:
  TaskOutput(task_id: task_id, block: true)
```

**Output:** `artifacts/{session}/findings/*.yaml`

**Quality Check:**
- count(findings/*.yaml) >= 3 (at least 3 categories produced output)
- Each file has valid YAML structure

**Checkpoint:**
```yaml
current_phase: "scan"
phase_states.scan: "completed"
workers:
  injection: completed
  auth: completed
  xss: completed
  sensitive-data: completed
  access-control: completed
  misconfig: completed
  dependencies: completed
  crypto: completed
```

**Next:** Phase 3 (or retry failed scanners)

---

## Phase 3: Triage

**Gate:**
```yaml
type: quality_threshold
condition: "count(findings/*.yaml) >= 3"
```

**Actions:**
1. Load all findings files
2. Invoke triage skill
3. Deduplicate, cross-reference, build attack chains
4. Assign final severity scores

**Orchestration:**
```
Skill(skill: "triage", args: |
  findings_dir: artifacts/{session}/findings/
  session_id: {session}
)
```

**Output:** `artifacts/{session}/triage.yaml`

```yaml
# triage.yaml schema
metadata:
  session_id: string
  total_raw_findings: number
  deduplicated_findings: number
  attack_chains_found: number
findings:
  - id: string
    original_ids: string[]
    severity: CRITICAL|HIGH|MEDIUM|LOW|INFO
    severity_score: number
    categories: string[]
    cwe: string[]
    location: {file, line}
    code_snippet: string
attack_chains:
  - id: string
    name: string
    severity: CRITICAL|HIGH
    steps: [...]
summary:
  by_severity: {critical, high, medium, low, info}
```

**Checkpoint:**
```yaml
current_phase: "triage"
phase_states.triage: "completed"
```

**Next:** Phase 4

---

## Phase 4: Quality Gate

**Gate:**
```yaml
type: file_exists
condition: "triage.yaml"
```

**Actions:**
1. Invoke security-quality-gate skill
2. Evaluate coverage, evidence quality, confidence, category completion
3. Route based on verdict

**Orchestration:**
```
Skill(skill: "security-quality-gate", args: |
  triage_path: artifacts/{session}/triage.yaml
  recon_path: artifacts/{session}/recon.yaml
)
```

**Output:** `artifacts/{session}/quality.yaml`

```yaml
# quality.yaml schema
verdict: PASS|WARN|FAIL
total_score: number
scores:
  coverage: {value, threshold, passed}
  evidence_quality: {value, threshold, passed}
  confidence_distribution: {value, threshold, passed}
  category_coverage: {value, threshold, passed}
critical_checks: {...}
issues: [...]
recommendations: [...]
```

**Routing:**
| Verdict | Action |
|---------|--------|
| PASS | Phase 5 |
| WARN | Phase 5 (with caveats noted in report) |
| FAIL | Report gaps, suggest re-scan |

**Checkpoint:**
```yaml
current_phase: "quality_gate"
phase_states.quality_gate: "completed"
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
2. Generate SECURITY_REPORT.md
3. Update state to completed

**Orchestration:**
```
Task(
  subagent_type: "general-purpose",
  prompt: |
    Load report-generator agent from .claude/agents/report-generator.md

    Generate security audit report:
    - triage_path: artifacts/{session}/triage.yaml
    - recon_path: artifacts/{session}/recon.yaml
    - quality_path: artifacts/{session}/quality.yaml
    - session_id: {session}
    - output_path: artifacts/{session}/SECURITY_REPORT.md
  description: "Generate security report"
)
```

**Output:** `artifacts/{session}/SECURITY_REPORT.md`

**Checkpoint:**
```yaml
current_phase: "report"
phase_states.report: "completed"
```

**Next:** None (terminal)

---

## State Management

### State File

Location: `artifacts/{session}/state.yaml`

```yaml
session_id: "audit-{YYYYMMDD}-{HHMMSS}"
target_path: "/path/to/repo"
workflow: "manager-audit"
current_phase: "scan"
phase_states:
  recon: completed
  scan: in_progress
  triage: pending
  quality_gate: pending
  report: pending
started_at: "2026-02-15T10:30:00Z"
last_updated: "2026-02-15T10:32:00Z"
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
1. Log which category scanner failed
2. Continue with remaining scanners
3. At phase end:
   - If completed >= 3 categories → continue
   - If completed < 3 → retry failed only
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
User: /audit /path/to/express-app

Phase 1: Reconnaissance
  ✓ Detected: Node.js, Express, MongoDB
  ✓ Found 12 routes, 3 middleware, 2 models
  ✓ 6 of 8 categories relevant (skipping: dependencies low priority, crypto low priority)
  → artifacts/audit-20260215-103000/recon.yaml

Phase 2: Category Scan
  ✓ Spawning 6 scanners in parallel
  ✓ findings/injection.yaml (2 findings: SQL injection, NoSQL injection)
  ✓ findings/auth.yaml (2 findings: JWT no expiry, hardcoded secret)
  ✓ findings/sensitive-data.yaml (2 findings: hardcoded API key, PII in logs)
  ✓ findings/access-control.yaml (1 finding: missing auth on admin route)
  ✓ findings/misconfig.yaml (1 finding: CORS wildcard)
  ✓ findings/xss.yaml (0 findings)

Phase 3: Triage
  ✓ 8 raw findings → 8 deduplicated
  ✓ 2 attack chains identified
  ✓ Final scores: 2 critical, 3 high, 2 medium, 1 low
  → artifacts/audit-20260215-103000/triage.yaml

Phase 4: Quality Gate
  ✓ Coverage: 88% (threshold: 70%)
  ✓ Evidence quality: 100% (threshold: 60%)
  ✓ Confidence distribution: 75% (threshold: 50%)
  → Verdict: PASS

Phase 5: Report
  ✓ Generated security report with 9 sections
  → artifacts/audit-20260215-103000/SECURITY_REPORT.md

Status: COMPLETED — 2 critical, 3 high, 2 medium, 1 low findings
```
