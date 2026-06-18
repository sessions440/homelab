# Changelog

---

## 2026-06-17 — Vaultwarden migration completed

- Migrated family Bitwarden cloud vaults to self-hosted Vaultwarden
- Personal vaults imported via JSON export; org vault recreated and imported separately
- Org members confirmed via web vault (appeared as "Needs Confirmation" rather
  than "Invited" — Vaultwarden auto-accepts invites when SMTP is disabled)
- One client device verified working; remaining member devices pending day-to-day confirmation
- `SIGNUPS_ALLOWED=false` set in `vaultwarden.env`
- Admin password reset during migration (original was mistyped at hash-generation time);
  reset procedure documented in `docs/services/vaultwarden.md`

## 2026-06-17 — Vaultwarden installed and running

- Installed Docker CE 29.5.3 + Compose v5 on vaultwarden LXC; enabled `keyctl=1` Proxmox feature
- Deployed Vaultwarden 1.36.0 via Docker Compose at `/opt/vaultwarden/`
- Admin token stored as argon2id hash in `/opt/vaultwarden/admin_token`, injected via `ADMIN_TOKEN_FILE`
  (env file interpolation in Compose v5 mangles `$argon2id$...` strings — file-based injection is the workaround)
- Updated Caddyfile on caddy LXC to `reverse_proxy 192.168.2.11:8080`
- Vaultwarden responding HTTP 200 on `127.0.0.1:8080` inside the LXC

## 2026-06-17 — Vaultwarden LXC provisioned

- Created LXC ID 110, hostname `vaultwarden`, IP `192.168.2.11/24`, 512 MB RAM, 8 GB disk, Debian 13
- `nesting=1` feature enabled (required for systemd 257)
- SSH key deployed; LXC reachable as `vaultwarden` host alias
- `apt update && apt upgrade` run on fresh template
- Vaultwarden binary not yet installed

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
