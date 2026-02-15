# Security Audit Pipeline

A multi-phase security audit pipeline that scans a codebase for vulnerabilities across 8 OWASP categories, builds attack chains, and generates an actionable report with file:line locations and fix code.

> **Framework Documentation:** See the main [README.md](../../README.md) for framework concepts and conventions.

Found 2 critical, 3 high vulnerabilities in 4 minutes -- each with `file:line` and fix code.

## Pattern: Subagent = Category

Domain expertise partitioning -- each scanner agent specializes in one vulnerability category (injection, auth, XSS, etc.) with its own grep patterns, false positive rules, and CWE mappings. Results are triaged across categories to detect attack chains that no single scanner would find alone.

```
                            ┌─────────────────┐
                            │   Manager       │
                            │ (orchestrator)  │
                            └────────┬────────┘
                                     │
         ┌──────────┬──────────┬─────┴─────┬──────────┬──────────┐
         │          │          │           │          │          │
         ▼          ▼          ▼           ▼          ▼          ▼
    ┌─────────┐┌─────────┐┌─────────┐┌─────────┐┌─────────┐┌─────────┐
    │injection││  auth   ││sensitive││ access  ││ misconf ││ crypto  │
    │ scanner ││ scanner ││  data   ││ control ││ scanner ││ scanner │
    └────┬────┘└────┬────┘└────┬────┘└────┬────┘└────┬────┘└────┬────┘
         │          │          │           │          │          │
         ▼          ▼          ▼           ▼          ▼          ▼
    ┌─────────┐┌─────────┐┌─────────┐┌─────────┐┌─────────┐┌─────────┐
    │findings ││findings ││findings ││findings ││findings ││findings │
    │  .yaml  ││  .yaml  ││  .yaml  ││  .yaml  ││  .yaml  ││  .yaml  │
    └─────────┘└─────────┘└─────────┘└─────────┘└─────────┘└─────────┘
```

### Why Pipeline > Single Prompt

| Single prompt | This pipeline |
|---------------|---------------|
| Hallucinates vulnerabilities from general knowledge | Every finding traced to actual code via Grep + Read |
| Misses cross-category attack chains | Triage phase detects compounding patterns across categories |
| Inconsistent severity ratings | CVSS-inspired scoring with defined factors |
| No evidence for findings | Every finding has file:line, code snippet, and CWE |
| Skips categories it "forgets" about | 8 parallel scanners, each with dedicated patterns |
| Varies wildly between runs | Structured categories, patterns, and thresholds |

## Quick Start

```bash
cd examples/security-audit
/manager-audit /path/to/your/repo
```

The pipeline will:
1. Map your codebase (languages, routes, auth, data flows)
2. Scan for vulnerabilities across up to 8 categories in parallel
3. Triage findings, detect attack chains, assign final severity
4. Quality-check the audit for coverage and evidence
5. Generate `SECURITY_REPORT.md` with every finding, fix, and priority

## Pipeline

```
Phase 1        Phase 2              Phase 3     Phase 4        Phase 5
Recon    ───▶  Scan ×N        ───▶  Triage ───▶ Quality  ───▶  Report
  │              │                    │            │              │
  ▼              ▼                    ▼            ▼              ▼
recon.yaml   findings/*.yaml    triage.yaml  quality.yaml  SECURITY_REPORT.md
```

## Phases

| Phase | Agent/Skill | Input | Output |
|-------|-------------|-------|--------|
| 1. Recon | reconnaissance | target path | recon.yaml |
| 2. Scan | category-scanner ×N | recon + patterns | findings/*.yaml |
| 3. Triage | triage | findings/*.yaml | triage.yaml |
| 4. Quality | security-quality-gate | triage + recon | quality.yaml |
| 5. Report | report-generator | triage + recon + quality | SECURITY_REPORT.md |

## Vulnerability Categories

| ID | OWASP | Category | What Scanner Looks For |
|----|-------|----------|----------------------|
| `injection` | A03:2021 | Injection | SQL/NoSQL/OS command injection, string concat in queries |
| `auth` | A07:2021 | Auth & Session | JWT issues, hardcoded secrets, insecure cookies |
| `xss` | A03:2021 | Cross-Site Scripting | dangerouslySetInnerHTML, unescaped output, missing CSP |
| `sensitive-data` | A02:2021 | Sensitive Data | Hardcoded keys, PII in logs, verbose errors |
| `access-control` | A01:2021 | Access Control | Missing auth middleware, IDOR, CORS wildcard |
| `misconfig` | A05:2021 | Misconfiguration | Debug mode, default credentials, missing headers |
| `dependencies` | A06:2021 | Vulnerable Deps | Outdated packages, unpinned versions |
| `crypto` | A02:2021 | Crypto Failures | MD5/SHA1, Math.random, hardcoded keys |

Not all categories apply to every project. The recon phase determines which are relevant based on the tech stack (e.g., XSS is skipped for API-only services).

## Severity Levels

| Level | Score | Criteria |
|-------|-------|---------|
| CRITICAL | 9.0-10.0 | Remote, unauthenticated, system compromise or RCE |
| HIGH | 7.0-8.9 | Significant data exposure or privilege escalation |
| MEDIUM | 4.0-6.9 | Limited impact, requires conditions or authentication |
| LOW | 2.0-3.9 | Defense-in-depth recommendation |
| INFO | 0.1-1.9 | Best practice, no direct impact |

Attack chains (e.g., missing auth + SQL injection) compound severity beyond individual findings.

## Structure

```
security-audit/
├── .claude/
│   ├── agents/
│   │   ├── category-scanner.md     # Worker: scans one vulnerability category
│   │   └── report-generator.md     # Worker: generates SECURITY_REPORT.md
│   ├── skills/
│   │   ├── manager-audit.md        # Manager: 5-phase pipeline orchestration
│   │   ├── reconnaissance.md       # Atomic: map codebase and attack surface
│   │   ├── triage.md               # Composite: deduplicate, chain, score
│   │   ├── security-quality-gate.md # Atomic: PASS/WARN/FAIL verdict
│   │   ├── evidence-protocol.md    # Domain: no-hallucination for findings
│   │   ├── vuln-categories.md      # Domain: 8 categories with patterns
│   │   ├── severity-classifier.md  # Domain: CVSS-inspired scoring
│   │   ├── silence-protocol.md     # Atomic: suppress chat, file-only output
│   │   ├── io-yaml-safe.md         # Atomic: safe YAML writing
│   │   └── anti-cringe.md          # Domain: suppress AI-typical phrasing
├── artifacts/                       # Runtime outputs (gitignored)
│   └── {session_id}/
│       ├── state.yaml              # Pipeline state for resume
│       ├── recon.yaml              # Codebase map and scan plan
│       ├── findings/*.yaml         # Per-category vulnerability findings
│       ├── triage.yaml             # Deduplicated, scored, chained findings
│       ├── quality.yaml            # Audit quality verdict
│       └── SECURITY_REPORT.md      # Deliverable
└── README.md
```

## Output Format

The final `SECURITY_REPORT.md` includes:

1. **Executive Summary** -- finding counts, top risks, attack chains
2. **Critical Findings** -- full detail with code, impact, fix
3. **High Findings** -- same detail level
4. **Medium Findings** -- condensed table format
5. **Low & Info Findings** -- summary table
6. **Attack Chains** -- step-by-step narratives
7. **Findings by Category** -- severity matrix
8. **Coverage & Methodology** -- files scanned, tools used
9. **Remediation Priority** -- ordered by severity and effort

Every finding includes:
- `file:line` location
- Actual code snippet from the source
- CWE and OWASP mapping
- Remediation with corrected code
- Effort estimate (low/medium/high)

## Demo

See `artifacts/demo_session/` for a complete example output from auditing a fictional Express.js API called "user-service". The demo found 2 critical, 3 high, 2 medium, and 1 low vulnerability, with 2 attack chains identified.

Key files:
- [`artifacts/demo_session/SECURITY_REPORT.md`](artifacts/demo_session/SECURITY_REPORT.md) -- the final report
- [`artifacts/demo_session/triage.yaml`](artifacts/demo_session/triage.yaml) -- triaged findings with attack chains
- [`artifacts/demo_session/recon.yaml`](artifacts/demo_session/recon.yaml) -- codebase reconnaissance

## Customization

### Adding a Vulnerability Category

1. Add the category definition to `vuln-categories.md` with grep patterns, glob patterns, vulnerable code examples, false positive indicators, and CWE IDs
2. Add the category ID to the selection logic in `reconnaissance.md`
3. The category-scanner worker handles it automatically -- it loads patterns from vuln-categories at runtime

### Adjusting Quality Thresholds

Edit `security-quality-gate.md`:
- `coverage` threshold (default: 0.7) -- lower for large monorepos
- `evidence_quality` threshold (default: 0.6) -- raise for compliance audits
- `confidence_distribution` threshold (default: 0.5) -- raise for fewer false positives

### Adjusting Severity Scoring

Edit `severity-classifier.md`:
- Modify scoring factors (attack vector, complexity, etc.)
- Add or change compounding rules for attack chains
- Adjust score ranges for severity levels

### Targeting Specific Languages

The category-scanner adapts automatically based on recon output. To focus on a specific stack:
- Edit grep patterns in `vuln-categories.md` for your language
- Adjust glob patterns to match your file structure
- Add framework-specific patterns (e.g., Django ORM vs raw SQL)

## Responsible Use

This pipeline is designed for **authorized security testing** of codebases you own or have explicit permission to audit. It helps development teams find and fix vulnerabilities in their own code.

- Run on your own projects or with written authorization
- Share reports only with authorized stakeholders
- Use findings to improve security, not to exploit vulnerabilities
- The pipeline identifies patterns -- human review is required to confirm exploitability

## Comparison: Security Audit vs Research Pipeline

| Dimension | Security Audit | Research Pipeline |
|-----------|---------------|-------------------|
| **Partitioning** | By vulnerability category | By research aspect |
| **Worker input** | Category patterns + codebase | Aspect description + queries |
| **Data source** | Local codebase (Grep/Read) | Web search (Exa/WebSearch) |
| **Aggregation** | Triage with attack chains | Synthesis with cross-references |
| **Quality gate** | Coverage + evidence + confidence | Saturation + diversity + tiers |
| **Use case** | Security auditing, code review | Research, analysis, reports |
