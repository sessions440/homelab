# Git Server

## Overview

| Item | Value |
|------|-------|
| LXC ID | 112 |
| Hostname | `git` |
| IP | `192.168.2.12` |
| OS | Debian 13 |
| RAM | 512 MB |
| Disk | 8 GB |
| Status | Running |

LAN-only bare git remote. No web frontend. Acts as a push/pull target for
personal repos from any machine on the LAN.

## Access

```
git@192.168.2.12:/srv/git/<repo>.git
# or, if home.arpa resolves on your machine:
git@git.home.arpa:/srv/git/<repo>.git
```

Interactive SSH sessions as the `git` user are intentionally disabled —
`git-shell` is the login shell, which only accepts git wire-protocol commands.
To administer the server, SSH as root: `ssh git` (using the `git` Host alias
in `~/.ssh/config`, which connects as root).

### SSH config requirement for git remotes

The `Host git` alias in `~/.ssh/config` is `User root` — it does not apply to
`git@192.168.2.12` remote URLs. A separate block is required so SSH picks the
right key for git operations:

```
Host 192.168.2.12
    IdentityFile ~/.ssh/human_homelab
```

Without this, SSH falls back to trying `~/.ssh/id_rsa` / `~/.ssh/id_ed25519`
and the connection is refused. See [docs/setup/ssh.md](../setup/ssh.md) for the
full config.

## Server layout

```
/srv/git/                        ← git user's home directory
├── .ssh/
│   └── authorized_keys          ← client public keys for push/pull access
├── git-shell-commands/
│   └── no-interactive-login     ← friendly error for interactive login attempts
└── <repo>.git/                  ← bare repositories live here
```

## Creating a new repository

SSH into the server as root, then:

```bash
git init --bare /srv/git/<repo>.git
chown -R git:git /srv/git/<repo>.git
```

From a client machine, add the remote and push:

```bash
git remote add origin git@192.168.2.12:/srv/git/<repo>.git
git push -u origin main
```

Or clone an existing repo directly:

```bash
git clone git@192.168.2.12:/srv/git/<repo>.git
```

## Client SSH access

Client public keys are stored in `/srv/git/.ssh/authorized_keys` — one key per
line. All listed keys have push/pull access to all repos; there is no per-repo
access control.

For general guidance on authorizing, revoking, and managing SSH keys, see
[docs/setup/ssh.md](../setup/ssh.md). The only git-server-specific detail is the
target path: use `/srv/git/.ssh/authorized_keys` (not `/root/.ssh/authorized_keys`)
and ensure ownership stays correct:

```bash
chown git:git /srv/git/.ssh/authorized_keys
chmod 600 /srv/git/.ssh/authorized_keys
```

### Keys currently authorized

| Key | Purpose |
|-----|---------|
| `claude_code_homelab` | Claude Code automated access |
| `human_homelab` | Human access from primary machine |

## Security notes

- Traffic is **cleartext on the server** — acceptable for LAN-only use, SSH-gated.
- SSH is the only access vector; no web UI, no open ports beyond 22.
- Password authentication is disabled (`PasswordAuthentication no`).
- Offsite encrypted backup is not yet configured (planned separately).

## Installed packages

| Package | Version |
|---------|---------|
| git | 2.47.3 |
| openssh-server | 10.0p1 |
