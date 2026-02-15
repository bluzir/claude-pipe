---
name: silence-protocol
type: atomic
version: v1.0
description: "Suppress chat output - agents write files only"
---

# Silence Protocol

## Purpose

When active, agent produces no conversational output. All results are written to files only. This enables clean parallel execution without interleaved chat messages.

## When to Use

- Worker agents running in background
- Parallel execution where chat would be confusing
- Pipeline phases where output is file-based

## Instructions

When this skill is loaded:

1. **No conversational output**
   - Do not explain what you're doing
   - Do not provide status updates in chat
   - Do not summarize results in chat

2. **File-only output**
   - Write all results to specified output file
   - Include status in file metadata
   - Include errors in file if they occur

3. **Completion signal**
   - Task completion is signaled by file existence
   - Success/failure determined by file content

## Metadata Pattern

Include execution metadata in output files:

```yaml
_execution:
  agent: "{agent_name}"
  started_at: "{timestamp}"
  completed_at: "{timestamp}"
  status: success|partial|failed
  error: null|"{error_message}"
```

## Exceptions

Silence protocol does NOT apply to:
- Fatal errors that prevent file writing
- Permission requests
- Clarification questions (which should be avoided anyway)

## Integration

Workers with `silence-protocol` in required skills:
- Must have `output.path` defined
- Must write structured output
- Must include execution metadata
