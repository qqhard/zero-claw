---
description: Supervisor commands (status/restart/start/stop/logs/screen/send/monitor). Mirrors the Telegram supervisor bot.
argument-hint: <status|restart|start|stop|logs|screen|send|monitor|help> [bot] [args...]
allowed-tools: Bash(node:*), Bash(dirname:*)
---

Run the project-local supervisor CLI with the user's arguments: `$ARGUMENTS`

Execute this as a single Bash invocation (it walks up from cwd to find the project root, then calls the supervisor CLI that ships alongside the running pm2 supervisor — guaranteed same version):

```bash
root=$PWD; while [ "$root" != "/" ] && [ ! -f "$root/ecosystem.config.cjs" ]; do root=$(dirname "$root"); done; \
[ -f "$root/ecosystem.config.cjs" ] || { echo "Not inside a zero-claw project (no ecosystem.config.cjs in any ancestor of $PWD)"; exit 1; }; \
[ -f "$root/supervisor/cli.mjs" ] || { echo "supervisor/cli.mjs missing in $root — this project's supervisor/ is pre-0.19.0. Run /zero-claw:upgrade to install it."; exit 1; }; \
node "$root/supervisor/cli.mjs" $ARGUMENTS
```

Print the CLI output verbatim. Do not paraphrase, reformat, or add commentary. If the CLI exits non-zero, just show its stderr message.
