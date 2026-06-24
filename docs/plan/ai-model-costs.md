# OpenCode Model Selection — Cost/Quality Analysis

> **Status:** Reference / Decision doc
> **Last updated:** 2026-06-24

---

## Context

OpenCode is used for all AI-assisted work on this homelab project, routed through
OpenRouter. The default model (`anthropic/claude-sonnet-latest`, currently resolving
to Claude Sonnet 4.5) produces excellent results but may be over-specified for the
actual work being done. This document records a cost/quality analysis conducted
2026-06-24 to identify cheaper alternatives worth trialling.

---

## Task Profile for This Project

Understanding what tasks the model actually performs shapes which tradeoffs matter:

| Task category | Frequency | Complexity |
|---|---|---|
| Config writing (Caddyfile, Docker Compose, systemd) | High | Low–medium |
| Sysadmin shell commands (install, service mgmt, SSH) | High | Low |
| Markdown documentation (runbooks, changelogs) | High | Low |
| Debugging (log interpretation, misconfig diagnosis) | Medium | Medium |
| Technical planning (GPU passthrough, local AI stack) | Low | Medium–high |

The work is **not** pushing the frontier of reasoning. There is no complex
algorithmic code, multi-file refactoring, or difficult mathematics. The primary
risk when using a cheaper model is subtle misconfiguration that leads to debugging
overhead — not catastrophic failure.

---

## Baseline: Current Model

| Model | OpenRouter ID | Input | Output |
|---|---|---|---|
| Claude Sonnet 4.5 | `anthropic/claude-sonnet-latest` | $3.00/1M | $15.00/1M |

Pricing as of 2026-06-24. Most cost is on output tokens.

---

## Candidates Evaluated

### 1. Google Gemini 2.5 Flash

| | |
|---|---|
| **OpenRouter ID** | `google/gemini-2.5-flash` |
| **Input price** | $0.30 / 1M tokens |
| **Output price** | $2.50 / 1M tokens |
| **Output cost reduction** | ~83% cheaper than Sonnet 4.5 |
| **Context window** | 1M tokens |
| **Released** | Jun 17, 2025 |
| **Knowledge cutoff** | Jan 2025 |

**Strengths:**
- Built-in "thinking" (reasoning) mode — controllable depth
- 1M context window is useful for long sessions with CLAUDE.md + full doc history
- Competitive SWE-bench scores; strong on coding and sysadmin tasks
- Largest cost reduction of the options reviewed

**Weaknesses / caveats:**
- Google model, not Anthropic — different behavioral patterns and tone
- Knowledge cutoff is Jan 2025 (via OpenRouter); all tech in use here predates that

**Verdict:** Best value option. The 83% output cost reduction is significant.
Try for routine config/docs sessions first.

---

### 2. Claude Haiku 4.5

| | |
|---|---|
| **OpenRouter ID** | `anthropic/claude-haiku-4-5` |
| **Input price** | $1.00 / 1M tokens |
| **Output price** | $5.00 / 1M tokens |
| **Output cost reduction** | ~67% cheaper than Sonnet 4.5 |
| **Context window** | 200K tokens |
| **Released** | Oct 15, 2025 |
| **Knowledge cutoff** | Jan 2025 |

**Strengths:**
- Same provider (Anthropic) — identical behavior, tone, tool-use patterns
- Anthropic claims Haiku 4.5 "matches Claude Sonnet 4's performance" on coding tasks
- SWE-bench Verified score >73%
- Lowest-risk swap if staying on Anthropic is a priority

**Weaknesses / caveats:**
- 200K context window (vs 1M on Sonnet 4.5 and Gemini 2.5 Flash)
  This is unlikely to be a constraint for typical sessions here.
- Extended thinking support exists but is a separate billable feature

**Verdict:** Lowest-risk alternative. Same provider, 67% cheaper output.
Good first candidate to trial if Google models are undesirable.

---

### 3. DeepSeek V3 0324

| | |
|---|---|
| **OpenRouter ID** | `deepseek/deepseek-chat-v3-0324` |
| **Input price** | $0.20 / 1M tokens |
| **Output price** | $0.77 / 1M tokens |
| **Output cost reduction** | ~95% cheaper than Sonnet 4.5 |
| **Context window** | 164K tokens |
| **Released** | Mar 24, 2025 |
| **Knowledge cutoff** | Jul 2024 |

**Strengths:**
- Exceptional price-to-performance ratio
- 685B MoE model — large effective capacity
- Strong coding benchmark scores

**Weaknesses / caveats:**
- Chinese model with data sovereignty implications. This project handles SSH
  keys, service credentials, and infrastructure topology — even via OpenRouter
  as intermediary, this warrants caution.
- Knowledge cutoff Jul 2024 (older than the others)
- Agentic tool-use reliability is somewhat less consistent than Anthropic models
- **Recommendation: do not use for sessions that involve SSH access to LXCs
  or any context involving actual credentials or key material.**

**Verdict:** Viable for offline/planning sessions (writing docs, reviewing config
snippets without secrets). Not recommended for agentic sessions touching live
infrastructure.

---

## Summary Comparison

| Model | Output price | vs Sonnet 4.5 | Context | Notes |
|---|---|---|---|---|
| Claude Sonnet 4.5 (current) | $15.00/1M | baseline | 1M | — |
| Gemini 2.5 Flash | $2.50/1M | **−83%** | 1M | Built-in thinking; Google |
| Claude Haiku 4.5 | $5.00/1M | **−67%** | 200K | Same provider; safe swap |
| DeepSeek V3 0324 | $0.77/1M | **−95%** | 164K | Privacy caveat; no live infra |

---

## Recommendations

1. **First trial:** `anthropic/claude-haiku-4-5` — lowest-risk swap, same
   provider, 67% output cost reduction. Test on a few config/docs sessions.

2. **If comfortable switching providers:** `google/gemini-2.5-flash` — 83%
   output cost reduction, equivalent or better context window. The built-in
   thinking mode is a genuine advantage for debugging sessions.

3. **Upgrade path:** Keep `anthropic/claude-sonnet-latest` available for the
   GPU passthrough and local AI stack implementation work (the most
   technically demanding upcoming tasks). Downgrade once those are stable.

4. **Avoid DeepSeek for agentic sessions** involving live infrastructure or
   any context where credentials appear.

---

## How to Change the Model in OpenCode

Edit `~/.config/opencode/opencode.jsonc` and set the `model` key:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-haiku-4-5"    // or "google/gemini-2.5-flash"
}
```

Restart OpenCode after editing. To revert to the current default, remove the
`model` key or set it back to `anthropic/claude-sonnet-latest`.
