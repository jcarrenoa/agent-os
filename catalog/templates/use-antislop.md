---
name: use-antislop
description: agent-os pipeline wrapper for The AntiSlop, written by /agent-os:extensions. Only applies when it is listed in an agent-os agent's Extensions block; it restricts The AntiSlop to auditing the prose that agent writes.
version: 1.0.0
---

# Use The AntiSlop (audit only)

Installed by agent-os from its recommended catalog (The AntiSlop, pinned commit). The AntiSlop skill itself is listed just before this one in your Extensions block. This wrapper comes later, so it overrides it where they conflict.

## How to apply it

1. Write your output as usual: requirements, ADR text, summaries.
2. Before returning it, review the prose you wrote in this task against The AntiSlop's patterns, and fix it in your own output (or in the files you created in this task).
3. Mention the review in your `EXTENSIONS APPLIED` line, e.g. "reviewed requirements text, removed 3 structural patterns".

## Limits

- **Audit only.** Do not use its Editor Mode on any file you did not write in this task, and do not run it over the codebase.
- **Never run its pattern-refresh section** (the `curl … | python3` commands that download patterns from Wikipedia).
- **Language.** Apply its structural patterns in any language. Apply its lexical patterns (specific English words and phrases, curly quotes and similar typography) only when you are writing in English.
- **Missing skill.** If The AntiSlop skill was not in your Extensions block or could not be read, report this wrapper as `not applicable: antislop not available`.
