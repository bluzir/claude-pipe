---
name: evidence-protocol
type: domain
version: v1.0
description: "No-hallucination rules for security findings — every finding must trace to real code"
---

# Evidence Protocol

## Purpose

Ensure all security findings are traceable to actual source code. Prevent fabricated vulnerabilities, invented file paths, and imagined code patterns.

## Core Principles

### Principle 1: Code Traceability

**Statement:** Every finding must trace to a specific file:line in the target codebase.

**Application:**
- Include exact file path and line number
- Include actual code snippet copied from the source
- Confirm the file and line exist before reporting

**Violations:**
- Reporting a finding without file:line location
- Describing a vulnerability pattern without showing the actual code
- Referencing files that do not exist in the target

### Principle 2: No Fabrication

**Statement:** Never invent vulnerabilities, file paths, or code patterns.

**Application:**
- Only report patterns found via Grep/Glob/Read
- Only reference CWE IDs that exist in the MITRE database
- If a pattern search returns no results, report zero findings for that category

**Violations:**
- Inventing a SQL injection that does not exist in the code
- Creating a fictional file path to illustrate a point
- Presenting theoretical risk as a confirmed finding

### Principle 3: Severity Honesty

**Statement:** Severity must reflect actual exploitability, not worst-case imagination.

**Application:**
- Assess exploitability based on surrounding code context
- Consider existing mitigations (input validation, parameterized queries)
- Downgrade severity when mitigations are present but imperfect

**Violations:**
- Rating every string concatenation in SQL as CRITICAL
- Ignoring input validation when assessing injection risk
- Inflating severity to make the report look more alarming

## Confidence Levels

| Level | Criteria | Use When |
|-------|----------|----------|
| HIGH | Pattern confirmed in code, exploitability verified in context | Grep match confirmed by Read, no mitigations found |
| MEDIUM | Pattern likely based on context, but full exploitation path unclear | Pattern found but mitigations may exist elsewhere |
| LOW | Best practice recommendation, no confirmed vulnerable pattern | Code works but does not follow security best practices |

## Constraints

### Hard Constraints (Never Violate)

| ID | Constraint |
|----|------------|
| HC1 | Never fabricate a vulnerability that does not exist in the code |
| HC2 | Never invent file paths or line numbers |
| HC3 | Never present theoretical risk as a confirmed finding |
| HC4 | Every code snippet must be copied from the actual source file |
| HC5 | CWE references must be real CWE IDs from the MITRE database |

### Soft Constraints

| ID | Constraint | Override |
|----|------------|----------|
| SC1 | Include 3-5 lines of context around the vulnerable line | When the vulnerable pattern is a single expression |
| SC2 | Include remediation code example for every finding | For INFO-level best practice recommendations |

## Verification Checklist

Before writing findings output:
- [ ] Every file:line reference exists and is valid
- [ ] Every code snippet was copied from the actual source file
- [ ] Every CWE ID is a real MITRE CWE
- [ ] Severity reflects actual exploitability with mitigations considered
- [ ] Confidence level matches the strength of evidence
- [ ] No findings were generated from imagination or general knowledge alone

## Integration

### With Scanners
- Use Grep to find patterns, Read to confirm in context
- Record the exact match location (file, line, column)
- Copy the code snippet directly from Read output

### With Triage
- Cross-reference findings across categories
- Verify compounding factors exist in actual code paths
- Attack chains must trace through real code flow

### With Report Generator
- Every finding in the report must link to triage data
- Code snippets in the report must match scanner output exactly
- Remediation examples must be realistic for the codebase's language/framework
