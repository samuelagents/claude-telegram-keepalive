# Keeping a Claude Code Telegram bot alive on macOS: three failures nobody warns you about

I run Claude Code as an always-on Telegram bot on a Mac, so I can talk to my
agent from my phone. The happy-path setup is a one-liner. Keeping it *reliably*
alive for weeks turned out to be three separate fights — and each failure mode
was invisible until I went looking. This is the post I wish I'd found.

If you're wiring any long-lived CLI agent to a chat channel on a Mac, at least
the first lesson will probably bite you too.

---

## The setup (30 seconds of context)

The official Telegram plugin turns Claude Code into a bot: a small `server.ts`
long-polls Telegram, and a `claude` process holds the session. You start it and
DM your bot. On a desktop you've just logged into, it Just Works.

The problem is "always-on." I want it running headless, surviving reboots, and
reachable when I'm away from the machine — which means launching it from places
that are *not* a fresh GUI login. That's where it falls apart.

---

## Failure 1: the keychain wall-off (the one that wastes your afternoon)

Start the bot over SSH, or from a `tmux` session you attached to remotely, and
you get some variant of:

```
Not logged in. Run `claude` to authenticate.
# or
security: SecKeychainUnlock: User interaction is not allowed.
```

…even though you authenticated days ago and `claude` works fine in your normal
terminal.

**Root cause:** Claude Code stores its OAuth credentials in the macOS **login
keychain**. The login keychain is unlocked for your *GUI login session*. An SSH
session — and any process tree parented by `sshd` — runs in a **different
security session** that has no access to that unlocked keychain. So the binary is
authenticated "as you," but the process literally cannot read its own token.

This is maddening because nothing about the error points at session isolation.
You'll go re-authenticate five times and it'll keep happening.

**The durable fix — get off the keychain entirely.** Mint a long-lived token and
hand it to the process via the environment, so it never touches the keychain:

```sh
# once: create a long-lived OAuth token
claude setup-token
# store it somewhere the bot can read at launch
printf 'CLAUDE_CODE_OAUTH_TOKEN=%s\n' "$TOKEN" > ~/.claude/claude-code-oauth.env
chmod 600 ~/.claude/claude-code-oauth.env
```

Then `source` it right before exec:

```sh
[[ -f ~/.claude/claude-code-oauth.env ]] && source ~/.claude/claude-code-oauth.env
exec claude --channels 'plugin:telegram@claude-plugins-official' ...
```

Now it works identically from SSH, cron, or launchd — no GUI session required.
(The stopgap, if you don't want a token file, is to start your `tmux` server
*from a GUI terminal* so it inherits the unlocked login keychain. It works, but
it's fragile: one reboot or one `tmux kill-server` from the wrong context and
you're back to square one.)

> **Lesson:** "authenticated" and "able to read its credentials" are different
> things on macOS. Anything you launch outside your GUI login session — SSH,
> cron, launchd — is walled off from the login keychain. Move secrets to an env
> file or a service the process can actually reach.

---

## Failure 2: only one process may hold the bot

Telegram allows exactly **one** `getUpdates` long-poll per bot token. Start a
second Claude session that also has the Telegram plugin enabled and the two fight
over the token — you get HTTP 409 conflicts and messages get answered by
whichever process won the race that second. On a machine where you also run
`claude` for normal work, this happens constantly.

**The trick: keep the plugin globally OFF, force it ON only for the bot.** Set
the Telegram plugin disabled in your global settings, so plain `claude` sessions
never grab the bot slot. Then force-enable it for the *one* dedicated bot session
via a flag-tier override:

```sh
claude --channels 'plugin:telegram@claude-plugins-official' \
       --settings '{"enabledPlugins":{"telegram@claude-plugins-official":true}}'
```

The `--settings` flag overrides user settings, so the bot keeps Telegram even
though it's globally off everywhere else. Result: structurally impossible for a
stray work session to steal the channel.

> **Lesson:** when a resource is single-holder, don't rely on "remembering not to
> start a second one." Make the default state safe (off) and opt the single
> intended holder in explicitly.

---

## Failure 3: the bot that thinks it replied (silent drift)

This is the nasty one, because nothing is "down." The process is alive, the
server is polling, logs look fine — but you message it and *nothing comes back to
your phone*.

What happens: a long-running bot session **drifts**. After enough turns it
starts emitting its reply as **plain text in its own terminal** instead of
*calling the Telegram send tool*. From the model's point of view it answered. The
message never leaves the machine. It's a silent false-negative — the worst kind
of failure for a notification channel, because you only discover it when you
realize you haven't heard from it in a while.

You cannot catch this with a liveness check. "Is the process up?" is yes. You
need to reset the session *before* drift sets in.

**The fix: a daily freshness restart that never interrupts an active turn.** A
fresh session resets channel state and context, which clears the drift. The only
risk is restarting mid-conversation and wiping context — so guard it:

```sh
# Claude Code renders "esc to interrupt" only while a turn is in progress.
# Its presence == mid-turn → skip and retry next cycle. Absent == safe to restart.
pane="$(tmux -L tgfix capture-pane -t tg -p)"
if echo "$pane" | grep -q 'esc to interrupt'; then
  echo "bot mid-turn, skipping restart"
else
  restart_bot   # fresh session
fi
```

Using the TUI's own "esc to interrupt" hint as an idleness signal is a hack, but
it's a *reliable* one — it's exactly the state the program uses to decide whether
it can be interrupted.

> **Lesson:** for a delivery channel, monitor **behavior, not liveness**. "The
> process is running" tells you nothing about "messages are arriving." Design for
> the silent false-negative, because that's the failure you won't notice.

---

## The health model: exactly one, and prove it

Tying it together, the keepalive loop doesn't ask "is a process alive?" — it
asserts a much stronger invariant and heals toward it every 20 seconds:

1. the bot's `tmux` session exists, **and**
2. its pane process is alive, **and**
3. that process is actually the channel bot (`--channels plugin:telegram`), **and**
4. there is **exactly one** `server.ts` running, **and**
5. that server is a **descendant of the bot's pane** (not an orphan from a
   previous run).

Anything else → tear down the strays and start one clean session. Two servers is
"split brain" and gets healed just like zero servers.

One more guardrail worth stealing: **conservative reaping.** When cleaning up
stray sessions, only kill ones that are *demonstrably* the bot (their command
line contains `--channels plugin:telegram`). A normal `claude` work session that
happens to share a name is left completely alone. When in doubt, don't kill — a
missed stray gets caught on the next heal cycle, but a wrongly-killed work
session loses your context.

## The wiring (launchd, two jobs)

- A **`watch`** agent (`KeepAlive` + `RunAtLoad`) runs the heal loop forever:
  one healthy bot, always.
- A **daily** agent at 05:00 runs the idle-guarded freshness restart from
  Failure 3.

Both export an explicit `PATH` and `HOME` in the plist, because launchd jobs
start with a minimal environment — another flavor of the same "you're not in your
login session" lesson from Failure 1.

---

## Takeaways

1. **macOS keychain ≠ available everywhere.** SSH / cron / launchd are walled off
   from your GUI login keychain. Use a token file the process can read.
2. **Single-holder resources need a structurally safe default.** Off globally,
   force-on for the one intended holder.
3. **Monitor behavior, not liveness.** The dangerous failures are the silent
   ones where everything "looks" up but nothing is being delivered.
4. **Heal toward an invariant, conservatively.** Assert "exactly one, and it's
   really mine," and never kill something you can't prove is a stray.

None of this is Telegram-specific, really — it's what it takes to run *any*
long-lived authenticated agent on a Mac and trust that it's actually working.
Hope it saves someone an afternoon.

---

*The runnable version of everything here — the keepalive script and the launchd
templates — is in this repo. See the [README](../README.md).*
