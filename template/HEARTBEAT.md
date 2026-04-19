# Heartbeat Tasks

_Your hourly checklist. Edit it freely — add items when you notice a recurring thing worth checking, remove them when they stop being useful. Keep the list short; it runs every hour._

See `CLAUDE.md` → "Heartbeat and Sleep" for scope, invariants, and how the cron is wired.

## Every heartbeat

- Send a brief online status to Telegram (plain text, no emoji). If the Telegram reply tool is unavailable this session (MCP disconnected), skip the ping and note the skip in the journal — but do NOT trigger the restart marker here. That goes at the end (see last bullet).
- Review the last hour of conversation for notable events.
- Write events to `journal/YYYY-MM-DD.md` using the journal format in `CLAUDE.md`. Tag with `(skills: x, y)` when applicable; tag `(candidate-skill: <slug>)` when the work could become a reusable skill.
- Write to `memory/*.md` immediately if a moment in the last hour produced long-term user-bot relationship content (new preference, feedback pattern, correction).
- If a wiki vault is configured: run the `llm-wiki` Capture → Ingest → Recompile loop for any last-hour world-knowledge triggers (finished `learn` session, multi-turn resolution, user-dropped raws). Silent when nothing qualifies.
- **Last step**: if the Telegram reply tool was unavailable this heartbeat (MCP disconnected), run `mkdir -p .zero-claw && touch .zero-claw/mcp-disconnected`. Supervisor polls for this marker and restarts the bot to reconnect MCP. This MUST be last so journal / memory / wiki writes above finish before the restart — otherwise the restart eats in-flight information.

## Proactive checks (rotate — not every heartbeat, every few)

- (add items here — e.g. "check inbox for urgent unread", "check calendar for events in next 24h", "git status on active projects")

## Do NOT disturb

- Outside waking hours — the cron shouldn't fire then, but if it does, stay silent.
- When the user is clearly busy / mid-conversation with another person.
- Triple-tap: don't send multiple heartbeats in a row for the same state.

## Notes to future-you

(anything you've learned about your own heartbeat behavior — patterns to avoid, things that worked)
