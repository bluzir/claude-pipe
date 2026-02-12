---
name: research-planner
type: atomic
version: v1.0
description: "Decompose topic into researchable aspects with queries"
input:
  required:
    - topic
  optional:
    - depth
    - focus_areas
output:
  type: data
  schema: plan.yaml
---

# Research Planner

## Purpose

Decompose a research topic into distinct aspects, each with targeted search queries. Produces a structured plan for parallel research.

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `topic` | string | Yes | Research topic |
| `depth` | enum | No | quick (3 aspects), medium (5), deep (7) |
| `focus_areas` | string[] | No | Areas to prioritize |

## Procedure

### Step 1: Topic Analysis

Analyze the topic to identify:
- Core concept
- Key dimensions (technical, market, social, etc.)
- Temporal aspects (current state, trends, future)
- Stakeholder perspectives

### Step 2: Aspect Generation

Generate aspects based on depth:

| Depth | Aspects | Focus |
|-------|---------|-------|
| quick | 3 | Core dimensions only |
| medium | 5 | Core + context |
| deep | 7 | Comprehensive coverage |

Each aspect must be:
- **Distinct:** No significant overlap with others
- **Researchable:** Can find sources via web search
- **Bounded:** Clear scope

### Step 3: Query Generation

For each aspect, generate 3-5 search queries:

**Query Types:**
- Definitional: "What is {concept}"
- Comparative: "{concept} vs {alternative}"
- Practical: "{concept} implementation"
- Expert: "{concept} best practices"
- Recent: "{concept} 2025 2026"

**Query Quality:**
- Specific enough to get relevant results
- Not so narrow that results are sparse
- Include temporal markers for freshness

### Step 4: Output

Generate plan.yaml with session metadata.

## Output Schema

```yaml
topic: "Original topic"
depth: quick|medium|deep
created_at: "timestamp"
session_id: "research_{date}_{random}"

aspects:
  - id: "aspect_1"
    name: "Aspect Name"
    description: "What this aspect covers"
    priority: 1  # 1 = highest
    queries:
      - "query 1"
      - "query 2"
      - "query 3"

  - id: "aspect_2"
    name: "Another Aspect"
    description: "Coverage description"
    priority: 2
    queries:
      - "query 1"
      - "query 2"

settings:
  max_sources_per_aspect: 15
  min_findings_per_aspect: 5
  min_aspects_for_synthesis: 3
```

## Examples

### Example 1: Medium Depth

**Input:**
```
topic: "AI agents orchestration patterns"
depth: medium
```

**Output:**
```yaml
topic: "AI agents orchestration patterns"
depth: medium
created_at: "2026-01-30T10:00:00Z"
session_id: "research_20260130_x7k9m"

aspects:
  - id: "architecture"
    name: "Architecture Patterns"
    description: "Structural patterns for multi-agent systems: hierarchical, mesh, swarm"
    priority: 1
    queries:
      - "multi-agent system architecture patterns"
      - "hierarchical vs mesh agent orchestration"
      - "agent swarm coordination patterns"

  - id: "communication"
    name: "Inter-Agent Communication"
    description: "How agents communicate: message passing, shared state, events"
    priority: 2
    queries:
      - "agent to agent communication protocols"
      - "multi-agent message passing patterns"
      - "shared state vs message passing agents"

  - id: "tools"
    name: "Orchestration Frameworks"
    description: "Frameworks and tools: LangGraph, AutoGen, CrewAI, Claude Code"
    priority: 2
    queries:
      - "LangGraph vs AutoGen vs CrewAI comparison 2026"
      - "multi-agent orchestration frameworks"
      - "AI agent framework comparison"

  - id: "challenges"
    name: "Challenges & Limitations"
    description: "Common problems: coordination, state management, debugging"
    priority: 3
    queries:
      - "multi-agent system challenges"
      - "AI agent debugging and observability"
      - "agent coordination problems"

  - id: "implementation"
    name: "Implementation Patterns"
    description: "Tool use, prompt chaining, memory, file-based state"
    priority: 3
    queries:
      - "AI agent tool use patterns production"
      - "agent memory architecture implementation"
      - "file-based state management agents"

settings:
  max_sources_per_aspect: 15
  min_findings_per_aspect: 5
  min_aspects_for_synthesis: 3
```

### Example 2: Quick Depth

**Input:**
```
topic: "Effective prompt engineering for code generation"
depth: quick
```

**Output:**
```yaml
topic: "Effective prompt engineering for code generation"
depth: quick
created_at: "2026-02-10T14:00:00Z"
session_id: "research_20260210_p3q8r"

aspects:
  - id: "techniques"
    name: "Structural Techniques"
    description: "Chain-of-thought, few-shot examples, system prompts, structured output"
    priority: 1
    queries:
      - "chain of thought prompting code generation"
      - "few-shot examples programming LLM best practices"

  - id: "error-reduction"
    name: "Error Reduction"
    description: "Grounding, validation loops, self-correction, hallucination prevention"
    priority: 1
    queries:
      - "reducing hallucination code generation LLM"
      - "LLM self-correction patterns code output"

  - id: "evaluation"
    name: "Evaluation Methods"
    description: "Benchmarks, pass@k, human eval, automated quality metrics"
    priority: 2
    queries:
      - "code generation evaluation benchmarks 2026"
      - "pass@k metric LLM coding assessment"

settings:
  max_sources_per_aspect: 15
  min_findings_per_aspect: 5
  min_aspects_for_synthesis: 3
```

### Example 3: Deep Depth

**Input:**
```
topic: "PostgreSQL performance tuning for analytics workloads"
depth: deep
```

**Output:**
```yaml
topic: "PostgreSQL performance tuning for analytics workloads"
depth: deep
created_at: "2026-02-12T09:00:00Z"
session_id: "research_20260212_k4m2n"

aspects:
  - id: "query-optimization"
    name: "Query Planning & Optimization"
    description: "EXPLAIN ANALYZE, planner hints, CTE strategies, join optimization"
    priority: 1
    queries:
      - "PostgreSQL query planner optimization analytics"
      - "CTE vs subquery performance PostgreSQL"
      - "PostgreSQL join strategies large tables"
      - "EXPLAIN ANALYZE interpreting slow queries"

  - id: "indexing"
    name: "Indexing Strategies"
    description: "B-tree, GIN, BRIN, partial indexes for analytical queries"
    priority: 1
    queries:
      - "PostgreSQL BRIN index analytics workload"
      - "partial index strategies Postgres large tables"
      - "GIN index JSONB performance PostgreSQL"
      - "covering indexes PostgreSQL optimization"

  - id: "partitioning"
    name: "Table Partitioning"
    description: "Range, list, hash partitioning; partition pruning trade-offs"
    priority: 2
    queries:
      - "PostgreSQL table partitioning analytics 2026"
      - "partition pruning performance PostgreSQL"
      - "declarative partitioning best practices"

  - id: "configuration"
    name: "Configuration Tuning"
    description: "shared_buffers, work_mem, parallel workers, JIT compilation"
    priority: 2
    queries:
      - "PostgreSQL configuration analytics workload"
      - "parallel query tuning PostgreSQL"
      - "JIT compilation PostgreSQL when to enable"
      - "work_mem tuning complex queries"

  - id: "data-modeling"
    name: "Data Modeling"
    description: "Columnar storage, materialized views, denormalization strategies"
    priority: 2
    queries:
      - "columnar extension PostgreSQL analytics"
      - "materialized view refresh strategies production"
      - "star schema vs flat tables PostgreSQL"

  - id: "monitoring"
    name: "Monitoring & Profiling"
    description: "pg_stat_statements, auto_explain, wait events, query fingerprinting"
    priority: 3
    queries:
      - "PostgreSQL performance monitoring production"
      - "pg_stat_statements analysis slow queries"
      - "auto_explain configuration PostgreSQL"

  - id: "infrastructure"
    name: "Hardware & Architecture"
    description: "Storage selection, read replicas, connection pooling, caching layers"
    priority: 3
    queries:
      - "PostgreSQL hardware sizing analytics workload"
      - "PgBouncer vs pgcat connection pooling comparison"
      - "read replica strategies PostgreSQL analytics"

settings:
  max_sources_per_aspect: 15
  min_findings_per_aspect: 5
  min_aspects_for_synthesis: 3
```

## Quality Criteria

- [ ] Aspects are distinct (no major overlap)
- [ ] Each aspect has 2-5 queries (matching depth)
- [ ] Queries are searchable (not too broad/narrow)
- [ ] Queries include temporal markers for freshness
- [ ] Priorities assigned (1 = core, 2 = context, 3 = peripheral)
- [ ] Settings included
