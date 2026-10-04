# dsh-lan-gate

LAN access gateway for the **DeepSeek Harness (DSH)** web GUI: a reverse proxy that runs inside
the DSH host process and serves the interface on `0.0.0.0:3088`, with device approval, token to
cookie binding, per-IP rate limiting and a phone layout mode.

Author: MaxEkb77 ([github.com/maxekb](https://github.com/maxekb)). The plugin is self-contained:
no subprocesses, no external dependencies, no telemetry and no outbound traffic.

Русская версия: [README.md](./README.md).

## What it does

- **Reverse proxy** on `LAN_GATE_HOST:LAN_GATE_PORT` (default `0.0.0.0:3088`) to the local DSH
  web server.
- **Device approval**: the first request from a new address must be approved on the machine
  running DSH; an approval lasts 90 days, after which the device asks again.
- **Token + cookie binding**: one approval binds one browser, and the token is claimed once.
- **Rate limiting**: per IP per minute (120 requests by default, `429` on overflow).
- **Access modes**: `auto` / `phone` / `desktop`; phone mode injects the compact layout.
- **Admin panel** inside the DSH Settings UI (Settings → LAN Access): device list, trusted
  subnets, denying access.

## Requirements

- DeepSeek Harness `>= 0.1.7-rc.2` (tested on `0.2.0-rc.2`).
- Node.js `>= 20`.
- The plugin uses the DSH `webServer` and `connection` services. It has no browser half: the
  admin panel is served by the host half.

## Install

As a package bundle (its `cordis.patch.yml` inserts the row):

```sh
dsh plugin --profile web add dsh-lan-gate
```

Or from the source directory:

```sh
dsh plugin --profile web add <path-to-the-project>
```

Restart DSH afterwards: the composition row is read when the process starts.

> **Important.** If the profile already mounts the gateway by file path (a `- id: lan-gate` row
> with `name: ../../lan-gate/lan-gate.mjs`), **replace** that row instead of adding a second one:
> two entries sharing one `id` conflict.

## Configuration

Environment variables:

| Variable | Meaning | Default |
|---|---|---|
| `LAN_GATE_HOST` | listen address | `0.0.0.0` |
| `LAN_GATE_PORT` | gateway port | `3088` |
| `LAN_GATE_TOKENS_FILE` | file with the login link | `$DSH_HOME/lan-gate-tokens.md` |
| `DSH_HOME` | DSH state directory | `~/.dsh` |

State files:

- `$DSH_HOME/lan-gate-state.json` — per-device decisions and the trusted subnet list (edited from
  the admin panel; created on first run);
- `$DSH_HOME/lan-gate-tokens.md` — the login link carrying the current process's one-time token.

Default trusted subnets are `192.168.1.` and `192.168.100.` (canonical form is CIDR, e.g.
`192.168.1.0/24`); the list lives in the state file and is edited in the admin panel.

## Security — read before enabling

- The plugin exposes the **entire DSH interface** to the local network. That is access to your
  agent, files and tools, not a status page.
- Protection rests on device approval, cookie binding and rate limiting. **Narrowing the trusted
  subnets is your responsibility**: do not leave them wider than necessary.
- `lan-gate-tokens.md` contains a **live login link in plain text**. Keep the directory private
  and never publish its contents.
- Do not expose the gateway port directly to the internet. For remote access put a TLS
  terminator and/or a VPN in front — the gateway targets the local network.
- There is no telemetry and nothing is sent out. Outbound connections in the code are exactly two
  kinds, both to the same machine: proxying a request to the local DSH web server (`http.request`
  in `sendUpstream`, `net.connect` for the WebSocket upgrade) and calls from the served admin
  panel to the gateway's own routes (`fetch('/lan-gate/...')`).

## Limitations

- State is a single JSON file; concurrent edits from several processes are not coordinated.
- One gateway instance per DSH process (a second row with the same `id` is a composition error).
- Phone mode affects the interface layout, not the feature set.

## License

GNU GPL v3 (`GPL-3.0-only`) — full text in [LICENSE](./LICENSE).
