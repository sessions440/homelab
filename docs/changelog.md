# Changelog

---

## 2026-06-16 — Caddy installed and TLS working

- Installed Caddy 2.11.4 from official apt repo on caddy LXC (`192.168.2.3`)
- Added `caddy-dns/cloudflare` plugin via `caddy add-package`
- Created `/etc/caddy/Caddyfile` with `bitwarden.{env.HOMELAB_DOMAIN}` virtual host
- Created systemd drop-in `/etc/systemd/system/caddy.service.d/env.conf` to
  load `/etc/caddy/caddy.env` and remove the `--environ` flag from ExecStart
- First Let's Encrypt certificate issued successfully via DNS-01 challenge
- **Issues encountered and resolved:**
  - Outbound port 53 (UDP+TCP) blocked to external IPs — fixed by setting
    `resolvers 192.168.2.1` in Caddyfile tls block
  - DNS propagation polling timing out against AdGuard — fixed by adding
    `propagation_delay 2m` to skip local polling
  - `--environ` flag in upstream Caddy systemd unit caused `CF_API_TOKEN` to
    appear in plaintext in the journal — fixed by overriding `ExecStart` in
    the drop-in to omit the flag
- SSH key passphrase removed from `~/.ssh/claude_code_homelab` (automated
  access key, passphrase provides no benefit)
