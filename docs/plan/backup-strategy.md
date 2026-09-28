# Backup & Recovery Strategy — Planning Doc

> **Status:** Planning — no deployment yet. Decisions below are architectural; execution instructions to follow in a dedicated session.
> **Last updated:** 2026-09-27

---

## Context / Goal

Motivation, sharpened by the parallel decision in `docs/plan/agent-privilege.md` to give an agent more direct (if scoped) access to Proxmox host operations: **effortless backup and restoration of the server, so a mistake — human or agent — is a quick rollback, not an incident.**

`docs/plan/proxmox-manual-backup.md` documents today's stopgap (manual `vzdump` to local disk — no scheduling, no retention, no verification, no offsite copy). This doc covers the intended upgrade path. Nothing here is final, and nothing here should be executed without a follow-up session; the goal right now is to have the shape of the plan right, not the commands.

## Two separate problems — they don't share an answer

1. **Guest backup** (VMs and LXCs) — well-served by **Proxmox Backup Server (PBS)**, regardless of host filesystem.
2. **Host-level backup** (the Proxmox hypervisor's own OS, `/etc/pve`, network config, installed drivers) — PBS does not address this at all. The right answer here depends on the host's root filesystem, confirmed as **ext4-on-LVM** (`pve-root`, via `findmnt /`) — not ZFS.

## Proxmox Backup Server (guest-level)

PBS isn't a from-scratch backup engine — it's still `vzdump` underneath, pointed at a smarter target. What it adds over the current manual `.tar.zst` approach:

- **Real incremental backups**, and this part is confirmed independent of the host filesystem type:
  - VMs: QEMU's own dirty-bitmap tracking, which lives in the VM's RAM — a shutdown drops it, and the next backup re-reads the full disk (still deduplicated against existing PBS chunks, just slower to compute)
  - LXCs: content-based chunking of a streamed `pxar` archive — not sensitive to the container's power state
- **Scheduling and retention policies** — closes the biggest gaps in the current manual doc
- **Backup verification**
- **File-level restore** — pull one file out of a backed-up VM/CT without a full restore

None of this requires ZFS. PBS's datastore is a self-managed, content-addressable chunk store that works on top of any filesystem; ZFS underneath a PBS datastore is a nice-to-have (checksumming catches bitrot in the backup archive itself) but not a requirement.

**Placement matters independent of everything above:** the PBS datastore should live on a **second, physically separate disk** from whatever's being backed up. A backup on the same single 466GB SSD it's protecting against protects against an agent or human mistake but not against that disk failing outright. Acquiring a second disk is a reasonable, low-regret first step regardless of how the rest of this plan lands.

## Host-level backup (the harder problem)

PBS has no equivalent for the hypervisor's own root filesystem. Options, by filesystem:

- **ext4-on-LVM (current state):**
  - A plain LVM snapshot of `pve-root` is possible, but needs snapshot space pre-reserved in advance, and restoring generally means booting into a rescue context — not a live, one-command rollback.
  - Full-disk imaging (Clonezilla, `dd`) is cruder and needs the host offline for a consistent image — realistic as an infrequent full backup, not a per-session habit.
  - What's already informally covered: `etckeeper`'s git history for `/etc` config drift, plus this repo's own documentation discipline, thorough enough to manually rebuild host package/driver state if it came to that — slow, not a rollback, but not starting from zero either.
- **ZFS root** would give native `zfs snapshot` / `zfs rollback` — fast, cheap, whole-host undo, entirely independent of PBS. This is the actual reason ZFS keeps coming up in this context. Getting it means reformatting the host's root filesystem — a real migration, not a toggle — and note that ZFS is *not* Proxmox's installer default for reasons of its own (ZFS's ARC cache wants real RAM to be worth anything, a single disk with no redundancy forfeits ZFS's self-healing benefit and keeps only its checksumming/snapshot benefits, and copy-on-write plus checksums generally means more writes than ext4 on a consumer SATA SSD without power-loss protection).

**Decision: deferred.** Staying on ext4 for now; no fast host-level rollback mechanism exists in the interim beyond what's listed above. The ZFS migration question should be revisited in the dedicated backup session as its own explicit yes/no, not as a side effect of anything else.

## Where restic fits

Restic isn't a competitor to PBS — it's the other tier. PBS is local, fast, Proxmox-native, and guest-aware (it understands what a VM is). Restic is a general-purpose encrypted backup tool with no concept of a VM at all — it ships file trees to remote storage with client-side encryption before anything leaves the LAN, the same posture as the `git-remote-gcrypt` setup.

Intended layering:

- **Tier 1 (local, fast):** PBS, on its own separate disk
- **Tier 2 (offsite, disaster-grade):** restic → rsync.net, targeting PBS's own datastore directory as the backup source (it's just a tree of content-addressed chunk files — restic's ordinary use case)

Recovering from a total local loss under this model: restore the PBS datastore from rsync.net, stand PBS back up, restore guests from it normally. Two steps, nothing exotic.

Proxmox also has a native PBS-to-remote-PBS sync as an alternative offsite mechanism, but it needs a second full PBS instance with real storage somewhere — heavier than pointing restic at rsync.net, which is already the preferred provider from earlier planning (see `docs/plan/proxmox-manual-backup.md`).

## Open questions / next steps (for the dedicated backup session)

- ZFS root migration: yes/no, and when
- Acquire a second physical disk; decide PBS datastore placement
- PBS deployment location (its own LXC/VM, or elsewhere)
- restic scheduling, pruning policy, and key handling
- Whether `docs/plan/proxmox-manual-backup.md` gets folded into this doc once PBS actually replaces it, or stays as historical record of the interim procedure

## References

- `docs/plan/proxmox-manual-backup.md` — current interim manual procedure; superseded in spirit by this doc, not yet in practice
- `docs/plan/agent-privilege.md` — the blast-radius risk this backup strategy exists as a safety net for
