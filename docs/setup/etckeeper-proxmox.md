# etckeeper on Proxmox

etckeeper puts `/etc` under git version control, with apt hooks that automatically
commit before and after every package operation. On Proxmox there is one important
setup quirk to handle before the tool works cleanly.

---

## Installation

```bash
apt install etckeeper
```

On a Debian-based system, installation automatically runs `etckeeper init` and makes
an initial commit containing all of `/etc` — including `/etc/pve`, which causes a
problem (see below).

---

## The `/etc/pve` problem

`/etc/pve` is not a real directory. It is a FUSE-based virtual filesystem (`pmxcfs`)
backed by a SQLite database at `/var/lib/pve-cluster/config.db`, replicated in real
time across cluster nodes. Because it is a FUSE mount:

- `chmod` on it fails with `Operation not permitted`
- Files inside it (e.g. `.rrd` round-robin databases) change constantly, causing
  etckeeper to block every subsequent `apt` operation with an "uncommitted changes"
  error

`/etc/pve` does not need to be tracked by etckeeper. Its backing store is covered by
`vzdump` and Proxmox's own replication.

### Fix

**Step 1 — exclude from git:**

Add `pve/` to `/etc/.gitignore`, *outside* the managed-by-etckeeper comment blocks:

```
# end section managed by etckeeper

pve/
```

**Step 2 — remove already-indexed contents:**

Because the initial auto-commit already added `pve/` to the git index, the `.gitignore`
alone is not enough. You must also untrack what was already staged:

```bash
cd /etc
git rm -r --cached pve/
git commit -m "untrack /etc/pve (pmxcfs FUSE mount, not a real directory)"
```

Verify the fix:

```bash
git status   # pve/ should not appear
```

---

## Workflow

### Before a major change

etckeeper's apt hooks auto-commit before and after every package install, but a
named manual commit gives a cleaner reference point:

```bash
etckeeper commit "pre-Nvidia driver install baseline"
```

### After a major change

```bash
cd /etc
git log --oneline -10          # review auto-commits from apt hooks
git diff HEAD~2 HEAD           # what changed across last two commits
git diff <hash1> <hash2>       # diff any two specific commits
```

### Manual config edits

etckeeper does not auto-commit manual file edits — only package operations trigger
the hooks. Commit manually after editing configs:

```bash
etckeeper commit "harden sshd: disable password auth"
# or use git directly
git -C /etc commit -am "description"
```

### Useful shorthands

```bash
etckeeper vcs diff             # runs git diff in /etc
etckeeper vcs log --oneline    # runs git log in /etc
```

---

## Security note

`/etc` contains sensitive files (`shadow`, private keys, API tokens in config files).
If you ever push this repo to a remote, use a private repository and audit what is
tracked with `git status` before pushing. The `PUSH_REMOTE` option in
`/etc/etckeeper/etckeeper.conf` can automate pushes after every commit if desired.

---

## Key files

| Path | Purpose |
|---|---|
| `/etc/etckeeper/etckeeper.conf` | Main config (VCS choice, auto-commit options) |
| `/etc/.gitignore` | Files/dirs excluded from tracking |
| `/etc/.etckeeper` | Metadata file tracking permissions/ownership git cannot store |
| `/var/lib/pve-cluster/config.db` | The actual pmxcfs backing store (not tracked by etckeeper) |
