# Troubleshooting

---

## Minecraft: `minecraft.socket` refusing to listen or cannot be enabled (2026-10-02)

**Symptom:** Running `systemctl enable --now minecraft.socket` produces `The unit files have no installation config` and/or `minecraft.socket: Socket service minecraft.service already active, refusing`.

**Cause:**
1. A `.socket` unit without an `[Install]` section (`WantedBy=sockets.target`) is static and cannot be enabled via `systemctl enable`.
2. When configuring `StandardInput=socket` with `Sockets=minecraft.socket`, systemd refuses to start the socket while the target service is already running.

**Fix:** Add `[Install]\nWantedBy=sockets.target` to `/etc/systemd/system/minecraft.socket`. Stop `minecraft.service` before enabling/starting `minecraft.socket`, then restart `minecraft.service`:
```bash
systemctl stop minecraft.service
systemctl enable --now minecraft.socket
systemctl start minecraft.service
```

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

---

## gcrypt push fails with `gpg: ... No secret key`, unrelated to SSH config (2026-09-03)

**Symptom:** Push to a `gcrypt::` remote gets past SSH auth and repo setup
fine (`Setting up new repository`, `Remote ID is ...`), then fails while
signing the manifest:

```
gcrypt: Requesting manifest signature
gpg: skipped "/Users/<user>/.ssh/id_ed25519.pub": No secret key
gpg: [stdin]: sign+encrypt failed: No secret key
error: failed to push some refs to '...'
```

Easy to mistake for an SSH problem given the `.ssh/` path in the error —
it isn't. SSH auth has already succeeded by this point in the push.

**Cause:** `git-remote-gcrypt` always **signs**, not just encrypts, the
manifest. It picks a signing key via:

```
git config --get remote.<name>.gcrypt-signingkey   # checked first
git config --path user.signingkey                  # fallback if unset
```

On a machine set up for the common modern GitHub workflow of signing
commits with an SSH key instead of real GPG (`git config gpg.format ssh`,
`user.signingkey ~/.ssh/id_ed25519.pub`), gcrypt blindly passes that
*path to an SSH public key file* to real `gpg -u` — which has no matching
GPG secret key, so the sign+encrypt operation fails outright.

**Fix:** Override the signing key for the gcrypt remote specifically, so
it never falls through to `user.signingkey`:

```bash
git config remote.<name>.gcrypt-signingkey <gpg-key-id>
```

Confirm the chosen key actually has a usable secret key on this machine
first — signing requires the private half to be physically present,
independent of whatever key(s) are configured as encryption recipients
via `gcrypt.participants` (signing and encryption-recipient are separate
roles here):

```bash
gpg --list-secret-keys <gpg-key-id>
```

**Related:** if this happens on a repo's first push, `remote.<name>.gcrypt-id`
has already been written to local git config before the crash (see the
"rrsync path doubling" entry above, "Residue to check after a failed
attempt") — clear it before retrying or the retry will fail differently
(`repository ID is set. Aborting`) even after the signing key is fixed.

---

## gcrypt push fails with `gpg: signing failed: Inappropriate ioctl for device` (2026-09-03)

**Symptom:** After the SSH/repo-setup and signing-key config are both
correct, the push still fails at the manifest-signing step:

```
gcrypt: Requesting manifest signature
gpg: signing failed: Inappropriate ioctl for device
gpg: [stdin]: sign+encrypt failed: Inappropriate ioctl for device
```

**Cause:** `git-remote-gcrypt`'s `rungpg()` helper is aware of this class
of error — its own comment says so — and tries to add `--no-tty` to avoid
it, but only when `$GPG_AGENT_INFO` is set:

```sh
if [ "x$GPG_AGENT_INFO" != "x" ]; then
    ${GPG} --no-tty $@
else
    ${GPG} $@
fi
```

`GPG_AGENT_INFO` was removed from GnuPG after 2.1 — agent discovery is
automatic now. On any modern GnuPG install (confirmed on 2.4.7) this
variable is never set, so the guard's condition is always false and
`--no-tty` never gets added, silently reproducing the exact failure the
comment describes. A real bug in the vendored script, separate from the
implicit-force-push one — see
[`vendor/git-remote-gcrypt/gpg-tty-patch-notes.md`](../../vendor/git-remote-gcrypt/gpg-tty-patch-notes.md)
for the root cause, a fix sketch, and upstream contribution notes.

**Fix — most reliable, sidesteps the tty/pinentry plumbing entirely:**
pre-authenticate with gpg-agent in a normal terminal before the push runs,
so it caches the already-unlocked key and the automated flow never needs
an interactive prompt to succeed:

```bash
echo test | gpg --clearsign --default-key <gpg-key-id> > /dev/null
```

**Alternative — fix the tty binding directly:**

```bash
export GPG_TTY=$(tty)
```

If that alone isn't sufficient, check `~/.gnupg/gpg-agent.conf` for a
working `pinentry-program`; a GUI pinentry (e.g. `pinentry-mac` on macOS,
via Homebrew) tends to be more robust than a tty-based one for this
specific git-spawns-script-spawns-gpg subprocess chain, since it doesn't
depend on inheriting a controlling terminal at all.

**Related:** clear `remote.<name>.gcrypt-id` again before retrying — this
failed attempt will have written a fresh one, same as the two entries
above.

---

## gcrypt manifest decrypt fails despite a "Good signature" line printing first (2026-09-11)

**Symptom:** `git push`/`pull` against a `gcrypt::` remote fails with:

```
gpg: ecdh failed in gcry_cipher_decrypt: Checksum error
gpg: Signature made ...
gpg: Good signature from "..." [ultimate]
gcrypt: Failed to decrypt manifest!
```

The `Good signature` line proves decryption already succeeded — GPG
cannot verify a signature without the plaintext — yet the script still
reports failure.

**Cause:** `gcrypt.participants` had more than one entry, encrypted with
hidden (`-R`) recipients (the default when `gcrypt.publish-participants`
is unset). GPG must try each local secret key against each recipient
packet in turn; a failed trial against a non-matching key logs the
"Checksum error" noise and leaves GPG's own process exit code non-zero,
even though a later trial against the correct key succeeds and produces
verified plaintext. `git-remote-gcrypt`'s `PRIVDECRYPT()` trusts that raw
exit code, so a fully successful decrypt gets treated as a hard failure.
Root cause and fix sketch:
[`vendor/git-remote-gcrypt/exit-code-patch-notes.md`](../vendor/git-remote-gcrypt/exit-code-patch-notes.md).

**Fix:**

```bash
git config --global gcrypt.publish-participants true
```

Then push once more from a client holding a real ref change (an
already-up-to-date push never invokes the helper's push path — see the
`GCRYPT_FULL_REPACK` entry below for why that matters) so the manifest
actually gets rewritten with visible recipients.

**Only safe to unset again once exactly one participant remains** in
`gcrypt.participants` — see the revocation entry below.

---

## Revoking a gcrypt participant: "Failed to verify manifest signature!" (2026-09-14)

**Symptom:** After narrowing `gcrypt.participants` from `{A, B}` down to
`{B}` (intending to revoke A's access), the very next push or pull fails:

```
gcrypt: Failed to verify manifest signature!
gcrypt: Only accepting signatories:  <B's key ID>
gcrypt: Failed to decrypt manifest!
```

**Cause:** `read_config()` derives two things from a single read of
`gcrypt.participants` in the same call: which keys future manifests get
*encrypted to*, and which signatures are *trusted* when verifying the
manifest currently on the remote. Narrowing the list changes both at
once. If the manifest sitting on the server was signed by A (the key
you're revoking) and you've already dropped A from the trusted-signer
set, every operation fails before it can get anywhere near writing a new
manifest — including the push meant to perform the revocation itself.
There is no code path that reaches manifest-rewriting logic without first
passing this verification step.

**Fix — revoke in two steps, not one:**

1. **With the full participant list still in place** (`{A, B}`), force a
   real push from a client holding B's secret key, so B becomes the
   manifest's signer. A push that reports `Everything up-to-date` does
   **not** count — git's ref comparison happens before the helper's push
   path is ever invoked, so nothing gets rewritten. Force an actual ref
   change first if needed:

   ```bash
   git commit --allow-empty -m "rekey checkpoint"
   git push --force
   ```

   Confirm the signer actually changed before proceeding:

   ```bash
   git pull   # should show "Signature made ... using EDDSA key <B's fingerprint>"
   ```

2. **Only now narrow the participant list**, since the current signer (B)
   is already inside the set you're narrowing to:

   ```bash
   git config --global gcrypt.participants "<B's fingerprint only>"
   git commit --allow-empty -m "revoke A"
   git push --force
   ```

Verification passes this time (current manifest is signed by B, and B is
in the accepted set), and the new manifest is encrypted to B alone.

**Scope of what this actually revokes:** only the server-side store going
forward. Anything A already fetched is unaffected — there's no remote
kill-switch for data a client has already decrypted locally. See
`docs/plan/encrypted-git.md` for the full reasoning if this is ever
revisited.

**Cleanup note:** the empty checkpoint/revoke commits used to force the
ref change are safe to drop afterward (`git reset --hard HEAD~N` +
force-push), but each rewrite leaves the prior packfile orphaned on the
server. Follow with a forced full repack (see next entry) to actually
reclaim that storage.

---

## `GCRYPT_FULL_REPACK=1` silently does nothing on an up-to-date push (2026-09-14)

**Symptom:** `GCRYPT_FULL_REPACK=1 git push --force` completes with
`Everything up-to-date` and no `gcrypt: Repacking remote ...` line —
looks successful, but no repack happened.

**Cause:** `GCRYPT_FULL_REPACK` is only read inside `repack_if_needed()`,
which is only called from `do_push()`. `do_push()` only runs if git's own
ref comparison (done before the remote-helper protocol invokes `push` at
all) finds an actual difference between local and remote refs. If nothing
changed, the helper's `push` verb is never called, so the env var has no
code path to act on — it isn't ignored, the function it lives in simply
never executes.

**Fix:** ensure there's a genuine ref change on the same push as the env
var:

```bash
git commit --allow-empty -m "trigger full repack"
GCRYPT_FULL_REPACK=1 git push --force
```

**Verifying a repack actually happened**, independent of the (sparse)
terminal output: inspect the server directory directly. A successful full
repack forces `Keeplist=` empty and consolidates every existing packfile
into exactly one new one, so a clean single-participant repo should show
exactly two files afterward — one packfile, one manifest:

```bash
ssh root@encrypted-git.home.arpa "ls -la /srv/gcrypt/<reponame>/"
```

**Efficiency note:** if a repack is already anticipated (e.g. right after
a revocation or history rewrite), starting with
`GCRYPT_FULL_REPACK=1 git push --force` directly — rather than an ordinary
push followed by a separate full-repack push — avoids an unnecessary
extra round-trip, as long as that first push carries a real ref change.

---

## gcrypt clone reports "Repository not found" for a repo that exists (2026-09-11)

**Symptom:** `git clone gcrypt::rsync://...` fails immediately:

```
gcrypt: Repository not found: rsync://gcrypt@<host>/<repo>
warning: You appear to have cloned an empty repository.
```

...even though the repo genuinely exists and other clients can reach it.

**Cause:** On a fresh clone, `$Repoid` is always unset locally (no prior
`remote.<name>.gcrypt-id` config exists yet), so `ensure_connected()`
takes this branch on any `GET` failure:

```sh
GET "$URL" "$Manifestfile" "$tmp_manifest" 2>| "$tmp_stderr" || {
    if ! isnull "$Repoid"; then
        cat >&2 "$tmp_stderr"
        ...
    else
        echo_info "Repository not found: $URL"
        return 0
    fi
}
```

The `else` branch — the one taken on every fresh clone attempt —
**discards `$tmp_stderr` entirely**. A genuinely-missing repo, a DNS
failure, an SSH auth failure, and (as diagnosed live in this homelab) a
plain typo in `~/.ssh/config`'s `Host` block all collapse into the exact
same generic message. The message is not trustworthy as a diagnosis on a
fresh clone — treat it as "the transport call failed for some reason,"
not "no repo here."

**Fix — test the transport layer directly, independent of git or gcrypt:**

```bash
which rsync                                  # confirms rsync itself is present
ssh -v gcrypt@<host> true                    # DNS failure vs. auth failure vs. clean forced-command exit
```

In the specific case that prompted this entry, the actual cause was a
typo in the client's `~/.ssh/config` `Host` block (must match the literal
hostname in the gcrypt URL exactly — see the existing "Permission denied"
entry above for the same underlying SSH-matching behavior). Once fixed,
the identical clone command succeeded with no other changes.

## gcrypt push signed by the wrong local key, despite `gcrypt.participants` being correct (2026-09-14)

**Symptom:** With two GPG secret keys (A, B) present in the local keyring
and `gcrypt.participants` set to A only, a push is nonetheless signed by
B. Subsequent push/pull attempts then fail with:

```
gpg: Signature made ...
gpg:                using EDDSA key <B's fingerprint>
gpg: Good signature from "..." [unknown]
gcrypt: Failed to verify manifest signature!
gcrypt: Only accepting signatories:  <A's key ID>
gcrypt: Failed to decrypt manifest!
```

Note this is a **different** bug from the hidden-recipient exit-code issue
(see `vendor/git-remote-gcrypt/exit-code-patch-notes.md`) even though the
surface symptom looks similar (`Good signature` followed by a hard
failure). Distinguish by the specific error: this one says
`Failed to verify manifest signature!` with a signer/accepted-signer
mismatch, not `Failed to decrypt manifest!` preceded by
`ecdh failed ... Checksum error` trial-decryption noise.

**Cause:** `gcrypt.participants` governs two things — who a manifest is
*encrypted to*, and whose signature is *trusted* on verify. It has no
bearing on which local secret key GPG actually *signs with* when
producing a new manifest. That's a separate, independent lookup in
`read_config()`:

```sh
Conf_signkey=$(git config --get "remote.$NAME.gcrypt-signingkey" '.+' ||
    git config --path user.signingkey || :)
```

If neither `remote.<name>.gcrypt-signingkey` nor `user.signingkey` is set,
`PRIVENCRYPT()` calls `gpg -se` with no `-u` flag, so GPG falls back to
its own default signing key — normally `default-key` in `~/.gnupg/gpg.conf`
if present, or otherwise **whichever secret key GPG lists first** when no
`gpg.conf` exists at all and no default is configured. On a keyring with
more than one secret key and no explicit signing-key override anywhere,
which key ends up signing a gcrypt push is effectively arbitrary from the
user's point of view — governed by GPG's own internal key ordering, not
by anything gcrypt-aware.

Confirmed on this homelab: no `~/.gnupg/gpg.conf` existed on the affected
AppVM, and key B simply appeared first in `gpg --list-secret-keys` — that
ordering, not any deliberate configuration, is what determined the
signer.

**Fix — force the signing key explicitly, per remote:**

```bash
git config remote.<name>.gcrypt-signingkey <A-fingerprint>
```

**Recovering from a repo already signed by the wrong key:** setting the
signing key only changes what happens on the *next* push — it cannot
retroactively fix verification of the manifest currently on the remote,
which is the same signer/recipient coupling documented in the "Revoking a
gcrypt participant" entry above. Two ways to recover, depending on whether
the repo's content is worth preserving:

- **If preserving content:** temporarily widen `gcrypt.participants` to
  include both A and B so verification of the existing (B-signed)
  manifest can succeed, set `remote.<name>.gcrypt-signingkey` to A, force
  a real push (an empty commit works if there's nothing else to push) so
  a correctly-signed manifest gets written, confirm via `git pull` that
  the signer is now A, then narrow `gcrypt.participants` back to A alone.
  This narrowing step is a single push here (unlike true revocation) since
  the manifest is already signed by A by the time you narrow.
- **If the repo is disposable (e.g. still mid-setup):** simpler to nuke
  the remote directory (`rm -rf /srv/gcrypt/<reponame>`), clear the local
  `remote.<name>.gcrypt-id`, set `gcrypt-signingkey` correctly first, and
  push fresh. Confirmed as a valid shortcut in this homelab when the repo
  had no content worth preserving through the recovery.

**Worth checking on any client with multiple GPG secret keys:** run
`gpg --list-secret-keys --keyid-format=long` and check for a `default-key`
line in `~/.gnupg/gpg.conf`. The same implicit-default behavior applies to
any plain `gpg --sign`/`gpg -se` invocation on that machine without an
explicit `-u`, not just through gcrypt — worth knowing which key is the
silent default before it surprises you somewhere else.
