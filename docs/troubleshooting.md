# Troubleshooting

---

## Vaultwarden: admin login rejected despite correct password (2026-06-17)

**Symptom:** `/admin` login returns "Invalid admin token" even when the correct
plaintext password is entered.

**Cause (most likely):** The password was mistyped when originally running
`vaultwarden hash` in an SSH terminal (copy-paste wasn't working at the time).
The hash in `/opt/vaultwarden/admin_token` is internally consistent but doesn't
match the intended password.

**Secondary cause:** Too many failed attempts triggers an in-memory rate-limit
lockout. A container restart clears it, but the underlying password mismatch
must still be resolved.

**Fix:** Reset the admin password — see the "Resetting the admin password"
section in `docs/services/vaultwarden.md`. When generating the new hash, paste
the password rather than typing it manually.

---

## Vaultwarden: 502 from Caddy after initial deploy (2026-06-17)

**Symptom:** `https://bitwarden.<domain>` returned HTTP 502. Vaultwarden was healthy inside its LXC but unreachable from the Caddy LXC.

**Cause:** The Docker port binding was `127.0.0.1:8080:80`, which only listens on the loopback interface inside the vaultwarden LXC. Caddy runs on a separate LXC (`192.168.2.3`) and cannot reach a loopback-only port on another host.

**Fix:** Change the binding to `8080:80` (all interfaces) in `docker-compose.yml` and recreate the container with `docker compose up -d --force-recreate`.

**Note:** `127.0.0.1:` binding is only safe when the reverse proxy and service share the same host. In a multi-LXC architecture, services must bind on all interfaces so the Caddy LXC can reach them. The LAN firewall provides the perimeter; direct exposure to the internet is not a concern.

---

## Caddy DNS-01 challenge: `connection refused` on port 53

**Symptom:** Caddy logs `read udp ...:53: read: connection refused` when
attempting cert issuance.

**Cause:** Outbound UDP/TCP port 53 to external IPs (e.g. `1.1.1.1`,
`2606:4700:58::...`) is blocked on the caddy LXC. This affects both IPv4 and
IPv6 external resolvers.

**Fix:** Set `resolvers 192.168.2.1` in the `tls` block of each Caddyfile
site. AdGuard Home at `192.168.2.1` is reachable on port 53 and forwards to
upstream resolvers.

---

## Caddy DNS-01 challenge: `timed out waiting for record to fully propagate`

**Symptom:** Caddy logs `timed out waiting for record to fully propagate; last error: <nil>`.
Cloudflare audit log shows create/delete cycles for `_acme-challenge` TXT
records — the API writes are succeeding but Caddy can't confirm propagation.

**Cause:** AdGuard Home at `192.168.2.1` does not return newly-created
Cloudflare TXT records within certmagic's propagation polling timeout. Let's
Encrypt's own resolvers can see the record fine.

**Fix:** Add `propagation_delay 2m` to the `tls` block. This skips the
local polling check entirely and waits a fixed time before signalling Let's
Encrypt to verify.

---

## sshd crashes on `systemctl reload` in Debian 13 LXC (2026-06-19)

**Symptom:** `systemctl reload ssh` returns an error; SSH connections refused immediately after. `systemctl status ssh` shows `fatal: Cannot bind any address`.

**Cause:** On Debian 13 LXCs, `reload` sends SIGHUP to the running sshd process, which re-execs itself. During re-exec, sshd expects `/run/sshd` (the privilege separation directory) to exist. In an LXC that hasn't been rebooted since the initial install, this directory may not have been created, causing the re-exec to fail and sshd to exit entirely.

**Fix:** From the Proxmox console (CT 112 → Console):

```bash
mkdir -p /run/sshd
systemctl start ssh
```

**Prevention:** Always use `systemctl restart ssh` instead of `reload` on Debian 13 LXCs. `restart` brings up a fresh process that creates `/run/sshd` itself; `reload` re-execs in-place and depends on the directory already existing.

---

## Caddy leaks secrets into systemd journal

**Symptom:** `CF_API_TOKEN` (or other env vars) visible in plaintext in
`journalctl -u caddy` output at startup.

**Cause:** The upstream Caddy systemd unit uses `--environ` in `ExecStart`,
which logs all environment variables. Once `EnvironmentFile=/etc/caddy/caddy.env`
is added via a drop-in, secrets are included in that dump.

**Fix:** Override `ExecStart` in the drop-in to remove `--environ`:

```ini
[Service]
EnvironmentFile=/etc/caddy/caddy.env
ExecStart=
ExecStart=/usr/bin/caddy run --config /etc/caddy/Caddyfile
```

The blank `ExecStart=` line is required to clear the inherited value before
setting the new one. Then clear the journal:

```bash
journalctl --rotate && journalctl --vacuum-time=1s
```

---

## Vaultwarden: blank/unresponsive vault in desktop app and browser extension, web vault unaffected (2026-08-14)

**Symptom:** On one Bitwarden client machine, the desktop app and browser
extension both show an empty vault with an unresponsive sidebar (can't
switch vaults, no items render), while the web vault at
`https://bitwarden.<domain>` works normally with all items visible.
Initially suspected as related to the `vaultwarden` LXC renumbering
(CT 110 → 111) performed the same week — the two are unrelated, see Cause.

**Cause:** Bitwarden shipped a coordinated client release (`2026.7.0` —
browser extension, desktop app, mobile, CLI, server) in late July 2026.
The 2026.7.0 browser extension and desktop app share a WASM SDK component
with a confirmed upstream bug against self-hosted Vaultwarden servers:
login and sync succeed, but the vault fails to render, while the web vault
(a separate codebase) is unaffected. This is a widely reported bug
(multiple open issues on `bitwarden/clients` and `dani-garcia/vaultwarden`),
not specific to this homelab.

Bitwarden's server release notes state client v2026.7.0 requires
**Vaultwarden 1.37.0+** for compatibility. This server was still on
**1.36.0** (installed 2026-06-17) when this was hit — the affected client
had silently auto-updated to 2026.7.0 in the background, independent of
any homelab-side change, which is why the timing coincided with (but was
not caused by) the CT renumbering.

**Fix:**

1. Upgrade Vaultwarden on the `vaultwarden` LXC (`192.168.2.11`) to
   `1.37.0` or later:

```bash
   cd /opt/vaultwarden
   docker compose pull
   docker compose up -d
   docker compose ps               # confirm the new image is running
   docker compose logs --tail=50   # check for startup errors
```

`docker-compose.yml` already points at `vaultwarden/server:latest`, so
`pull` is sufficient — no compose file edit needed. 2. On the affected client, force a sync (or restart the app) and confirm
the vault now renders. 3. **If still broken after the server upgrade** (reported by some users
even on 1.37.0+): the confirmed workaround is downgrading that specific
client to `2026.6.1` via the browser extension store's version history,
or the desktop installer archive from Bitwarden's GitHub releases — a
manual step on the affected machine, outside what SSH access to the LXC
can address. 4. Update the version noted in `docs/services/vaultwarden.md` and add a
`docs/changelog.md` entry once resolved.

**Note:** not caused by, and unrelated to, the `vaultwarden` LXC ID
renumbering (110 → 111) performed the same week — included here for the
record since the timing was initially misleading.

**Resolved.** Verified success from a Bitwarden client.