# `git-remote-gcrypt` — stale `GPG_AGENT_INFO` tty check, patch notes

> Written to save a future coding agent (or human) the archaeology of
> re-deriving this from the script. Applies to the vendored copy at
> `vendor/git-remote-gcrypt/git-remote-gcrypt`, commit `a5ff704...` per
> `SOURCE.md`. If re-vendoring a newer commit later, re-check whether this
> is still accurate. See
> [`force-push-patch-notes.md`](force-push-patch-notes.md) for the sibling
> write-up on the implicit-force-push gap and for general upstream
> contribution notes (canonical repo, contact paths) that apply equally
> here — not repeated in full below.

---

## The observed behavior

A push can fail late, at the manifest-signing step, with:

```
gpg: signing failed: Inappropriate ioctl for device
gpg: [stdin]: sign+encrypt failed: Inappropriate ioctl for device
```

despite SSH auth, repo setup, and the signing-key config all being
correct. Confirmed reproducible on a stock macOS + Homebrew GnuPG 2.4.7
setup, invoking `git push` from a normal interactive terminal — i.e. not
an exotic environment, just a completely ordinary client.

---

## Root cause

`rungpg()`, the script's single choke point for every `gpg` invocation:

```sh
rungpg()
{
	if isnonnull "$Conf_gpg_args"; then
		set -- "$Conf_gpg_args" "$@"
	fi
	# gpg will fail to run when there is no controlling tty,
	# due to trying to print messages to it, even if a gpg agent is set
	# up. --no-tty fixes this.
	if [ "x$GPG_AGENT_INFO" != "x" ]; then
		${GPG} --no-tty $@
	else
		${GPG} $@
	fi
}
```

The comment shows the author correctly anticipated this exact failure
mode and added a guard for it — `--no-tty` tells gpg not to attempt
writing status output to the calling process's controlling terminal,
which is unnecessary and can fail when gpg-agent (via pinentry) is
actually handling any interactive passphrase prompting out-of-band.

The guard is gated on `$GPG_AGENT_INFO` being non-empty. That variable
was how *pre-2.1* GnuPG communicated the running agent's socket location
to client processes. **GnuPG 2.1 (released 2014) changed agent discovery
to a fixed, predictable socket path and stopped setting this variable
entirely.** Every currently-shipping GnuPG release is 2.1+ — Debian,
Homebrew, and every other mainstream distribution channel have been on
the 2.x line for years. In practice this means `$GPG_AGENT_INFO` is
**never** set on any environment this tool is realistically run in today,
so the `if` branch is dead code: the condition is always false, `--no-tty`
never gets added, and gpg is invoked exactly the way the author's own
comment describes as broken.

This is a straightforward case of a workaround whose trigger condition
quietly stopped being true out from under it, not a design tradeoff
anyone is relying on.

---

## Sketch of a fix

Given gpg-agent has been mandatory, unconditional infrastructure in every
supported GnuPG version for a decade, the check itself is no longer doing
useful work — `--no-tty` is safe to pass unconditionally:

```sh
rungpg()
{
	if isnonnull "$Conf_gpg_args"; then
		set -- "$Conf_gpg_args" "$@"
	fi
	# Modern GnuPG (2.1+) always runs against gpg-agent, which handles
	# any interactive passphrase prompting via pinentry, out-of-band
	# from gpg's own stdio. gpg itself never needs to write to the
	# calling process's controlling terminal, so --no-tty is always
	# safe. (Historically this was gated on $GPG_AGENT_INFO, which
	# pre-2.1 GnuPG used to signal agent availability; that variable
	# hasn't been set by any GnuPG release since 2.1 and the gate had
	# become permanently false.)
	${GPG} --no-tty "$@"
}
```

Notes on this sketch:

- **This is a much smaller, lower-risk change than the force-push
  patch** — one function, no new state, no backend-specific behavior to
  reconcile. A reasonable first patch to actually attempt if easing into
  contributing to this codebase.
- **Quoting fix included almost incidentally:** the original also uses
  unquoted `$@` (`${GPG} --no-tty $@` / `${GPG} $@`), which is subject to
  word-splitting on any argument containing whitespace. The sketch above
  quotes it (`"$@"`). Worth doing in the same patch since it's touching
  this exact line anyway, but call it out explicitly in any patch
  description as a separate, deliberate fix rather than bundling it
  silently with the `GPG_AGENT_INFO` change.
- **Legacy GnuPG 1.x theoretical regression:** GnuPG 1.4.x (the last 1.x
  line) could, in principle, run without an agent at all, in which case
  the old conditional's intent (only add `--no-tty` when an agent is
  actually present) had real meaning. GnuPG 1.x has been unmaintained
  and absent from mainstream distributions for years at this point,
  so treating agent-availability as a given seems like the right
  pragmatic call — but worth stating explicitly in any upstream patch
  description, since it's a real (if narrow) behavior change for anyone
  still on 1.x, not just a bugfix from their point of view.
- **This script is POSIX `sh`, not bash** — same constraint as the
  force-push patch. This particular fix doesn't need any bash-only
  constructs regardless, so no real risk here.

---

## Upstream contribution notes

Same situation and same contact-path guidance as the force-push gap — see
[`force-push-patch-notes.md`](force-push-patch-notes.md) for the full
writeup (canonical repo `git.spwhitton.name` currently unreachable per
Debian's `vcswatch` and independent AUR reports; direct email to Sean
Whitton or a Debian BTS bug are more likely to land than a GitHub PR
against the mirror).

Given how small and self-contained this particular fix is, it may be
worth bundling both patches (this one and the force-push fix) into a
single contribution if either is ever actually submitted upstream, rather
than sending them separately — reduces the number of times a mostly-quiet
maintainer needs to context-switch to review something from this project.
