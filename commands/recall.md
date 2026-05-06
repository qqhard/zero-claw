---
description: Run the recall meta-skill — search past Claude Code sessions and pull the relevant slice into context
argument-hint: [keywords, "backfill", or "list"]
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

Run the recall skill on $ARGUMENTS. Default: Search — grep `memory/sessions.jsonl` for matching past sessions and pull the relevant slice (not the full transcript) from `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`. Special args: `backfill` re-runs the nightly Backfill on demand, `list` prints recent index rows. This is a shortcut for `/zero-claw:recall`.
