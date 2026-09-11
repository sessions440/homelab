# `git-remote-gcrypt` — implicit force-push, patch notes

> Written to save a future coding agent (or human) the archaeology of
> re-deriving this from the script. Applies to the vendored copy at
> `vendor/git-remote-gcrypt/git-remote-gcrypt`, commit `a5ff704...` per
> `SOURCE.md`. If re-vendoring a newer commit later, re-check whether this
> is still accurate — line numbers especially will drift. See
> [`gpg-tty-patch-notes.md`](gpg-tty-patch-notes.md) for a second,
> smaller, unrelated bug found in the same script.

---

## The observed behavior

Every ordinary push (anything that isn't an explicit `git push --force`)
prints:

```
gcrypt: Due to a longstanding bug, this push implicitly has --force.
gcrypt: Consider explicitly passing --force, and setting
gcrypt: gcrypt's require-explicit-force-push git config key.
```

The existing `gcrypt.require-explicit-force-push` config only gates
*whether you're forced to type `--force` consciously* — it does not add
real fast-forward checking. Even with it enabled, once you do pass
`--force` (which the message tells you to do), the push proceeds exactly
as before: unconditional overwrite, no comparison against the remote's
prior state.

**Practical consequence:** two clients with diverged local history,
pushing the same repo, will silently clobber each other. No error, no
extra warning. This matters here specifically because
[`docs/services/encrypted-git.md`](../../docs/services/encrypted-git.md)
deliberately supports multi-client access via `gcrypt.participants`.

---

## Root cause — it's a real gap, not an architectural wall

The remote-as-opaque-blob-storage design (no git-aware server) might
suggest fast-forward checking is *impossible* here — there's no server to
ask "is this a fast-forward?" the way a normal git remote does. That's not
actually the constraint. The information needed for a real check is
already fetched and decrypted locally on every push; it just isn't used
for this purpose.

### Where the data already exists

In `ensure_connected()`, the manifest is fetched and decrypted, then:

```sh
filter_to @Refslist "$Hex40 *" "$manifest_"
```

`Refslist` now holds every ref the remote currently has, as
`<sha1> <refname>` lines — this is the remote's authoritative prior
state, decrypted and sitting in a local shell variable before `do_push`
even starts.

### Where it gets thrown away

In `do_push()`, for each `src:dst` pair being pushed:

```sh
while IFS=: read -r src_ dst_
do
    if [ $(echo "$src_" | cut -c1) != + ]
    then
        force_passed=false
    fi

    src_=${src_#+}
    filter_to ! @Refslist "$Hex40 $dst_" "$Refslist"

    if isnonnull "$src_"
    then
        append_to @r_revlist "$src_"
        obj_=$(xfeed "$src_" safe_git_rev_parse)
        append_to @Refslist "$obj_ $dst_"
    fi
done <<EOF
$1
EOF
```

`filter_to ! @Refslist "$Hex40 $dst_" "$Refslist"` removes the *existing*
line for this ref from `Refslist` — discarding the old SHA entirely —
and the next few lines unconditionally append the *new* SHA in its place.
The old value is never compared against the new one. There's no
`merge-base`, no ancestor check, nothing. This is the entire "bug": the
old ref value passes through this function on its way to being deleted,
and nobody looks at it first.

The `force_passed` logic that follows only checks whether *git itself*
told the remote helper this was a forced push (a leading `+` on the
refspec, per the git remote-helper protocol) — it has nothing to do with
whether the push is actually safe.

---

## Sketch of a fix

Before the existing `filter_to ! @Refslist ...` line discards the old
entry, capture it and compare:

```sh
old_line_=$(xfeed "$Refslist" xgrep -E "^$Hex40 $dst_\$")
old_sha_=${old_line_%% *}

filter_to ! @Refslist "$Hex40 $dst_" "$Refslist"

if isnonnull "$src_"
then
    append_to @r_revlist "$src_"
    obj_=$(xfeed "$src_" safe_git_rev_parse)

    if isnonnull "$old_sha_" && [ "$force_passed" != "explicit-for-this-ref" ]
    then
        if ! git cat-file -e "$old_sha_" 2>/dev/null
        then
            echo_die "Cannot verify fast-forward for $dst_: local repo doesn't have $old_sha_. Fetch first."
        elif ! git merge-base --is-ancestor "$old_sha_" "$obj_"
        then
            echo_die "Non-fast-forward push to $dst_ rejected (old: $old_sha_, new: $obj_). Use --force if intended."
        fi
    fi

    append_to @Refslist "$obj_ $dst_"
fi
```

Notes on this sketch, deliberately left rough rather than finished:

- **`force_passed` is currently a single flag for the whole push
  invocation**, not per-ref (a push can touch multiple refs in one
  invocation via multiple `push` lines in the same protocol exchange —
  see `gcrypt_main_loop`'s inner `while read input_inner` loop collecting
  multiple `push ...` lines before calling `do_push` once). A correct
  patch needs per-`dst_` force tracking (check the leading `+` on *this*
  refspec, not a script-global flag), not reuse of the existing
  `force_passed` variable as-is.
- **The "local repo doesn't have `old_sha_`" case matters and is not an
  edge case to skip.** If another client advanced the remote ref past
  what you've fetched, your local git may not have that commit object at
  all, and `git merge-base --is-ancestor` requires both objects to be
  present. Treat "can't verify" as "refuse, tell the user to fetch" —
  the same posture a normal git server takes when you're behind.
- **Where the fast-forward check itself belongs is debatable**: inline in
  `do_push` (shown above) is the minimal-diff option, but it's shared
  code across *every* backend (local, rsync, sftp, gitception,
  experimental rclone) — a bug in this logic affects all of them
  identically, which is good (one fix, no backend-specific
  divergence) but means testing needs to cover more than just the
  rsync backend this homelab actually uses.
- **This script is POSIX `sh`, not bash** — no arrays, no `[[`, no bash-only
  string ops. Match the existing idioms already in the file
  (`xfeed`, `filter_to`, `setvar`/`append_to` as the return-value
  convention, `isnull`/`isnonnull`/`iseq` as the comparison helpers)
  rather than introducing new patterns.
- **Backward compatibility:** consider whether this should be gated
  behind a new config (e.g. `gcrypt.check-fast-forward`) defaulting to
  on, versus just always being correct. Given the current behavior is a
  straightforward bug (silent data loss on divergence) rather than a
  documented feature anyone relies on, defaulting to *always check*
  seems right — but flag this explicitly in any patch description since
  it's a behavior change for existing users of the tool, not just a bug
  fix from their point of view if they were unknowingly relying on
  last-write-wins semantics.

---

## If this gets fixed for real: upstream contribution notes

- **Canonical repo:** `git.spwhitton.name`, maintained by Sean Whitton (a
  Debian Developer). As of this writing, the Debian package tracker's
  `vcswatch` reports it can't reach this host, and an independent AUR
  comment reports the same — see
  [`docs/plan/encrypted-git.md`](../../docs/plan/encrypted-git.md) for
  the full maintenance-risk writeup. **Don't assume a GitHub PR against
  the `spwhitton/git-remote-gcrypt` mirror will be seen** — it's a
  mirror, not necessarily monitored.
- **More reliable contact paths, in rough order of likely success:**
  1. Direct email to Sean Whitton (findable via his Debian Developer
     identity / personal site) with a patch attached or a link to a
     public branch.
  2. A bug filed against the Debian `git-remote-gcrypt` package
     (`reportbug git-remote-gcrypt` on a Debian system, or via
     `bugs.debian.org`) — Debian maintainers often pick up patches
     through the BTS even when upstream itself is slow.
  3. A GitHub PR against the mirror, as a fallback / for visibility, but
     treat it as the least likely to get a response given the above.
- **Realistic sequencing:** patch this homelab's own vendored copy
  first, and validate it against real usage here. Whether it ever gets
  submitted upstream — and whether upstream ever merges it — is a
  separate, lower-priority question. The vendoring decision in
  `docs/plan/encrypted-git.md` was explicitly made so this homelab isn't
  blocked on upstream's pace either way.
