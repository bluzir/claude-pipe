# Contributing to claude-pipe

Thank you for your interest in contributing to claude-pipe! This document provides guidelines and instructions for contributing.

## How to Contribute

### Reporting Issues

If you find a bug or have a feature request:

1. Check if an issue already exists
2. If not, create a new issue with:
   - Clear, descriptive title
   - Steps to reproduce (for bugs)
   - Expected vs actual behavior
   - Your environment (OS, Claude Code version)

### Suggesting Enhancements

We welcome suggestions for:
- New templates (agents, skills)
- Schema improvements
- Documentation clarifications
- New orchestration patterns

Please open an issue to discuss before implementing significant changes.

### Pull Requests

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Make your changes
4. Ensure your changes follow the conventions below
5. Commit with clear messages
6. Push to your fork
7. Open a Pull Request

## Development Setup

### Prerequisites

- [Claude Code CLI](https://claude.ai/code) installed and configured
- Git
- Text editor with Markdown and YAML support

### Getting Started

```bash
# Clone the repository
git clone https://github.com/bluzir/claude-pipe.git
cd claude-pipe

# Explore the structure
ls -la framework/
```

### Testing Changes

To test new templates or agents:

1. Copy template to a test project's `.claude/` directory
2. Run with Claude Code
3. Verify output follows expected patterns
4. Check state files are created correctly

## Code Style

### Markdown Conventions

- Use ATX-style headers (`#`, `##`, etc.)
- One sentence per line in prose (for better diffs)
- Code blocks with language specifier (` ```yaml `, ` ```markdown `)
- Tables for structured comparisons
- ASCII diagrams for architecture (no external images)

### YAML Conventions

```yaml
# Use 2-space indentation
key:
  nested_key: value

# Quote strings with special characters
description: "Contains: colons and other special chars"

# Use explicit types when ambiguous
count: 5          # number
enabled: true     # boolean
version: "1.0"    # string (quoted to avoid float interpretation)
```

### Agent/Skill Frontmatter

```yaml
---
name: agent-name           # lowercase, hyphenated
type: researcher           # from taxonomy
model: sonnet              # sonnet|opus|haiku
tools: [Read, Write, Grep] # required tools
---
```

### File Naming

| Type | Pattern | Example |
|------|---------|---------|
| Agent | `{role}.md` | `aspect-researcher.md` |
| Skill (atomic) | `{action}.md` | `slop-check.md` |
| Skill (manager) | `manager-{workflow}.md` | `manager-research.md` |
| Schema | `{entity}.schema.yaml` | `agent.schema.yaml` |
| Template | `{type}.template.md` | `researcher.template.md` |

## Pull Request Process

### Before Submitting

- [ ] Changes follow the style conventions
- [ ] Templates validate against their schemas
- [ ] Documentation is updated if needed
- [ ] Commit messages are clear and descriptive

### PR Description Template

```markdown
## Summary
Brief description of changes.

## Changes
- List of specific changes made

## Testing
How you tested these changes.

## Related Issues
Fixes #123
```

### Review Process

1. Maintainers will review within 1 week
2. Address feedback in additional commits
3. Once approved, maintainer will merge
4. Your contribution will be in the next release

## Areas for Contribution

### High Priority

- Additional agent templates for common patterns
- Real-world example pipelines
- Schema validation tooling
- Documentation improvements

### Good First Issues

- Typo fixes
- Documentation clarifications
- Adding examples to existing templates
- Translating documentation

## Questions?

If you have questions about contributing:

1. Check existing issues and discussions
2. Open a new issue with the "question" label
3. Be patient - maintainers respond as time permits

## Code of Conduct

- Be respectful and constructive
- Focus on the technical merits
- Help others learn and grow
- Assume good intentions

Thank you for contributing!
