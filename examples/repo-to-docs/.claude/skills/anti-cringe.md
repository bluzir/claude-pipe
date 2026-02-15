---
name: anti-cringe
type: domain
version: v1.0
description: Suppress AI-typical phrases and hedging patterns
---

# Anti-Cringe Filter

Eliminate language patterns that signal AI-generated content.
Apply to any text destined for human readers.

## Banned Phrases

Remove or rewrite any occurrence of:

- "It's important to note..."
- "It's worth noting..."
- "It should be noted..."
- "In conclusion..."
- "In summary..."
- "As we can see..."
- "This is a significant..."
- "It is crucial to understand..."
- "One cannot overstate..."
- "At the end of the day..."
- "Moving forward..."

## Hedging Limits

Limit hedging qualifiers to **one per paragraph maximum**:
- "perhaps", "possibly", "might", "could potentially"
- "it seems", "it appears", "arguably"

If a claim needs hedging, hedge it once with precision.
Do not stack qualifiers ("it might possibly perhaps...").

## Replacement Patterns

| Instead of | Write |
|------------|-------|
| "It's important to note that X" | "X" |
| "In conclusion, we find that Y" | "Y" |
| "There is a significant body of evidence suggesting Z" | "Evidence shows Z" |
| "It could potentially be the case that W" | "W may [reason]" |
| "This represents a paradigm shift in..." | State the specific change |

## Tone Target

- Direct statements over throat-clearing
- Specific claims over vague gestures
- Let evidence speak without editorializing
- Professional, not robotic. Clear, not cold.
