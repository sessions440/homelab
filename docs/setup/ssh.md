# SSH Setup

## Key strategy

Two separate keys serve distinct purposes:

| Key | Purpose | Passphrase |
|-----|---------|-----------|
| `~/.ssh/ai_homelab` | Claude Code automated access (root on service LXCs) | None (automation key) |
| `~/.ssh/human_homelab` | Human interactive access (root on service LXCs, git user on git LXC) | Recommended |

Keeping them separate means Claude Code access can be revoked independently of your personal access, and vice versa.

## Generating a key

```bash
ssh-keygen -t ed25519 -C "<label>" -f ~/.ssh/<keyname>
```

This produces `~/.ssh/<keyname>` (private) and `~/.ssh/<keyname>.pub` (public). Never share or commit the private key.

## SSH config (~/.ssh/config)

The `Host` aliases are reserved for human interactive access — they let you type `ssh caddy` instead of `ssh -i ~/.ssh/... root@192.168.2.3`. Claude Code uses IP addresses with an explicit `-i ~/.ssh/ai_homelab` flag and does not need aliases.

```
# Bare hostnames reserved for human interactive access (User root).
# Claude Code uses IPs with explicit -i ~/.ssh/ai_homelab flag; no aliases needed.

Host caddy
    HostName 192.168.2.3
    User root
    IdentityFile ~/.ssh/human_homelab
    ServerAliveInterval 60

Host vaultwarden
    HostName 192.168.2.11
    User root
    IdentityFile ~/.ssh/human_homelab

Host git
    HostName 192.168.2.12
    User root
    IdentityFile ~/.ssh/human_homelab

# git remote URLs use git@192.168.2.12 (not the Host alias above, which is
# User root). This block ensures the correct key is used for those connections.
Host 192.168.2.12
    IdentityFile ~/.ssh/human_homelab
```

Add a new `Host` block for each container when it's provisioned. Never add a block for `192.168.2.2` (Proxmox host).

The `git` *user* on the git LXC (`git@192.168.2.12`) requires a separate `Host 192.168.2.12` block because SSH matches on the literal string in the URL — the `Host git` alias (which is `User root`) does not match. The IP block has no `User` directive so git can supply the user via the remote URL (see [docs/services/git.md](../services/git.md)).

## Conventions

- **Password authentication is disabled** (`PasswordAuthentication no` in `/etc/ssh/sshd_config`) on all service LXCs. SSH key access only.
- Claude Code uses `ai_homelab` for all automated SSH operations.
- Human interactive sessions use `human_homelab`.
- New containers get both keys added to `/root/.ssh/authorized_keys` at provisioning time.

## Adding keys to a new container

At provisioning, add both public keys and disable password auth:

```bash
# Add both keys
ssh-copy-id -i ~/.ssh/ai_homelab.pub root@<ip>
ssh-copy-id -i ~/.ssh/human_homelab.pub root@<ip>

# Disable password auth
ssh root@<ip> "sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config && systemctl restart ssh"
```

`ssh-copy-id` handles creating `~/.ssh/` and `authorized_keys` with correct permissions if they don't exist. If the container doesn't yet have your key (first login), add `-o PasswordAuthentication=yes` to fall back to the password for that one command.

## Authorizing a key on an existing container

```bash
cat ~/.ssh/<keyname>.pub | ssh <host> "cat >> ~/.ssh/authorized_keys"
```

For containers where the target user is not root, replace `~` with the user's home path, e.g. `/srv/git` for the `git` user on the git LXC.

## Revoking a key

Open `authorized_keys` on the target container and delete the relevant line:

```bash
ssh <host> "nano /root/.ssh/authorized_keys"
```

Each line in `authorized_keys` ends with the comment field from when the key was generated (e.g. `ai_homelab` or `human_homelab`). That comment is how you identify which line to delete.

To view all currently authorized keys on a container:

```bash
ssh <host> "cat /root/.ssh/authorized_keys"
```

## Renaming or replacing a key

SSH keys can't be renamed in the cryptographic sense — the public key identity is fixed. "Renaming" means:

1. Generate a new key with the new name/label.
2. Authorize the new key on all relevant containers (see above).
3. Update `~/.ssh/config` to point `IdentityFile` at the new key.
4. Revoke the old key from all containers (see above).
5. Delete the old private key file locally.

There is no shortcut — the old and new keys must coexist on the containers during the transition period so you don't lock yourself out.

## Key management at scale

At ~4–5 containers with infrequent provisioning, manual management is straightforward. A helper script (`scripts/sync-authorized-keys.sh`) may be worth writing when the container count grows or key changes become frequent. Ansible's `ansible.posix.authorized_key` module is the right tool if full configuration management is ever adopted.

## Jump box / bastion assessment

A bastion host (single SSH entry point that proxies into service containers) is standard practice for large fleets or internet-exposed infrastructure. For this homelab it is overkill:

- All containers are on a trusted LAN (`192.168.2.0/24`)
- No SSH ports are forwarded from the internet (CGNAT + no port forwards)
- The git LXC requires direct SSH access from every client machine for git operations, so a bastion would require a `ProxyJump` directive in every client's `~/.ssh/config` — complexity with no real gain

The meaningful security wins here are key-only auth (done) and never forwarding SSH to the internet (architectural guarantee via CGNAT).
