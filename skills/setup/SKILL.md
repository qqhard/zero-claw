---
name: setup
description: "First-run setup wizard for Zero-Claw. Triggers: 'setup bot', 'configure assistant', 'zero-claw setup', or when ecosystem.config.cjs has empty SUPERVISOR_BOT_TOKEN."
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
  - TaskCreate
  - TaskUpdate
---

# Zero-Claw Setup

Interactive setup wizard. Guide the user step by step.

Plugin root is available as `$CLAUDE_PLUGIN_ROOT`.

## Before Starting

Use TaskCreate to create **all** tasks upfront so the user sees the full picture:

1. "Choose language"
2. "Check prerequisites (tmux, node, pm2)"
3. "Check Telegram plugin"
4. "Create two Telegram bots (main + supervisor)"
5. "Configure Telegram plugin"
6. "Get Telegram user ID"
7. "Collect user info (name, timezone, assistant name)"
8. "Choose working directory"
9. "Generate project files"
10. "Start supervisor"
11. "Verify and launch"

Mark each task `in_progress` when starting it, `completed` when done.

## Steps

1. **Language**: Ask the user's preferred language. Continue in that language.

2. **Prerequisites**: Check installed tools:
   ```bash
   tmux --version
   node --version
   pm2 --version
   ```
   If anything is missing, tell the user what to install and stop.

3. **Telegram plugin**: Check if the Telegram plugin is installed:
   ```bash
   claude plugins list 2>/dev/null | grep telegram
   ```
   If not installed, run `claude plugins install telegram`.

4. **Create two Telegram bots**: Guide the user to open [@BotFather](https://t.me/BotFather) and create two bots. Explain why two:
   - **Main bot** — your assistant's face, for daily conversation. Suggest names based on the assistant name chosen later (or ask to come back and rename). Username like `myname_bot` or `myname_assistant_bot`.
   - **Supervisor bot** — remote control for when the main bot is unresponsive. Lets you restart, check status, view logs. Username like `myname_supervisor_bot`.
   
   For each bot, tell the user to paste the **entire BotFather response** — parse the token yourself using regex: `/\d+:[A-Za-z0-9_-]{35,}/`. Clearly label which is which when confirming back.

5. **Configure Telegram plugin**: Run `/telegram:configure` and guide the user to paste the **main bot** token to pair the channel.

6. **User ID**: Ask the user to message [@userinfobot](https://t.me/userinfobot) on Telegram. They can paste the entire reply — extract the numeric `Id` field yourself.

6. **User info**:
   - **User's name**: Auto-detect from Telegram profile or system info. Confirm with the user, don't make them type it.
   - **Timezone**: Ask for timezone (e.g. `Asia/Singapore`). Try to auto-detect from system (`timedatectl` or `TZ` env) and offer as default.
   - **Assistant name**: Ask the user to name their assistant. Suggest 3-5 names from mythology, folklore, or fiction — pick randomly from diverse cultures and pantheons each time (Greek, Norse, Egyptian, Hindu, Chinese, Japanese, Celtic, Mesopotamian, etc.). For each suggestion, give a one-line reason why the name fits an AI assistant (e.g. knowledge, wisdom, communication, protection). Let the user pick one or type their own.

8. **Working directory**: Default to `~/<assistant-name-lowercase>` (e.g. if assistant is "Thoth", default `~/thoth`). Let the user confirm or change.

9. **Generate files** in the working directory:
   - Copy `$CLAUDE_PLUGIN_ROOT/template/CLAUDE.md` → `CLAUDE.md`, fill in all placeholders (assistant name, user name, timezone, language).
   - Copy `$CLAUDE_PLUGIN_ROOT/supervisor/` → `supervisor/`, run `npm install`.
   - Generate `ecosystem.config.cjs` with the collected values. Use `<assistant-name-lowercase>` as the `TMUX_SESSION` name and the **supervisor bot token**.
   - Copy `$CLAUDE_PLUGIN_ROOT/start.sh` → `start.sh`, make executable.
   - Create `memory/MEMORY.md` (empty).
   - Create `memory/journal/` directory.
   - Initialize git repo.

10. **Start supervisor**: Run `pm2 start ecosystem.config.cjs && pm2 save`.

11. **Summary**: Tell the user everything is ready. Show how to launch (using assistant name as tmux session):
    ```bash
    tmux new-session -s <name> -c ~/<name> './start.sh'
    ```
    This attaches the user directly into the tmux session so they can watch Claude Code start up. To detach later: `Ctrl-b d`. To re-attach: `tmux attach -t <name>`.
    
    Also remind them of supervisor bot commands: `/status`, `/restart`, etc.
