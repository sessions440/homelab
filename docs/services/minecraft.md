# Minecraft Server

Self-hosted Minecraft Java Edition server for LAN-only play. Internet exposure
(via WireGuard or a tunnel) is planned for later, once this is stable.

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

- **2026-08-14:** Java 25 (`openjdk-25-jre-headless`) installed. Dedicated user `minecraft` created with home `/opt/minecraft`. Minecraft Vanilla Server 26.2 downloaded, EULA accepted, and configured as a systemd service (`minecraft.service`). Verified active and listening on port `25565`.
- **2026-08-13:** LXC provisioned. SSH keys (`ai_homelab`, `human_homelab`)
  authorized; password authentication disabled.

---

## External Access (Internet)

**Status:** Decision made — VPS+FRP. Not yet implemented (blocked on manual
VPS provisioning, see `docs/setup/vps-relay.md`).

### Decision

Goal: let a whitelisted set of players outside the LAN connect alongside LAN
players, without granting them any broader network access — this rules out
WireGuard-into-LAN, which is the right tool for personal remote access but
the wrong tool for handing out to game-server guests (it would give every
player network-level presence on the same subnet as Vaultwarden, the git
server, etc.).

Two relay approaches were compared:

- **playit.gg** — turnkey third-party tunnel service purpose-built for game
  hosting. Genuinely solid option: zero install for connecting players (they
  enter the assigned address directly into vanilla Minecraft's server list —
  no client software needed on their end), generous free tier, open-source
  host-side agent (note: the agent is open source, the relay backend itself
  is not). Considered and rejected in favor of the VPS route below, primarily
  because it doesn't generalize to future non-Minecraft services and keeps a
  third party in the data path.
- **Self-hosted VPS + FRP** (chosen) — an Oracle Cloud Always Free VPS relays
  TCP:25565 back to this LXC over an FRP tunnel. Zero recurring cost, becomes
  reusable infrastructure for any future internet-facing service, and is a
  deliberate upskilling opportunity. Full rationale and generic setup steps:
  **[`docs/setup/vps-relay.md`](../setup/vps-relay.md)**.

Latency between the two options was assessed as roughly a wash for Minecraft
specifically — a well-chosen VPS region performs comparably to playit's
network, and Minecraft's server-authoritative movement tolerates the
20–50ms range either approach implies without being disqualifying.

### Setup Plan

> Written as instructions for a coding agent with SSH access to this LXC and
> to the VPS provisioned per `docs/setup/vps-relay.md`. Assumes the VPS
> already exists, both firewall layers are open for TCP 25565, and `frps` is
> running (see that doc's Parts 1–3). Keep this section updated as ground
> truth once execution starts, matching the pattern used for the original
> server install below.

1. Install `frpc` on this LXC following `docs/setup/vps-relay.md` Part 3
   ("On the home LXC").

2. Add a Minecraft-specific proxy block to `/etc/frp/frpc.toml`:
   ```toml
   [[proxies]]
   name = "minecraft"
   type = "tcp"
   localIP = "127.0.0.1"
   localPort = 25565
   remotePort = 25565
   ```

3. Enable and start `frpc`:
   ```bash
   systemctl daemon-reload
   systemctl enable --now frpc
   systemctl status frpc
   ```

4. **DNS record needed (human confirmation required — Cloudflare changes are
   not agent-executable per AGENTS.md hard limits):** an **A record**,
   `minecraft.{env.HOMELAB_DOMAIN}` → `<VPS_PUBLIC_IP>`, **DNS-only (grey
   cloud)**, not proxied. Unlike the HTTP-based services behind Caddy,
   Cloudflare's proxy only terminates HTTP(S) traffic — a proxied record
   would break a raw TCP game connection. This is also the one record in this
   homelab that's *intentionally* publicly resolvable and connectable: the
   "security through inaccessibility" model used for other internal
   subdomains doesn't apply here by design. The whitelist below and this
   VPS's patch discipline carry the security load for this service instead.

5. **Whitelist enforcement** (do this regardless of, and ideally before, step 4):
   ```
   # in the Minecraft console, or via RCON if configured
   whitelist add <username>
   ```
   Also confirm `server.properties` has:
   ```
   white-list=true
   enforce-whitelist=true
   ```

6. Test from an external network (mobile data, not LAN wifi) connecting to
   `minecraft.<domain>:25565`.

7. Update this doc's Status section, add a `docs/changelog.md` entry, and
   update `AGENTS.md`'s inventory (see proposed "External Infrastructure"
   table addition raised alongside this doc).

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
- External access: see "External Access (Internet)" section above — decision
  made for VPS+FRP, not yet implemented.
- No automated backups yet; covered by the manual `vzdump` procedure in
  `docs/plan/proxmox-manual-backup.md` until restic + rsync.net lands.
