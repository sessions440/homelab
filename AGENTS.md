# Homelab — Claude Code Context

---

## ⚠️ Hard Limits

- **Do NOT access the Proxmox host** (`192.168.2.2`) via SSH or any other means.
  Proxmox-level operations (VM creation, storage pools, network bridges) are
  handled manually. Use Claude chat for guidance on those tasks.
- **Do NOT modify Cloudflare DNS records** without explicit written confirmation.
- Do not run `docker compose down -v` or any command that destroys named volumes
  without explicit written confirmation in this session.
- Do not modify networking configs: `/etc/network/interfaces`, `/etc/netplan/`,
  firewall rules, or bridge configs on any host.
- Do not delete or overwrite `.env` files.
- If a planned action could cause data loss or service downtime, **stop and confirm**
  before proceeding.

---

## Secrets Management

All sensitive values — domain name, API tokens, passwords — are injected via
environment variables. **Claude Code must never write a literal value for these
into any file it creates or edits**, and must never run commands whose output
would reveal them (e.g. `env | grep DOMAIN`, `curl` against the live domain,
`dig` on the real hostname, or `caddy validate` output that echoes it).

Verification steps that would expose the real domain must be flagged for the
human to run manually.

---

## Server Overview

| Item       | Value                                     |
| ---------- | ----------------------------------------- |
| Hypervisor | Proxmox VE 9.2.3                          |
| Proxmox IP | `192.168.2.2` (off-limits to Claude Code) |
| LAN subnet | `192.168.2.0/24`                          |

### Hardware

- Intel Core i7-7700 @ 3.60GHz, 64-bit
- 32GB RAM
- 466GB SSD (Samsung 850 EVO)
- NVIDIA GTX 1060 6GB

---

## Network Topology

- **Router:** OpenWrt (D-Link DIR-885L)
- **DNS:** AdGuard Home on port 53 with conditional upstream for `home.arpa`
- **DHCP:** dnsmasq, port 5335 only
- **Public DNS:** Cloudflare (domain delegated from Namecheap; free tier)
- **Static IPs** assigned to all servers (see inventory below); dynamic clients
  get MAC-based DHCP leases outside the static range
- **External connectivity:** Starlink residential (CGNAT, dynamic IP)

### DNS Architecture Notes

- Internal subdomains (e.g. `caddy.home.arpa`) resolve via AdGuard → `home.arpa`
  conditional upstream
- Public subdomains for internal services (e.g. `bitwarden.<yourdomain.com>`) use
  real Let's Encrypt certs issued via DNS-01 ACME challenge through Cloudflare API.
  These resolve to LAN IPs and are unreachable from outside — security through
  inaccessibility, not obscurity.
- Existing public DNS records remain proxied (Cloudflare orange cloud).
  Internal-only subdomains are DNS-only (grey cloud).
- DKIM CNAME records are DNS-only by requirement (proxying breaks email auth).

---

## VM / LXC Inventory

| Name          | ID  | Type               | IP             | RAM    | Disk   | OS        | Purpose                             |
| ------------- | --- | ------------------ | -------------- | ------ | ------ | --------- | ----------------------------------- |
| caddy         | 100 | LXC (unprivileged) | `192.168.2.3`  | 512 MB | 8 GB   | Debian 13 | Reverse proxy (Caddy)               |
| immich        | 210 | VM                 | `192.168.2.10` | 12 GB  | 150 GB | Debian 13 | Docker host: Immich (deprioritized) |
| vaultwarden   | 111 | LXC (unprivileged) | `192.168.2.11` | 512 MB | 8 GB   | Debian 13 | Password manager (Vaultwarden)      |
| git | 112 | LXC (unprivileged) | `192.168.2.12` | 512 MB | 8 GB | Debian 13 | Bare git remote (LAN-only) |
| minecraft     | 113 | LXC (unprivileged) | `192.168.2.13` | 8192 MB | 32 GB  | Debian 13 | Minecraft server (LAN-only)          |

### LXC Provisioning Notes (Debian 13)

Debian 13 ships with systemd 257, which requires the `nesting` feature enabled
or systemd services may misbehave. Always set at creation or immediately after:

```bash
pct set <id> --features nesting=1
```

Datacenter-level DNS is configured in Proxmox UI → Datacenter → Options → DNS.
Leave the DNS field blank in individual CT/VM wizards to inherit from there.

---

## Service Inventory

### caddy — `192.168.2.3`

- **Status:** Running. Caddy 2.11.4 installed with `caddy-dns/cloudflare` plugin; TLS working
- **Purpose:** Reverse proxy for all internal services; handles TLS termination
  via Let's Encrypt DNS-01 challenge (Cloudflare plugin)
- **Architecture:** All internal HTTPS services route through Caddy. Backend
  services are isolated in their own VMs/LXCs and are not directly exposed.
- **Docs:** `docs/services/caddy.md`

### immich — `192.168.2.10`

- **Status:** Bare Debian 13 VM; Docker not yet installed; deprioritized
- **Planned runtime:** Docker + Docker Compose
- **Docs:** `docs/services/immich.md`

### vaultwarden — `192.168.2.11`

- **Status:** Running. Vaultwarden 1.36.0 via Docker Compose; Caddy proxying to `192.168.2.11:8080`
- **Purpose:** Self-hosted Bitwarden-compatible password manager
- **Runtime:** Docker Compose at `/opt/vaultwarden/`; admin token via `ADMIN_TOKEN_FILE`
- **Docs:** `docs/services/vaultwarden.md`

### git — `192.168.2.12`

- **Status:** Running. git 2.47.3 + openssh-server; `git` user with `git-shell`; password auth disabled
- **Purpose:** LAN-only bare git remote. No web frontend.
  Acts as a push/pull target for personal repos from client machines.
- **Runtime:** `git` + `openssh-server`; dedicated `git` user with `git-shell`
- **Access:** `git@git.home.arpa:/srv/git/<repo>.git`
- **Encryption:** Cleartext on server. LAN-only exposure; SSH-gated.
  Offsite encrypted backup planned separately.
- **Docs:** `docs/services/git.md`

### minecraft — `192.168.2.13`

- **Status:** LXC provisioned (CT 113, Debian 13, 2 cores, 8192MB RAM, 512MB swap, 32GB disk, unprivileged, `nesting=1`). SSH keys deployed, password auth disabled. Java/server install pending — see `docs/services/minecraft.md` for the execution plan.
- **Purpose:** LAN-only Minecraft Java Edition server. Internet exposure (WireGuard/tunnel) planned for later.
- **Runtime:** Bare JVM via systemd (no Docker) — planned unit `minecraft.service`
- **Docs:** `docs/services/minecraft.md`

---

## SSH Access

Two keys are in use:

| Key | User | Method |
|-----|------|--------|
| `~/.ssh/ai_homelab` | Claude Code | IP address + explicit `-i` flag; no Host aliases |
| `~/.ssh/human_homelab` | Human interactive | `Host` aliases in `~/.ssh/config` |

Claude Code connects via IP with an explicit `-i` flag, for example:

```bash
ssh -i ~/.ssh/ai_homelab root@192.168.2.3
```

The `Host` aliases in `~/.ssh/config` (caddy, vaultwarden, git) are for human
interactive access and use `human_homelab`. Claude Code does not use them.
**Do not add a Host alias block for `192.168.2.2` (Proxmox host).**

See `docs/setup/ssh.md` for full key strategy, provisioning steps, and config.

---

## Conventions

### Reverse Proxy (Caddy)

- All internal HTTPS services get a `bitwarden.{env.HOMELAB_DOMAIN}`-style
  virtual host block — always using env interpolation, never a literal domain
  (see Secrets Management above)
- Caddyfile lives at `/etc/caddy/Caddyfile` on the caddy LXC
- Caddy's systemd unit loads secrets from `EnvironmentFile=/etc/caddy/caddy.env`
- Each new service gets its own subdomain block; Caddy handles cert renewal
  automatically
- Internal `home.arpa` names are for LAN convenience; HTTPS certs use the
  public subdomain via the env var

### Caddyfile Convention

All Caddyfile blocks must use Caddy's native env interpolation syntax — never
a literal domain:

```caddyfile
# CORRECT
bitwarden.{env.HOMELAB_DOMAIN} {
    reverse_proxy localhost:8080
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
}

# WRONG — never write a literal domain
bitwarden.example.com {
    ...
}
```

Caddy resolves `{env.HOMELAB_DOMAIN}` at runtime from the environment loaded
by systemd. Claude Code writes only the template; the human populates the env
file manually (see setup instructions in `docs/setup/caddy-env.md`).

#### Defined Variables

| Variable         | Contains                         | Set in                 |
| ---------------- | -------------------------------- | ---------------------- |
| `HOMELAB_DOMAIN` | Public domain (e.g. example.com) | `/etc/caddy/caddy.env` |
| `CF_API_TOKEN`   | Cloudflare API token (DNS-01)    | `/etc/caddy/caddy.env` |

### Docker Compose

- Compose files live at `/opt/<service>/docker-compose.yml`
- Environment variables in `/opt/<service>/.env` (not committed to this repo)
- Prefer named Docker volumes; document their names in the service's doc file
- To restart a service: `docker compose -f /opt/<service>/docker-compose.yml restart`
- To view logs: `docker compose -f /opt/<service>/docker-compose.yml logs -f --tail=100`

### Documentation (mandatory)

After **every task**, update docs before ending the session:

1. If a service was configured or changed → update `docs/services/<service>.md`
2. Add a dated entry to `docs/changelog.md`
3. If something failed or behaved unexpectedly → add to `docs/troubleshooting.md`
4. If a new VM or LXC was provisioned → update the inventory table in this file

### End of Session

At the end of every session, propose a `git commit` covering all changes made
during the session before signing off. The human approves and runs it, or asks
Claude Code to run it. Claude Code may run the commit directly once the human
has given explicit approval in the session.

### Snapshots

Human takes Proxmox snapshots manually before authorizing any high-risk operation.
If a task is high-risk and no snapshot has been confirmed, flag it before proceeding.

---

## Repository Structure

```
homelab/
├── CLAUDE.md                    ← this file; keep current
├── docs/
│   ├── architecture.md          ← network diagram, hardware, design decisions
│   ├── changelog.md             ← dated log of all changes
│   ├── troubleshooting.md       ← problems encountered and resolutions
│   ├── plan/
│   │   ├── local-ai-stack.md    ← Pending local AI stack plan and decision log
│   ├── setup/
│   │   ├── etckeeper-proxmox.md ← etckeeper installation gotchas for Proxmox
│   │   ├── ssh.md               ← SSH keypair setup and deployment notes
│   │   └── caddy-env.md         ← manual env file setup on caddy LXC
│   └── services/
│       ├── caddy.md             ← reverse proxy setup, Caddyfile, cert notes
│       ├── vaultwarden.md       ← (stub)
│       ├── immich.md            ← setup history, config notes, lessons learned
│       └── minecraft.md         ← (stub)
└── scripts/                     ← helper scripts, if any
```

This repo intentionally does **not** contain secrets, `.env` files, or private keys.

---

## Wiki Pipeline Note

Claude chat logs from this project are being harvested for synthesis into a
structured knowledge wiki (separate repo). When writing doc updates, favour:

- Factual, self-contained explanations (assume no prior context)
- Explicit rationale for non-obvious decisions
- Full commands rather than abbreviated hints
- Dated entries in changelog so the wiki timeline is accurate

---

## Known History / Context

- Proxmox host at `192.168.2.2`; Proxmox UI accessible at `https://192.168.2.2:8006`
- AdGuard + OpenWRT DNS stack already configured and working (`home.arpa`), except `home.arpa` not working on the Claude Code machine due to conflict with VPN. Troubleshooting is a future project. In the meantime use IP addresses directly whenever `home.arpa` fails.
- DNS delegated from Namecheap to Cloudflare (free tier); nameservers updated
  2025-06. Existing public records retained and proxied. DKIM CNAMEs left as
  DNS-only per Cloudflare warning.
- Caddy LXC (ID 100) provisioned 2025-06 via Proxmox UI; SSH pubkey for Claude
  Code uploaded at creation. `nesting=1` feature set post-creation due to
  systemd 257 warning on Debian 13.
- immich renumbered from `192.168.2.3` to `192.168.2.10` to free up low IP
  for Caddy (core infrastructure convention)
- Immich setup deprioritized; Docker not installed on immich
- Minecraft VM not yet provisioned
- git server LXC planned; threat model: cleartext acceptable for LAN-only use, SSH-gated. Offsite encrypted backup to follow.

## Inexperience with coding agents

The human is currently very new to coding agents. Err on the side of human intervention for now. Feel free to make suggestions, including edits to `AGENTS.md`. Explain yourself.

## Git authorship convention

Commits in this repo use two identities — no personal names or real email addresses:

| Author | Name | Email |
|---|---|---|
| Human-authored (any human involvement) | `Human` | `human@localhost` |
| AI-authored (solely by AI) | `AI` | `ai@localhost` |

When running git commits, always pass `--author` explicitly to set the correct identity regardless of the local git config. Example:

```bash
git commit --author="AI <ai@localhost>" -m "message"
git commit --author="Human <human@localhost>" -m "message"
```

Note: existing history uses `none` as the email for all commits (pre-convention).
New commits should use `ai@localhost` or `human@localhost` per the table above.