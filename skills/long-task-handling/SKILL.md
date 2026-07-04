---
name: long-task-handling
description: Use when a task may exceed 2 minutes or risks hanging. Set 3-minute provider timeouts, require progress checkpoints every 60s, and enforce mandatory user response after every tool call.
license: MIT
metadata:
  source: https://github.com/mayoralito/opencode-config
---

## Quick start
Before any potentially long task:
1. Set provider `timeout: 180000` (3 min) — override the default 10 min.
2. Break work into <60s chunks with explicit checkpoints.
3. After every tool call, immediately respond to the user with status + next step.

## Workflow
1. Detect long-running risk (builds, tests, migrations, large refactors, subagents on big codebases).
2. State expected duration and set 3-min timeout.
3. Run one bounded step, then report.
4. Ask user to continue or adjust before next step.
5. Never chain multiple long operations without explicit approval.

## Rules
- Never leave the conversation on a tool call.
- Never rely on 10-minute timeouts for interactive work.
- Use `task` tool `timeout` parameter for subagents on large codebases.
- Default to Claude for complex/long work; route boilerplate to faster models via task-routing skill.