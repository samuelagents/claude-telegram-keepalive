# claude-telegram-keepalive

Keep a [Claude Code](https://claude.com/claude-code) **Telegram bot** alive and
healthy on macOS — surviving reboots, SSH sessions, and the silent failures that
make an "always-on" agent quietly stop delivering.

Running the official Telegram plugin once is a one-liner. Running it *reliably
for weeks* means fighting three non-obvious failure modes. This repo is the small
zsh keepalive + launchd setup I landed on, plus a write-up of why each piece
exists so you can adapt it.

> 📖 **The full story:** [`docs/three-failures.md`](docs/three-failures.md) —
> the keychain wall-off, single-holder token competition, and silent send-drift.

## The three failures, in one paragraph

1. **Keychain isolation.** Claude Code keeps its OAuth token in the macOS login
   keychain, which is only unlocked for your GUI login session. Start the bot
   over SSH / cron / launchd and the process can't read its own credentials
   ("Not logged in"). → Fix: a long-lived token in an env file, sourced at launch.
2. **Single holder.** Telegram allows one long-poll per bot token; a second
   Claude session with the plugin enabled causes HTTP 409 conflicts. → Fix: keep
   the plugin globally **off**, force it **on** for the one bot session only.
3. **Silent send-drift.** A long-running session eventually replies as plain text
   instead of calling the send tool — the bot "thinks" it answered, your phone
   stays empty. Liveness checks can't see this. → Fix: a daily, idle-guarded
   freshness restart.

## Install

```sh
# 1. the keepalive script
cp bin/claude-telegram-tgfix ~/.local/bin/
chmod +x ~/.local/bin/claude-telegram-tgfix

# 2. (Failure 1) a long-lived token so the bot never needs the keychain
claude setup-token
printf 'CLAUDE_CODE_OAUTH_TOKEN=%s\n' "<token-from-above>" > ~/.claude/claude-code-oauth.env
chmod 600 ~/.claude/claude-code-oauth.env

# 3. (Failure 2) keep the plugin OFF globally so work sessions don't steal the slot
#    (the bot force-enables it per-session via --settings; nothing to do but
#     make sure it isn't enabled in your global ~/.claude settings)

# 4. start it
~/.local/bin/claude-telegram-tgfix start
~/.local/bin/claude-telegram-tgfix status
```

### Run it under launchd (survives reboot + daily freshness restart)

The two plists in [`launchd/`](launchd/) are templates with a `__HOME__`
placeholder. Substitute your home dir and install:

```sh
for f in launchd/*.plist.template; do
  out=~/Library/LaunchAgents/$(basename "${f%.template}")
  sed "s#__HOME__#$HOME#g" "$f" > "$out"
  launchctl unload "$out" 2>/dev/null || true
  launchctl load "$out"
done
```

- `ai.claude.telegram-keepalive` — the `watch` heal loop (one healthy bot, always).
- `ai.claude.telegram-restart` — the idle-guarded daily freshness restart (05:00).

## Commands

| command | what it does |
| --- | --- |
| `start` | start the bot if absent; leave a healthy one alone |
| `restart` | tear down strays and start a fresh session |
| `restart-idle` | freshness restart, but skip if the bot is mid-turn |
| `stop` | stop the bot session and its server |
| `status` | print token / session / server state |
| `watch` | long-running heal loop (launchd mode) |

## Config

All overridable via environment (defaults shown):

| var | default | meaning |
| --- | --- | --- |
| `CLAUDE_OAUTH_ENV` | `~/.claude/claude-code-oauth.env` | token file sourced at launch |
| `TGFIX_SOCKET` | `tgfix` | dedicated tmux socket |
| `TGFIX_SESSION` | `tg` | tmux session name |
| `TGFIX_PLUGIN` | `plugin:telegram@claude-plugins-official` | `--channels` value |
| `TGFIX_PLUGIN_KEY` | `telegram@claude-plugins-official` | key for `--settings` |
| `TELEGRAM_STATE_DIR` | `~/.claude/channels/telegram` | plugin state dir |
| `TGFIX_SERVER_MATCH` | `server.ts` | `pgrep -f` pattern for the poller |
| `TGFIX_HEAL_INTERVAL` | `20` | seconds between `watch` heals |

## The health model

`watch` doesn't ask "is a process alive?" — it asserts a stronger invariant and
heals toward it: the tmux session exists, its pane really is the channel bot,
there is **exactly one** server, and that server **descends from the pane** (not
an orphan from a previous run). Anything else gets torn down and restarted clean.
Reaping is **conservative** — it only kills processes it can prove are the bot
(`--channels plugin:telegram`), never your own `claude` work sessions.

## Caveats

- **macOS + zsh only.** It leans on the macOS keychain behavior, `launchctl`,
  and a couple of zsh-isms.
- **Unofficial.** Not affiliated with Anthropic. It wraps the official Telegram
  plugin from the outside; it doesn't modify it. Plugin internals (the
  `server.ts` poller, state dir) may change between plugin versions — adjust the
  config vars if so.
- It assumes you run the bot on its own dedicated tmux socket so it can manage
  the session lifecycle without touching your other terminals.

## License

MIT
