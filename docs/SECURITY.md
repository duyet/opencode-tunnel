# Security notes

Read this before pointing the tunnel at anything that matters.

## Threat model

This script publishes an **OpenCode server** to the public internet. An OpenCode
server is not a read-only API. Anyone who authenticates can:

- run arbitrary shell commands (`/api/shell`, session `shell` routes)
- read and write files in and around `OC_DIR` (`/api/fs/*`)
- create sessions, run prompts, and burn your model provider quota
- reach anything the agent's own credentials can reach

So the URL plus the password **is** a remote shell with your permissions.
Treat it accordingly.

## What actually protects you

One thing only: HTTP Basic auth, enforced by OpenCode itself via
`OPENCODE_SERVER_PASSWORD`. There is no second factor, no rate limiting, no
lockout, no IP allowlist, and no audit log of failed attempts on a Quick Tunnel.

The password is 256 bits of entropy from `secrets.token_urlsafe(32)`, so
brute force is not the threat model. **Disclosure** is: a URL pasted into a
chat, a CI log, a screenshot, a shared dotfile.

## Hardening checklist

- [x] `OC_HOST` defaults to `127.0.0.1` — the tunnel is the only ingress path.
      Do not set `0.0.0.0`; that exposes the port to the whole LAN, bypassing
      the tunnel entirely.
- [x] `OPENCODE_SERVER_PASSWORD` is set in the child environment only, so it
      never appears in a `ps` listing or a shell history.
- [x] State directory is mode `700`; `config` and `status.env` are mode `600`.
- [x] `.gitignore` blocks `config`, `status.env`, pids, logs, and
      `*password*` / `*secret*` / `*token*` patterns.
- [ ] Set OpenCode's `permissions` to deny-by-default for destructive tools if
      you do not trust every URL holder. See
      <https://opencode.ai/v2/docs/permissions/>.
- [ ] Rotate with `./opencode-tunnel rotate` whenever the URL is disclosed.
      Rotation restarts the tunnel, so the hostname changes too — the old URL
      dies immediately.
- [ ] Prefer a **named tunnel + Cloudflare Access** for anything long-lived.
      Access gives real identity (email, SSO, groups) instead of one shared
      password.

## Why the password is safe to print once

Basic auth needs the secret on every request, so the agent that starts the
tunnel has to hand it back to you. The script prints it to **stdout only**,
after the endpoint has been verified to reject unauthenticated requests. It is
never written to a log file, never echoed into a URL query string, and never
placed in the repository.

If you need it without displaying it:

```bash
# read it without printing
PASS=$(grep '^OC_PASSWORD=' ~/.local/state/opencode-tunnel/config | cut -d= -f2-)

# hand it straight to a client, no shell history
read -rs PASS && OPENCODE_PASSWORD="$PASS" opencode --server "$(./opencode-tunnel url)"
```

## What a Quick Tunnel does *not* give you

- **No stable hostname.** A new random `*.trycloudflare.com` name on every
  restart. DNS allowlists and bookmarks break each time.
- **No uptime guarantee.** It dies with the process and nobody is paged.
- **No account, no SLA.** Cloudflare's accountless terms explicitly reserve the
  right to investigate for violations.
- **No Access integration.** Identity-based auth is unavailable without an
  account, so Basic auth is the only option.

For anything you intend to keep running, use a named tunnel.

## Reporting a leak

If the URL and password both leak, the fastest safe move is:

```bash
./opencode-tunnel rotate
```

This invalidates the old password **and** retires the old hostname in one step.
Then check the OpenCode logs at `~/.local/share/opencode/log/opencode.log`
(`role=server`) for sessions you do not recognise.

## Redacting before sharing output

If you paste `status` or `logs` output into a bug report, strip:

- the `trycloudflare.com` hostname
- `~/.local/state/opencode-tunnel/config`
- any `Authorization` header
- anything from `~/.local/share/opencode/log/opencode.log` (prompts, file
  contents, provider keys)
