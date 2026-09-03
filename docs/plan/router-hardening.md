# Router Hardening — Planning & Execution Doc

> **Status:** Planned — not yet started
> **Last updated:** 2026-09-02

---

## Context

Follow-up from a resolved false-alarm security investigation — see
`docs/troubleshooting.md` → "Router: blank-password root SSH login,
initially mistaken for intrusion". The router itself wasn't compromised,
but the investigation surfaced two real gaps worth closing, plus a
storage-lesson from further back:

1. Root password was blank until manually rotated 2026-09-02.
2. Router's log buffer is RAM-only — the only evidence from the
   above incident was lost the moment the router was rebooted (for an
   unrelated connectivity fix the same evening).
3. (Older, half-remembered) A prior chat session helped diagnose local
   router or AdGuard storage filling up with logs — persistent local
   logging was avoided as a result. Worth digging up before assuming
   remote syslog is a clean fix; confirm the old failure mode doesn't
   recur in a different form.

---

## Open Questions

- [ ] Is dropbear (SSH) reachable from WAN, or LAN-only? Not yet checked.
- [ ] Is SSH key-only auth (disabling password auth) safe to enable now
  that `ai_homelab`/`human_homelab` keys are authorized? Any lockout risk?
- [ ] What's the right remote syslog target — existing LXC or new one?

---

## Planned Work

### 1. WAN exposure check (read-only recon)

Check `/etc/config/dropbear` for interface binding, cross-reference
`/etc/config/firewall` to confirm whether WAN→router SSH is actually
reachable. Report only — no changes.

### 2. SSH key-only auth

Research the exact `/etc/config/dropbear` change to disable password
auth, given both homelab keys are already in
`/etc/dropbear/authorized_keys`. Propose the change and flag any
lockout risk before applying. **Do not apply without explicit
confirmation** — losing SSH access to the router is a bad failure mode.

### 3. Remote syslog

Router's local log is RAM-only (`logd`), cleared on every reboot.
Proposal to investigate:

- Confirm OpenWrt's logging config (`/etc/config/system` —
  `log_type`/`log_ip`/`log_port` options) supports forwarding to a
  remote syslog receiver.
- Propose which existing homelab LXC should run the receiver
  (rsyslog or syslog-ng), or whether a new minimal LXC is warranted.
  This is a low-traffic router — keep it minimal, no need for a
  dedicated logging stack.
- **Before proposing a specific approach:** check whether the earlier
  "storage filled up" issue (context above, memory incomplete) was
  actually about the *router's own* local flash, or about AdGuard's
  local query-log storage — these have different fixes and it matters
  which one prompted the earlier change. If findable in past chat
  history, resolve this before designing the fix; if not findable,
  flag as an open assumption in the proposal.
- Do **not** provision anything — this task ends at a written proposal
  for human review.

---

## Execution Log

*(empty — not yet started)*