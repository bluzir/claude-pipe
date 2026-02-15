---
name: codebase-scanner
type: atomic
version: v1.0
description: "Scan repository structure, detect language/framework, enumerate modules"
input:
  required:
    - target_path
output:
  type: plan
  schema: plan.yaml
---

# Codebase Scanner

## Purpose

Analyze a repository to detect its language, framework, and module structure. Produce a plan for documentation generation.

## Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `target_path` | string | Absolute path to the repository root |

## Procedure

### Step 1: Detect Language and Framework

Check for marker files in the repository root:

| Marker File | Language | Framework Detection |
|-------------|----------|---------------------|
| `package.json` | JavaScript/TypeScript | Check dependencies for express, react, next, etc. |
| `tsconfig.json` | TypeScript | (supplements package.json) |
| `pyproject.toml` | Python | Check for django, flask, fastapi, etc. |
| `requirements.txt` | Python | Scan for framework packages |
| `setup.py` | Python | Legacy Python project |
| `go.mod` | Go | Check module path |
| `Cargo.toml` | Rust | Check dependencies |
| `pom.xml` | Java | Maven project |
| `build.gradle` | Java/Kotlin | Gradle project |
| `Gemfile` | Ruby | Check for rails, sinatra, etc. |
| `composer.json` | PHP | Check for laravel, symfony, etc. |

If multiple markers exist, use the primary one (e.g., `package.json` over `tsconfig.json`).

### Step 2: Enumerate Modules

Apply language-specific module detection:

**JavaScript/TypeScript:**
- Each directory with `index.ts`, `index.js`, or its own `package.json`
- Top-level `src/` subdirectories
- Named exports from barrel files

**Python:**
- Each directory with `__init__.py`
- Standalone `.py` files in `src/` or project root
- Packages listed in pyproject.toml

**Go:**
- Each directory containing `.go` files
- Package declarations in source files

**Rust:**
- `src/lib.rs` and `src/main.rs` as entry points
- Each directory in `src/`
- Each crate in a workspace

**General fallback:**
- Top-level directories containing source files
- Group by directory structure

### Step 3: Analyze Each Module

For each detected module:

1. **List files** — all source files in the module directory
2. **Estimate LOC** — count non-empty, non-comment lines
3. **Identify entry point** — main file (index, mod, __init__, lib)
4. **Note imports/exports** — scan for cross-module dependencies
5. **Brief description** — infer from directory name, README, or top-level comments

### Step 4: Plan Documentation Sections

Generate doc sections based on modules and project structure:

| Section Type | Condition | Source |
|-------------|-----------|--------|
| `architecture` | Always | Cross-module analysis |
| `getting-started` | Always | Package manager + entry point |
| `api-reference` | Always | All public APIs aggregated |
| `module` | Per module | One section per module |

### Step 5: Write Plan

Write to `artifacts/{session}/plan.yaml`.

## Output Schema

```yaml
project:
  name: string            # From package.json name, directory name, etc.
  language: string         # javascript, typescript, python, go, rust, java
  framework: string|null   # express, react, django, etc. or null
  root: string            # Absolute path to repo root

modules:
  - id: string            # kebab-case identifier
    name: string           # Human-readable name
    path: string           # Relative path from root
    files: string[]        # File list (relative to module path)
    entry_point: string    # Main file
    loc_estimate: number   # Approximate lines of code
    description: string    # Brief description from analysis

doc_sections:
  - id: string
    type: architecture|getting-started|api-reference|module
    source_module: string|null  # null for cross-cutting sections
    title: string
```

## Quality Check

- modules.length >= 1
- Each module has files.length >= 1
- doc_sections includes architecture, getting-started, api-reference
- Each module has a corresponding doc_section of type `module`

## Constraints

- Do not read file contents deeply in this phase — structure only
- LOC estimates can be approximate (count lines, not parse AST)
- Module descriptions should be brief (one sentence)
- If a directory has no source files, skip it
