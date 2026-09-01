# CLAUDE.md

Project-wide instructions for Claude Code. These are permanent and apply to every session in this repository.

## Chat history logging (mandatory)

This repository keeps a persistent, append-only transcript of every prompt/response exchange at `.chat-history/log.md`. It is graded evidence — treat it as required output, not optional bookkeeping.

### 1. At session start

Read `.chat-history/log.md` before doing anything else, to recover context from previous sessions. If the file or the `.chat-history/` directory does not exist, create them (the file starts empty apart from entries you append).

### 2. After every response

Append one entry to `.chat-history/log.md` using this exact format:

```
---
- timestamp: "<ISO 8601 timestamp if available, otherwise estimate based on conversation order>"
- user_prompt: "<the user's original prompt>"
- assistant_response_summary: "<summary of what you generated or answered for this prompt>"
- files_affected: "<comma-separated list of files created or modified, or none>"
```

### 3. Rules

- **Never delete or rewrite previous entries.** The file is append-only. New entries go at the end.
- **Never skip an exchange.** Every prompt/response pair gets logged, including short answers, clarifying questions, refusals, and responses where no files changed.
- **Be precise about `files_affected`.** List only files explicitly created or modified during that response. Use repo-relative paths. If nothing was written, the value is `none`. Do not list files you merely read, searched, or considered. Do not list `.chat-history/log.md` itself.
- **Keep `assistant_response_summary` concise but specific.** Name the actual functions, endpoints, manifests, workflow jobs, or key decisions involved — not "helped with the code".
- **Quote the user's prompt verbatim** in `user_prompt`. If it is long, keep it intact rather than paraphrasing; escape or collapse embedded newlines so the entry stays on one line.
- **Do all of this silently.** Never ask for confirmation before reading or appending to the log, and do not narrate the logging step in your response to the user.
- Use a real ISO 8601 timestamp (e.g. `2026-09-01T16:48:00Z`) when the current time is known; otherwise estimate one that preserves conversation order relative to the previous entry.

### 4. Timing

Write the log entry as the final action of the turn, after all other tool calls, so `files_affected` reflects everything that actually changed.
