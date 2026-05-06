---
name: recall
description: "Cross-session memory — index every Claude Code session's session_id + topic + summary in `memory/sessions.jsonl`, so a fresh session can find and pull the relevant slice of past conversations after a quota cut, restart, MCP disconnect, or manual stop. Triggered by Session Start (Begin), heartbeat (Update), sleep (Backfill), and on user reference to past chats (Search). Triggers: 'recall', 'remember when we talked about', 'we discussed before', '我们之前聊过', '上次说的', '/zero-claw:recall'."
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
---

# Recall (meta-skill)

Cross-session memory for the bot. Claude Code already stores every session's full transcript at `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl` — `recall` adds the index and the retrieval protocol that turns those files into recoverable memory after a session ends.

## Why this exists

A Telegram session can die mid-conversation (quota, restart, MCP disconnect, daily reboot). The next session has zero context. Without `recall`, the bot apologizes ("I don't remember"); with `recall`, the bot finds the right past session, pulls the relevant slice, and continues.

## Files it owns

- **`memory/sessions.jsonl`** — append-mostly index, one JSON object per session. Owned exclusively by this skill.
- **`memory/sessions.archive.jsonl`** — aged-out rows moved here by Backfill. Anti-bloat without data loss.

It does NOT touch:

- `~/.claude/projects/.../*.jsonl` — read-only, Claude Code owns these.
- `memory/*.md` — owned by heartbeat / sleep.
- `journal/*` — raw events, never rewritten.
- `SOUL.md`, `USER.md`, `CLAUDE.md`.

## Index row schema

```jsonl
{"session_id":"abc-123","started":"2026-05-06T10:30:00Z","ended":"2026-05-06T11:45:00Z","topic":"session 恢复设计","summary":"用户提出 session 中断后失忆问题；设计了 recall meta-skill，索引存 sessions.jsonl，search 默认拉相关切片不拉全文。"}
```

Fields:

- `session_id` — UUID matching the JSONL filename in `~/.claude/projects/<encoded-cwd>/`.
- `started` — ISO-8601 UTC.
- `ended` — ISO-8601 UTC, or `null` if the session is still live.
- `topic` — short free-form phrase, user's language. No length / format rules.
- `summary` — short — a sentence or two is plenty. User's language. May be `null` for live sessions before the first heartbeat fires.

## Locating the current session's transcript

```bash
# Encode the cwd: replace / and . with -, mirroring Claude Code's projects dir naming.
ENC=$(pwd | tr '/.' '-')
PROJDIR="${HOME}/.claude/projects/${ENC}"
# Fallback: if encoding doesn't match (rare — dotted/odd paths), pick the projects dir
# whose most-recent JSONL was modified in the last 5 minutes (it's almost certainly us).
[ -d "$PROJDIR" ] || PROJDIR=$(ls -td "$HOME"/.claude/projects/*/ 2>/dev/null | head -1)
SID=$(ls -t "$PROJDIR"/*.jsonl 2>/dev/null | head -1 | xargs -n1 basename | sed 's/\.jsonl$//')
echo "$SID"
```

## Operations

### Begin — called from `CLAUDE.md` Session Start

Fast and silent. Just registers the session.

1. Determine `session_id` (script above).
2. If `memory/sessions.jsonl` already has a row for this `session_id`, skip (idempotent — Session Start may run twice in some flows).
3. Else append `{session_id, started: <now-iso>, ended: null, topic: null, summary: null}` to `memory/sessions.jsonl`.
4. If `memory/` doesn't exist yet, `mkdir -p memory && touch memory/sessions.jsonl`.

Do NOT read or summarize anything in Begin. Keep it under 1 second.

### Update — called from `HEARTBEAT.md`

Once per heartbeat, refresh `topic` + `summary` for the current session.

1. Determine current `session_id`.
2. Skim what you've discussed with the user since session start (or since the last Update). You're *in* the session — your conversation context already has it. Don't re-read the JSONL transcript.
3. Write a short `topic` (a phrase) and `summary` (a sentence or two). User's language. Free-form.
4. Atomic rewrite of `sessions.jsonl`:
   - Read the file into memory.
   - Replace the row matching `session_id` (preserve `started`; leave `ended` as null).
   - If no matching row exists, call Begin first then proceed.
   - Write to `memory/sessions.jsonl.tmp`, then `mv` over the real file.

Refusing to police summary length is intentional — the agent picks what's worth remembering.

### Backfill — called from `SLEEP.md`

The safety net for sessions that died before any Update fired. Runs once per night.

Two passes plus an aging step:

**Pass 1 — Catch-up (any age)**: for every JSONL file in `~/.claude/projects/<encoded-cwd>/` that has NO row in `sessions.jsonl`, generate one. This catches first-run (where the index is empty but past sessions exist) and any sessions missed by Begin.

**Pass 2 — Refresh (last 48h)**: for every JSONL modified in the last 48h whose row exists but has `summary: null` or `ended: null` → re-summarize. This catches sessions that crashed before the first heartbeat could write a summary.

For each session being summarized:

```bash
JSONL="$PROJDIR/${SID}.jsonl"
LINES=$(wc -l < "$JSONL")
# For huge transcripts, head + tail is plenty for a 1-2 sentence summary.
if [ "$LINES" -gt 400 ]; then
  head -200 "$JSONL"
  echo "..."
  tail -200 "$JSONL"
else
  cat "$JSONL"
fi
```

Then synthesize `topic` + `summary` from the content. Set `ended` to the file's mtime in ISO-8601 UTC if the file hasn't been written to in the last hour (heuristic: session is over). Otherwise leave `ended: null`.

**Aging step**: after both passes, scan `sessions.jsonl`:

- Any row with `ended` older than 90 days → append to `memory/sessions.archive.jsonl`, remove from `sessions.jsonl`.
- If `sessions.archive.jsonl` doesn't exist, create it (empty).

Aged rows stay grep-able if the user asks for ancient context — Search falls through to the archive on no-hit. Just not loaded into the active index.

Always atomic-rewrite (`tmp` + `mv`). Never partial writes.

### Search — called when the user references past conversation

Triggers (any of):

- "我们之前聊过 X" / "上次说的那个" / "remember when we discussed Z".
- User asks about something not in current context that might be in a past one.
- Direct: `/zero-claw:recall <query>`.

Steps:

1. **Extract keywords** from the user's reference. Be liberal — better to over-match and filter than miss.
2. **Index search** — grep `memory/sessions.jsonl` for keyword matches in `topic` and `summary` (also match by date if the user said "上周二" / "yesterday"):
   ```bash
   grep -i 'KEYWORD' memory/sessions.jsonl
   ```
   If zero hits, fall through to `memory/sessions.archive.jsonl`.
3. **Confirm if ambiguous**. Multiple candidates → briefly list them ("找到 3 条候选：A (5/4 聊 X), B (5/2 聊 Y), C…要拉哪个？") and wait for the user to pick. Single clear hit → proceed without asking.
4. **Pull the RELEVANT SLICE — not the full transcript**. Once a session is selected:
   - Open `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`.
   - `grep -n 'KEYWORD'` to find anchor line numbers inside the JSONL.
   - For each anchor, read a ±20-line window (`Read` with `offset` and `limit`).
   - Dedupe overlapping windows, concatenate.
   - If the user asked open-endedly ("remind me what we said") with no specific keyword → take the **last 100 lines** (recency) plus the first 30 lines (the topic statement is usually near the start).
5. **Surface to user as prose, not raw JSONL**. Say what you found in plain language ("在 5/4 那次 session 你说过 X，我们当时定了 Y"), then continue. Don't dump JSON lines.
6. **Default is sliced, not full**. Only pull the entire transcript if the user explicitly asks ("把整段拉出来", "give me the full thing").

## JSONL editing notes

JSONL doesn't support in-place row updates. The pattern for Update / Backfill / Aging:

1. Read the entire file.
2. Build a `dict[session_id] → row`, last-write-wins.
3. Apply your modifications (replace / insert / drop).
4. Write to a temp file (`memory/sessions.jsonl.tmp`).
5. `mv` over the real file (atomic).

For thousands of rows this is sub-second. Aging keeps the active file well under 10k rows. If you ever see it climb past that, drop the threshold from 90 days.

## Safety

- Never `git push`.
- Never write to `~/.claude/projects/.../*.jsonl` — Claude Code owns it.
- Never block on transcript files. Read while the session is live is fine; do NOT write.
- Search is read-only.
- All writes go through `tmp` + `mv` — partial writes if the bot is killed mid-Update would corrupt the index.

## Failure modes

- **`~/.claude/projects/<encoded-cwd>/` doesn't exist**: cwd is wrong, or fresh install. Begin / Backfill skip gracefully and log a 1-line note in today's journal.
- **No JSONL files at all**: same.
- **`sessions.jsonl` is corrupted JSON**: rename to `sessions.jsonl.broken-YYYY-MM-DD`, start fresh. The next Backfill re-derives every row from JSONL transcripts.
- **`session_id` collision** (shouldn't happen — UUIDs): keep the row with the most recent `ended`, drop the other.

## Manual invocation

User-invocable for explicit requests:

- `/zero-claw:recall <query>` — runs Search with `<query>` as keyword.
- `/zero-claw:recall backfill` — re-runs Backfill on demand (useful right after first install).
- `/zero-claw:recall list` — print the last 20 rows of `sessions.jsonl` so the user can browse.
