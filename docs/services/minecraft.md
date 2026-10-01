# Minecraft Server

Self-hosted Minecraft Java Edition server. LAN play works directly; internet
access for a whitelisted set of outside players goes through a **playit.gg**
tunnel (see "External Access" below).

 **TODO** split this doc into `docs/plan/minecraft.md` (the "why?") and `docs/services/minecraft.md` (the "how?") following the pattern of `encrypted-git.md`
---

## Infrastructure

| Item     | Value                                                        |
| -------- | ------------------------------------------------------------- |
| LXC ID   | 113                                                            |
| IP       | `192.168.2.13/24`                                              |
| OS       | Debian 13                                                      |
| Cores    | 2                                                               |
| RAM      | 8192 MB                                                         |
| Swap     | 512 MB                                                          |
| Disk     | 32 GB                                                           |
| Type     | Unprivileged, `nesting=1`                                       |
| SSH host | `minecraft` (add to `~/.ssh/config` — see docs/setup/ssh.md)   |

`nesting=1` was enabled post-creation to resolve the systemd 257 warning
common to all Debian 13 LXCs in this homelab (see AGENTS.md → LXC
Provisioning Notes).

---

## Status

- **2026-09-29:** External-access decision changed from self-hosted VPS + FRP to **playit.gg** (see "External Access"). Not yet implemented; blocked only on the human creating a playit.gg account and claiming an agent.
- **2026-08-14:** Java 25 (`openjdk-25-jre-headless`) installed. Dedicated user `minecraft` created with home `/opt/minecraft`. Minecraft Vanilla Server 26.2 downloaded, EULA accepted, and configured as a systemd service (`minecraft.service`). Verified active and listening on port `25565`.
- **2026-08-13:** LXC provisioned. SSH keys (`ai_homelab`, `human_homelab`)
  authorized; password authentication disabled.

---

## External Access (Internet)

**Status:** Decision made — **playit.gg**. Not yet implemented.

### Decision history

Goal: let a whitelisted set of players outside the LAN connect alongside LAN
players, without granting them any broader network access. Home internet is
Starlink (CGNAT), so inbound connections are impossible regardless of port
forwarding; some outbound-initiated tunnel is required. This also rules out
WireGuard-into-LAN, which would give every player network-level presence on
the same subnet as Vaultwarden, the git server, etc.

1. **First choice (abandoned): self-hosted VPS + FRP.** Chosen for
   reusability, zero recurring cost, and upskilling. Blocked when Oracle Cloud
   Always Free rejected the signup (support ticket unresolved). Google Cloud's
   Always Free e2-micro was considered next (Standard network tier required
   to get 200 GB/month free egress, plus a billing budget alert) but never
   provisioned.
2. **Pivot (current): playit.gg.** A turnkey third-party tunnel service built
   for game hosting. Players enter an address in vanilla Minecraft's server
   list — no client software on their end. The host-side agent is open
   source (the relay backend is not). Free tier is sufficient for a small
   whitelist.

**What was given up by the pivot:**
- A third party sits in the data path (acceptable: nothing sensitive crosses
  this tunnel, and Minecraft's `online-mode=true` encrypts the session).
- No reusable VPS with a controllable public IP / reverse DNS (e.g. for a
  future mail relay). If that need returns, revisit VPS+FRP — the generic
  mechanics are preserved in [`docs/setup/vps-relay.md`](../setup/vps-relay.md),
  which is now **shelved, not deleted**.
- No custom domain on the free tier (players use a playit-assigned address).
  Custom domains via playit are a paid feature — verify current terms before
  relying on one.

**Not affected by this pivot:** offsite backup (restic + rsync.net or similar)
remains a fully separate decision. Future HTTP services (e.g. Miniflux) can
use Cloudflare Tunnel instead; playit is for raw game traffic.

### Things to know before starting

- **Account claim is a human step.** The agent prints a claim URL on first
  run; the human opens it in a browser and logs in/registers at playit.gg.
  The coding agent cannot and should not do this.
- **The agent secret key is a credential.** It lives in the agent's config
  file on the LXC (expected: `/etc/playit/playit.toml` — confirm with
  `dpkg -L playit`). Anyone holding it can run a tunnel as this account.
  Keep it out of the repo, out of docs, and out of any command output the
  coding agent captures (same policy as `caddy.env` — see AGENTS.md → Secrets
  Management). If it leaks, delete the agent in the playit dashboard and
  claim a new one.
- **Enable 2FA / use a strong unique password on the playit account.** Account
  compromise = someone can repoint your tunnel.
- **All connections appear to come from the local agent** (127.0.0.1), not
  the player's real IP. Consequences: `ban-ip` and IP-based logging are
  useless. The **whitelist (username-based) is the real access control.**
  Playit supports PROXY protocol to forward real IPs, but vanilla Minecraft
  doesn't parse it — only proxies like Velocity/BungeeCord do. Not needed here.
- **The public address is not secret and is scannable.** Treat the server as
  publicly reachable: whitelist on, `online-mode=true`, keep the server jar
  updated.
- **Pick the region closest to you and your players** when creating the
  tunnel (North America East for Toronto), rather than a default/anycast
  region if a choice is offered — latency depends on it.
- **Free-tier limits change.** Check the current tunnel count, bandwidth,
  and inactivity/expiry rules on playit.gg's pricing/FAQ pages when signing
  up, rather than trusting this doc. Record anything binding in the Status
  section once known.
- **Address stability:** the assigned address is tied to the account/agent.
  Deleting the tunnel or agent may change it and require re-sharing with
  players.

### Setup Plan

> Steps 1–2 are **human-only** (account and claim). Steps 3+ can be handed to
> the coding agent with SSH access to this LXC, *except* anything that would
> print the agent secret. Keep this section updated as ground truth once
> execution starts.

1. **(Human) Create a playit.gg account.** Enable 2FA.

2. **(Human) Confirm whitelist is already enforced** before any tunnel exists
   (do this first, not last):
   ```
   # in the Minecraft console
   whitelist add <username>
   ```
   and confirm `server.properties` has:
   ```
   white-list=true
   enforce-whitelist=true
   online-mode=true
   ```
   Restart the server if these were changed.

3. **Install the agent** on this LXC (Debian apt repo per playit's docs —
   re-check https://playit.gg/support/run-on-linux/ for the current commands
   before running, in case the repo URL or package name changed):
   ```bash
   apt install -y gpg curl
   curl -SsL https://playit-cloud.github.io/ppa/key.gpg | gpg --dearmor | tee /etc/apt/trusted.gpg.d/playit.gpg >/dev/null
   echo "deb [signed-by=/etc/apt/trusted.gpg.d/playit.gpg] https://playit-cloud.github.io/ppa/data ./" | tee /etc/apt/sources.list.d/playit-cloud.list
   apt update
   apt install -y playit
   ```
   If the LXC's `apt` warns about the repo, review the playit GitHub repo
   before proceeding.

4. **(Human, interactive) Claim the agent.** SSH in as the human (`ssh minecraft`)
   and run:
   ```bash
   systemctl enable --now playit.service
   playit setup
   ```
   Open the printed claim URL in a browser, log in, and complete the
   "Create agent" flow. Do not paste the claim URL or any resulting secret
   into a coding-agent session.

5. **(Human, dashboard) Create the tunnel:** Dashboard → Tunnels → Add Tunnel
   → type **Minecraft Java (game)**, region closest to you, select this
   agent. Local address `127.0.0.1`, local port `25565` (the default).
   Copy the public address it shows.

6. **Verify the agent is running and enabled at boot:**
   ```bash
   systemctl status playit
   systemctl is-enabled playit
   journalctl -u playit --no-pager -n 30   # agent logs; check no secret is echoed before sharing output
   ```

7. **Test from an external network** (mobile data, not LAN wifi) using the
   public address in the Minecraft server list, with a whitelisted account.
   Also test with a *non-whitelisted* account and confirm it is rejected.

8. **Update docs:** refresh Status above with the actual free-tier limits and
   the fact that the tunnel is live (do **not** record the secret; the public
   address is fine to record or omit per preference), add a
   `docs/changelog.md` entry, remove the `vps-relay` row from AGENTS.md's
   External Infrastructure table (or mark it shelved), and mark
   `docs/setup/vps-relay.md` as shelved at the top.

### Operations

- Restart the agent: `systemctl restart playit`
- Logs: `journalctl -u playit -f`
- Rotate/revoke: delete the agent in the playit dashboard, then re-run
  `playit setup` on the LXC to claim a fresh one.
- The tunnel only carries traffic while both the agent and
  `minecraft.service` are running. LAN players connect directly to
  `192.168.2.13:25565` and are unaffected by playit outages.

---

## Setup Plan (remaining)

> Intended to be executed by a coding agent with SSH access to this LXC.
> (Currently trialing Gemini 3.7 Flash via OpenCode with "low" reasoning. This choice differs from what's in [`docs/plan/ai-model-costs.md`](../plan/ai-model-costs.md), which is two months out of date as of 2026-08-14.)
> Written as instructions for that agent — keep this section updated as ground truth once execution starts.

### ⚠️ Java version — decide before installing

Minecraft moved to calendar versioning in 2026 (current release: **26.1**).
**Current Java Edition server releases (26.1+) require Java 25**, not Java
21 — many guides still in circulation (including some from mid-2026)
incorrectly cite Java 21 and will produce `UnsupportedClassVersionError`
against a current server jar. Java 21 is still correct **only** if
deliberately running an older server line (1.20.5–1.21.x).

| Path | Minecraft version | Java | apt package |
|---|---|---|---|
| **A — latest vanilla (assumed default below)** | 26.1 | 25 | `openjdk-25-jre-headless` |
| **B — older / mod-friendly line** | 1.21.x | 21 | `openjdk-21-jre-headless` |

Debian 13 ships both `openjdk-25-jdk` and `openjdk-21-jdk` in its default
repos — no external repository needed for either path. `-headless` variants
are sufficient; no GUI dependencies needed for a dedicated server.

If going modded (Paper/Fabric/Forge/NeoForge) instead of vanilla, check that
loader's supported Java version first — support for the Java 25 line may
lag vanilla, since 26.1 is very recent.

### Steps

1. **Install Java** (pick per the table above):
```bash
   apt update
   apt install -y openjdk-25-jre-headless   # or openjdk-21-jre-headless for path B
   java -version   # confirm it reports the expected major version
```

2. **Create a dedicated service user** (mirrors the git LXC's non-root
   service-user pattern):
```bash
   useradd -r -m -d /opt/minecraft -s /usr/sbin/nologin minecraft
   chown minecraft:minecraft /opt/minecraft
```

3. **Download the server jar** from the official version manifest at
   https://www.minecraft.net/en-us/download/server — get the exact link
   for the chosen version from there; don't guess a URL, they change per
   release.
```bash
   sudo -u minecraft -i
   cd /opt/minecraft
   wget <official-server-jar-url> -O server.jar
```

4. **Accept the EULA / first run:**
```bash
   java -jar server.jar nogui   # exits immediately, writes eula.txt
   sed -i 's/eula=false/eula=true/' eula.txt
```

5. **Configure `server.properties`:** `server-port=25565` (fine for
   LAN-only), `motd`, `difficulty`, `gamemode`, `max-players` to taste;
   leave `online-mode=true` unless you have a specific reason not to.

6. **systemd unit** at `/etc/systemd/system/minecraft.service`:
```ini
   [Unit]
   Description=Minecraft Server
   After=network.target

   [Service]
   WorkingDirectory=/opt/minecraft
   User=minecraft
   ExecStart=/usr/bin/java -Xms6G -Xmx6G -jar server.jar nogui
   Restart=on-failure
   RestartSec=10
   TimeoutStopSec=60

   [Install]
   WantedBy=multi-user.target
```
   `-Xmx6G` leaves ~2GB of the 8GB allocation for OS/JVM overhead outside
   the heap. `TimeoutStopSec=60` gives the server time to save the world
   cleanly on stop rather than being killed mid-write.
```bash
   systemctl daemon-reload
   systemctl enable --now minecraft
   journalctl -u minecraft -f   # watch for the "Done" startup message
```

7. **Test from a LAN client:** connect to `192.168.2.13:25565`. No Caddy
   or firewall changes needed — LAN-to-LAN traffic on an already-trusted
   subnet.

8. **Update docs:** refresh the Status section above, add a
   `docs/changelog.md` entry, and update the inventory/service tables in
   `AGENTS.md`.

---

## Notes

- No Caddy entry — Minecraft's protocol isn't HTTP.
- External access: see "External Access (Internet)" above — playit.gg tunnel,
  not yet implemented. Earlier VPS+FRP plan shelved (`docs/setup/vps-relay.md`).
- No automated backups yet; covered by the manual `vzdump` procedure in
  `docs/plan/proxmox-manual-backup.md` until restic + rsync.net lands.
