# AGENTS.md

Instructions for an AI agent working **in this repository**.

## What this repo is

A single self-contained script, `opencode-tunnel`, that publishes a local
OpenCode server over HTTPS via a Cloudflare Quick Tunnel with HTTP Basic auth.
There is no build step, no dependencies to install, and no test suite.

```
opencode-tunnel     the only executable; all logic lives here
README.md           user-facing docs
PROMPT.md           copy-pasteable one-shot prompts
docs/SECURITY.md    read before changing anything about auth or binding
.env.example        every supported setting, with defaults
```

## Rules

1. **Never commit a secret.** The password lives only in
   `~/.local/state/opencode-tunnel/config` (mode 600). If you ever need to
   change where state lives, keep it outside the repo. `.gitignore` is a safety
   net, not permission to be careless.
2. **Keep `OC_HOST` on loopback.** `127.0.0.1` is what makes the authenticated
   tunnel the only ingress path. Do not "fix" this to `0.0.0.0`.
3. **Do not disable the self-verification.** `start` deliberately polls the
   public URL until it returns 401 unauthenticated and 200 authenticated. A
   fresh quick-tunnel DNS record is not resolvable for several seconds; the
   retry loop exists for that reason. Removing it produces false failures.
4. **Do not print the password to stdout before verification passes.** It is
   printed once, after the 401/200 check succeeds.
5. **Never pass `OPENCODE_SERVER_PASSWORD` on a command line.** It is exported
   into the child environment only, so it stays out of `ps` and shell history.
6. **Prefer editing the existing script over adding files.** It is deliberately
   one file so it can be audited and copied without path assumptions.
7. **Update `PROMPT.md` when you change the interface.** Those prompts are the
   product for other people's agents; a renamed flag or subcommand makes them
   wrong.

## Verifying a change

There is no test harness. Verify by hand, using a throwaway state directory and
a non-default port so you do not disturb a running instance:

```bash
OT_STATE_DIR=/tmp/ot-test OC_PORT=4097 OC_DIR=/workspace ./opencode-tunnel setup
OT_STATE_DIR=/tmp/ot-test OC_PORT=4097 OC_DIR=/workspace ./opencode-tunnel start
OT_STATE_DIR=/tmp/ot-test ./opencode-tunnel status
OT_STATE_DIR=/tmp/ot-test ./opencode-tunnel stop
rm -rf /tmp/ot-test
```

`start` must end with `no creds: HTTP 401` and `with creds: HTTP 200`. If it
does not, the change is not ready — do not weaken the check to make it pass.

Check the script parses before running anything:

```bash
bash -n opencode-tunnel
```

## Shell conventions in this file

- `set -euo pipefail`, `die`/`info`/`step` helpers, `"$var"` everywhere.
- Lowercase locals; `UPPER_CASE` for config and exported state.
- `umask 077` before writing any file that can contain the password.
- Prefer `printf` over `echo` (the password is URL-safe base64, but `printf` is
  still the habit to keep).
