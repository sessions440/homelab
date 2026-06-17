# Vaultwarden

Self-hosted Bitwarden-compatible password manager.

---

## Infrastructure

| Item     | Value                               |
| -------- | ----------------------------------- |
| LXC ID   | 110                                 |
| IP       | `192.168.2.11`                      |
| OS       | Debian 13                           |
| RAM      | 512 MB                              |
| Disk     | 8 GB                                |
| SSH host | `vaultwarden` (via `~/.ssh/config`) |

LXC is unprivileged with `nesting=1` and `keyctl=1` features enabled.
`nesting=1` is required for systemd 257 on Debian 13; `keyctl=1` is required for Docker.

---

## Status

- **2026-06-17:** LXC provisioned. Docker installed. Vaultwarden 1.36.0 running via Docker Compose. Caddy proxying to this LXC.

---

## Runtime

Vaultwarden runs as a Docker container managed by Docker Compose.

- Compose file: `/opt/vaultwarden/docker-compose.yml`
- Config: `/opt/vaultwarden/vaultwarden.env`
- Admin token: `/opt/vaultwarden/admin_token` (argon2id hash, bind-mounted read-only into container)
- Data volume: Docker named volume `vaultwarden_vaultwarden_data` (SQLite DB, attachments)

```bash
# Start / stop
docker compose -f /opt/vaultwarden/docker-compose.yml up -d
docker compose -f /opt/vaultwarden/docker-compose.yml down

# Logs
docker compose -f /opt/vaultwarden/docker-compose.yml logs -f --tail=100
```

### docker-compose.yml

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    volumes:
      - vaultwarden_data:/data
      - ./admin_token:/admin_token:ro
    ports:
      - "8080:80"
    env_file:
      - path: vaultwarden.env
        required: true
    environment:
      - ADMIN_TOKEN_FILE=/admin_token

volumes:
  vaultwarden_data:
```

### Admin token note

The `ADMIN_TOKEN` value is an argon2id PHC string (generated with
`docker run --rm -it vaultwarden/server /vaultwarden hash --preset owasp`).
It cannot be passed via an env file because Docker Compose v5 performs shell
variable interpolation on env file values, which mangles the `$argon2id$...`
string. The fix is to write the raw token to a file and pass the path via
`ADMIN_TOKEN_FILE`, which Vaultwarden reads directly.

---

## Caddy Integration

`/etc/caddy/Caddyfile` on the caddy LXC (`192.168.2.3`):

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

---

## Ports

| Port | Protocol | Purpose                                                |
| ---- | -------- | ------------------------------------------------------ |
| 8080 | TCP      | Vaultwarden HTTP (internal only; Caddy terminates TLS) |
| 3012 | TCP      | WebSocket notifications (optional, not yet wired up)   |
