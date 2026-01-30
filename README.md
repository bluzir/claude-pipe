# Agent Orchestration Framework v1.0

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Compatible-blue.svg)](https://claude.ai/code)

Conventions and templates for organizing multi-agent systems on top of Claude Code primitives.

> **New to the framework?** Check out [QUICKSTART.md](QUICKSTART.md) for a 5-minute getting started guide.

---

## Table of Contents

- [Philosophy](#philosophy)
- [Why Claude Code](#why-claude-code)
- [Who Is This For](#who-is-this-for)
- [The Problem](#the-problem)
- [Framework Architecture](#framework-architecture)
- [Agent Taxonomy](#agent-taxonomy)
- [Data Layers (L0-L3)](#data-layers-l0-l3)
- [Orchestration (Deep Dive)](#orchestration-deep-dive)
- [Interface Schema](#interface-schema-autogen-ui)
- [Framework Structure](#framework-structure)
- [Usage](#usage)
- [When to Use This Framework](#when-to-use-this-framework)
- [Adoption Path](#adoption-path)
- [Principles (TL;DR)](#principles-tldr)

---

## Philosophy

```
Markdown is the new JavaScript.
YAML is the new PostgreSQL.
```

**Markdown** is code for LLMs. Instructions, procedures, constraints. LLMs execute markdown like a runtime executes JS.

**YAML** is data. Structured, versionable, human-readable. Replaces databases for most agent workflows.

### Approach: Framework + Domain → Pipeline

You don't write agents by hand. You:

1. **Provide the framework** — conventions, schemas, templates
2. **Provide domain knowledge** — specifics of your area (health, research, marketing)
3. **Ask Claude to package it into a pipeline**

```
┌─────────────────┐   ┌─────────────────┐
│   Framework     │ + │  Domain Data    │
│   (conventions) │   │  (your context) │
└────────┬────────┘   └────────┬────────┘
         │                     │
         └──────────┬──────────┘
                    ▼
         ┌─────────────────────┐
         │   Claude Code       │
         │   generates         │
         │   pipeline          │
         └─────────────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │  Working agents     │
         │  + skills           │
         │  + workflows        │
         └─────────────────────┘
```

**Why this works:**

- LLMs understand conventions better than API docs
- Markdown templates = examples for in-context learning
- YAML schemas = constraints for structured output
- Framework = shared language between you and Claude

**Example prompt:**

```
I have a framework (framework/README.md) and domain knowledge
about health tracking (data/config/user_profile.yaml, training-science.md).

Create a pipeline that:
1. Reads daily log
2. Validates against directives
3. Generates recommendations

Use framework conventions: agents, skills, data layers.
```

Claude creates:
- `manager-daily-review.md` (orchestration skill)
- `log-analyzer.md` (worker agent)
- `recommendation-generator.md` (worker agent)
- `dashboard.yaml` (interface)

Everything follows conventions. Everything is compatible with other pipelines.

---

## Why Claude Code

### The Core Advantage: Iteration Speed

When you're building agents for tasks that have never been automated before, you're on the frontier. There's no playbook. No proven architecture. No reference implementation to copy.

The only way forward is iteration: try something → see what breaks → fix it → repeat.

The system that lets you iterate fastest wins.

### Self-Improving System

Claude Code isn't just a tool you use. Combined with this framework, it becomes a system that improves itself.

```
You: "This agent is hallucinating sources"

Claude Code:
  1. Reads agent definition
  2. Identifies the problem
  3. Adds grounding-protocol skill
  4. Tests the fix
  5. Commits the change

Elapsed: 2 minutes
```

The same system that runs your agents can modify your agents. No context switching. No deployment pipeline. No waiting.

When you find a bug in your pipeline, you don't file a ticket. You describe it, and the system fixes itself.

### Text Files = Velocity

Why markdown and YAML instead of code and databases?

**Immediate feedback loop:**
```
Edit an agent → test immediately
No compilation. No deployment. No restart.
```

**Human-readable diffs:**
```diff
- model: haiku
+ model: sonnet

- max_sources: 5
+ max_sources: 10
```

You can review agent changes like code changes. Your git history tells the story of how your agents evolved.

**LLM-native format:**
```
Claude Code can read agent definitions.
Claude Code can write agent definitions.
The format is the interface.
```

No serialization. No ORM. No API layer between Claude and your agent configurations.

**Version control:**
```
git checkout HEAD~5 -- .claude/agents/my-agent.md
```

Roll back an agent to last week's version. Branch to experiment. Merge improvements from a colleague.

This is why markdown and YAML beat code and databases for agent development: they remove every obstacle between "I have an idea" and "it's running."

---

## Who Is This For

You're building agents on Claude Code and encountering:
- Pipelines with multiple steps (research → analyze → generate)
- Need to reuse knowledge between agents
- Questions like "where to store state?" and "how to resume after failure?"
- Desire for structure, but without heavy frameworks

This framework is a set of **conventions**, not a library. You use it as a reference and adapt it to your needs.

---

## The Problem

### Why Multi-Agent Systems Are Hard

**1. Coordination Explosion**

One agent = prompt + tools. Simple.

Two agents = who calls whom? how to pass data? what if one fails?

Five agents = exponential complexity. Each can affect each other.

```
Agents:     1    2    3    4    5
Complexity: O(1) O(n) O(n²) ...
```

**2. No Agreed Patterns**

In web development there's MVC, REST, microservices. Everyone understands these.

In agent development:
- Some make monolithic agents with 3000 lines
- Some split into 50 micro-agents
- Some write in LangChain, some in AutoGen, some in raw API

No common language. No reusable patterns.

**3. State Management Chaos**

Where does state live between agent calls?

| Option | Problem |
|--------|---------|
| Context window | Lost in long sessions, expensive |
| In-memory | Lost on restart |
| Database | Overkill, requires infrastructure |
| Files | What format? who owns them? how to version? |

**4. Debugging Hell**

A pipeline of 5 agents fails on step 4.

- Where exactly did it break?
- How to reproduce?
- How to restart from midpoint?
- What was the intermediate data?

Observability in agent systems is an unsolved industry problem.

**5. Knowledge Duplication**

Every agent contains copy-pasted instructions:
- "Don't hallucinate"
- "Verify sources"
- "Format output as YAML"
- "Don't use AI-typical phrases"

Code reuse was solved long ago. Knowledge reuse hasn't been.

---

### Approaches That Don't Work

**Monolithic Agent**
```
One giant system prompt with all logic.
```
- Context overflow on complex tasks
- Can't test parts individually
- One change = risk of breaking everything
- No parallelism

**Micro-Agents**
```
Each action = separate agent.
```
- Coordination overhead
- Context loss between calls
- Debugging 50 agents = nightmare
- Latency from sequential calls

**Heavy Frameworks (LangChain, AutoGen)**
```
Use a framework for everything.
```
- Leaky abstractions
- Hard to customize for your needs
- Dependent on someone else's roadmap
- Documentation lags behind code

---

### How This Framework Solves These Problems

| Problem | Solution |
|---------|----------|
| Coordination explosion | **Flat hierarchy**: ROOT orchestrates, Workers execute |
| No patterns | **Taxonomy**: 4 architectural roles, 8 functional roles |
| State chaos | **Data Layers**: L0→L1→L2→L3 with clear contracts |
| Debugging | **File-based state**: inspect, resume, replay |
| Knowledge duplication | **Skills**: shared libraries for instructions |

**Key Insight: Convention over Configuration**

Claude Code already has primitives:

| Primitive | What It Does |
|-----------|--------------|
| Task tool | Spawn subagent |
| Skill tool | Load knowledge pack |
| MCP | External integrations |
| Files | Persistent state |

No need to invent a runtime. You need **conventions** for how to use these primitives together.

**Architectural Constraint as Feature**

```
Claude Code: Subagents CANNOT spawn other subagents
```

This looks like a limitation, but it's a design decision:
- Forces flat hierarchy (easier to debug)
- ROOT = single point of control
- Workers = pure executors (easier to test)

**Skills = Knowledge Reuse**

```
Agent A + grounding-protocol = Agent A that doesn't hallucinate
Agent B + grounding-protocol = Agent B that doesn't hallucinate
```

A Skill is not an agent. A Skill is a set of instructions loaded into context.
This is knowledge reuse.

**Files = Reliable State**

```
Why files > memory:
✓ Inspectable (humans can read)
✓ Versionable (git-friendly)
✓ Resumable (restart from any point)
✓ Testable (mock data for testing)
```

---

## Framework Architecture

### Mental Model (Like an OS)

```
┌─────────────────────────────────────────────────────┐
│                     USER                            │
│                  (slash commands)                   │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│                    ROOT                             │
│              (Claude Code CLI)                      │
│                                                     │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │  Skill  │  │  Skill  │  │  Skill  │  Skills   │
│   │(manager)│  │(atomic) │  │(domain) │  (libs)   │
│   └─────────┘  └─────────┘  └─────────┘           │
│                                                     │
│   ┌─────────────────────────────────────────────┐  │
│   │              Task tool                       │  │
│   │         (spawn subagents)                   │  │
│   └─────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌───────────┐  ┌───────────┐  ┌───────────┐
│  Worker   │  │  Worker   │  │  Worker   │
│ (subagent)│  │ (subagent)│  │ (subagent)│
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │              │              │
      ▼              ▼              ▼
┌───────────────────────────────────────────┐
│                  FILES                     │
│           (L0 → L1 → L2 → L3)             │
│         (persistent state layer)          │
└───────────────────────────────────────────┘
```

### Key Principles

**1. ROOT orchestrates, Workers execute**

ROOT (the main Claude Code process) is the only one that can:
- Spawn subagents (Task tool)
- Load skills (Skill tool)
- Coordinate phases

Workers are isolated executors:
- Receive task
- Execute
- Write result to file
- Terminate

**2. Skills are shared libraries**

A Skill is NOT an agent. A Skill is a set of instructions loaded into context.

```
Agent A + Skill X = Agent A with knowledge X
Agent B + Skill X = Agent B with knowledge X
```

Skill types:
| Type | Scope | Example |
|------|-------|---------|
| atomic | Single operation | tier-weights, slop-check |
| composite | Combination of atomic | source-evaluation |
| domain | Knowledge pack | training-science, grounding-protocol |
| manager | Workflow instructions | manager-research |

**3. Files are state**

Everything persists to YAML files:
- Reproducible: can restart from any point
- Inspectable: humans can read and edit
- Testable: can substitute mock data
- Versionable: git-friendly

**4. Layers have contracts**

```
L0 (Config)      →  L1 (Directives)  →  L2 (Operational)  →  L3 (Artifacts)
user_profile.yaml   plan.yaml           aspects/*.yaml       FINAL_REPORT.md
                    directives.yaml     synthesis.yaml
```

Contracts:
- L1 constraints ALWAYS override L2 decisions
- L3 artifacts MUST reference L2 sources
- L2 CAN BE regenerated from L1 + external data

**5. Manager = Skill, not Agent**

Manager doesn't run as a subagent.
Manager is a skill that ROOT reads and executes the instructions itself.

Why:
- Subagents cannot spawn subagents
- ROOT must see the whole pipeline
- Debugging is easier when orchestration is in one place

---

## Agent Taxonomy

### By Architectural Role

| Role | Execution | Can Spawn | Purpose |
|------|-----------|-----------|---------|
| **commander** | ROOT | Yes | Entry point, slash commands |
| **manager** | ROOT skill | No* | Workflow instructions |
| **worker** | Subagent | No | Isolated task executor |
| **utility** | Inline skill | No | Stateless procedure |

*Manager = Skill loaded into ROOT, ROOT spawns workers.

### By Functional Role

| Role | Pattern | Example |
|------|---------|---------|
| explorer | Wide search → Structured output | topic-explorer |
| researcher | Queries → Evaluate → Extract | aspect-researcher |
| scorer | Input → Criteria → Scores | source-evaluator |
| aggregator | N inputs → Synthesis | findings-synthesizer |
| validator | Input → Rules → Verdict | quality-gate |
| generator | Context → Template → Artifact | report-generator |
| transformer | Format A → Format B | yaml-to-markdown |
| operator | Command → State change | log-manager |

---

## Data Layers (L0-L3)

| Layer | Name | Mutability | Lifecycle | Example |
|-------|------|------------|-----------|---------|
| **L0** | Config | User editable | Persistent | user_profile.yaml |
| **L1** | Directives | Agent → User approved | Session persistent | plan.yaml |
| **L2** | Operational | Agent generated | Session scoped | aspects/*.yaml |
| **L3** | Artifacts | Append-only | Permanent | FINAL_REPORT.md |

### Layer Contracts

```yaml
L0 → L1:
  trigger: "Strategic review"
  process: "Analyze L0 → Generate directives → User approves"
  validation: "L1 must not contradict L0"

L1 → L2:
  trigger: "Operational command"
  process: "Load L1 constraints → Execute → Write L2"
  validation: "L1 ALWAYS wins over L2 preferences"

L2 → L3:
  trigger: "Quality gate PASS"
  process: "Aggregate L2 → Quality check → Generate artifact"
  validation: "L3 must trace to L2 sources"
```

---

## Orchestration (Deep Dive)

Orchestration is how ROOT coordinates execution of multi-step pipelines. This is the core part of the framework.

### Constraint: Flat Hierarchy

```
                    ┌──────────────────────────────────┐
                    │  Claude Code Constraint:         │
                    │  Subagents CANNOT spawn          │
                    │  other subagents                 │
                    └──────────────────────────────────┘
```

This is not a bug, it's a feature. The constraint forces architecture:

```
✓ ROOT → Worker (allowed)
✗ Worker → Worker (NOT allowed)
```

**Consequences:**
- All orchestration logic lives in ROOT
- Workers = pure executors, no coordination logic
- Debugging is easier: single point of control

### Working Around the Constraint: Manager Skills

Problem: ROOT doesn't know the workflow but must orchestrate.

Solution: Manager Skill = instructions for ROOT.

```
┌────────────────────────────────────────────────────┐
│ ROOT loads manager-research.md as a Skill          │
│                                                    │
│ Manager Skill contains:                            │
│ - Description of phases                            │
│ - Gate conditions                                  │
│ - Instructions on how to spawn workers             │
│ - State management rules                           │
│                                                    │
│ ROOT reads instructions and EXECUTES THEM ITSELF   │
│ (doesn't delegate to another agent)                │
└────────────────────────────────────────────────────┘
```

**Manager ≠ Agent.** Manager is a knowledge pack that ROOT interprets.

### Orchestration Primitives

ROOT has three primitives for orchestration:

**1. Task tool (spawn)**
```
Task(
  subagent_type: "general-purpose",
  prompt: "Load X agent, do Y",
  run_in_background: true|false
)
→ Returns: task_id
```

**2. TaskOutput (wait)**
```
TaskOutput(
  task_id: "xxx",
  block: true
)
→ Returns: agent output
```

**3. Skill tool (load knowledge)**
```
Skill(skill: "my-skill")
→ Loads skill instructions into ROOT context
```

### Orchestration Patterns

#### Pattern 1: Fan-out (Parallel Execution)

When: N independent tasks that can run in parallel.

```
ROOT reads plan.yaml (5 aspects)
  │
  │  ┌─────── Single message with multiple Task calls ───────┐
  │  │                                                        │
  │  │  Task(worker-1, background: true) → task_id_1         │
  │  │  Task(worker-2, background: true) → task_id_2         │
  │  │  Task(worker-3, background: true) → task_id_3         │
  │  │  Task(worker-4, background: true) → task_id_4         │
  │  │  Task(worker-5, background: true) → task_id_5         │
  │  │                                                        │
  │  └────────────────────────────────────────────────────────┘
  │
  │  Wait phase:
  │  TaskOutput(task_id_1, block: true)
  │  TaskOutput(task_id_2, block: true)
  │  ... etc
  │
  ▼
Continue to next phase
```

**Important:** All Task calls must be in ONE message for parallel execution.

**Code in manager skill:**
```markdown
## Phase 2: Parallel Research

**Orchestration:**
For each aspect in plan.aspects:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load aspect-researcher agent.
      Research: {aspect.name}
      Output: artifacts/{session}/aspects/{aspect.id}.yaml
    run_in_background: true
  )

Wait for all tasks to complete.
```

#### Pattern 2: Pipeline (Sequential Phases)

When: Each phase depends on the previous phase's result.

```
Phase 1        Gate         Phase 2        Gate         Phase 3
┌──────┐    ┌──────┐      ┌──────┐     ┌──────┐      ┌──────┐
│ Plan │───▶│Check │───▶  │Research│───▶│Check │───▶  │Synth │
└──────┘    │exists│      └──────┘     │quality│      └──────┘
    │       └──────┘          │        └──────┘           │
    ▼                         ▼                           ▼
plan.yaml              aspects/*.yaml              synthesis.yaml
```

**Gate = transition condition between phases:**
```yaml
gate:
  type: file_exists
  condition: "plan.yaml"

gate:
  type: quality_threshold
  condition: "count(aspects/*.yaml) >= 3"

gate:
  type: custom
  condition: "quality.verdict in ['PASS', 'WARN']"
```

#### Pattern 3: Quality Loop

When: Need to achieve a certain quality level, possibly through multiple iterations.

```
┌─────────────────────────────────────────────────┐
│                                                 │
│    ┌──────────┐     ┌───────────┐     ┌─────┐  │
└───▶│ Research │────▶│ Synthesis │────▶│ QG  │──┤
     └──────────┘     └───────────┘     └─────┘  │
                                           │      │
                                    PASS   │      │ FAIL
                                    ───────┘      │
                                                  │
                                    ┌─────────────┘
                                    ▼
                              Gap Analysis
                              (what's missing?)
                                    │
                                    └──── back to Research
                                          with specific gaps
```

**Logic in manager skill:**
```markdown
## Quality Gate Routing

After quality-gate skill returns:

| Verdict | Action |
|---------|--------|
| PASS | Continue to report generation |
| WARN | Continue with caveats noted in report |
| FAIL | Analyze gaps → Re-run research for specific gaps |

On FAIL:
1. Read quality.yaml for missing areas
2. Generate targeted queries for gaps
3. Spawn additional researchers for gap areas only
4. Re-run synthesis with new data
5. Re-check quality
```

#### Pattern 4: Fork-Join

When: Multiple parallel branches that then merge.

```
              ┌──────────────┐
              │   Planning   │
              └──────┬───────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Branch  │ │ Branch  │ │ Branch  │
    │   A     │ │   B     │ │   C     │
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         └───────────┼───────────┘
                     ▼
              ┌──────────────┐
              │    Merge     │
              │  (synthesis) │
              └──────────────┘
```

**Code:**
```markdown
## Fork Phase
Spawn in parallel:
- Task(branch-a-worker, ...)
- Task(branch-b-worker, ...)
- Task(branch-c-worker, ...)

Wait for all.

## Join Phase
Gate: All branch outputs exist
Action: Synthesis skill merges results
```

#### Pattern 5: Conditional Routing

When: Different paths depending on results.

```
              ┌──────────────┐
              │   Analyze    │
              └──────┬───────┘
                     │
              ┌──────┴──────┐
              │  Condition  │
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Path A  │  │ Path B  │  │ Path C  │
   │(simple) │  │(medium) │  │(complex)│
   └─────────┘  └─────────┘  └─────────┘
```

**Code:**
```markdown
## Routing Logic

After analysis phase:
- IF complexity_score < 30 → Path A (simple pipeline)
- IF complexity_score < 70 → Path B (standard pipeline)
- ELSE → Path C (full deep research)

Each path has different:
- Number of aspects
- Source requirements
- Quality thresholds
```

### State Management

Orchestration requires state for:
- Resume after interrupt
- Track progress
- Debug failures

**State file structure:**
```yaml
# artifacts/{session}/state.yaml

session_id: "research_20260130_abc"
workflow: "manager-research"
started_at: "2026-01-30T10:00:00Z"
last_updated: "2026-01-30T10:35:00Z"

current_phase: "synthesis"

phase_states:
  planning: completed
  research: completed      # All workers done
  synthesis: in_progress   # Currently running
  quality_gate: pending
  report: pending

# Optional: track individual workers
workers:
  aspect_1: completed
  aspect_2: completed
  aspect_3: failed        # This one failed
  aspect_4: completed
  aspect_5: completed

error: null               # Or error message if failed
```

**State transitions:**
```
pending → in_progress → completed
                     → failed
                     → partial (some workers succeeded)
```

**Resume logic:**
```markdown
## On Resume

1. Read state.yaml
2. Find current_phase
3. Check phase_states:
   - If current_phase is "in_progress" → resume from there
   - If current_phase is "failed" → offer retry options
   - If current_phase is "completed" → move to next
4. Skip all "completed" phases
5. For "partial" phases → only re-run failed workers
```

### Error Handling

#### Worker Failure

```
Worker fails
    │
    ├── Log error to state.yaml
    │
    ├── Continue with other workers
    │
    └── At phase end:
        │
        ├── If completed >= minimum → continue to next phase
        │
        └── If completed < minimum → phase fails
```

**Code:**
```markdown
## Error Handling: Research Phase

On worker failure:
1. Update state: workers.{aspect_id} = "failed"
2. Log error: workers_errors.{aspect_id} = "{error}"
3. Continue with remaining workers

At phase end:
- Count completed workers
- IF completed >= plan.settings.min_aspects → Phase completed
- ELSE → Phase failed, halt pipeline

Recovery option:
- User can run: `/research retry-failed {session}`
- This re-runs only failed workers
```

#### Phase Failure

```
Phase fails
    │
    ├── Update state.yaml with error
    │
    ├── Halt pipeline
    │
    └── Report to user:
        - What phase failed
        - Why it failed
        - What was completed
        - Suggested actions
```

### Observability

What to log for debugging:

```yaml
# Per worker
worker_logs:
  aspect_1:
    started_at: timestamp
    completed_at: timestamp
    status: completed
    output_file: "aspects/aspect_1.yaml"
    metrics:
      queries_run: 3
      sources_found: 24
      sources_used: 8

# Per phase
phase_logs:
  research:
    started_at: timestamp
    completed_at: timestamp
    workers_total: 5
    workers_succeeded: 4
    workers_failed: 1
    duration_seconds: 180

# Aggregate
pipeline_metrics:
  total_sources: 32
  total_findings: 48
  quality_score: 0.84
  duration_seconds: 600
```

### Best Practices

**1. Always use state.yaml**

Even for simple pipelines. It's insurance against interrupts.

**2. Set timeouts**

Workers can hang. Set timeouts:
```
Task(..., timeout: 300000)  # 5 minutes
```

**3. Minimum thresholds, not exact counts**

```yaml
# Good: flexible
min_aspects_for_synthesis: 3

# Bad: rigid
required_aspects: 5  # Fails if one worker fails
```

**4. Idempotent workers**

A worker that restarts should produce the same result.
Check if output exists → skip or overwrite.

**5. Clear phase boundaries**

Each phase should:
- Read from specific files
- Write to specific files
- Have no side effects on other phases

---

## Interface Schema (Autogen UI)

Defines HOW to render data, not the data itself.

```yaml
sources:                              # Data binding
  state: "artifacts/{session}/state.yaml"
  synthesis: "artifacts/{session}/synthesis.yaml"

pages:
  - id: overview
    sections:
      - layout: grid                  # Layout type
        fields:
          - source: synthesis         # Which file
            path: quality_metrics.saturation  # JSONPath
            type: progress_bar        # Widget type
            label: "Saturation"
            threshold: 50
```

Render adapters transform to target:
| Target | Output |
|--------|--------|
| CLI | ASCII tables, progress bars |
| Chat | Markdown |
| API | JSON |
| Web | React components |

---

## Framework Structure

```
framework/
├── README.md                         # This file
├── LICENSE                           # MIT License
├── QUICKSTART.md                     # Getting started guide
├── CONTRIBUTING.md                   # Contribution guidelines
├── schemas/                          # YAML conventions
│   ├── agent.schema.yaml             # Agent definition
│   ├── skill.schema.yaml             # Skill definition
│   ├── workflow.schema.yaml          # Pipeline definition
│   ├── interface.schema.yaml         # UI definition
│   └── data-layers.schema.yaml       # L0-L3 conventions
├── templates/                        # Starting points
│   ├── agents/
│   │   ├── worker.template.md
│   │   ├── researcher.template.md
│   │   └── generator.template.md
│   └── skills/
│       ├── atomic.template.md
│       ├── composite.template.md
│       ├── domain.template.md
│       └── manager.template.md
└── examples/
    └── research-pipeline/            # Complete working example
```

---

## Usage

### 1. Create an Agent

```bash
cp framework/templates/agents/researcher.template.md \
   .claude/agents/my-researcher.md
# Edit for your task
```

### 2. Create a Skill

```bash
cp framework/templates/skills/atomic.template.md \
   .claude/skills/my-skill.md
# Define input → procedure → output
```

### 3. Create a Workflow (Manager Skill)

```bash
cp framework/templates/skills/manager.template.md \
   .claude/skills/manager-mypipeline.md
# Describe phases, gates, orchestration
```

### 4. Create an Interface

```yaml
# .claude/interfaces/dashboard.yaml
sources:
  data: "artifacts/{session}/synthesis.yaml"
pages:
  - id: main
    sections:
      - layout: card
        fields:
          - source: data
            path: key_metric
            type: stat
```

---

## When to Use This Framework

**Good fit:**
- Multi-step research pipelines
- Data processing workflows (gather → analyze → generate)
- Any task with phases and quality gates
- Projects where intermediate state needs persistence
- Teams that want shared conventions across agents

**Not a good fit:**
- Simple single-agent tasks
- Real-time chat applications
- Tasks without intermediate artifacts
- When you need sub-millisecond latency

---

## Adoption Path

### Level 1: Conventions Only
Use naming conventions and file structure without formal schemas.
```
.claude/
├── agents/          # Your agents
├── skills/          # Your skills
└── interfaces/      # Your dashboards
artifacts/
└── {session}/       # Runtime outputs
```

### Level 2: Schemas + Templates
Use templates for consistency. Validate agent and skill structure.

### Level 3: Full Pipeline
Manager skills for orchestration. Quality gates. State management.

---

## Principles (TL;DR)

1. **ROOT orchestrates, Workers execute** — no nested spawns
2. **Skills are shared libraries** — knowledge reuse
3. **Files are state** — everything persists to YAML
4. **Layers have contracts** — L1 constrains L2, L2 feeds L3
5. **Interface ≠ Data** — schema describes presentation, not content
6. **Manager = Skill** — workflow instructions for ROOT, not separate agent

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to contribute to this project.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
