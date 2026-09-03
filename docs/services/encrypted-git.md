# Encrypted Git (git-remote-gcrypt)

Opaque ciphertext storage backend for git repos too sensitive for the
plaintext LAN-only git server ([`docs/services/git.md`](git.md)), even though
that server is itself LAN-only and SSH-gated. This is the "Tier 2" half of a
two-tier model — see [`docs/plan/encrypted-git.md`](../plan/encrypted-git.md)
for the full rationale and tool comparison (FOKS, git-crypt, git-secret,
age-based tools) that led here.

**The core property this design gives you:** this server never sees
plaintext. Encryption happens entirely client-side, before anything crosses
the network. Whoever has SSH access to this LXC — a family member, a future
homelab-admin LLM agent, an attacker who roots the box — only ever sees
GPG-encrypted blobs with content-hash filenames. There is no cleartext
window to protect, so (unlike the original mount/unmount-toggle idea this
design replaced) access control to the host is not load-bearing for
confidentiality here.

---

## Infrastructure

| Item     | Value                                          |
| -------- | ----------------------------------------------- |
| LXC ID   | 114                                              |
| Hostname | `encrypted-git`                                  |
| IP       | `192.168.2.14`                                   |
| OS       | Debian 13                                        |
| RAM      | 512 MB                                           |
| Disk     | 8 GB                                             |
| Type     | Unprivileged, `nesting=1`                        |
| SSH host | `encrypted-git.home.arpa` (see note below on why the exact hostname string matters) |

This LXC runs **no git software whatsoever** — not git itself, not
`git-shell`. It only needs `openssh-server` and `rsync`. The remote is,
literally, an authenticated directory that receives files.

---

## Server setup

### Dedicated service account

Parallel to the `git` user on the plaintext server, but restricted to rsync
transfers only — there is no reason this account needs a usable interactive
shell for its actual purpose (see the shell gotcha below for why it still
needs *a* shell).

```bash
useradd -r -m -d /srv/gcrypt -s /bin/sh gcrypt
mkdir -p /srv/gcrypt/.ssh
chmod 700 /srv/gcrypt/.ssh
```

**Why `-s /bin/sh` and not `/bin/false` or `/usr/sbin/nologin`:** even with a
forced `command=` in `authorized_keys` (below), `sshd` still executes that
command via the account's configured login shell — effectively
`<shell> -c "<forced command>"`. A shell like `/bin/false` exits immediately
without ever running its argument, so the forced command silently never
executes and the connection just closes. This doesn't weaken the access
restriction — `no-pty` plus the forced command already fully constrain what
the session can do — the shell field is just plumbing `sshd` needs in order
to invoke anything at all, forced or not.

### Restricting SSH to rsync-only (`rrsync`)

Debian 13 ships `rrsync` pre-built at `/usr/bin/rrsync` — no separate install
or `cp` step needed. (Older guides reference copying it into place from
`/usr/share/rsync/scripts/rrsync`; that path doesn't exist on Debian 13's
`rsync` package. Check with `dpkg -L rsync | grep rrsync` if this ever
changes on a future Debian release.)

Add each authorized client key to `/srv/gcrypt/.ssh/authorized_keys`,
wrapped with a forced command scoping it to `/srv/gcrypt` and disabling
everything else the session doesn't need:

```
command="/usr/bin/rrsync /srv/gcrypt",no-pty,no-agent-forwarding,no-X11-forwarding,no-port-forwarding ssh-ed25519 AAAA... <client-key-comment>
```

```bash
chown -R gcrypt:gcrypt /srv/gcrypt
chmod 600 /srv/gcrypt/.ssh/authorized_keys
```

Even if a client key or client machine were compromised, the blast radius is
"can read/write ciphertext files under `/srv/gcrypt`," not "has a shell on
this box."

### Keys currently authorized

| Key | Purpose |
|-----|---------|
| `ai_homelab` | Claude Code automated access |
| `human_homelab` | Human access from primary machine |
| per-AppVM key | The LAN-only Qubes AppVM that holds the GPG private key and performs actual pushes/fetches uses its own dedicated key, distinct from `human_homelab` |

`ai_homelab` is authorized here on the same basis as every other service
LXC in this homelab — confidentiality is handled by the encryption itself,
not by restricting SSH access, so there's no reason to treat this box
differently from `caddy`/`vaultwarden`/`git` on that front. See
[`docs/plan/encrypted-git.md`](../plan/encrypted-git.md) for the reasoning
that led to (and away from) tighter access control here.

---

## Client setup

Any machine that will `push`/`fetch` against a `gcrypt::` remote needs three
things: the `git-remote-gcrypt` script on `PATH`, the `rsync` binary
installed, and (for the sensitive repos this backend exists for) the GPG
private key.

### 1. Install `git-remote-gcrypt`

This tool is **not** apt-installable and is deliberately not depended on as
a live upstream package — see
[`docs/plan/encrypted-git.md`](../plan/encrypted-git.md) for why. A vetted
copy is vendored in this repo at
[`vendor/git-remote-gcrypt/`](../../vendor/git-remote-gcrypt/), with
`SOURCE.md` recording exactly which upstream commit it was pulled from.

```bash
mkdir -p ~/.local/bin
cp vendor/git-remote-gcrypt/git-remote-gcrypt ~/.local/bin/
chmod +x ~/.local/bin/git-remote-gcrypt
```

Confirm `~/.local/bin` is on `PATH`, then:

```bash
which git-remote-gcrypt
```

Git invokes it automatically as an external remote helper whenever it sees
a `gcrypt::` URL scheme — no further registration needed.

**If the client machine has no direct internet access** (e.g. a
network-restricted Qubes AppVM), fetch the script on a machine that does
have internet, then transfer it in without a network hop — on Qubes, via
`qvm-copy-to-vm <target-appvm> git-remote-gcrypt` (or the file manager's
"Copy to Other AppVM" action), landing in `QubesIncoming/<source-vm>/`.

### 2. Install `rsync` on the client

`git-remote-gcrypt`'s rsync backend shells out to the real `rsync` binary
locally, not just on the server. On a normal client this is `apt install
rsync` and done.

**On a network-restricted Qubes AppVM**, `apt` can't reach the internet
directly. Install at the **TemplateVM** level instead, so the AppVM
inherits it:

```bash
# from dom0, or a shell in the TemplateVM directly:
qvm-run -u root <template-name> "apt update && apt install -y rsync"
```

Then fully restart the AppVM — not just its applications — since Qubes
AppVMs boot from a fresh template snapshot each time:

```bash
qvm-shutdown <appvm-name>
qvm-start <appvm-name>
```

If the TemplateVM was already running, shut it down too before the AppVM
restart picks up the change. Confirm with `which rsync` / `rsync --version`
on the AppVM afterward.

If the target AppVM shares a template with other AppVMs, note that they'll
all pick up `rsync` too — harmless in practice (it's an unremarkable,
ubiquitous utility) but worth a beat of thought given how deliberately
compartmentalized this setup otherwise is.

### 3. SSH config — hostname must match exactly

SSH matches `Host` blocks in `~/.ssh/config` against the **literal hostname
string** used on the command line — and the hostname embedded in a
`gcrypt::rsync://` URL is exactly that literal string. A shorter alias
(e.g. `Host encrypted-git`) will **not** match a URL that spells out
`encrypted-git.home.arpa`.

```
Host encrypted-git.home.arpa
    User root
    IdentityFile ~/.ssh/<client-key>
```

(Client connects to the box as `root` here only for interactive
administration — the actual gcrypt transfers authenticate as the `gcrypt`
user via the key embedded in the `rsync://gcrypt@...` URL itself, which
sshd resolves through its own `authorized_keys` matching regardless of this
`Host` block's `User` line.)

### 4. GPG key

Generate a **dedicated** key for this purpose — do not reuse a personal or
communication identity:

```bash
gpg --full-generate-key
```

- Kind: ECC (sign and encrypt), Curve 25519
- Passphrase: yes — this key is operated interactively, so a passphrase is
  a real second factor here (unlike an unattended automation key)
- Real name / email: non-identifying label is fine (e.g. `homelab-gcrypt`);
  this is never published to a keyserver

Immediately after generating:

```bash
gpg --list-secret-keys --keyid-format=long   # find the KEYID / fingerprint
gpg --gen-revoke <KEYID> > gcrypt-revocation-cert.asc
gpg --export-secret-keys --armor <KEYID> > gcrypt-private-key-backup.asc
```

Back up **both files to durable storage outside the originating machine**
before any real (non-test) data goes through this pipeline. There is no
password-reset equivalent for a lost private key — losing the only copy
means every repo encrypted to it is permanently unrecoverable. The
revocation cert doesn't help with recovery either; it only lets you declare
the key dead going forward.

Set the default recipient for every gcrypt push from this client:

```bash
git config --global gcrypt.participants "<key-fingerprint>"
```

---

## Basic operations

### Adding a new repo

**On the server**, as root:

```bash
mkdir -p /srv/gcrypt/<reponame>
chown gcrypt:gcrypt /srv/gcrypt/<reponame>
```

**On the client:**

```bash
git remote add origin "gcrypt::rsync://gcrypt@encrypted-git.home.arpa/<reponame>"
git push origin master
```

**Path gotcha:** the path in the URL is relative to `rrsync`'s configured
root (`/srv/gcrypt`), **not** the real absolute server path. Use
`gcrypt::rsync://gcrypt@encrypted-git.home.arpa/<reponame>` — never
`.../srv/gcrypt/<reponame>`, which double-prepends the root
(`rrsync` adds its root to whatever path the client sends) and fails with
something like:

```
rsync: [Receiver] change_dir#3 "/srv/gcrypt/srv/gcrypt" failed: No such file or directory (2)
```

### Cloning / fetching an existing repo

```bash
git clone "gcrypt::rsync://gcrypt@encrypted-git.home.arpa/<reponame>"
```

Works from any client machine with the vendored script, `rsync`, and (for
repos using the default participant config) the GPG private key installed
— multi-client use is a designed gcrypt use case, not a workaround.

### Deleting a repo

There's no gcrypt-native delete operation — the remote is just a directory
of opaque files. On the server, as root:

```bash
rm -rf /srv/gcrypt/<reponame>
```

Permanent, no soft-delete. Any client still holding a local clone is
unaffected until its next fetch/push attempt against that remote, which
will then fail as if the repo never existed.

### Inspecting what's actually stored (sanity check)

Every file gcrypt writes — the manifest and each packfile — is named by
content hash, not a fixed filename like `manifest`. List what's there and
inspect any of them:

```bash
ssh root@encrypted-git.home.arpa "ls -al /srv/gcrypt/<reponame>/"
ssh root@encrypted-git.home.arpa "cat /srv/gcrypt/<reponame>/<hash>" | head -c 200
```

Should print binary/base64-looking noise — no filenames, no recognizable
git object headers, no plaintext content.

---

## Repacking behavior

`git-remote-gcrypt` periodically consolidates accumulated packfiles: roughly
every 25 pushes (a fixed, hardcoded pack-count threshold in the script, not
size- or time-based), it downloads and decrypts everything accumulated so
far, repacks it together with the new push's objects into one fresh pack,
re-encrypts, uploads, and deletes the old packfiles. Expect a bandwidth
spike on that push — roughly double the size of an equivalent fresh clone
(a full download plus a full upload), not a bug. See
[`docs/plan/encrypted-git.md`](../plan/encrypted-git.md) for the full
mechanics and sizing reasoning.

---

## Security notes

- Server never has access to plaintext — encryption is entirely
  client-side. SSH/host access to this LXC does not grant access to
  repo contents.
- `gcrypt` account is restricted to `rrsync` scoped to `/srv/gcrypt` via a
  forced SSH command; no interactive use of that account is possible even
  though it technically has a shell (see the shell gotcha above for why
  that shell exists at all).
- Password authentication is disabled (`PasswordAuthentication no`),
  matching every other LXC in this homelab.
- The one real single point of failure is the GPG private key: back it up
  outside its originating machine immediately after generation (see Client
  setup above). Rotating or re-homing the key across clients is cheap by
  comparison — see `docs/plan/encrypted-git.md` for why.
- Offsite backup of `/srv/gcrypt` itself is not yet configured — same
  restic + rsync.net upgrade path noted for the plaintext git server
  applies here, and is arguably more valuable here precisely because this
  is the one copy of the sensitive data outside the source AppVM.

---

## Installed packages

| Package | Version |
|---------|---------|
| openssh-server | (Debian 13 default) |
| rsync | 3.4.1 (Debian 13 default) |
