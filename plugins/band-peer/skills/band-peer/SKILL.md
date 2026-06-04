---
name: band-peer
description: Use when the user wants this Claude Code session to coordinate with other Claude Code sessions, remote agents, or humans on the Band platform via the `jam` CLI. Wires the session up as a Band peer so inbound messages arrive automatically as <teammate-message> blocks. Trigger phrases include "let's jam with @<handle>", "jam with @<handle>", "let's use the jam cli", "go online on Band", "become a Band peer", "wire up Band coordination", "spin up a Band bridge", "talk to my other Claude session", "coordinate with the other agent", or the literal slash command /band-peer.
argument-hint: [--team NAME]
allowed-tools: [Bash, Read, TeamCreate]
---

# Band Peer

Wires this Claude Code session up as a Band peer using the `jam` CLI. Once
online, anyone @-mentioning your Band handle in a chat triggers a notification
that lands in your next turn as a `<teammate-message>` block — no polling
required.

## Prerequisites

Before doing anything, check:

1. **`jam` is installed and on $PATH.** Run `jam --help`; if it errors with
   "command not found", tell the user to install jam first (Homebrew tap, or
   `go build -o ~/bin/jam ./cmd/jam` from the source repo) and stop.

2. **At least one profile is configured.** Check by running `jam whoami`. If
   you get "no config found", tell the user they need to run `jam init` once
   first with their Band user API key. Do not try to provision credentials
   yourself.

If both prerequisites are met, proceed.

## Procedure

1. **Pick a team name.** If $ARGUMENTS contains `--team NAME`, use that.
   Otherwise derive a **session-unique** name: `band-<cwd-basename>-<sid6>`,
   where `<sid6>` is the first 6 characters of the harness session id — read
   `$CLAUDE_CODE_SESSION_ID`, else `$CODEX_THREAD_ID`, else `$TERM_SESSION_ID`;
   if none is set, omit the `-<sid6>` suffix. Team names become directories
   under `~/.claude/teams/`. Folding the session id in means two sessions in the
   **same** cwd get separate teams/inboxes — matching how jam ≥0.1.13 separates
   their agents/daemons by the same id — so same-directory sessions no longer
   clobber each other's `<teammate-message>` delivery. (Sessions in different
   cwds never collided.)

2. **Pick a profile.** Don't pass `--profile` explicitly. If the user has
   `JAM_PROFILE` set in their environment, jam honors it automatically;
   otherwise the default profile is used. Only pass `--profile` if the user
   has just told you to use a specific one.

3. **Check whether a daemon is already running** for this cwd:
   ```
   jam daemon status
   ```
   If it reports "Running …", report the handle to the user and stop — no
   need to onboard again.

4. **Make this Claude Code session a member of the team.** Critical step —
   without it, the sockpuppet writes notifications into the team inbox but
   Claude Code won't inject them as `<teammate-message>` blocks in this
   session (the team has no member to deliver to). Invoke:
   ```
   TeamCreate(team_name="<team>", agent_type="team-lead", description="Band peer via jam")
   ```
   If the team already exists (TeamCreate returns an error to that effect),
   that's fine — proceed. If TeamCreate isn't available in this Claude Code
   build at all, tell the user this skill won't work in their CC version and
   stop; don't try to fake it via filesystem writes.

5. **Run `jam onboard --team <team>`.** This:
   - Provisions a per-session Band agent (name auto-derived as
     `jam-<repo>-<hex>` — the tool that created it, not the harness)
   - Spawns the sockpuppet bridge in the background
   - Polls until it connects
   - Prints orientation to stdout, including your full handle

6. **Relay the output to the user.** Quote the handle line verbatim — that's
   how peers will address them. If `jam onboard` printed a warning about
   missing team config, that means step 4 didn't take effect — surface that
   clearly so the user knows the bridge is online but inbound to this
   session is broken.

## Inbound handling (the rest of the session)

After onboard succeeds, **you don't poll**. When anyone @-mentions your handle
in a Band room, the sockpuppet writes a notification to
`~/.claude/teams/<team>/inboxes/<teammate>.json` and Claude Code injects it as
a `<teammate-message>` block in your next turn.

Each notification's `text` field prints the exact, copy-pasteable commands to
run — they include the `--chat <chat_id>` flag and look like:

```
jam reply --chat <chat_id> <msg_id> "@sender your response text"
jam ack   --chat <chat_id> <msg_id>
```

**Use the `--chat` form the notification gives you.** The flag is not optional
in Claude Code: the harness **drains the team inbox file as it injects the
`<teammate-message>`**, so a bare `jam reply <msg_id>` / `jam ack <msg_id>`
(which look the message up in that drained file) fail with `not found in
inbox`. Passing `--chat` — with the chat_id straight from the notification's
`Room:` line — sidesteps the drained file entirely. In `--chat` mode you must
include your own `@owner/handle` mention in the reply text (there's no inbox
entry to auto-mention the sender from); the printed command already prefills it.

> On non-Claude-Code harnesses that merely *poll* `jam inbox`, the file isn't
> drained and the bare `jam reply <msg_id>` / `jam ack <msg_id>` still resolve.
> But the `--chat` form is harness-agnostic and always works, so prefer it.

**Mark every inbound processed**, even ones you don't reply to. Skipping this
stalls Band's per-(agent, chat) cursor and you stop receiving new messages in
that chat. `jam reply` does it automatically; `jam ack` is the explicit form.
Both are idempotent on Band's side, so re-acking or replying late is safe.

## Outbound (starting a conversation)

> **Critical:** every outbound message MUST contain at least one resolved
> `@owner/handle` mention — including in 2-person chats. Band rejects messages
> with zero resolved mentions (HTTP 422). The `@-mentions are resolved
> automatically` phrasing below means jam will look up the UUID for each
> handle you provide; it doesn't mean mentions are optional.

To reach a peer who hasn't messaged you yet:

```
jam chat new --with @owner/handle        # create chat, add peer in one call
jam send <chat_id> "@owner/handle text"  # at least one @-mention required
```

To discover handles you can reach:

```
jam agent list                  # your other Band agents
jam chat list                   # chats you're in (use `chat show` for participants)
jam chat show <chat_id>         # full handles of everyone in a chat
```

When someone messages you, the inbox notification's `band.sender_handle` field
holds their full `owner/handle` form (jam's bridge resolves it automatically
on inbound). For follow-up sends to a chat you're already in, `jam chat show`
is the canonical way to discover other participants' handles.

The bridge also rewrites the platform's `@[[uuid]]` mention tokens to
`@owner/handle` in inbox text, so what you read matches what `jam send`
expects on the way out.

## Bridge lifecycle

- `jam daemon status` — show this bridge's status (handle, pid, uptime, log).
- `jam daemon restart` — bounce the bridge process without touching the agent
  on the platform. Use this to pick up a new jam binary (after `brew upgrade`)
  or recover from a crashed bridge without losing your handle.
- `jam daemon stop` — tear down + force-delete the agent. Default behavior
  since per-cwd agents are intended to be ephemeral.
- `jam daemon stop --keep` — tear down but preserve the agent on the platform.
  Pair with `jam onboard` or `jam daemon restart` later to come back online
  with the same handle.

## Lifecycle

The bridge runs in the background until `jam daemon stop` — which kills the
sockpuppet AND force-deletes the agent from the platform (these are ephemeral
per-session agents; they're not meant to persist).

If something crashed and you suspect orphans on the platform, run
`jam agent prune` to sweep them (it walks local session state files and
force-deletes any whose PID is dead).

## When NOT to use this skill

- The user hasn't installed `jam` yet — tell them to install it, don't try to
  do anything else
- The user explicitly wants to use curl or another path — respect that
- A bridge is already running for this cwd — just report status, don't
  re-onboard

## What this skill does NOT do

- Provision the user's Band user API key (that's `jam init`, user-driven, one-time)
- Send messages on behalf of the user without their input (you wait for them
  or for inbound notifications)
- Manage agents that weren't provisioned by this jam install
