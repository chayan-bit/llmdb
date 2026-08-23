# CLAUDE.md - llmdb

Project-specific instructions for Claude Code.
Extends the global `~/.claude/CLAUDE.md`; does not replace it.

## What this project is

llmdb is a **mechanistic-interpretability and latent-space safety-auditing toolkit for local LLMs**.
It treats transformer weights as a queryable graph database and provides a GDB-style workflow to inspect, trace, and surgically edit model internals on CPU / Apple Silicon, no GPU required.

It is built on top of [LARQL](https://github.com/chrishayuk/larql) (Apache-2.0), which already provides the inference engine, the `vindex` weight format, residual/activation hooks, and a Python binding surface.
llmdb is the tooling and product layer on top of those primitives.

## Positioning (read before writing docs or marketing copy)

The defensible product is **"GDB for local LLMs + pre-release safety auditing of open-weight models."**
Lead with the interpretability and auditing story, not an offensive-exploit story.

The offensive-sounding component (`LatentROP`) is reframed as a **defensive latent-gadget auditor**:

- A **latent gadget** is an internal pathway (an FFN neuron or gate direction) that reliably pushes the model toward an unsafe behavior (emit an unintended tool-call, comply with a jailbreak, leak context).
- The tool runs a **red-then-blue loop**: discover gadgets, demonstrate a chain that triggers the behavior, then **mitigate** (ablation overlay) and **monitor** (runtime latent IDS).
- The valid threat model is **white-box auditing of open-weight models you ship or own** - finding which internal circuits can be triggered to misbehave, and patching them before release.
  Do NOT frame this as attacking remote/commercial APIs or systems the user does not control. That framing fails a threat-model check and has no real audience.

When in doubt, the defensive/auditing framing wins over the exploit framing.

## Architecture

```text
llmdb/
├── external/larql/   # vendored fork of LARQL (Apache-2.0) - inference + vindex + hooks
├── lcp/              # Latent Control Plane: thin Python wrapper over LARQL hooks
├── llmdb/            # Interactive GDB-style CLI debugger (the headline tool)
└── latent_rop/       # Defensive latent-gadget auditor (scanner, eval, mitigation, monitor)
```

LARQL already exposes (via `larql._native.WalkModel`):
`capture_residuals`, `forward_with_capture`, `forward_steer`, `forward_ablate`, `patch_activations`, `logit_lens`, `project_through_unembed`, `gate_knn`, `embedding_neighbors`, `generate_with_hooks`, plus `.vlp` patch overlays.

**Build on these. Do not reimplement them.**
The only known primitive LARQL lacks is mid-forward attention mask/bias surgery ("attention pinning"); that is the one item requiring Rust work in `larql-inference`, and it should be deferred and ideally contributed upstream.

## Build order (Python-first, Rust last)

1. **Scanner** - rank gate/FFN vectors by causal effect on a target-behavior token (`gate_knn` + `logit_lens` candidates, verified causally with `forward_ablate`). Pure Python.
2. **Eval harness** - measure perplexity / small-benchmark delta so any mitigation is provably non-destructive. Non-negotiable.
3. **Mitigation overlay generator** - compile unsafe gadgets into a `.vlp` patch that neutralizes them.
4. **Latent IDS** - runtime monitor that flags when activations traverse known gadget directions. The real differentiator.
5. **Chain compiler** - order gadgets into a multi-step pathway, verify via `forward_steer`.
6. **Debugger REPL** - thin wrapper over the existing hooks; ship the headline `why "<token>"` / `patch` demo early.
7. **Attention pinning (Rust)** - only if steering + ablation prove insufficient.

Steps 1-6 require zero Rust.
Do not start with Rust or with reimplementing primitives that already exist.

## LARQL dependency rules (CRITICAL)

LARQL is **Apache-2.0**. When working with `external/larql/`:

- Keep its `LICENSE` and any `NOTICE` file intact; retain all copyright/attribution notices.
- State modifications: record changes in a `NOTICE` or `CHANGES` file in the fork.
- Do NOT strip git history to "detach" or relabel it as original work.
- Prefer contributing the attention-hook addition upstream to reduce single-maintainer bus-factor risk.
- Your own code in `lcp/`, `llmdb/`, `latent_rop/` is yours and may carry your own license; only LARQL-derived code is bound by Apache-2.0 attribution.

## Conventions

- Follow all global rules in `~/.claude/CLAUDE.md` and `~/.claude/rules/common/`.
- TDD: failing test first, then implement; target >=80% coverage.
- Many small files (200-400 lines typical, 800 max); functions < 50 lines.
- Immutable patterns; validate all model/file inputs at boundaries.
- Conventional commits (`feat:`, `fix:`, ...); no Co-Authored-By trailers.
- Never use the em dash; use a plain dash.
- In long Markdown, put each full sentence on its own line.
- Keep the ethical/threat-model framing accurate in every doc and demo: open-weight auditing only.
