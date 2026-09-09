# Troubleshooting

---

## Vaultwarden: admin login rejected despite correct password (2026-06-17)

**Symptom:** `/admin` login returns "Invalid admin token" even when the correct
plaintext password is entered.

**Cause (most likely):** The password was mistyped when originally running
`vaultwarden hash` in an SSH terminal (copy-paste wasn't working at the time).
The hash in `/opt/vaultwarden/admin_token` is internally consistent but doesn't
match the intended password.

**Secondary cause:** Too many failed attempts triggers an in-memory rate-limit
lockout. A container restart clears it, but the underlying password mismatch
must still be resolved.

**Fix:** Reset the admin password — see the "Resetting the admin password"
section in `docs/services/vaultwarden.md`. When generating the new hash, paste
the password rather than typing it manually.

---

## Vaultwarden: 502 from Caddy after initial deploy (2026-06-17)

**Symptom:** `https://bitwarden.<domain>` returned HTTP 502. Vaultwarden was healthy inside its LXC but unreachable from the Caddy LXC.

**Cause:** The Docker port binding was `127.0.0.1:8080:80`, which only listens on the loopback interface inside the vaultwarden LXC. Caddy runs on a separate LXC (`192.168.2.3`) and cannot reach a loopback-only port on another host.

**Fix:** Change the binding to `8080:80` (all interfaces) in `docker-compose.yml` and recreate the container with `docker compose up -d --force-recreate`.

**Note:** `127.0.0.1:` binding is only safe when the reverse proxy and service share the same host. In a multi-LXC architecture, services must bind on all interfaces so the Caddy LXC can reach them. The LAN firewall provides the perimeter; direct exposure to the internet is not a concern.

---

## Caddy DNS-01 challenge: `connection refused` on port 53

**Symptom:** Caddy logs `read udp ...:53: read: connection refused` when
attempting cert issuance.

**Cause:** Outbound UDP/TCP port 53 to external IPs (e.g. `1.1.1.1`,
`2606:4700:58::...`) is blocked on the caddy LXC. This affects both IPv4 and
IPv6 external resolvers.

**Fix:** Set `resolvers 192.168.2.1` in the `tls` block of each Caddyfile
site. AdGuard Home at `192.168.2.1` is reachable on port 53 and forwards to
upstream resolvers.

---

## Caddy DNS-01 challenge: `timed out waiting for record to fully propagate`

**Symptom:** Caddy logs `timed out waiting for record to fully propagate; last error: <nil>`.
Cloudflare audit log shows create/delete cycles for `_acme-challenge` TXT
records — the API writes are succeeding but Caddy can't confirm propagation.

**Cause:** AdGuard Home at `192.168.2.1` does not return newly-created
Cloudflare TXT records within certmagic's propagation polling timeout. Let's
Encrypt's own resolvers can see the record fine.

**Fix:** Add `propagation_delay 2m` to the `tls` block. This skips the
local polling check entirely and waits a fixed time before signalling Let's
Encrypt to verify.

---

## sshd crashes on `systemctl reload` in Debian 13 LXC (2026-06-19)

**Symptom:** `systemctl reload ssh` returns an error; SSH connections refused immediately after. `systemctl status ssh` shows `fatal: Cannot bind any address`.

**Cause:** On Debian 13 LXCs, `reload` sends SIGHUP to the running sshd process, which re-execs itself. During re-exec, sshd expects `/run/sshd` (the privilege separation directory) to exist. In an LXC that hasn't been rebooted since the initial install, this directory may not have been created, causing the re-exec to fail and sshd to exit entirely.

**Fix:** From the Proxmox console (CT 112 → Console):

```bash
mkdir -p /run/sshd
systemctl start ssh
```

**Prevention:** Always use `systemctl restart ssh` instead of `reload` on Debian 13 LXCs. `restart` brings up a fresh process that creates `/run/sshd` itself; `reload` re-execs in-place and depends on the directory already existing.

---

## Caddy leaks secrets into systemd journal

**Symptom:** `CF_API_TOKEN` (or other env vars) visible in plaintext in
`journalctl -u caddy` output at startup.

**Cause:** The upstream Caddy systemd unit uses `--environ` in `ExecStart`,
which logs all environment variables. Once `EnvironmentFile=/etc/caddy/caddy.env`
is added via a drop-in, secrets are included in that dump.

**Fix:** Override `ExecStart` in the drop-in to remove `--environ`:

```ini
[Service]
EnvironmentFile=/etc/caddy/caddy.env
ExecStart=
ExecStart=/usr/bin/caddy run --config /etc/caddy/Caddyfile
```

The blank `ExecStart=` line is required to clear the inherited value before
setting the new one. Then clear the journal:

```bash
journalctl --rotate && journalctl --vacuum-time=1s
```

---

## Vaultwarden: blank/unresponsive vault in desktop app and browser extension, web vault unaffected (2026-08-14)

**Symptom:** On one Bitwarden client machine, the desktop app and browser
extension both show an empty vault with an unresponsive sidebar (can't
switch vaults, no items render), while the web vault at
`https://bitwarden.<domain>` works normally with all items visible.
Initially suspected as related to the `vaultwarden` LXC renumbering
(CT 110 → 111) performed the same week — the two are unrelated, see Cause.

**Cause:** Bitwarden shipped a coordinated client release (`2026.7.0` —
browser extension, desktop app, mobile, CLI, server) in late July 2026.
The 2026.7.0 browser extension and desktop app share a WASM SDK component
with a confirmed upstream bug against self-hosted Vaultwarden servers:
login and sync succeed, but the vault fails to render, while the web vault
(a separate codebase) is unaffected. This is a widely reported bug
(multiple open issues on `bitwarden/clients` and `dani-garcia/vaultwarden`),
not specific to this homelab.

Bitwarden's server release notes state client v2026.7.0 requires
**Vaultwarden 1.37.0+** for compatibility. This server was still on
**1.36.0** (installed 2026-06-17) when this was hit — the affected client
had silently auto-updated to 2026.7.0 in the background, independent of
any homelab-side change, which is why the timing coincided with (but was
not caused by) the CT renumbering.

**Fix:**

1. Upgrade Vaultwarden on the `vaultwarden` LXC (`192.168.2.11`) to
   `1.37.0` or later:

```bash
   cd /opt/vaultwarden
   docker compose pull
   docker compose up -d
   docker compose ps               # confirm the new image is running
   docker compose logs --tail=50   # check for startup errors
```

`docker-compose.yml` already points at `vaultwarden/server:latest`, so
`pull` is sufficient — no compose file edit needed. 2. On the affected client, force a sync (or restart the app) and confirm
the vault now renders. 3. **If still broken after the server upgrade** (reported by some users
even on 1.37.0+): the confirmed workaround is downgrading that specific
client to `2026.6.1` via the browser extension store's version history,
or the desktop installer archive from Bitwarden's GitHub releases — a
manual step on the affected machine, outside what SSH access to the LXC
can address. 4. Update the version noted in `docs/services/vaultwarden.md` and add a
`docs/changelog.md` entry once resolved.

**Note:** not caused by, and unrelated to, the `vaultwarden` LXC ID
renumbering (110 → 111) performed the same week — included here for the
record since the timing was initially misleading.

**Resolved.** Verified success from a Bitwarden client.

---

## encrypted-git setup: `rrsync` not at the path older guides expect (2026-09-01)

**Symptom:** `/usr/share/rsync/scripts/rrsync` doesn't exist on the
`encrypted-git` LXC (Debian 13).

**Cause:** Many guides describe `rrsync` as a script template that needs
copying into place and `chmod +x`'d. On Debian 13's `rsync` package, it
ships pre-built and ready to use directly at `/usr/bin/rrsync` — no copy
step needed.

**Fix:** Point the SSH forced command at `/usr/bin/rrsync` directly. If
this changes again on a future Debian release, `dpkg -L rsync | grep
rrsync` will show the actual installed path.

---

## git-remote-gcrypt: `rsync: not found` on a network-restricted client (2026-09-01)

**Symptom:** `git push` to a `gcrypt::rsync://` remote fails with
`.../git-remote-gcrypt: 289: rsync: not found`.

**Cause:** `git-remote-gcrypt`'s rsync backend shells out to a local
`rsync` binary on the *client* as well as the server. The client in this
case was a Qubes AppVM with LAN-only network access, so `apt install
rsync` from inside the AppVM itself couldn't reach Debian's mirrors.

**Fix:** Install `rsync` at the TemplateVM level (which has network
access), then fully restart the AppVM — not just its applications —
since Qubes AppVMs boot from a fresh template snapshot each time:

```bash
qvm-run -u root <template-name> "apt update && apt install -y rsync"
qvm-shutdown <appvm-name>
qvm-start <appvm-name>
```

If the TemplateVM was already running, shut it down first. Confirm with
`which rsync` on the AppVM afterward.

**Note:** if the target AppVM shares a template with other AppVMs, they
inherit `rsync` too — harmless in practice, but worth checking given how
compartmentalized this particular setup otherwise is.

---

## gcrypt push: `Permission denied (publickey)` despite a correct key (2026-09-01)

**Symptom:** `git push` to a `gcrypt::` remote fails with
`gcrypt@<host>: Permission denied (publickey)`, even though the key is
correctly authorized on the server and works fine for plain `ssh` tests
using a differently-named `Host` alias.

**Cause:** SSH matches `Host` blocks in `~/.ssh/config` against the literal
hostname string used on the command line. The hostname embedded in a
`gcrypt::rsync://...` URL is exactly that literal string — a shorter alias
(e.g. `Host encrypted-git`) does not match a URL spelling out
`encrypted-git.home.arpa`, so the wrong (or no) identity file gets used.

**Fix:** The `Host` block must be spelled exactly as the hostname appears
in the gcrypt remote URL:

```
Host encrypted-git.home.arpa
    IdentityFile ~/.ssh/<key>
```

---

## rrsync forced command silently never runs — 0 bytes transferred (2026-09-01)

**Symptom:** SSH authentication succeeds, but the connection closes
immediately with `rsync: connection unexpectedly closed (0 bytes received
so far) [sender]` / `rsync error: unexplained error (code 255)`.

**Cause:** The `gcrypt` service account's login shell was `/bin/false`.
Even with a forced `command=` in `authorized_keys`, `sshd` still executes
that command via the account's configured shell
(`<shell> -c "<forced command>"`). `/bin/false` exits immediately without
ever running its argument, so the forced `rrsync` command silently never
executes and the session just dies.

**Fix:** Set the account's shell to something that actually execs its
argument:

```bash
usermod -s /bin/sh gcrypt
```

This does not weaken the access restriction — `no-pty` plus the forced
`command=` already fully constrain the session. The shell is only the
plumbing `sshd` needs to invoke anything, forced or not.

---

## rrsync path doubling: `change_dir ".../srv/gcrypt/srv/gcrypt" failed` (2026-09-01)

**Symptom:** `git push` to a freshly-configured `gcrypt::rsync://` remote
fails with:

```
rsync: [Receiver] change_dir#3 "/srv/gcrypt/srv/gcrypt" failed: No such file or directory (2)
```

**Cause:** `rrsync <root>` (configured in the server's forced SSH command,
e.g. `command="/usr/bin/rrsync /srv/gcrypt"`) prepends its configured root
to whatever path the client requests — it does not chroot in the
traditional sense. If the gcrypt remote URL *also* includes the real
absolute server path, the root gets applied twice.

**Fix:** The path in a `gcrypt::rsync://` URL must be relative to
`rrsync`'s configured root, not the real filesystem path on the server:

```
gcrypt::rsync://gcrypt@encrypted-git.home.arpa/<reponame>
```

never

```
gcrypt::rsync://gcrypt@encrypted-git.home.arpa/srv/gcrypt/<reponame>
```

The real directory (created via `mkdir`/`chown` on the server) keeps its
full absolute path — only the client-facing URL path changes.

**Alternate symptom — porting an existing remote:** the doubled path
doesn't always surface as an explicit `change_dir` error. When re-pointing
an existing repo's `origin` at a gcrypt remote (e.g. via a shell-history
command that still has the wrong path baked in), the failure can instead
look like a generic, earlier-stage failure:

```
gcrypt: Repository not found: rsync://gcrypt@<host>/srv/gcrypt/<reponame>
gcrypt: Setting up new repository
error: failed to push some refs to '...'
```

Same root cause, same fix (drop the `/srv/gcrypt` prefix from the URL) —
this just fails before rsync ever prints the more explicit `change_dir`
message.

**Residue to check after a failed attempt like this:** `git-remote-gcrypt`
writes a `remote.<name>.gcrypt-id` value to local git config as soon as it
decides "no repo found, must be new" — before any network I/O, and
regardless of whether the push subsequently succeeds. If you retry against
a corrected URL under the *same* remote name, that stale ID can make the
script misinterpret the correct-but-still-repo-less URL as "a real repo
that's since disappeared," and abort instead of creating it. Check for and
clear it before retrying:

```bash
git config --get remote.origin.gcrypt-id   # or whatever your remote is named
git config --unset remote.origin.gcrypt-id # if it printed anything
```

No server-side cleanup is needed in this scenario — `rrsync`'s root
confinement means any artifact from a failed attempt is, at worst, a
harmless empty directory under `/srv/gcrypt`, and the actual content
upload never runs until after the path/repo-id checks succeed.

---

## Router: blank-password root SSH login, initially mistaken for intrusion (2026-09-02)

**Symptom:** OpenWrt syslog showed a successful root SSH login with a
blank password from a LAN IP (`192.168.2.147`), preceded by two failed
"nonexistent user" attempts, followed by a LuCI login visiting the DHCP
leases page — initially investigated as a possible intrusion.

**Cause:** Self-triggered. Shell history confirmed `ssh 192.168.2.1`
(defaults to local username, rejected as nonexistent user — likely run
twice, deduped in history) followed by `ssh root@192.168.2.1`
(succeeded, root's password was in fact blank at the time). The LuCI
visit was the same troubleshooting session, checking DHCP leases while
diagnosing an unrelated Android Wi-Fi connectivity issue. Qubes
(`sys-net`) was ruled out as the source via MAC address and
NetworkManager topology before the human-error explanation was found.

**Fix:** Root password rotated to a non-blank value (unrelated to
whether this was actually an intrusion — it was a real gap regardless).

**Note:** `192.168.2.147`'s MAC (`9A:3A:EF:E3:B0:35`, locally-administered
bit set) doesn't match any known homelab device. This and 4 other DHCP
leases with blank hostnames and distinct randomized MACs remain
unexplained — likely phones/guest devices with MAC-privacy features, but
unconfirmed. Lease history was lost to a router reboot before this could
be resolved. Not currently worth further investigation; revisit if a
similar pattern recurs.