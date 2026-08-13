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

- **2026-08-13:** LXC provisioned. SSH keys (`ai_homelab`, `human_homelab`)
  authorized; password authentication disabled. No Minecraft software
  installed yet.

---

## Setup Plan (remaining)

> Intended to be executed by a coding agent (trialing Gemini 2.5 Flash via
> OpenCode) with SSH access to this LXC. Written as instructions for that
> agent — keep this section updated as ground truth once execution starts.

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
- Not exposed beyond the LAN yet. Starlink's CGNAT means future internet
  exposure needs WireGuard or a tunnel service — plain port forwarding
  isn't possible.
- No automated backups yet; covered by the manual `vzdump` procedure in
  `docs/plan/proxmox-manual-backup.md` until restic + rsync.net lands.