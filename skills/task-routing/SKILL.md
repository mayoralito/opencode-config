---
name: task-routing
description: Use when deciding which model to invoke. Route fast models to formatting, tests, boilerplate, dependency updates, and CI tweaks. Route precise models to bug hunting and core algorithm work.
license: MIT
metadata:
  source: https://github.com/mayoralito/opencode-config
---

## Quick start
Before starting work, classify the task:
- Grok: mass formatting, test generation, docstrings, boilerplate, dependency bumps, CI/CD tweaks.
- Claude: bug hunting, core algorithm design, complex refactors, cryptographic code.

## Workflow
1. Ask or determine task type.
2. Select model per the chart above.
3. If the task is ambiguous, default to Claude.
4. Confirm the chosen model with the user when switching.

## 80/20 Rule
Use Grok where speed beats perfection. Never use Grok for security-sensitive or correctness-critical code.