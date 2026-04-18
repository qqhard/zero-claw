# Zero-Claw

A zero-infrastructure recipe for turning Claude Code into a personal AI assistant via Telegram.

## Discussion context (read this first)

When the user asks about bot behavior, skill triggering, heartbeat flow, wiki/memory ops, or anything about how this thing *runs* — they mean the **installed-bot runtime** (zero-claw plugin loaded inside the bot's tmux session, Telegram plugin active), **not** the source-repo Claude session you're probably in right now.

In the source repo, skills under `skills/` are **not** auto-loaded by your current session — they become user-invocable only after `claude plugins install` and only inside the bot. So if the user says "why didn't `/llm-wiki` trigger" or "搜 wiki 应该走什么 skill," don't answer from your current session's skill list — reason about the installed-bot side.

Default assumption: **installed-bot runtime**. Only switch to "source repo dev env" framing if the user explicitly says so ("in this repo", "in the dev setup", etc.).

## Philosophy

Don't build what already exists. Compose.

| Component | Role | Source |
|-----------|------|--------|
| Claude Code | Brain | Anthropic subscription |
| Telegram plugin | Mouth | `claude plugins install telegram` |
| tmux | Body | System tool |
| Supervisor | Heart monitor | This project |
| CLAUDE.md | System mechanism | Plugin template (identical per bot) |
| SOUL.md / USER.md / CRONTAB.md | Personalization | User-defined (per bot) |
| memory/ | Memory | Git-tracked files |

## Design Principles

1. **No wheels** — Don't reimplement what Claude Code, tmux, pm2, or Telegram already do. The only custom code is the supervisor.
2. **Markdown is the app** — Bot behavior is defined in natural language, not code. `CLAUDE.md` is the system mechanism (shared across bots, upgraded by replacement); `SOUL.md` / `USER.md` / `CRONTAB.md` / `HEARTBEAT.md` / `SLEEP.md` carry personalization.
3. **Plugins are folders** — A skill is a folder with a `SKILL.md`. No npm install, no API registration. Claude Code auto-discovers it.
4. **Memory follows git** — `memory/` is git-tracked. Clone the repo, get the memory. No external database.
5. **Minimal code, maximum leverage** — Every line of code should justify why Claude Code can't do it natively. If Claude can handle it via markdown instructions, don't write code for it.
6. **Framework follows the plugin; personalization stays in side files** — Three pieces are pure framework — on upgrade they are replaced verbatim from `$CLAUDE_PLUGIN_ROOT` with at most a tiny graft of preserved values:
   - `supervisor/` (code only) — zero customization, straight overwrite. The supervisor knows nothing user-specific at compile time; every bot-aware behavior (which bots exist, their tokens / tmux sessions / work dirs, watchdog intervals, sleep & restart schedules, timezone, context-usage thresholds) is read at runtime from `ecosystem.config.cjs`. So the *code* is framework, the *config* is personalization, and the supervisor "adapts" to each project purely through env vars.
   - bot's `CLAUDE.md` — system mechanism, identical to `template/CLAUDE.md`. Only the two cron expressions in the Heartbeat/Sleep table are preserved.
   - meta-skills (currently `evolve`, `llm-wiki`, `learn`) — zero customization, straight overwrite of `<bot>/.claude/skills/<name>/`.

   Everything user-specific lives elsewhere: `SOUL.md`, `USER.md`, `CRONTAB.md`, `HEARTBEAT.md`, `SLEEP.md`, `memory/`, `journal/`, and `ecosystem.config.cjs` (supervisor token, Telegram user_id, BOTS list with per-bot token / session / work dir, schedule times, timezone, watchdog/context thresholds). If you're tempted to edit a framework file inside a bot or `supervisor/`, the change belongs upstream in this repo's template, not in the bot — otherwise the next `/zero-claw:upgrade` will silently revert it. The `upgrade` and `upgrade-meta-skill` skills enforce this contract.

## First-Run Setup

On first launch, detect unconfigured state and guide the user interactively. The canonical sequence lives in `skills/setup/SKILL.md` — this section is a high-level summary.

**Detection**: the bot directory's `CLAUDE.md` still matches `template/CLAUDE.md` (fresh install). As of 0.18 an empty `SUPERVISOR_BOT_TOKEN` is NOT an "unconfigured" signal — supervisor runs headless by default and the Supervisor remote-control bot is an opt-in final step.

**Flow** (summary):

1. Greet; pick language.
2. Check prerequisites (tmux, node, pm2, bun) and Telegram plugin.
3. Ask for the **assistant bot token** via @BotFather — one bot at this stage. Collect Telegram user_id, name, timezone.
4. Generate files. `ecosystem.config.cjs` ships with `SUPERVISOR_BOT_TOKEN` empty; `ALLOWED_USERS` is pre-populated so step 6 below only needs the token.
5. Start supervisor headless: `pm2 start ecosystem.config.cjs && pm2 save` — verify `pm2 logs` shows `Supervisor started [headless — no remote control bot]`. Launch the assistant bot and pair it over Telegram.
6. **Optional**: offer to add a Supervisor remote-control bot. Only if the user accepts, create a second BotFather bot, write `SUPERVISOR_BOT_TOKEN` into `ecosystem.config.cjs`, `pm2 restart <dirname>-supervisor --update-env`, and verify `/help` reaches it.

**Important**: Do NOT auto-launch `start.sh` during first-run setup — the user is already in an interactive Claude session. Just prepare everything so the next `./start.sh` works.

## Project Structure

```
zero-claw/
├── DESIGN.md               # Architecture deep-dive (Chinese)
├── CLAUDE.md               # This file (project identity + setup logic)
├── start.sh                # 1-line launcher
├── ecosystem.config.cjs    # pm2 config (generated by setup)
├── supervisor/             # Supervisor bot
├── template/               # Files copied into each new bot dir:
│                           #   CLAUDE.md (system — verbatim)
│                           #   HEARTBEAT.md, SLEEP.md, CRONTAB.md,
│                           #   SOUL.md, USER.md
│                           #   (all customized during setup)
├── skills/                 # Plugin skills: setup, add-bot, upgrade,
│                           # evolve, learn, llm-wiki, ...
├── commands/               # /zero-claw:<name> slash commands
└── plugins/                # Optional add-on skills
```

## Development Guidelines

- Keep the supervisor small. If it grows, something probably belongs in `template/CLAUDE.md` (mechanism) or a side file instead.
- Write English code and comments. DESIGN.md is Chinese.
- Don't add features that only benefit one user's setup. Keep the template generic.
- Test with: supervisor bot commands work, Claude Code starts/stops cleanly, memory persists across restarts.
- Bumping version: update `package.json`, `.claude-plugin/plugin.json`, and `.claude-plugin/marketplace.json` in the same commit. All three must stay in sync.
