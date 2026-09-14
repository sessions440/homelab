# `git-remote-gcrypt` — hidden-recipient trial-decryption exit code, patch notes

> Written to save a future coding agent (or human) the archaeology of
> re-deriving this from the script. Applies to the vendored copy at
> `vendor/git-remote-gcrypt/git-remote-gcrypt`, commit `a5ff704...` per
> `SOURCE.md`. Third bug found in this script, alongside
> [`force-push-patch-notes.md`](force-push-patch-notes.md) and
> [`gpg-tty-patch-notes.md`](gpg-tty-patch-notes.md) — upstream contact
> paths and canonical-repo caveats are the same for all three; not
> repeated here.

---

## The observed behavior

A manifest that decrypts and verifies correctly is still reported as a
hard failure:

```
gpg: ecdh failed in gcry_cipher_decrypt: Checksum error
gpg: Signature made ...
gpg:                using EDDSA key ...
gpg: Good signature from "..." [ultimate]
gcrypt: Failed to decrypt manifest!
```

Note the ordering: GPG cannot print `Good signature from ...` without
having *already* decrypted the plaintext successfully. The crypto
genuinely worked. `git-remote-gcrypt` reports failure anyway.

Reproduced live in this homelab (2026-09-11) on a repo with two
`-R`-encrypted (hidden) participants; confirmed absent once reduced to a
single participant or once participants are encrypted with `-r`
(visible) instead.

---

## Root cause

Recipients are hidden (`-R`) by default — see `read_config()`, which only
switches to `-r` when `gcrypt.publish-participants` is true. With hidden
recipients, GPG has no key ID to go on and must try each candidate secret
key in the local keyring against each PKESK packet until one matches.
Every failed trial against a non-matching key logs exactly the
`ecdh failed ... Checksum error` noise seen above — this is expected,
harmless, and unavoidable with hidden recipients when more than one
participant is configured.

The bug is in how `PRIVDECRYPT()` interprets the outcome:

```sh
PRIVDECRYPT()
{
	local status_=
	exec 4>&1 &&
	status_=$(rungpg --status-fd 3 -q -d 3>&1 1>&4) &&
	xfeed "$status_" grep "^\[GNUPG:\] ENC_TO " >/dev/null &&
	...
}
```

`status_=$(...)` in POSIX `sh` takes on the **exit status of the command
substitution**, which is GPG's own process exit code. GPG's exit code
reflects "did anything error during this invocation," not "did the
requested operation ultimately succeed" — a failed trial decryption
against the wrong recipient counts against it even when a later trial
against the right recipient succeeds and produces verified plaintext.
Because the `&&` chain gates on that raw exit code *before* ever
inspecting `$status_` for the `ENC_TO` status line that would prove
success, a manifest that decrypted perfectly fine gets treated identically
to one that didn't decrypt at all.

This only manifests with **more than one hidden recipient** — a single
recipient never needs a failed trial first, and visible (`-r`) recipients
carry the real key ID, so GPG picks correctly on the first attempt
regardless of participant count.

---

## Sketch of a fix

Stop trusting GPG's raw exit code as the success signal; rely on the
machine-readable status-fd lines it already emits for exactly this
purpose:

```sh
PRIVDECRYPT()
{
	local status_= rc_=
	exec 4>&1
	status_=$(rungpg --status-fd 3 -q -d 3>&1 1>&4)
	rc_=$?

	if xfeed "$status_" grep "^\[GNUPG:\] DECRYPTION_OKAY" >/dev/null &&
	   xfeed "$status_" grep "^\[GNUPG:\] ENC_TO " >/dev/null
	then
		xfeed "$status_" grep -e "$1" >/dev/null || {
			echo_info "Failed to verify manifest signature!"
			echo_info "Only accepting signatories: ${2:-(none)}"
			return 1
		}
	else
		echo_info "GPG decryption did not report success (gpg exit $rc_)"
		return 1
	fi
}
```

`DECRYPTION_OKAY` is the specific status-fd token GnuPG emits when the
overall decryption succeeded, independent of how many trial attempts it
took to get there. Checking for it (rather than the process exit code)
correctly separates "some non-target key didn't match, as expected with
hidden recipients" from "decryption genuinely failed."

Notes on this sketch:

- Smallest of the three known bugs in terms of blast radius — it only
  ever bites multi-participant + hidden-recipient configurations, which
  is exactly the workaround already documented in
  `docs/services/encrypted-git.md` (`gcrypt.publish-participants true`).
  A fix here would make that workaround unnecessary rather than required.
- Same POSIX `sh` constraint as the other two patches — no bashisms.
- Worth bundling with the `gpg-tty` fix if either is ever submitted
  upstream, per the reasoning already given in that file's notes.

---

## Practical workaround (already in use, no script change needed)

Set `gcrypt.publish-participants true` whenever more than one participant
is configured. This avoids the bug's trigger condition entirely rather
than fixing the underlying exit-code handling. Tradeoff: recipient key
IDs become visible in the manifest rather than hidden — see
`docs/services/encrypted-git.md` for the corresponding note in this
homelab's threat model.

**Caveat confirmed by hands-on testing:** this workaround is safe to
*unset* only when exactly one participant remains configured — a single
recipient never triggers the multi-trial path regardless of hidden vs.
visible. Re-enable it the moment a second participant is added back.
