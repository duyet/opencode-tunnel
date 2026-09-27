# opencode-tunnel

Publish a local [OpenCode](https://opencode.ai) server to the internet over
HTTPS using a **Cloudflare Quick Tunnel**, protected by HTTP Basic auth.

No domain, no account, no Cloudflare login. You get a random
`*.trycloudflare.com` URL that works immediately.

## Quickstart

```bash
./opencode-tunnel setup     # verify opencode + cloudflared, generate a password
./opencode-tunnel start     # start server + tunnel, verify, print the URL
```

`start` prints the public URL, the username, the generated password, and the
exact command to attach a client.

```
  public URL : https://adventures-subdivision-winter-behavior.trycloudflare.com
  local      : http://127.0.0.1:4096
  user       : opencode
  no creds   : HTTP 401   (expect 401)
  with creds : HTTP 200   (expect 200)

  READY — password (store it safely):
  <generated>

  attach a client with:
    OPENCODE_PASSWORD='<generated>' opencode --server https://...trycloudflare.com
```

## Commands

| Command | What it does |
| --- | --- |
| `./opencode-tunnel setup` | Check prerequisites, generate credentials |
| `./opencode-tunnel start` | Start server + tunnel, verify end to end |
| `./opencode-tunnel status` | Health check, local and public |
| `./opencode-tunnel url` | Print the public URL only (for scripts) |
| `./opencode-tunnel rotate` | Generate a new password and restart |
| `./opencode-tunnel logs` | Tail server and tunnel logs |
| `./opencode-tunnel stop` | Tear everything down |

## Connecting a client

The auth is HTTP Basic, so the password belongs in an `Authorization` header.
There is no login form.

```bash
# TUI / CLI pointed at the remote server
OPENCODE_PASSWORD='<password>' opencode --server https://<url>

# Raw HTTP
curl -u 'opencode:<password>' https://<url>/api/info
```

```ts
// @opencode/client
import { OpenCode } from "@opencode/client"

const client = OpenCode.make({
  baseUrl: "https://<url>",
  headers: { authorization: `Basic ${btoa("opencode:<password>")}` },
})

const session = await client.session.create({ location: { directory: "/workspace" } })
await client.session.prompt({ sessionID: session.id, text: "Hello" })
```

## Configuration

Every setting has a default. See `.env.example`, or export the variables:

| Variable | Default | Notes |
| --- | --- | --- |
| `OC_DIR` | `$PWD` | Workspace the agent may operate in |
| `OC_HOST` | `127.0.0.1` | Keep on loopback so the tunnel is the only ingress |
| `OC_PORT` | `4096` | Local opencode port |
| `OC_PASSWORD` | generated | Basic auth password |
| `OT_STATE_DIR` | `~/.local/state/opencode-tunnel` | Config, pids, logs (mode 700) |

## Where state lives

```
~/.local/state/opencode-tunnel/
├── config          # OC_* settings incl. the password — mode 600
├── status.env      # current public URL — mode 600
├── server.pid
├── tunnel.pid
└── logs/
```

**No secret is ever written inside this repository.** `.gitignore` blocks
`config`, `status.env`, pids, logs, and anything matching `*password*`,
`*secret*`, `*token*` as a safety net.

## Security

Read [docs/SECURITY.md](docs/SECURITY.md) before exposing anything. The short
version:

- Anyone holding the URL **and** the password has full control of the machine:
  arbitrary shell execution, file writes, and your model provider credentials.
- Basic auth is the only gate. There is no rate limiting, no lockout, no
  second factor.
- Rotate with `./opencode-tunnel rotate` the moment the URL is disclosed.
- `OC_HOST` stays on `127.0.0.1` so the tunnel is the only way in — do not bind
  `0.0.0.0`.
- Treat `~/.local/state/opencode-tunnel/config` as a credential file.

## Quick Tunnel caveats

- The hostname is **random and changes every restart**. Not for production.
- **No uptime guarantee.** The URL dies with the process.
- Accountless: no SLA, and Cloudflare reserves the right to investigate
  accountless tunnel usage under its Online Services Terms.
- For a stable hostname on your own domain, use a **named tunnel** instead:
  `cloudflared tunnel login`, `cloudflared tunnel create <name>`,
  `cloudflared tunnel route dns <name> <name>.<your-domain>`, then
  `cloudflared tunnel run <name>`. Optionally front it with Cloudflare Access
  for real identity-based auth.

## Troubleshooting

**Public URL returns 502 / connection refused**
The server died or is still starting. `./opencode-tunnel status`, then
`./opencode-tunnel logs`.

**Public URL does not resolve right after `start`**
A fresh quick-tunnel DNS record takes a few seconds to become resolvable.
`start` already polls until it answers, so this normally resolves itself. To
force another attempt, `./opencode-tunnel rotate`.

**`401` even with the password**
You are probably using the wrong username; it is always `opencode`. Confirm
with `./opencode-tunnel status`, which tests both cases.

**Port already in use**
Set another port: `OC_PORT=5000 ./opencode-tunnel start`.

**`cloudflared: command not found`**
Install it from
<https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/>.
