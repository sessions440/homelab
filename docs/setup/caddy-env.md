# Caddy Environment File Setup

This step must be performed **manually by the human** before handing off Caddy
configuration to Claude Code. It ensures that the real domain name and
Cloudflare API token are never transmitted to Anthropic (see Secrets Management
in CLAUDE.md).

---

## What you need first

### Cloudflare API Token

Create a scoped token at https://dash.cloudflare.com/profile/api-tokens:

1. Click **Create Token** → **Create Custom Token**
2. **Permissions:** Zone → DNS → Edit
3. **Zone Resources:** Include → Specific zone → your domain
4. No IP filtering required unless you want extra hardening
5. Copy the token — it is only shown once

---

## Steps on the caddy LXC

SSH in manually (not via Claude Code):

```bash
ssh root@192.168.2.3
```

Create the secrets directory and env file:

```bash
mkdir -p /etc/caddy
cat > /etc/caddy/caddy.env << 'EOF'
HOMELAB_DOMAIN=your-real-domain.com
CF_API_TOKEN=your-cloudflare-api-token
EOF
```

Lock down permissions (readable only by root):

```bash
chmod 600 /etc/caddy/caddy.env
chown root:root /etc/caddy/caddy.env
```

Verify the file looks right (do not share this output with Claude Code):

```bash
cat /etc/caddy/caddy.env
```

Exit the SSH session.

---

## What happens next

When Claude Code installs Caddy, it will:

1. Create `/etc/caddy/Caddyfile` using `{env.HOMELAB_DOMAIN}` and
   `{env.CF_API_TOKEN}` — no literal values
2. Drop a systemd override at `/etc/systemd/system/caddy.service.d/env.conf`
   containing:

   ```ini
   [Service]
   EnvironmentFile=/etc/caddy/caddy.env
   ```

3. Reload systemd and start Caddy

Caddy reads the env file at startup and interpolates the variables. The running
config will use your real domain; the files Claude Code touches never will.

---

## Notes

- `/etc/caddy/caddy.env` is not committed to the homelab git repo (secrets are
  never in the repo — see CLAUDE.md)
- If you rotate the Cloudflare token, update this file and restart Caddy:
  `systemctl restart caddy`
- If you ever need to re-confirm the file is in place before a Claude Code
  session, check existence only: `ls -l /etc/caddy/caddy.env` — do not `cat`
  it in a context Claude Code can see
