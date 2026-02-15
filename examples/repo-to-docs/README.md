# Repo-to-Docs Pipeline

Run `/docs` on any repo. Get a docs site in 3 minutes.

> **Framework Documentation:** See the main [README.md](../../README.md) for framework concepts and conventions.

## Pattern: Subagent = Module

Structural partitioning of a codebase into modules, each documented in parallel by a dedicated worker agent. Results are assembled into cross-cutting pages, quality-gated, and emitted as a complete documentation set.

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
    │  Documenter     │    │  Documenter     │    │  Documenter     │
    │ (module: routes)│    │ (module: models)│    │ (module: auth)  │
    └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
             │                      │                      │
             ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  API signatures │    │  API signatures │    │  API signatures │
    │  usage examples │    │  usage examples │    │  usage examples │
    │  diagrams       │    │  diagrams       │    │  diagrams       │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Why Pipeline > Single Prompt

- **Context limits**: Large codebases exceed a single context window. Splitting by module keeps each worker focused.
- **Hallucination isolation**: A single-prompt approach halluccinates APIs when context is full. Per-module workers read actual source files and extract real signatures.
- **Parallel speed**: Modules are documented simultaneously. A 10-module repo finishes as fast as a 1-module repo.
- **Quality gating**: Automated checks catch missing modules and undocumented exports before final output.

## Quick Start

```bash
cd examples/repo-to-docs
/manager-docs /path/to/your/repo
```

The pipeline scans the repo, documents each module, assembles cross-cutting pages, runs quality checks, and emits final markdown.

## Pipeline

```
Phase 1        Phase 2              Phase 3        Phase 4        Phase 5
Scan     ───▶  Document ×N   ───▶  Assemble ───▶  Quality  ───▶  Emit ×N
  │               │                    │              │              │
  ▼               ▼                    ▼              ▼              ▼
plan.yaml    modules/*.yaml      assembly.yaml   quality.yaml   docs/*.md
```

```
┌─────────────┐     ┌─────────────────┐     ┌───────────┐
│    Scan     │────▶│ Document ×N     │────▶│  Assemble │
│  (scanner)  │     │ (N documenters) │     │           │
└─────────────┘     └─────────────────┘     └─────┬─────┘
                                                  │
                    ┌─────────────────┐     ┌─────▼─────┐
                    │   Emit ×N      │◀────│  Quality  │
                    │  (N writers)   │     │   Gate    │
                    └─────────────────┘     └───────────┘
```

## Phases

| Phase | Agent/Skill | Input | Output |
|-------|-------------|-------|--------|
| 1. Scan | codebase-scanner | target repo path | plan.yaml |
| 2. Document | module-documenter ×N | plan.modules | modules/*.yaml |
| 3. Assemble | docs-assembler | modules/*.yaml + plan.yaml | assembly.yaml |
| 4. Quality | docs-quality-gate | assembly + modules + plan | quality.yaml |
| 5. Emit | docs-writer ×N | assembly.yaml + modules/*.yaml | docs/*.md |

## Structure

```
repo-to-docs/
├── .claude/
│   ├── agents/
│   │   ├── module-documenter.md     # Worker: documents one module
│   │   └── docs-writer.md          # Worker: writes one markdown file
│   ├── skills/
│   │   ├── manager-docs.md         # Manager: 5-phase pipeline orchestration
│   │   ├── codebase-scanner.md     # Atomic: repo → module inventory
│   │   ├── docs-assembler.md       # Composite: cross-cutting doc assembly
│   │   ├── docs-quality-gate.md    # Atomic: PASS/WARN/FAIL verdict
│   │   ├── code-grounding.md       # Domain: no-hallucination for code docs
│   │   ├── silence-protocol.md     # Atomic: suppress chat, file-only output
│   │   ├── io-yaml-safe.md         # Atomic: safe YAML writing
│   │   └── anti-cringe.md          # Domain: suppress AI-typical phrasing
├── artifacts/                       # Runtime outputs (gitignored)
│   └── {session_id}/
│       ├── state.yaml              # Pipeline state for resume
│       ├── plan.yaml               # Module inventory + doc plan
│       ├── modules/*.yaml          # Per-module documentation data
│       ├── assembly.yaml           # Cross-cutting assembly
│       ├── quality.yaml            # Quality verdict
│       └── docs/                   # Final markdown output
│           ├── README.md           # Architecture overview
│           ├── getting-started.md  # Installation + quick start
│           ├── api-reference.md    # Full API index
│           └── modules/            # Per-module documentation
│               ├── {module}.md
│               └── ...
└── README.md
```

## Output Format

The pipeline produces a `docs/` directory with:

| File | Content |
|------|---------|
| `docs/README.md` | Architecture overview with Mermaid dependency graph |
| `docs/getting-started.md` | Installation, quick example, key concepts |
| `docs/api-reference.md` | Flat API index across all modules |
| `docs/modules/{id}.md` | Per-module docs: API reference, examples, internal diagrams |

Every API signature traces to a `file:line` in the source. No invented APIs, no hallucinated parameters.

## Supported Languages

| Language | Module Detection | Framework Detection |
|----------|-----------------|---------------------|
| TypeScript/JavaScript | `index.ts` / `package.json` per directory | Express, React, Next.js |
| Python | `__init__.py` per directory | Django, Flask, FastAPI |
| Go | Directories with `.go` files | Standard library patterns |
| Rust | `src/` directories, workspace crates | Actix, Axum, Rocket |
| Java | Maven/Gradle modules | Spring Boot |

## Demo

See `artifacts/demo_session/` for a complete run documenting a fictional Express.js API (`user-service`) with three modules: Routes, Middleware, and Models.

Key demo files:
- `artifacts/demo_session/plan.yaml` -- scanned module inventory
- `artifacts/demo_session/modules/routes.yaml` -- structured API data for routes
- `artifacts/demo_session/assembly.yaml` -- cross-cutting assembly
- `artifacts/demo_session/quality.yaml` -- PASS verdict
- `artifacts/demo_session/docs/` -- final markdown output

## Data Flow

```yaml
L1 (Directives):
  - plan.yaml            # Module inventory, doc sections

L2 (Operational):
  - modules/routes.yaml   # Per-module API documentation
  - modules/models.yaml
  - modules/middleware.yaml
  - assembly.yaml         # Cross-cutting sections
  - quality.yaml          # Quality verdict

L3 (Artifacts):
  - docs/README.md        # Architecture overview
  - docs/getting-started.md
  - docs/api-reference.md
  - docs/modules/*.md     # Per-module pages
```

## Customization

To adapt this pipeline:

1. **Module detection** -- Edit `codebase-scanner.md` to add language-specific module detection rules
2. **Documentation depth** -- Modify `module-documenter.md` to include/exclude private APIs, add more examples, or change output format
3. **Quality thresholds** -- Adjust pass/warn/fail boundaries in `docs-quality-gate.md`
4. **Output format** -- Customize `docs-writer.md` for your doc site (Docusaurus, MkDocs, VitePress)
5. **Domain skills** -- Add specialized skills (e.g., `openapi-extractor.md` for REST APIs, `graphql-schema.md` for GraphQL)
6. **Cross-cutting sections** -- Modify `docs-assembler.md` to generate different overview pages (changelog, migration guide, FAQ)

## Comparison: Repo-to-Docs vs Research Pipeline

| Dimension | Repo-to-Docs | Research Pipeline |
|-----------|-------------|-------------------|
| **Partitioning** | Structural (by module) | Semantic (by meaning) |
| **Worker input** | Source files in a directory | Search queries for an aspect |
| **Parallelism** | Document N modules + Emit N files | Research N aspects |
| **Aggregation** | Assembly of cross-cutting pages | Synthesis of cross-aspect patterns |
| **Quality gate** | Coverage, API completeness, diagrams | Saturation, diversity, evidence depth |
| **Use case** | Code documentation | Research reports |
