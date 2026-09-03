# Encrypted Git — Planning & Decision Log

> **Status:** Implemented and validated end-to-end (test repo push/clone/inspection all confirmed).
> **Last updated:** 2026-09-01

---

## Context / Goal

The plaintext git server (`docs/services/git.md`, `192.168.2.12`) is
LAN-only, SSH-gated, and cleartext on disk — an accepted threat model for
most repos, but not comfortable for data sensitive enough to warrant
protection even from other things with LAN/SSH-level access: other family
members on the network, and any future LLM agent that administers this
homelab (an anticipated, not hypothetical, concern — this repo is already
partly agent-administered).

Two additional concerns shaped the requirement:

- **Expanding attack surface:** internet-facing services planned elsewhere
  in this homelab (VPS+FRP for Minecraft, possible future WireGuard remote
  access) increase the chance of *some* LXC or the Proxmox host being
  compromised, which could enable lateral movement to the git LXC even
  though it isn't directly internet-facing.
- **Accidental exposure control:** wanting a structural guarantee that
  sensitive repo content isn't legible to anything with SSH/host access,
  rather than relying on discipline about who's granted that access and
  when.

An early idea was a mount/unmount toggle — keep sensitive data on a
separate volume, unmounted except during active use, combined with
selectively revoking LLM-agent SSH access while it's mounted. This was
**superseded by choosing client-side encryption instead**: if the server
never has plaintext to begin with, there's no window to protect and no
manual lock/unlock discipline to maintain. It's also a strictly stronger
guarantee — safe even against root-level compromise of the LXC, not just
against an SSH session during a permissive window. See "Security
architecture" below for the resulting two-tier model.

---

## Tools evaluated

### FOKS (foks.pub)

Founded by Keybase.io co-founders, MIT licensed, post-quantum crypto
(Curve25519 + ML-KEM). Federated user@host identity model with DNS-verified
host chains — federation here is about identity/key distribution, not git
object transfer, analogous to SMTP servers routing mail rather than bytes.
Client-side packfile creation preserves per-push delta compression (the
whitepaper explicitly addresses and corrects an assessment that wrongly
assumed no compression at all); remaining inefficiency is the lack of
server-side repacking across pushes, plus an upfront packfile-index fetch
cost that scales with repo history.

Server footprint is heavy: 11 separate processes (probe, reg, user,
merkle_query, kv_store, beacon, merkle_batcher, merkle_builder,
merkle_signer, queue, internal_ca) plus PostgreSQL, abstracted by
`foks-tool standup` + Docker Compose but still a real resource commitment.

**Not chosen:** the federation/post-quantum crypto investment doesn't pay
for itself for a single-user homelab with no near-term need to share
repos across independent servers.

### git-crypt / git-secret

Transparent (git-crypt) or manual (git-secret) per-file content encryption
via smudge/clean filters or explicit hide/reveal commands. Both explicitly
**not** designed for whole-repo encryption — filenames, commit messages,
and repo structure stay plaintext, and git-crypt's own documentation
recommends git-remote-gcrypt for whole-repo needs. Full native git
performance is preserved since only blob *contents* are touched, but this
doesn't match the actual requirement here (protecting more than just file
contents).

**Not chosen:** wrong tool for whole-repo confidentiality.

### age-based alternatives (git-agecrypt, agebox)

Investigated specifically to avoid GPG's UX. `git-agecrypt` is the
age-based equivalent of git-crypt but its own author explicitly disclaims
production use pending an audit. `agebox` is oriented at GitOps
secrets-in-CI workflows, not whole-repo private use, and is manual rather
than transparent.

**Not chosen:** no production-trustworthy age-based whole-repo tool exists
yet. GPG's UX friction was accepted as the cost of the only mature option.

### git-remote-gcrypt — chosen

Git remote helper that treats the remote as opaque GPG-encrypted blob
storage. The local repo is completely normal git (plaintext, full
diff/log/history) — encryption happens only at the push/fetch boundary.
Backends: local, `rsync://`, `sftp://`, arbitrary git URL (via a
`gitception` layer), experimental `rclone://`.

Critical distinction confirmed by reading the source directly (not just
the docs): the `rsync://` backend supports genuinely incremental transfer
for ordinary pushes, while the arbitrary-giturl and `sftp://` backends
re-upload the **entire history on every push** — explicitly documented as
suitable only for small repos. This ruled out anything but the `rsync://`
backend for anything beyond trivial credential-store-sized repos.

For a home LAN target, `rsync://` over SSH to a plain directory needs
**no git-aware software on the server at all** — see
`docs/services/encrypted-git.md`.

---

## Maintenance-risk assessment

Before committing, checked whether the project is actually maintained,
since the GitHub mirror (`github.com/spwhitton/git-remote-gcrypt`) showed
no commits in roughly two years at the time of review:

- The canonical repository is `git.spwhitton.name`, maintained by Sean
  Whitton (a long-standing Debian Developer); GitHub is only a mirror.
- The last actual upstream release was 1.5 in August 2022 — closer to four
  years stale than two, though the Debian packaging changelog shows an
  unreleased 1.6 entry, indicating some work exists but hasn't shipped.
- The Debian package tracker's `vcswatch` currently reports it cannot
  reach `git.spwhitton.name` (connection reset) to check VCS status. A
  community comment on the AUR package independently confirms the
  canonical repo has been unreachable and asks whether the project is
  still maintained.
- Despite this, it's still packaged and shipping in current Debian,
  Ubuntu, and Arch (AUR), and third-party guides referencing it are still
  being published.

**Verdict:** single-maintainer tool with real bus-factor risk and a
currently-flaky canonical repo — not actively developed, but not broken
either. The mitigating factor: it's a single ~970-line POSIX shell script
with no hidden dependency tree, genuinely readable end-to-end in an
evening. This substantially de-risks the "what if upstream disappears"
scenario compared to a typical dependency.

**Decision:** proceed, but don't depend on continued upstream/apt
availability. **Vendor a pinned copy** into this repo
(`vendor/git-remote-gcrypt/`, with `SOURCE.md` recording the exact commit
and fetch date) rather than installing it from a live package source. If a
bug or incompatibility ever surfaces, the small size of the script makes
self-patching (with coding-agent help) realistic rather than a
last resort.

---

## Repack performance — corrected model

Initial hypothesis (pre-implementation): since most commits are
append-style ("add a new file") with low internal redundancy, history size
roughly tracks repo-size-on-disk, so a worst-case repack-triggered push
should be "no worse than a fresh clone."

Reading the actual script logic corrected this in two ways:

**1. The trigger is a fixed pack *count*, not size or time.** A hardcoded
`Repack_limit=25` (not exposed via git config, only forceable via the
`GCRYPT_FULL_REPACK` env var) counts non-consolidated packfiles on the
remote. Each ordinary push adds one new packfile; once more than 25
non-kept packfiles have accumulated, the *next* push triggers a repack.
For an append-heavy, single-purpose-commit usage pattern, this means a
repack roughly every 25 pushes — predictable and infrequent, not scaling
per-push.

**2. The cost of a repack event is worse than "equivalent to one clone."**
When triggered, the script first **downloads and decrypts every
non-kept packfile currently on the remote**, extracts their object lists,
combines them with the new push's objects, repacks everything into one
fresh pack with normal git delta compression, re-encrypts with a new key,
uploads that single consolidated pack, and only then deletes the old
packfiles remotely. Data moves in **both directions** — a full download of
everything accumulated since the last repack, plus a full upload of the
new consolidated pack. That's closer to **~2x the size of an equivalent
fresh clone**, not equal to one.

The original hypothesis about history size tracking repo size on disk is
still correct and still useful — it determines how large that periodic ~2x
event actually is — but doesn't affect *frequency*, which is governed
purely by push count.

**Practical takeaway:** for modest-sized "glorified backup" repos pushed
occasionally, a ~2x-clone-sized transfer every 25 pushes is a non-issue
even on a residential uplink. This would only become a real concern for a
single repo growing into multi-gigabyte territory with frequent pushes.
Since `Repack_limit` is a plain shell variable in the (now vendored, hence
freely editable) script, it can be tuned per-repo needs later if that ever
becomes relevant.

---

## Security architecture — two-tier model

| | Tier 1 (existing) | Tier 2 (this doc) |
|---|---|---|
| Server | `git` LXC, `192.168.2.12` | `encrypted-git` LXC, `192.168.2.14` |
| Server-side content | Plaintext | Opaque GPG ciphertext only |
| Software on server | `git`, `git-shell` | `openssh-server`, `rsync` only |
| Confidentiality mechanism | LAN isolation + SSH gating | Client-side encryption (server access is irrelevant to confidentiality) |
| `ai_homelab` access | Authorized | Authorized — see below |
| Use case | Bulk / non-sensitive repos | Repos sensitive enough to want protection even from other LAN/SSH-privileged parties |

**Why a dedicated, separate LXC rather than adding gcrypt storage onto the
existing `git` LXC:** isolation from the established cleartext server (no
risk of the two tiers' concerns bleeding into each other operationally),
and reversibility — the entire tier can be deprovisioned by deleting one
LXC without touching anything else in the homelab, if git-remote-gcrypt is
ever abandoned in favor of something else.

**Why `ai_homelab` is authorized on `encrypted-git` despite the original
motivating concern being "don't expose sensitive data to an LLM agent":**
once the server is encryption-only, SSH/host access to it doesn't grant
access to anything readable — the blast radius of any compromise
(agent-driven or otherwise) is limited to ciphertext and directory
structure. Restricting `ai_homelab` specifically here, while authorizing it
identically on every other service LXC in this homelab, wasn't doing real
protective work and was inconsistent with the homelab's existing access
policy. The confidentiality guarantee comes entirely from the encryption
design above this table, not from access control to the host.

---

## Client-side key and script topology

- **GPG key** is generated and lives on a dedicated, network-restricted
  (LAN-only) Qubes AppVM — separate from the AppVM used for Claude
  Code / coding-agent work. This isn't a gcrypt requirement, but a
  deliberate compartmentalization choice consistent with this homelab's
  general Qubes boundary philosophy.
- **Re-homing or copying the key to additional clients is cheap and fully
  supported** — multi-client access to the same repos via
  `gcrypt.participants` is a designed use case, not a workaround. An
  earlier draft of this plan overstated the cost of this, conflating it
  with *key rotation*.
- **Key rotation is also cheaper than initially assumed:** the manifest is
  a fixed-name file that gets **fully overwritten on every push**, not
  versioned. Changing `gcrypt.participants` and pushing once re-encrypts
  the manifest going forward — there's no need to bulk re-encrypt
  historical packfiles.
- **The one genuine single point of failure** is losing the only copy of
  the private key — there is no recovery mechanism, unlike a password.
  Mitigated by backing up both the private key and its revocation
  certificate to durable storage outside the originating AppVM
  immediately after generation, before any real (non-test) data flows
  through the pipeline. Longer-term, this backup should fold into the
  planned restic + rsync.net offsite backup work.
- **`git-remote-gcrypt` itself is installed per-client** (copied from the
  vendored copy in this repo into `~/.local/bin` on each machine that
  needs it), not centrally — see `docs/services/encrypted-git.md` for the
  install steps, including the Qubes-specific detour needed when the
  target client has no direct internet access of its own.

---

## Execution Log

- **2026-09-01:** `encrypted-git` LXC (ID 114, `192.168.2.14`, Debian 13,
  unprivileged, `nesting=1` set at creation via the Proxmox web UI)
  provisioned. `gcrypt` service account created with `rrsync`-restricted,
  forced-command-only SSH access. Dedicated GPG key generated on the
  LAN-only AppVM, backed up (with revocation cert) to durable storage
  outside that AppVM. `git-remote-gcrypt` (commit `a5ff704...`, fetched
  2026-09-01) vendored into this repo and installed on the AppVM via a
  Qubes disposable + `qvm-copy-to-vm` (the AppVM itself has no direct
  internet access). `rsync` installed on the AppVM by way of its
  TemplateVM. End-to-end validated with a throwaway test repo: push,
  clone, and manual inspection of server-side files confirmed only
  ciphertext with content-hash filenames is ever stored. Several setup
  gotchas encountered and resolved — see `docs/troubleshooting.md`.
