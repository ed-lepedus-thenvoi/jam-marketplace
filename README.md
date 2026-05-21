# jam-marketplace

Claude Code plugin marketplace for the [`jam`](https://github.com/ed-lepedus-thenvoi/jam) CLI — a tool that lets Claude Code sessions coordinate with remote agents and humans on the [Band](https://band.ai) platform.

## Plugins

### band-peer

Adds a `/band-peer` slash command that wires the current Claude Code session as a Band peer. After invocation, inbound messages directed at the session arrive automatically as `<teammate-message>` blocks; outbound is `jam send` / `jam reply`.

## Install

Via the `jam` CLI (recommended — does both `marketplace add` and `install` for you):

```bash
jam plugin install
```

Manually:

```bash
claude plugin marketplace add ed-lepedus-thenvoi/jam-marketplace
claude plugin install band-peer@jam-marketplace
```

The plugin requires the `jam` CLI to be installed and on `$PATH` and a profile to be configured via `jam init`. See the [jam README](https://github.com/ed-lepedus-thenvoi/jam) for setup.
