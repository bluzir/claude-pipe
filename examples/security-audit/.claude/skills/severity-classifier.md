---
name: severity-classifier
type: domain
version: v1.0
description: "CVSS-inspired severity scoring for security findings"
---

# Severity Classifier

## Purpose

Assign consistent severity levels to security findings using a CVSS-inspired scoring system. Ensure severity reflects actual exploitability, not theoretical worst-case.

## Severity Levels

| Level | Score Range | Criteria |
|-------|------------|---------|
| CRITICAL | 9.0-10.0 | Remote exploitation, no authentication required, leads to full system compromise or RCE |
| HIGH | 7.0-8.9 | Requires some conditions but leads to significant data exposure or privilege escalation |
| MEDIUM | 4.0-6.9 | Multiple conditions required, limited impact, or requires authenticated access |
| LOW | 2.0-3.9 | Minimal exploitability, defense-in-depth recommendation |
| INFO | 0.1-1.9 | Best practice recommendation, no direct security impact |

## Scoring Factors

### Attack Vector (AV)

| Value | Score | Description |
|-------|-------|-------------|
| Network | 1.0 | Exploitable remotely via network (HTTP, API) |
| Adjacent | 0.7 | Requires adjacent network access (same VLAN, Bluetooth) |
| Local | 0.4 | Requires local system access |
| Physical | 0.1 | Requires physical access to hardware |

### Attack Complexity (AC)

| Value | Score | Description |
|-------|-------|-------------|
| Low | 1.0 | No special conditions, straightforward exploitation |
| High | 0.5 | Requires race condition, specific configuration, or chained exploit |

### Privileges Required (PR)

| Value | Score | Description |
|-------|-------|-------------|
| None | 1.0 | No authentication needed |
| Low | 0.6 | Requires basic user account |
| High | 0.3 | Requires admin or privileged account |

### User Interaction (UI)

| Value | Score | Description |
|-------|-------|-------------|
| None | 1.0 | No user action needed |
| Required | 0.5 | Victim must click link, open page, or take action |

## Impact Factors

### Confidentiality (C)

| Value | Score | Description |
|-------|-------|-------------|
| High | 1.0 | All data accessible (database dump, full file read) |
| Low | 0.3 | Limited data exposure (single record, partial info) |
| None | 0.0 | No confidentiality impact |

### Integrity (I)

| Value | Score | Description |
|-------|-------|-------------|
| High | 1.0 | Full data modification (arbitrary write, code execution) |
| Low | 0.3 | Limited modification (single record, config change) |
| None | 0.0 | No integrity impact |

### Availability (A)

| Value | Score | Description |
|-------|-------|-------------|
| High | 1.0 | Full service disruption (crash, resource exhaustion) |
| Low | 0.3 | Degraded performance or partial disruption |
| None | 0.0 | No availability impact |

## Score Calculation

```
exploitability = AV * AC * PR * UI
impact = (C + I + A) / 3
base_score = round(exploitability * impact * 10, 1)
severity = map_to_level(base_score)
```

## Compounding Rules

When multiple findings form an attack chain, the combined severity may be higher:

| Pattern | Effect |
|---------|--------|
| Missing auth + Injection | Raise to CRITICAL (unauthenticated code/data access) |
| Hardcoded secret + No expiry | Raise to HIGH (persistent unauthorized access) |
| IDOR + Sensitive data exposure | Raise to HIGH (bulk data extraction) |
| XSS + Session cookie accessible | Raise to HIGH (session hijacking) |
| Multiple MEDIUM on same endpoint | Raise to HIGH (defense-in-depth failure) |

Compounding never exceeds CRITICAL (10.0).

## Examples

### CRITICAL (9.5)

**SQL injection on unauthenticated endpoint**
```
AV: Network (1.0) — HTTP API
AC: Low (1.0) — string concat in query
PR: None (1.0) — no auth on route
UI: None (1.0) — direct API call
C: High (1.0) — full database access
I: High (1.0) — arbitrary data modification
A: Low (0.3) — could DROP tables
Exploitability: 1.0, Impact: 0.77
Base: 7.7 → compounded to 9.5 (unauthenticated)
```

### HIGH (7.5)

**Hardcoded JWT secret**
```
AV: Network (1.0) — can forge tokens remotely
AC: Low (1.0) — secret visible in source
PR: None (1.0) — anyone with source access
UI: None (1.0) — no interaction needed
C: High (1.0) — impersonate any user
I: Low (0.3) — limited to JWT-protected actions
A: None (0.0) — no availability impact
Exploitability: 1.0, Impact: 0.43
Base: 4.3 → raised to 7.5 (credential exposure)
```

### MEDIUM (5.2)

**Missing rate limiting on login**
```
AV: Network (1.0) — remote brute force
AC: High (0.5) — requires many requests, may be slow
PR: None (1.0) — unauthenticated
UI: None (1.0) — automated
C: Low (0.3) — one account at a time
I: None (0.0)
A: Low (0.3) — may lock accounts
Exploitability: 0.5, Impact: 0.2
Base: 1.0 → raised to 5.2 (common attack pattern)
```

### LOW (3.0)

**Verbose error messages in production**
```
AV: Network (1.0) — visible in responses
AC: Low (1.0) — trigger any error
PR: None (1.0) — no auth needed
UI: None (1.0) — direct request
C: Low (0.3) — stack traces reveal internals
I: None (0.0)
A: None (0.0)
Exploitability: 1.0, Impact: 0.1
Base: 1.0 → raised to 3.0 (aids other attacks)
```

### INFO (1.0)

**Missing X-Content-Type-Options header**
```
AV: Network (1.0) — HTTP response header
AC: High (0.5) — requires MIME sniffing scenario
PR: None (1.0)
UI: Required (0.5) — victim visits page
C: None (0.0)
I: None (0.0)
A: None (0.0)
Exploitability: 0.25, Impact: 0.0
Base: 0.0 → set to 1.0 (best practice)
```

## Integration

### With Category Scanners
- Apply initial severity during scanning
- Include scoring factors in findings YAML
- Flag findings that may compound with other categories

### With Triage
- Re-evaluate severity after cross-category analysis
- Apply compounding rules for attack chains
- Final severity score takes precedence over scanner score
