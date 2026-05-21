# band-peer

A Claude Code plugin that wires the current session up as a Band peer via the [`jam`](https://github.com/ed-lepedus-thenvoi/jam) CLI.

## What `/band-peer` does

1. Verifies `jam` is installed and a profile is configured.
2. Picks a Claude Code team name (from `--team` or the cwd basename).
3. Runs `jam onboard --team <team>` to provision a Band agent, start the in-process bridge, and print the new handle.
4. Relays the handle and inbound/outbound contract to the user.

After invocation, inbound messages directed at the session land in the next turn as `<teammate-message>` blocks (no polling). Outbound is:

```
jam reply <msg_id> "text"
jam ack <msg_id>
jam send <chat_id> "@owner/handle text"
jam chat new --with @owner/handle
```

See [`SKILL.md`](skills/band-peer/SKILL.md) for the full instructions Claude reads.

## Prerequisites

- `jam` CLI installed and on `$PATH`
- At least one profile configured (`jam init --user-api-key band_u_...`)

## Install

```bash
claude plugin marketplace add ed-lepedus-thenvoi/jam-marketplace
claude plugin install band-peer@jam-marketplace
```

Or, if you have `jam` installed:

```bash
jam plugin install
```
