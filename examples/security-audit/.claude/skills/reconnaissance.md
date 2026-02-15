---
name: reconnaissance
type: atomic
version: v1.0
description: "Phase 1 — map target codebase structure, attack surface, and relevant scan categories"
input:
  required:
    - target_path
    - session_id
output:
  type: recon
  schema: recon.yaml
---

# Reconnaissance

## Purpose

Map the target codebase to determine languages, frameworks, entry points, auth boundaries, data flows, and which vulnerability categories are relevant. This shapes the scan plan for Phase 2.

## Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `target_path` | string | Root directory of the target codebase |
| `session_id` | string | Current audit session ID |

## Procedure

### Step 1: Detect Languages and Frameworks

Use Glob to find marker files:

```
# Languages
*.js, *.ts          → JavaScript/TypeScript
*.py                → Python
*.go                → Go
*.rb                → Ruby
*.java              → Java
*.php               → PHP
*.rs                → Rust

# Frameworks (by marker files)
package.json        → Node.js (check for express, fastify, koa, next, etc.)
requirements.txt    → Python (check for flask, django, fastapi, etc.)
go.mod              → Go (check for gin, echo, fiber, etc.)
Gemfile             → Ruby (check for rails, sinatra, etc.)
pom.xml             → Java (check for spring, etc.)
Cargo.toml          → Rust (check for actix, rocket, etc.)

# Package managers
package.json        → npm/yarn
requirements.txt    → pip
Pipfile             → pipenv
go.mod              → go modules
Cargo.toml          → cargo
pom.xml             → maven
Gemfile             → bundler
```

### Step 2: Map Attack Surface

**Entry Points:** Use Grep to find HTTP routes, CLI handlers, event listeners:

```
# Express routes
app\.(get|post|put|delete|patch)\(
router\.(get|post|put|delete|patch)\(

# Fastify routes
fastify\.(get|post|put|delete|patch)\(

# CLI handlers
commander|yargs|meow|process\.argv

# Event listeners
\.on\(|addEventListener|EventEmitter
```

**Auth Boundaries:** Find authentication/authorization middleware:

```
# Middleware patterns
passport|jwt|auth|authenticate|authorize|isLoggedIn|requireAuth|protect

# Decorator patterns (Python/Java)
@login_required|@requires_auth|@RolesAllowed
```

**Data Flows:** Map source → sink paths:

```
# Sources: req.body, req.query, req.params, process.env, database reads
# Sinks: database writes, res.send, file writes, exec, eval
# Through: middleware, validation, sanitization
```

### Step 3: Inventory Files

Use Glob to count file types:

```
source_files: **/*.{js,ts,py,go,rb,java,php,rs}
test_files: **/*.{test,spec}.{js,ts,py} + **/test/** + **/__tests__/**
config_files: **/*.{json,yaml,yml,toml,ini,cfg,env}
vendor_dirs: node_modules/, vendor/, .venv/, target/, dist/
```

### Step 4: Determine Relevant Categories

For each of the 8 vulnerability categories, assess relevance:

| Category | Relevant When | Skip When |
|----------|--------------|-----------|
| `injection` | App has database queries or exec calls | Static site, no DB |
| `auth` | App has authentication/sessions | No auth, no sessions |
| `xss` | App renders HTML templates or returns user content | API-only with JSON |
| `sensitive-data` | App handles credentials, PII, or API keys | No data handling |
| `access-control` | App has routes with different permission levels | Single-user CLI tool |
| `misconfig` | App has deployment configs | Library/SDK |
| `dependencies` | App has dependency files | No dependencies |
| `crypto` | App uses crypto operations | No crypto usage |

Assign priority based on attack surface:
- **high**: Category directly relevant, multiple patterns found
- **medium**: Category relevant but limited exposure
- **low**: Category marginally relevant, few patterns expected

### Step 5: Write Output

Write `recon.yaml` to `artifacts/{session_id}/recon.yaml`.

## Output Schema

```yaml
project:
  name: string
  root: string
  languages: string[]
  frameworks: string[]
  package_managers: string[]

attack_surface:
  entry_points:
    - type: http_route|cli_handler|event_listener|cron
      path: string
      file: string
      method: string|null
  auth_boundaries:
    - type: middleware|decorator|guard
      file: string
      protects: string[]
  data_flows:
    - source: string
      sink: string
      through: string[]

dependency_files:
  - path: string
    type: npm|pip|go|cargo|maven

scan_categories:
  - id: string
    relevant: boolean
    reason: string
    priority: high|medium|low

file_inventory:
  total_files: number
  source_files: number
  test_files: number
  config_files: number
  vendor_dirs: string[]
```

## Gate

Phase 1 passes when:
- `scan_categories` has at least 3 entries with `relevant: true`
- `file_inventory.source_files` > 0
- `attack_surface.entry_points` has at least 1 entry

If fewer than 3 categories are relevant, the target may be too simple for a full audit. Report to user with recommendation.

## Quality Criteria

- [ ] All detected languages have framework check
- [ ] Entry points mapped with file:line
- [ ] Auth boundaries identified (or confirmed absent)
- [ ] Each category has a relevance reason
- [ ] Vendor dirs identified for exclusion
