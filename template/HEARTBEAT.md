# Heartbeat Checklist

_Read this at every heartbeat. Edit it freely — it's yours._

Each heartbeat, go through the items below. Skip anything that doesn't apply right now. Add new items when you notice a recurring thing worth checking; remove them when they stop being useful. Keep the list short — it runs every hour.

## Every heartbeat

- Send a brief online status to Telegram (plain text, no emoji)
- Review recent conversation for notable events
- Write events to `journal/YYYY-MM-DD.md` with `(skills: x, y)` tags. Also tag `(candidate-skill: <slug>)` when the work could become a reusable skill — feeds evolve's upgrade signal.
- Write to `memory/*.md` immediately if a moment in the last hour produced long-term user-bot relationship content (new preference, feedback pattern, correction). Don't wait for EOD.
- If a wiki vault is configured: run the `llm-wiki` Capture → Ingest → Recompile loop for any last-hour world-knowledge triggers (finished `learn` session, multi-turn resolution, user-dropped raws). Silent when nothing qualifies.

## Proactive checks (rotate — not every heartbeat, every few)

- (add items here — e.g. "check inbox for urgent unread", "check calendar for events in next 24h", "git status on active projects")

## Last heartbeat of the day

- Memory maintenance: distill today's journal into `memory/*.md`, prune superseded entries (budget min(2 files, 5%)), keep `memory/MEMORY.md` under 200 lines.
- Run the `evolve` skill — it maintains the skill library only (upgrade + retire at 90-day unused threshold).
- If a wiki vault is configured: promote any world-knowledge that accidentally landed in `memory/` to the vault (Capture → Ingest → Recompile), then run `llm-wiki` Lint and surface findings.

## Do NOT disturb

- Between the user's sleep hours — the cron shouldn't fire then, but if it does, stay silent
- When the user is clearly busy / mid-conversation with another person
- Triple-tap: don't send multiple heartbeats in a row for the same state

## Notes to future-you

(anything you've learned about your own heartbeat behavior — patterns to avoid, things that worked)
