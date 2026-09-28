# Local AI Inference Stack — Planning & Execution Doc

> **Status:** Implementation in progress — GPU host driver installed, LXC passthrough pending
> **Last updated:** 2026-09-27

---

## Context / Goal

Setting up a purely local AI inference stack on the home LAN to enable semantic search and AI chat over a sensitive personal note vault (a folder of markdown files, 5+ years of disorganized personal logs). No data should leave the machine or LAN. This is a milestone toward a longer-term goal of building a Karpathy-style LLM wiki from those notes.

---

## Proposed Stack

1. **New LXC (or VM if required)** on Proxmox with NVIDIA GTX 1060 GPU access.

2. **`llama-server`** (llama.cpp) running as a service inside it, exposing the OpenAI-compatible API on a LAN IP. Start with bare llama-server (single model, single process). If/when multiple models are needed, drop **llama-swap** in front — it's a single Go binary pointed at the same llama-server invocations, so the migration is trivial.

3. **UI progression:** Start with the llama-server built-in web UI (zero additional deployment; chat history is browser-local). Migrate to Open WebUI when server-side chat persistence, multi-device access, or tunable RAG over the markdown vault is needed. At migration time, document how to configure Open WebUI's knowledge base for the markdown vault (chunking/overlap settings, document collection setup, llama-server connection) — noting that, per the Data Architecture decision below, Open WebUI should run client-side (or on a client-adjacent trusted machine), not on the inference LXC itself.

4. **Document source:** the markdown vault is never copied to server-side storage — see **Data Architecture** below. (This supersedes an earlier version of this plan that called for bind-mounting the vault into the container via a Proxmox `mp` entry.)

5. **Models (settled — see Decisions Log):**
   - **Chat:** Qwen3 8B, Q4_K_M quantization (~5.0GB VRAM)
   - **Embeddings:** nomic-embed-text-v1.5, F16 GGUF (~262MB VRAM)
   - Both are loaded sequentially via llama-swap, not simultaneously.

---

## Data Architecture — Stateless Server, Client-Held Vault

**Constraint:** the inference LXC (server side) must never hold a persistent, at-rest copy of vault content — encrypted or not. The server is a dumb, stateless compute backend; the vault's plaintext lives only client-side.

**Why the earlier bind-mount plan was dropped:** no real benefit was identified for it over a plain network `git clone`/`pull`, and it carried genuine costs, checked directly rather than assumed — Proxmox won't snapshot a container that has a host bind mount, and configuring a bind mount requires authenticating as `root@pam` specifically, which a scoped API token cannot do (see `docs/plan/agent-privilege.md`). It also tightly couples two containers at the storage layer for no identified gain. Treated as an under-examined early-planning placeholder, not a deliberate tradeoff.

**Architecture**, modeled on how a stateless cloud LLM API actually works (checked against Anthropic's own documented API behavior, not just assumed — by default, conversation content is not retained, and any persistence is a narrow, purpose-built, opt-in feature layered on top of a stateless baseline):

- The vault's plaintext lives only client-side — wherever Obsidian + Syncthing already put it. No copy is created on the inference LXC or the Proxmox host, bind-mounted or otherwise.
- A RAG/agent harness runs on the client (or a client-adjacent trusted machine — e.g. a dedicated Qubes AppVM, matching how the gcrypt GPG key is deliberately kept off any server), not on the inference LXC. AnythingLLM and Open WebUI (already in this plan for UI purposes) both support pointing their chat backend at a remote OpenAI-compatible endpoint while keeping their own document/vector store wherever they run.
- **Embeddings computed client-side where practical.** `nomic-embed-text-v1.5` (~262MB) is cheap enough to run without a GPU. Doing this client-side means the inference LXC never sees vault content in bulk — only the specific chunks relevant to a given query, at query time. Narrow caveat: dense embedding vectors aren't perfectly opaque (partial text recovery from embeddings is a real, if narrow, research area) — so if embeddings are ever computed server-side instead, the resulting vectors shouldn't be allowed to persist or get logged there.
- Only the per-query retrieved chunks plus the user's prompt cross the wire to `llama-server` for the actual completion. No conversation or document content is written to disk on the server; verbose/debug request logging must stay off, or be redirected to a `tmpfs` mount if logging is ever needed for debugging.
- **TLS in front of `llama-server`.** Plaintext vault content crosses the LAN on every query, and "trusted LAN" isn't a given here (see the unresolved mystery-MAC entries in `docs/troubleshooting.md`). Reuse the existing Caddy pattern — an internal virtualhost fronting `llama-server` — rather than treating LAN-only as sufficient on its own.
- **Swap disabled or encrypted on the inference LXC.** "Plaintext only in RAM" holds only if RAM is never paged to disk under memory pressure — worth confirming once the model, KV cache, and everything else are actually running on a 6GB-VRAM box.
- **Bulk-query performance (e.g. "search the whole vault") is handled by `llama-server`'s own context/KV-cache reuse (`--cache-reuse`, slot save/restore) — not by persistent server-side storage.** A rejected alternative was encrypted-at-rest storage with on-demand decryption (GPG or `age`, decrypting to a `tmpfs` target so plaintext never touches disk) — technically workable, but it requires a decrypt-capable key to live on the inference LXC, reintroducing real server-side key custody for what turned out to be a caching/performance problem rather than a data-architecture requirement. Revisit only if the stateless model hits an actual throughput wall it can't solve.

**Residual risk floor, stated plainly:** a live compromise of the inference LXC during an active request could still scrape process memory. This is the same floor any stateless server has — not a gap specific to this design — and a much smaller and shorter-lived exposure than plaintext sitting on disk indefinitely.

---

## Key Open Questions

- [x] ~~**Model selection:** best chat model and embedding model for this VRAM budget, optimized for Q&A over personal notes rather than coding.~~ **RESOLVED 2026-06-24. See Decisions Log.**

- [x] ~~**GPU in Proxmox:** LXC cgroup device passthrough vs KVM VM with VFIO PCIe passthrough~~ **DECIDED: LXC cgroup passthrough.** Host NVIDIA driver installed and verified — see [Proxmox Host — NVIDIA Driver Setup](../setup/proxmox-nvidia-driver-setup.md) for the full install log, decisions, and gotchas. **Remaining:** wire the GPU into the actual inference LXC (privileged container, `/dev/nvidia*` bind mounts, matching CUDA toolkit inside the container) — not yet done.

- [x] ~~**Vault data architecture:** should sensitive vault content ever be persisted server-side on the inference LXC, encrypted or not?~~ **RESOLVED 2026-09-27.** See Decisions Log and "Data Architecture" above: stateless, client-held, never persisted server-side.

---

## Constraints

- No data leaves the LAN; no cloud model APIs for the sensitive vault
- **The inference server (`llama-server`) must remain stateless — no persistent server-side copy of vault content, encrypted or not**
- Prefer LXC over VM unless GPU access requires otherwise
- Prefer documented, actively maintained software
- Claude Code is available for implementation via SSH into LXCs; avoid running it on the Proxmox host itself
- A `CLAUDE.md` file documents the homelab for Claude Code sessions

---

## Longer-Term Context

The ultimate goal is a Karpathy-style LLM wiki: a git repo of structured markdown compiled from raw notes using AI assistance. The sensitive vault wiki will use the local inference stack. A separate wiki based on Claude.ai chat history may use Anthropic models since that data is already on their servers.

**Khoj** is worth evaluating at that stage as it's purpose-built for personal knowledge base indexing. **LibreChat** is the leading candidate for a multi-provider cloud chat frontend (Claude.ai replacement), separate from the inference stack, and used to harvest raw notes for ingestion into the wiki.

---

## Decisions Log

### 2026-09-27 — Data Architecture: Stateless, Client-Held Vault (supersedes bind-mount plan)

**Decision: the vault never lands on server-side storage, in any form.** Dropped the original bind-mount plan (no identified benefit; broke CT snapshots; required `root@pam`). Considered and rejected server-side encrypted-at-rest storage with on-demand decryption (GPG/`age` to a `tmpfs` target) — workable, but it puts a decrypt-capable key on the inference LXC for what was ultimately a caching/performance motivation, not a real data-architecture need. Adopted instead: a client-side (or client-adjacent) RAG/agent harness holds the vault and decides what to send per request; `llama-server` is a stateless compute backend that never sees more than the current query's context. See "Data Architecture" section above for the full design, including the hardening steps (client-side embeddings, TLS via Caddy, swap handling, native KV-cache reuse for bulk-query performance).

### 2026-07-24 — GPU Passthrough Approach

**Decision: LXC cgroup device passthrough**, not VFIO/KVM. Rationale: this is a headless CUDA inference workload with no need for display output or full hardware isolation; VFIO adds VM overhead and IOMMU-group risk (consumer boards often bundle the GPU with PCH devices) for no real benefit here. LXC cgroup passthrough grants the container direct access to host-owned `/dev/nvidia*` device nodes — near bare-metal throughput, no hypervisor layer.

Host-side driver install (NVIDIA 580.173.02 via `.run` installer + DKMS, persistence daemon, udev rules) is complete. Full walkthrough, gotchas, and command history: **[Proxmox Host — NVIDIA Driver Setup](../setup/proxmox-nvidia-driver-setup.md)**.

Still open: the per-container wiring (privileged LXC creation, cgroup device permissions, CUDA toolkit version matched to host driver 580.173.02, llama-server CUDA build). Concrete prep notes — required config lines, base template tradeoff, and a known boot-race gotcha — are documented in [Proxmox Host — NVIDIA Driver Setup](../setup/proxmox-nvidia-driver-setup.md) under "Next Step: LXC GPU Wiring".

### 2026-06-24 — Model Selection


#### Chat model: Qwen3 8B Q4_K_M

**Source:** `bartowski/Qwen3-8B-GGUF` on Hugging Face.

Qwen3 8B is the current best-in-class 7–9B model for general Q&A on consumer hardware. Key properties:

- Q4_K_M GGUF file is ~5.0GB — fits within 6GB VRAM with room for KV cache at moderate context lengths
- Native thinking mode (chain-of-thought on demand), togglable per-request: `--chat-template-kwargs '{"enable_thinking":false}'` to disable for speed
- 128K context window (architecture supports it; VRAM limits practical usage to ~4K–6K tokens on this hardware)
- Apache 2.0 license
- Actively maintained GGUF builds

**Close second:** Llama 3.3 8B Q4_K_M (~4.9GB). Marginally better on English instruction-following; slightly worse on reasoning benchmarks. A one-line model path swap in llama-server if Qwen3 8B proves unsatisfactory.

#### Embedding model: nomic-embed-text-v1.5 (F16 GGUF)

~262MB, negligible VRAM footprint. Runs via the same llama-server binary with the `--embedding` flag. Properties:

- 8192-token context window — important for embedding longer note chunks without aggressive splitting
- Matryoshka Representation Learning: embedding dimensions can be truncated (768 → 512 → 256) with minimal quality loss, useful if vector storage becomes a concern later
- English-only; fine for this vault

**llama.cpp quirk:** The GGUF version defaults to 2048 token context. To unlock the full 8192, pass: `--rope-scaling yarn --rope-freq-scale .75`

The newer `nomic-embed-text-v2-moe` (~1.9GB) adds multilingual support and a MoE architecture — overkill for an English personal notes vault. Skip for now.

#### VRAM budget summary

| Role | Model | Quant | Approx VRAM |
|---|---|---|---|
| Chat | Qwen3 8B | Q4_K_M | ~5.0GB |
| KV cache (chat, 4K ctx) | — | — | ~0.3–0.5GB |
| Embeddings | nomic-embed-text-v1.5 | F16 | ~262MB |

Chat and embeddings are loaded sequentially via llama-swap, not simultaneously. Total never exceeds ~5.5GB for chat mode or ~262MB for embedding mode.

**llama-server context flag:** `-c 4096` is recommended as the starting cap. This leaves ~0.5–1GB of VRAM headroom. Can be relaxed to `-c 6144` experimentally, but watch VRAM usage before committing.

#### Context window expectations for RAG Q&A

Each RAG query turn is largely independent — conversation history does not accumulate the way it does in a long Claude.ai session. However, retrieved note chunks are injected into context per-turn and can add 2000–4000 tokens. Net expectation: **less total context pressure than a typical long Claude.ai session**, but more per-turn than a simple chat message. A 4096-token cap is almost certainly sufficient for this use case.

#### VRAM monitoring

```bash
# Live dashboard, refreshes every 2 seconds
watch -n 2 nvidia-smi

# Loggable output
nvidia-smi --query-gpu=memory.used,memory.free --format=csv -l 2
```

If VRAM is exhausted, llama.cpp begins offloading layers to CPU RAM over PCIe — throughput drops dramatically (reported 5–10× slowdown). A sudden crawl in response speed is the primary symptom; the above commands provide the smoking gun. No built-in alert system; a shell script checking `memory.free` below a threshold could be added later.

---

## Education for the human

**Decision order rationale:** Model selection and GPU passthrough setup are largely independent. Model selection is constrained by VRAM (6GB, fixed regardless of passthrough method), so it was addressed first as a desk exercise while GPU passthrough work is pending.

**Why "8B" fits despite the apparent VRAM tightness:** The 7B/8B labels are marketing, not exact parameter counts. What matters for VRAM is the GGUF file size. Qwen3 "8B" (8.2B actual parameters) is architecturally efficient (GQA attention) and its Q4_K_M file is ~5.0GB — comparable to or smaller than some models labelled "7B." Always check file size, not the parameter label.

---

## Execution Log

- **2026-06-26 → 2026-07-24:** Proxmox host NVIDIA driver (580.173.02) installed and verified via DKMS; persistence daemon and udev rules in place. `etckeeper` installed on the Proxmox host to track `/etc` changes going forward. Full details: [Proxmox Host — NVIDIA Driver Setup](../setup/proxmox-nvidia-driver-setup.md).
- **2026-09-27:** Data architecture decided (stateless, client-held vault) — see Decisions Log. No infrastructure changed; this was a planning-only session.
- **Next:** create the inference LXC and wire GPU cgroup passthrough into it.
