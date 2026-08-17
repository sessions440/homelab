# Changelog

---

## 2026-08-17 — Vaultwarden upgraded to 1.37.1

- Upgraded Vaultwarden container on `vaultwarden` LXC (`192.168.2.11`) from 1.36.0 to 1.37.1 via `docker compose pull` and `docker compose up -d`
- Verified container health check and `/alive` HTTP 200 response
- Resolves upstream client rendering issue with Bitwarden 2026.7.0+ desktop and browser extensions
- Verified success from a Bitwarden client.

## 2026-08-14 — Minecraft Java Edition server installed (Decision Path A)

- Installed `openjdk-25-jre-headless` (Java 25.0.4) on `minecraft` LXC (`192.168.2.13`)
- Created system user `minecraft` (`/opt/minecraft`, `/usr/sbin/nologin`)
- Downloaded Minecraft Java Edition server release 26.2 from Mojang official version manifest to `/opt/minecraft/server.jar`
- Accepted EULA in `/opt/minecraft/eula.txt`
- Created and enabled systemd unit `/etc/systemd/system/minecraft.service` with `-Xms6G -Xmx6G`
- Verified service running and world generated; TCP port 25565 open and responding to LAN connections

## 2026-08-13 — Minecraft LXC provisioned

- Proxmox housekeeping: Vaultwarden LXC ID change 110 -> 111 to conform to convention that LXC ID xyy has LAN IP address 192.168.2.yy
- Created CT 113, hostname `minecraft`, IP `192.168.2.13/24`, Debian 13, 2 cores, 8192MB RAM, 512MB swap, 32GB disk, unprivileged
- Enabled `nesting=1` post-creation (Debian 13 / systemd 257 warning, same pattern as other CTs)
- SSH keys (`ai_homelab`, `human_homelab`) authorized; password authentication disabled
- Java/server install deferred to a coding-agent session (trialing Gemini 2.5 Flash via OpenCode) — see `docs/services/minecraft.md` for the execution plan
- **Note for the record:** Minecraft moved to calendar versioning in 2026; current releases (26.1+) require **Java 25**, not the Java 21 commonly cited in older guides — flagged explicitly in the service doc

## 2026-08-12 — Fix: corrected SSH Access section in CLAUDE.md

- SSH Access section previously showed a `Host caddy` alias with `IdentityFile ~/.ssh/ai_homelab`,
  implying Claude Code uses Host aliases — it does not
- Rewrote section to accurately describe the two-key system: `ai_homelab` used by
  Claude Code via explicit `-i` flag on IPs; `human_homelab` used by humans via Host aliases
- No functional change; documentation-only correction

## 2026-08-12 — SSH key renamed: claude_code_homelab → ai_homelab

- Renamed Claude Code automation SSH key files from `~/.ssh/claude_code_homelab`
  to `~/.ssh/ai_homelab`; key comment updated to `ai_homelab` via `ssh-keygen -c -C`
- Old files preserved at `~/.ssh/claude_code_homelab.old` and `~/.ssh/claude_code_homelab.pub.old`
- Key identity (public key fingerprint) is unchanged; no re-authorization needed on any container
- Updated all references in: `docs/setup/ssh.md`, `docs/services/git.md`, `CLAUDE.md`,
  `.claude/settings.local.json`, `~/.ssh/config`
- Connectivity verified: `ssh -i ~/.ssh/ai_homelab root@192.168.2.3` responds correctly

## 2026-06-24 — Research: OpenCode model cost/quality analysis

- Reviewed cost and capability of cheaper OpenRouter models as alternatives to
  Claude Sonnet 4.5 (`anthropic/claude-sonnet-latest`, $15/1M output tokens)
- Analysis tailored to actual task profile of this project (config writing,
  sysadmin shell commands, documentation, light debugging)
- Three candidates evaluated:
  - **Claude Haiku 4.5** — 67% cheaper output, same Anthropic provider, 200K ctx
  - **Gemini 2.5 Flash** — 83% cheaper output, Google, 1M ctx, built-in thinking
  - **DeepSeek V3 0324** — 95% cheaper, flagged as unsuitable for agentic sessions
    involving live infrastructure due to data sovereignty concerns
- Findings recorded in `docs/plan/ai-model-costs.md`

## 2026-06-22 — Docs: clarified AppVM SSH key installation for git server

- Added two subsections to `docs/services/git.md` under "Client SSH access":
  - "Adding a key from a machine with root access" (existing flow, now explicit)
  - "Adding a key from a machine without root access (e.g. an AppVM)" — covers
    the case where the new machine cannot SSH as root; key must be installed via
    a primary machine that has root access, since `git-shell` blocks interactive
    login and password auth is disabled

## 2026-06-19 — git LXC provisioned and configured

- Created LXC ID 112, hostname `git`, IP `192.168.2.12/24`, 512 MB RAM, 8 GB disk, Debian 13
- `nesting=1` feature enabled (required for systemd 257)
- Installed git 2.47.3 and openssh-server 10.0p1
- Created `git` user with `/usr/bin/git-shell` as login shell, home at `/srv/git`
- Set up `/srv/git/.ssh/authorized_keys`; `claude_code_homelab` key authorized
- `git-shell-commands/no-interactive-login` script added
- Password authentication disabled (`PasswordAuthentication no`)
- **Issue encountered:** `systemctl reload ssh` caused `fatal: Cannot bind any address` — sshd
  exited and `/run/sshd` privilege separation directory was not recreated. Fixed by running
  `mkdir -p /run/sshd && systemctl start ssh` from Proxmox console. Use `systemctl restart ssh`
  not `reload` in LXC environments.
- SSH setup notes written to `docs/setup/ssh.md`
- `human_homelab` key created; authorized on git LXC (`/srv/git/.ssh/authorized_keys`) and all existing containers (`/root/.ssh/authorized_keys`)
- Host aliases in `~/.ssh/config` reassigned to `human_homelab`; `Host 192.168.2.12` block added for git remote URL compatibility
- Claude Code switched to using IPs with explicit `-i ~/.ssh/claude_code_homelab` flag; no SSH aliases needed

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
