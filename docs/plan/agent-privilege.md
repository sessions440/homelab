# Proxmox Host — Agent Privilege Model

> **Status:** Decided — RBAC-scoped automation chosen over full root. Not yet implemented; no concrete privilege list or wrapper script built.
> **Last updated:** 2026-09-27

---

## Context / Goal

Everything below the Proxmox host level already runs autonomously: `ai_homelab` has root SSH on every service LXC, and `opencode` handles execution tasks against them directly. The host itself does not — per the existing Hard Limit in `AGENTS.md` ("Do NOT access the Proxmox host via SSH or any other means"). Anything at the host level (VM/LXC creation, storage, snapshots, network bridges) is fully manual: Claude chat gives guidance, the human copy-pastes into a terminal.

Goal: reduce that friction by giving an agent some form of direct host-level execution, without recreating the two risks that motivated the original Hard Limit:

- **Confidentiality** — sensitive data leaving the LAN via a cloud-hosted agent with filesystem-level access to the host.
- **Blast radius** — an agent action (a buggy multi-step plan, a prompt injection, a plain mistake) causing irreversible damage to the hypervisor: destroyed storage, corrupted `/etc/pve` cluster config, broken networking, deleted VMs/LXCs.

## Candidates considered

### 1. Full standing root (extend the `ai_homelab` pattern to the host)

Simplest to implement. Rejected.

**Confidentiality:** originally the sharper objection, on the strength of the inference LXC's now-abandoned plan to bind-mount the personal notes vault (see `docs/plan/local-ai-stack.md`) — full host root would have meant trivial plaintext read access to years of personal notes from a cloud-hosted agent, directly contradicting the reason that stack is local in the first place. This objection is resolved: the inference LXC's architecture was redesigned around a stateless, ephemeral backend that holds no server-side copy of vault content at all. Confidentiality is no longer the disqualifying factor it first looked like.

**Blast radius:** unaffected by the fix above, and does the actual disqualifying work now. This risk has nothing to do with what data lives on the host — it's about the availability and irreversibility of services the household depends on (Vaultwarden, Minecraft). A single bad plan or injected instruction with full root can destroy storage, corrupt the pmxcfs-backed cluster config, misconfigure networking and lock out the box, or delete guests outright. Full standing root has no structural boundary against any of this — only the agent's own judgment does, every session, indefinitely.

### 2. Scoped Proxmox API token (RBAC) — chosen

Proxmox has a native permission system: custom roles built from specific privileges (e.g. `VM.Allocate`, `VM.Config.Disk`, `VM.Config.Network`, `VM.Snapshot`, `Datastore.AllocateSpace`, `VM.Audit`), assignable to an API token with privilege separation so the token can't exceed the role. This maps directly onto the actual, repetitive friction — creating, resizing, and snapshotting VMs/LXCs — without a standing shell.

**Known limitation:** some operations require authentication as the literal `root@pam` user regardless of assigned privileges — most notably, bind-mounting a host directory into a container. A scoped token is refused on these even with a permissive role. RBAC cannot cover everything; a residual category of host-level operations genuinely needs root.

### 3. Forced-command wrapper for the residual root-only sliver

For operations RBAC can't reach (bind mounts, some storage/network plumbing, host package installs), the candidate pattern is the one already running for `encrypted-git`'s `gcrypt` account: a dedicated key whose only capability is a small allowlist script — specific vetted `pct`/`qm`/`pvesm` subcommands, no general shell, no arbitrary `pct exec`. Not built.

Given how small this residual category now looks — the bind-mount case was the main concrete example, and it no longer exists now that the inference LXC doesn't bind-mount anything — this is lower priority than originally scoped. May not be worth building at all if the remaining root-only operations stay as infrequent as they currently appear.

## Decision

**Adopt a scoped Proxmox API token (RBAC) as the primary mechanism for host-level agent automation. Do not grant standing full root access to the Proxmox host.** The existing `AGENTS.md` Hard Limit against direct host access stands until the RBAC token is built and proven; at that point the limit should be narrowed to cover only what the token's role doesn't grant, rather than removed outright.

Residual root-only operations (bind mounts, network config, host package/driver installs, `/etc/pve` edits) remain manual / chat-guided, matching existing Hard Limits. Their low frequency means the productivity cost of keeping them manual is small — this is the main reason RBAC alone is expected to close most of the friction gap without the wrapper-script layer being needed immediately.

**Rationale, in short:** the confidentiality objection to full root was real but has been engineered away elsewhere (stateless inference architecture). The blast-radius objection was never about data and doesn't go away with that fix — it's addressed structurally by RBAC, which also ages well: it won't need to be re-litigated as more sensitive services (e.g. Immich, once actually populated with photos) come online, the way "the agent already had root anyway" would.

**Adjacent note on trust boundaries, surfaced while working through this:** the secrecy of `caddy.env` (real domain, Cloudflare token) from the agent today is enforced by the `AGENTS.md` Secrets Management policy, not by an OS-level wall — `ai_homelab` already has root SSH on the caddy LXC, so reading that file would only require disregarding a written instruction, not defeating a technical control. Host root wouldn't have made this specific secret meaningfully more exposed than it already is. Noted here because it's a useful data point on how much of this homelab already runs on instruction-following versus hard boundaries, which bears on how much *additional* policy-based (rather than structural) trust is reasonable to extend at the host level.

## Not yet decided / next steps

- Enumerate the actual PVE privilege list for the automation role and validate it against real day-to-day CT/VM lifecycle work (not yet attempted)
- Decide whether the forced-command wrapper for the residual root-only sliver is worth building at all now that the bind-mount case has disappeared, or whether "stays manual" is an acceptable permanent answer
- No `docs/services/agent-privilege.md` yet — there is no concrete "how" to document until the privilege list above is drafted and tried against real usage

## References

- `docs/plan/local-ai-stack.md` — the stateless inference architecture that resolved the confidentiality objection
- `docs/plan/backup-strategy.md` — a pre-session snapshot/rollback safety net is a complementary, not alternative, mitigation for the blast-radius risk this doc addresses
- `AGENTS.md` — current Hard Limits and Secrets Management sections
