# Caddy Reverse Proxy

**LXC ID:** 100  
**IP:** `192.168.2.3`  
**OS:** Debian 13 (trixie)  
**Status:** Running, TLS working

---

## Installation

Caddy is installed from the official Caddy apt repository (not the older Debian
package), with the `caddy-dns/cloudflare` plugin added via `caddy add-package`.

```bash
# Add official Caddy apt repo
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install -y caddy

# Add Cloudflare DNS plugin (replaces the binary in-place)
caddy add-package github.com/caddy-dns/cloudflare
```

Installed version: **2.11.4** with `dns.providers.cloudflare v0.2.4`

---

## Caddyfile

Location: `/etc/caddy/Caddyfile`

```caddyfile
bitwarden.{env.HOMELAB_DOMAIN} {
	reverse_proxy 192.168.2.11:8080

	tls {
		dns cloudflare {env.CF_API_TOKEN}
		resolvers 192.168.2.1
		propagation_delay 2m
	}
}
```

**Why `resolvers 192.168.2.1`:** Outbound UDP/TCP port 53 to external IPs is
blocked on this LXC. The local AdGuard instance at `192.168.2.1` is accessible
on port 53 and is used as the resolver.

**Why `propagation_delay 2m`:** AdGuard does not return freshly-created
Cloudflare TXT records within certmagic's propagation polling timeout. Setting
a fixed delay bypasses the polling check; Let's Encrypt uses its own public
resolvers to verify and sees the record correctly.

---

## Secrets

Caddy reads secrets from `/etc/caddy/caddy.env` via a systemd drop-in. This
file is **not** committed to the repo and must be created manually — see
`docs/setup/caddy-env.md`.

| Variable         | Contains                         |
| ---------------- | -------------------------------- |
| `HOMELAB_DOMAIN` | Public domain (e.g. example.com) |
| `CF_API_TOKEN`   | Cloudflare API token (DNS-01)    |

---

## Systemd Configuration

Two drop-in files under `/etc/systemd/system/caddy.service.d/`:

**`env.conf`:**
```ini
[Service]
EnvironmentFile=/etc/caddy/caddy.env
ExecStart=
ExecStart=/usr/bin/caddy run --config /etc/caddy/Caddyfile
```

The blank `ExecStart=` clears the upstream value before setting the override.
The `--environ` flag is intentionally removed from the upstream unit's
`ExecStart` — it logs all environment variables at startup, which would expose
`CF_API_TOKEN` in plaintext in the systemd journal.

Reload and restart after any changes:
```bash
systemctl daemon-reload && systemctl restart caddy
```

---

## TLS / Certificate Status

- **Issuer:** Let's Encrypt (production)
- **Challenge type:** DNS-01 via Cloudflare API
- **First issued:** 2026-06-16
- **Auto-renewal:** Managed by Caddy/certmagic

---

## Adding a New Service

1. Add a new block to `/etc/caddy/Caddyfile`:

   ```caddyfile
   service.{env.HOMELAB_DOMAIN} {
       reverse_proxy <LXC-IP>:<port>

       tls {
           dns cloudflare {env.CF_API_TOKEN}
           resolvers 192.168.2.1
           propagation_delay 2m
       }
   }
   ```

2. Also add a DNS A record in Cloudflare pointing `service.<domain>` to the
   LAN IP of the caddy LXC (`192.168.2.3`) — DNS-only (grey cloud), not proxied.

3. Reload Caddy:
   ```bash
   systemctl reload caddy
   ```
