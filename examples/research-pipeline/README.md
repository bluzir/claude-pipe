# Research Pipeline Example

A complete multi-phase research pipeline demonstrating the **"subagent = aspect"** pattern for parallel deep research with source quality evaluation, slop detection, and grounded report generation.

> **Framework Documentation:** See the main [README.md](../../README.md) for framework concepts and conventions.

## Pattern: Subagent = Aspect

Semantic decomposition of a research topic into independent aspects, each researched in parallel by a dedicated worker agent. Results are synthesized across aspects, quality-gated, and compiled into a grounded report.

```
                            ┌─────────────────┐
                            │   Manager       │
                            │ (orchestrator)  │
                            └────────┬────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  Researcher     │    │  Researcher     │    │  Researcher     │
    │ (aspect: arch)  │    │ (aspect: tools) │    │ (aspect: risks) │
    └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
             │                      │                      │
             ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  8-15 findings  │    │  8-15 findings  │    │  8-15 findings  │
    │  tiered sources │    │  tiered sources │    │  tiered sources │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Key Characteristics

- **Semantic partitioning**: Split topic by meaning, not by count
- **True parallelism**: All aspect researchers run concurrently
- **Source tiering**: Every source classified S/A/B/C/D/X with recency weights
- **Slop detection**: AI-generated content flagged and downweighted
- **Grounded output**: Every claim traceable to a URL
- **Quality gate**: Automated pass/warn/fail before report generation

## Prerequisites

### Exa MCP Server

This pipeline uses [Exa](https://exa.ai) for web search via MCP. Setup:

1. **Get an Exa API key** at [exa.ai](https://exa.ai)

2. **Add Exa MCP server** to your Claude Code settings.

   Project-level (`.claude/settings.local.json`):
   ```json
   {
     "permissions": {
       "allow": [
         "mcp__exa__web_search_exa",
         "mcp__exa__crawling_exa"
       ]
     },
     "mcpServers": {
       "exa": {
         "command": "npx",
         "args": ["-y", "exa-mcp-server"],
         "env": {
           "EXA_API_KEY": "your-key-here"
         }
       }
     }
   }
   ```

   Or user-level (`~/.claude/settings.json`) if you want Exa available globally.

3. **Verify** — start Claude Code and confirm `mcp__exa__web_search_exa` appears in available tools.

> **Without Exa**: You can adapt `aspect-researcher.md` to use the built-in `WebSearch`/`WebFetch` tools instead. Exa gives better results for technical research (semantic search, content extraction), but the pipeline works with any search provider.

## Structure

```
research-pipeline/
├── .claude/
│   ├── agents/
│   │   ├── aspect-researcher.md    # Worker: researches one aspect via Exa
│   │   └── report-generator.md     # Worker: generates grounded report
│   ├── skills/
│   │   ├── manager-research.md     # Manager: 5-phase pipeline orchestration
│   │   ├── research-planner.md     # Atomic: topic → aspects + queries
│   │   ├── synthesis.md            # Composite: cross-aspect aggregation
│   │   ├── quality-gate.md         # Atomic: PASS/WARN/FAIL verdict
│   │   ├── grounding-protocol.md   # Domain: no-hallucination rules
│   │   ├── silence-protocol.md     # Domain: suppress chat, file-only output
│   │   ├── search-safeguard.md     # Atomic: Exa API retry + jitter
│   │   ├── io-yaml-safe.md         # Atomic: safe YAML writing
│   │   ├── yaml-repair.md          # Atomic: auto-fix broken YAML
│   │   ├── phase-checkpoint.md     # Atomic: git checkpoint per phase
│   │   └── resume-checkpoint.md    # Atomic: restore from checkpoint
├── workflows/
│   └── research.yaml               # Declarative workflow definition
├── templates/                      # Templates for extending the pipeline
│   └── skills/
│       ├── checkpoint.template.md
│       └── repair.template.md
├── artifacts/                      # Runtime outputs (gitignored)
│   └── {session_id}/
│       ├── state.yaml              # Pipeline state for resume
│       ├── plan.yaml               # Decomposed topic
│       ├── aspects/*.yaml          # Per-aspect findings
│       ├── synthesis.yaml          # Aggregated insights
│       ├── quality.yaml            # Quality verdict
│       └── FINAL_REPORT.md         # Deliverable
└── README.md
```

## Running

```
# Default: medium depth (5 aspects)
/research "AI agents orchestration patterns"

# Quick (3 aspects)
/research quick "REST API design best practices"

# Deep (7 aspects)
/research deep "Trade-offs between microservices and monoliths in 2026"
```

## Pipeline

```
Phase 1        Phase 2           Phase 3       Phase 4        Phase 5
Planning  ───▶ Research ×N ───▶ Synthesis ───▶ Quality ───▶ Report
   │              │                 │             │            │
   ▼              ▼                 ▼             ▼            ▼
plan.yaml    aspects/*.yaml   synthesis.yaml  quality.yaml  FINAL_REPORT.md
```

```
┌─────────────┐     ┌─────────────────┐     ┌───────────┐
│  Planning   │────▶│ Parallel Research│────▶│ Synthesis │
│  (planner)  │     │ (N researchers)  │     │           │
└─────────────┘     └─────────────────┘     └─────┬─────┘
                                                  │
                    ┌─────────────────┐     ┌─────▼─────┐
                    │  Report Gen     │◀────│  Quality  │
                    │  (generator)    │     │   Gate    │
                    └─────────────────┘     └───────────┘
```

## Phases

| Phase | Agent/Skill | Input | Output |
|-------|-------------|-------|--------|
| 1. Planning | research-planner | topic, depth | plan.yaml |
| 2. Research | aspect-researcher ×N | plan.aspects | aspects/*.yaml |
| 3. Synthesis | synthesis | aspects/*.yaml | synthesis.yaml |
| 4. Quality | quality-gate | synthesis.yaml | quality.yaml |
| 5. Report | report-generator | synthesis + plan + quality | FINAL_REPORT.md |

## Source Quality

Every source is evaluated on two dimensions before being used.

### Tier Classification

| Tier | Weight | Examples |
|------|--------|---------|
| S | 1.0 | github.com, arxiv.org, official docs, RFCs |
| A | 0.8 | Personal tech blogs, dev.to, Hacker News, lobste.rs |
| B | 0.6 | Medium (with named author), Stack Overflow |
| C | 0.4 | News sites, tech aggregators |
| D | 0.2 | Generic content sites |
| X | 0.0 | SEO farms — skipped entirely |

### Recency Weights

| Age | Weight |
|-----|--------|
| <6 months | 1.0 |
| 6-18 months | 0.8 |
| 18-36 months | 0.6 |
| >36 months | 0.4 |

### Slop Detection

Sources are checked for AI-generated content:

- **>80% AI content** → skip entirely
- **>60% AI content** → flagged, lower weight
- **<60%** → used normally

This prevents the pipeline from laundering LLM output back through search results into the report.

## Data Flow

```yaml
L1 (Directives):
  - plan.yaml           # Decomposed topic, constraints

L2 (Operational):
  - aspects/arch.yaml    # Research findings per aspect
  - aspects/tools.yaml
  - aspects/risks.yaml
  - synthesis.yaml       # Aggregated insights
  - quality.yaml         # Quality verdict

L3 (Artifacts):
  - FINAL_REPORT.md      # Deliverable
```

## Example Aspect Decompositions

### "AI agents orchestration patterns" (medium — 5 aspects)

| Aspect | Description | Sample Queries |
|--------|-------------|----------------|
| Architecture Patterns | Hierarchical, mesh, swarm structures | "multi-agent architecture patterns", "hierarchical vs mesh agent orchestration" |
| Inter-Agent Communication | Message passing, shared state, events | "agent communication protocols", "shared state vs message passing agents" |
| Orchestration Frameworks | LangGraph, AutoGen, CrewAI, Claude tools | "LangGraph vs AutoGen vs CrewAI 2026", "multi-agent orchestration frameworks" |
| Challenges & Limitations | Coordination, state management, debugging | "multi-agent system challenges", "agent debugging and observability" |
| Implementation Patterns | Tool use, prompt chaining, memory | "AI agent tool use patterns", "agent memory architecture production" |

### "Effective prompt engineering for code generation" (quick — 3 aspects)

| Aspect | Description | Sample Queries |
|--------|-------------|----------------|
| Structural Techniques | Chain-of-thought, few-shot, system prompts | "chain of thought prompting code generation", "few-shot examples programming LLM" |
| Error Reduction | Grounding, validation, self-correction loops | "reducing hallucination code generation", "LLM self-correction patterns code" |
| Evaluation Methods | Benchmarks, human eval, pass@k | "code generation evaluation benchmarks 2026", "pass@k metric LLM coding" |

### "PostgreSQL performance tuning for analytics" (deep — 7 aspects)

| Aspect | Description | Sample Queries |
|--------|-------------|----------------|
| Query Optimization | EXPLAIN ANALYZE, planner hints, CTE strategies | "PostgreSQL query planner optimization analytics", "CTE vs subquery performance" |
| Indexing | B-tree, GIN, BRIN, partial indexes | "PostgreSQL BRIN index analytics workload", "partial index strategies Postgres" |
| Partitioning | Range, list, hash; partition pruning | "PostgreSQL table partitioning 2026", "partition pruning performance Postgres" |
| Configuration | shared_buffers, work_mem, parallel workers | "PostgreSQL config tuning analytics", "parallel query settings Postgres" |
| Data Modeling | Columnar storage, materialized views | "columnar extension PostgreSQL", "materialized view refresh strategies" |
| Monitoring | pg_stat_statements, auto_explain, wait events | "PostgreSQL monitoring production", "pg_stat_statements query analysis" |
| Infrastructure | Storage, replicas, connection pooling | "PostgreSQL hardware sizing analytics", "PgBouncer vs pgcat comparison" |

## Key Concepts Demonstrated

- **Manager skill** orchestrating the pipeline from ROOT
- **Parallel fan-out** for the research phase
- **Sequential gates** between phases
- **Quality loop** with PASS/WARN/FAIL routing
- **Source tiering** (S/A/B/C/D/X) with recency and slop weights
- **Grounding protocol** — every claim traceable to a source
- **Silence protocol** — clean parallel execution without chat noise
- **File-based state** for resume capability
- **Git checkpointing** for time-travel debugging

## Customization

To adapt this pipeline:

1. **Topic decomposition** — Edit `research-planner.md` to change how aspects are generated, or add domain-specific heuristics
2. **Source evaluation** — Modify tier definitions in `aspect-researcher.md` for your domain (e.g., add PubMed as S-tier for medical research)
3. **Quality thresholds** — Adjust pass/warn/fail boundaries in `quality-gate.md`
4. **Report format** — Customize `report-generator.md` for your output style (executive brief, technical report, etc.)
5. **Search provider** — Replace Exa MCP calls with `WebSearch`/`WebFetch` or another MCP search server
6. **Domain skills** — Add specialized skills (e.g., `academic-protocol.md` for scholarly research, `compliance-check.md` for regulated domains)

## Comparison: Research vs Batch Classifier

| Dimension | Research Pipeline | Batch Classifier |
|-----------|-------------------|-------------------|
| **Partitioning** | Semantic (by meaning) | Data (by count) |
| **Worker input** | Research aspect with queries | Fixed-size batch of items |
| **Parallelism** | All aspects simultaneously | All batches simultaneously |
| **Aggregation** | Synthesis with cross-referencing | Counts and statistics |
| **Quality gate** | PASS/WARN/FAIL with 4 metrics | None (direct aggregation) |
| **Use case** | Research, analysis, reports | Classification, extraction, tagging |
