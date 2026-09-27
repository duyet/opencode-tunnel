# One-shot prompts

Copy one of these into any AI agent (or hand to a person) to stand up the
tunnel in a single step. They are written to be self-contained: the agent needs
no other context.

---

## 1. Default — start the tunnel

> Set up a publicly reachable HTTPS URL for a local OpenCode server using a
> Cloudflare Quick Tunnel.
>
> 1. Clone this repository and `cd` into it.
> 2. Run `./opencode-tunnel setup`, then `./opencode-tunnel start`.
> 3. When it finishes, report back **verbatim**:
>    - the public `https://*.trycloudflare.com` URL
>    - the HTTP Basic auth username (`opencode`)
>    - the generated password
>    - the ready-to-run `opencode --server ...` client command
>    - the `./opencode-tunnel status` output proving 401 without credentials
>      and 200 with them
>
> Do not commit anything. Do not write the password into any file inside the
> repository. If the user asks how to shut it down, the answer is
> `./opencode-tunnel stop`.

---

## 2. Pin a specific workspace directory

> Set up the OpenCode Cloudflare Quick Tunnel from this repository, but scope the
> agent to `/srv/my-app` instead of the current directory:
>
> ```bash
> OC_DIR=/srv/my-app ./opencode-tunnel setup
> OC_DIR=/srv/my-app ./opencode-tunnel start
> ```
>
> Then report the public URL, the username, the password, and the client
> command. Confirm with `./opencode-tunnel status` that the endpoint returns
> 401 without credentials and 200 with them.

---

## 3. Use a non-default port

> Bring up the OpenCode tunnel on local port 5000 instead of 4096, because
> 4096 is already taken:
>
> ```bash
> OC_PORT=5000 ./opencode-tunnel setup
> OC_PORT=5000 ./opencode-tunnel start
> ```
>
> Report the public URL, username, password, client command, and the
> `status` verification.

---

## 4. Keep an existing password

> Start the OpenCode tunnel but reuse the password already stored in
> `~/.local/state/opencode-tunnel/config`. If no password exists yet, generate
> one first. Do not rotate a password that is already in use.
>
> ```bash
> ./opencode-tunnel start
> ```
>
> Report the public URL, username, password, and client command.

---

## 5. Rotate a disclosed password

> Someone had the URL and password for the running OpenCode Quick Tunnel. Rotate
> the password, which also restarts the tunnel and therefore changes the public
> URL:
>
> ```bash
> ./opencode-tunnel rotate
> ```
>
> Report the new public URL, the new password, and remind me that the old URL
> is now dead and the old password is invalid.

---

## 6. Diagnose an unhealthy tunnel

> The OpenCode Cloudflare tunnel in this repository is not working. The public
> URL returns 502 (or times out, or 401s unexpectedly).
>
> 1. Run `./opencode-tunnel status` and paste the output.
> 2. Run `./opencode-tunnel logs` and inspect the last 50 lines of both the
>    `opencode-serve` and `cloudflared` logs.
> 3. Check that the local server is listening on the configured port and that
>    `/api/info` returns 401 unauthenticated and 200 with
>    `-u opencode:<password>`.
> 4. If the cloudflared process died, restart with `./opencode-tunnel start`.
> 5. Tell me the root cause and the minimal fix. Do not change the bind address
>    away from `127.0.0.1`.

---

## 7. Promote to a stable named tunnel (own domain)

> The Quick Tunnel works, but the random `trycloudflare.com` hostname changes on
> every restart. Replace it with a named Cloudflare Tunnel on a domain I control,
> and put Cloudflare Access in front of it for real identity-based auth.
>
> 1. Run `cloudflared tunnel login` and wait for me to complete the browser
>    authorisation.
> 2. Create the tunnel and route DNS for a hostname I will provide.
> 3. Replace the `cloudflared tunnel --url ...` invocation in `opencode-tunnel`
>    with `cloudflared tunnel run <name>` plus an ingress rule pointing at
>    `http://127.0.0.1:<port>`.
> 4. Keep `OC_HOST=127.0.0.1` and keep the opencode password in place.
> 5. Show me the diff before applying it, and keep a fallback to the Quick
>    Tunnel behind a flag.

---

## Notes for whoever writes prompts like these

- Always name the exact command. An agent should never improvise the tunnel
  invocation.
- Always ask for the `status` verification. The script self-verifies, but
  reporting 401-vs-200 is what proves auth is actually enforced.
- Never ask the agent to commit credentials, and never let `OT_STATE_DIR`
  point inside the repository.
- If the machine is reachable over the tailnet, mention `./opencode-tunnel
  stop` — an exposed agent server is a remote shell.
