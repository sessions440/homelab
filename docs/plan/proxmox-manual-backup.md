# Proxmox Manual Backup Procedure

A partial solution — better than nothing, worth doing before risky host-level changes (e.g. kernel module / driver installs).

---

## What to Back Up

Proxmox has no single "back up everything" tool. Two separate things need covering:

### 1. Guest backups (`vzdump`)

LXC containers and VMs. Use the web UI:

- **UI method:** Right-click any LXC/VM → *Backup Now*
- **Mode:** `snapshot` recommended for consistency
- **Output:** `.tar.zst` archives land in `/var/lib/vz/dump/` by default

Or via CLI:

```bash
vzdump <vmid> --mode snapshot --compress zstd
```

### 2. Host config (manual tar)

Proxmox doesn't back up the PVE host itself. Key directories:

| Path | Contents |
|---|---|
| `/etc/pve/` | Cluster config, VM/LXC definitions |
| `/etc/network/` | Host network config |
| `/etc/modprobe.d/` | Kernel module options (e.g. Nvidia blacklists) |
| `/etc/modules-load.d/` | Modules loaded at boot |

```bash
tar czf /var/backups/pve-host-$(date +%Y%m%d).tar.gz \
  /etc/pve \
  /etc/network \
  /etc/modprobe.d \
  /etc/modules-load.d
```

---

## Copying to Another Machine on the LAN

Since the Proxmox host's SSH key isn't authorized on other machines, it's easier to pull from the destination rather than push from the host.

On the **destination machine**:

```bash
# Pull host config tar
scp root@192.168.2.x:/var/backups/pve-host-YYYYMMDD.tar.gz /path/to/backup/

# Pull guest backups
scp root@192.168.2.x:/var/lib/vz/dump/*.tar.zst /path/to/backup/
```

Use the Proxmox root password when prompted.

---

## Restoration

**Host config:**
```bash
tar xzf pve-host-YYYYMMDD.tar.gz -C /
```

**Guest (LXC):**
```bash
pct restore <vmid> /path/to/backup.tar.zst
```

**Guest (VM):**
```bash
qmrestore /path/to/backup.vma.zst <vmid>
```

Or re-import via the web UI.

---

## Limitations of This Approach

This is a **partial solution**. Gaps vs. a complete backup service:

- **No scheduling** — must be run manually
- **No retention policy** — backups accumulate until disk fills
- **No offsite copy** — doesn't survive disk failure, fire, or theft
- **No verification** — untested backups are just hopes

The natural upgrade path is **restic + rsync.net**: adds scheduling (systemd timer), retention policies (7d/4w/6m/2y), offsite storage, and restore verification. Worth setting up eventually; this procedure is a stopgap until then.

---

## When to Run This

At minimum, before any risky host-level operation:

- Kernel/driver changes (e.g. Nvidia dkms install)
- Proxmox version upgrades
- Network config changes
- Adding/removing storage
