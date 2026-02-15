---
name: code-grounding
type: domain
version: v1.0
description: "No-hallucination rules for source code documentation"
---

# Code Grounding Protocol

## Purpose

Ensure all generated documentation is traceable to actual source code. Prevent hallucinated APIs, fabricated signatures, and invented functionality.

## Core Principles

### Principle 1: Source Traceability

**Statement:** Every documented API must trace to a specific source file and line number.

**Application:**
- Link every function/class/type to `file:line`
- Include actual signatures copied from source
- Reference real import paths

**Violations:**
- Documenting a function without verifying it exists
- Describing parameters not in the actual signature
- Listing exports that don't appear in source

### Principle 2: No Fabrication

**Statement:** Never invent APIs, signatures, parameters, return types, or module descriptions.

**Application:**
- Only document functions/classes that exist in source files
- Only list parameters present in actual signatures
- Only describe behavior observable from code and comments

**Violations:**
- Inventing function signatures ("probably takes a config object")
- Fabricating import paths
- Describing non-existent functionality
- Making up default values

### Principle 3: Code Over Paraphrase

**Statement:** Prefer actual code snippets over paraphrased descriptions.

**Application:**
- Use real signatures from source, not rewritten versions
- Show actual usage patterns found in tests or examples
- Quote comments directly when describing intent

**Violations:**
- Rewriting a signature in a "cleaner" form that doesn't match source
- Inventing usage examples with untested patterns
- Paraphrasing a comment into something it doesn't say

## Constraints

### Hard Constraints (Never Violate)

| ID | Constraint |
|----|------------|
| HC1 | Never invent function or method signatures |
| HC2 | Never fabricate import paths or module names |
| HC3 | Never describe functionality that doesn't exist in source |
| HC4 | Never present inferred behavior as documented behavior |
| HC5 | Never invent parameter types or return types |

### Soft Constraints

| ID | Constraint | Override |
|----|------------|----------|
| SC1 | Prefer exact code snippets over descriptions | When snippet is too long (>20 lines) |
| SC2 | Include file:line for every API item | When line number changes frequently |
| SC3 | Show real usage from tests/examples | When no tests exist — note this gap |

## Verification Checklist

Before outputting documentation:
- [ ] Every documented function exists in source
- [ ] Every signature matches the actual code
- [ ] Every type annotation is real
- [ ] Every import path is valid
- [ ] Every code example is based on actual patterns
- [ ] Gaps in documentation are acknowledged, not filled with guesses

## Integration

### With Module Documenters
- Read all source files before documenting
- Extract signatures directly from code
- Cross-reference imports against actual file structure

### With Docs Writers
- Verify all API references against module YAML data
- Flag any description that can't be traced to source
- Use actual code blocks, not invented pseudocode

### With Assembly
- Architecture diagrams must reflect real module dependencies
- Import relationships must match actual import statements
- No invented modules or connections
