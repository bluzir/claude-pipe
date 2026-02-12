# Quickstart Guide

Get up and running with the Agent Orchestration Framework in 5 minutes.

## Prerequisites

- [Claude Code CLI](https://claude.ai/code) installed and authenticated
- Basic familiarity with markdown and YAML
- A project directory to work in

## Step 1: Set Up Project Structure

Create the standard directory structure for Claude Code:

```bash
mkdir -p .claude/agents
mkdir -p .claude/skills
mkdir -p artifacts
```

Your project should look like:
```
your-project/
├── .claude/
│   ├── agents/         # Agent definitions
│   └── skills/         # Skill definitions
└── artifacts/          # Runtime outputs (gitignored)
```

## Step 2: Copy Framework (Optional)

You can either reference the framework documentation or copy it locally:

```bash
# Option A: Copy framework to your project
cp -r path/to/framework ./framework

# Option B: Just reference README.md when prompting Claude
# "Use conventions from framework/README.md"
```

## Step 3: Create Your First Agent

Copy the researcher template:

```bash
cp framework/templates/agents/researcher.template.md \
   .claude/agents/my-researcher.md
```

Edit `.claude/agents/my-researcher.md`:

```markdown
---
name: my-researcher
type: researcher
model: sonnet
tools: [Read, Write, Grep, WebFetch, WebSearch]
---

# My Researcher

You are a research agent focused on [YOUR DOMAIN].

## Input
- `query`: Topic to research
- `output_path`: Where to write results

## Procedure
1. Search for relevant sources
2. Evaluate source quality
3. Extract key findings
4. Write structured output

## Output Format
Write YAML to `{output_path}`:
```yaml
query: "{query}"
sources:
  - url: "..."
    title: "..."
    tier: A|B|C
findings:
  - claim: "..."
    source: "..."
    confidence: high|medium|low
```
```

## Step 4: Create Your First Skill

Create an atomic skill for a common operation:

```bash
cp framework/templates/skills/atomic.template.md \
   .claude/skills/source-check.md
```

Edit `.claude/skills/source-check.md`:

```markdown
---
name: source-check
type: atomic
scope: "Evaluate source credibility"
---

# Source Check

## Input
- URL to evaluate

## Procedure
1. Check domain reputation
2. Look for author credentials
3. Check publication date
4. Identify potential bias

## Output
Return:
- `tier`: S|A|B|C|D|X
- `reason`: Brief explanation
```

## Step 5: Run Your Agent

In Claude Code, invoke your agent:

```
/my-researcher

Research the topic "sustainable agriculture practices"
and write output to artifacts/research-001.yaml
```

Claude will load your agent definition and execute the research task.

## Step 6: Create a Simple Pipeline

For multi-step workflows, create a manager skill:

```bash
cp framework/templates/skills/manager.template.md \
   .claude/skills/manager-basic-research.md
```

Edit `.claude/skills/manager-basic-research.md`:

```markdown
---
name: manager-basic-research
type: manager
scope: "Coordinate basic research pipeline"
---

# Basic Research Pipeline

## Phases

### Phase 1: Planning
1. Read user query
2. Generate 3-5 research aspects
3. Write plan to `artifacts/{session}/plan.yaml`

### Phase 2: Research
For each aspect in plan:
1. Spawn `my-researcher` agent with aspect
2. Write to `artifacts/{session}/aspects/{aspect_id}.yaml`

**Orchestration:** Run all researchers in parallel.

### Phase 3: Synthesis
1. Read all aspect files
2. Combine findings
3. Write `artifacts/{session}/synthesis.yaml`

### Phase 4: Output
1. Generate final report
2. Write `artifacts/{session}/REPORT.md`
```

## Step 7: Run the Pipeline

```
/manager-basic-research

Research topic: "Impact of remote work on productivity"
Session: research-001
```

## What's Next?

### Learn More
- Read the full [README.md](README.md) for detailed concepts
- Explore [examples/research-pipeline](examples/research-pipeline/) for a complete implementation

### Customize
- Add domain-specific skills for your use case
- Create agents for different functional roles
- Build quality gates for your pipelines

### Adopt Gradually
1. **Level 1**: Use conventions (file structure, naming)
2. **Level 2**: Use templates (schemas, validation)
3. **Level 3**: Full pipelines (managers, state, quality gates)

## Troubleshooting

### Agent not found
- Ensure file is in `.claude/agents/`
- Check filename matches agent name in frontmatter
- Verify markdown frontmatter is valid YAML

### Output not written
- Check `artifacts/` directory exists
- Verify agent has Write tool in frontmatter
- Look for error messages in Claude Code output

### Pipeline stuck
- Check `artifacts/{session}/state.yaml` for phase status
- Look for failed workers
- Verify gate conditions are met

## Quick Reference

| Task | Command |
|------|---------|
| Create agent | `cp templates/agents/*.template.md .claude/agents/` |
| Create skill | `cp templates/skills/*.template.md .claude/skills/` |
| Run agent | `/agent-name <args>` |
| Run pipeline | `/manager-name <args>` |
| Check state | Read `artifacts/{session}/state.yaml` |

---

Ready to dive deeper? Check out the [full documentation](README.md).
