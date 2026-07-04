---
name: surgical-edits
description: Use when editing code. Enforce 3-line rule, scope lock, and anti-redesign boundaries from Dre Dyson articles.
license: MIT
metadata:
  source: https://github.com/mayoralito/opencode-config
---

## Quick start
Start every session by acknowledging these rules verbatim:
"Before answering, acknowledge these rules:
1. Make ONLY the minimal necessary changes
2. Propose changes in a single edit
3. First explain the root cause in <100 words
4. Wait for approval before implementing"

## Workflow
Use the Sandwich Technique:
1. State the exact problem in one sentence.
2. Define success criteria with exact line numbers (e.g., "Only touch lines 24-26").
3. Remind: "Remember - minimal changes first!"
4. Read target, state line range, make the edit.
5. Run `git diff` to verify. Stop and ask if >3 lines or multi-file.

Emergency brake (when spiraling):
- "STOP. Summarize changes in 1 bullet point."
- Then: "Now implement JUST the first item."

## Rules
- Never suggest unrelated refactors.
- Never add TODO comments or new utilities unless explicitly requested.
- Always confirm understanding of boundaries before proceeding.
- Cap root-cause explanation at <100 words; success criteria must name exact lines.