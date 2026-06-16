# Troubleshooting

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
