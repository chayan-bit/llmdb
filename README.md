# 🧠 llmdb: A Latent Debugger & Safety Auditor for Local LLMs

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Rust](https://img.shields.io/badge/rust-nightly-orange.svg)](https://www.rust-lang.org/)

**llmdb is "GDB for local LLMs."**
Step through any transformer on your laptop, watch exactly why it predicts each token, surgically edit its knowledge, and audit the internal circuits that can make it misbehave - all on CPU / Apple Silicon, with no GPU.

It is built on top of [LARQL](https://github.com/chrishayuk/larql) (Apache-2.0), which treats model weights as a queryable graph database (a `vindex`) and exposes residual/activation hooks over a Rust engine with Python bindings.
llmdb is the interpretability, debugging, and safety-auditing layer on top of those primitives.

---

## 🛠️ The Tool Suite

This monorepo contains three coupled tools:

1. **`LCP` (Latent Control Plane):** A thin Python wrapper over LARQL's hook API. Read and write the residual stream, capture activations, steer, ablate, and apply `.vlp` patch overlays through one clean interface.

2. **`llmdb` (The Debugger):** An interactive, GDB-style CLI for mechanistic interpretability. Step through transformer layers, set breakpoints on token probabilities, trace why a token was predicted, and live-patch model knowledge.

3. **`latent_rop` (The Latent-Gadget Auditor):** A **defensive** safety tool. It discovers "latent gadgets" - internal pathways (FFN neurons / gate directions) that reliably push the model toward an unsafe behavior - demonstrates how they chain, and then helps you **mitigate** them with ablation overlays and **monitor** them at runtime.

---

## 🛡️ What "ROP-style" means here (defensive reframe)

The auditor borrows the *vocabulary* of binary exploitation (gadgets, chains) because the structure is analogous, but it is built strictly for **defensive, white-box auditing of open-weight models you own or ship.**

| Binary exploitation | Latent analogue in llmdb |
|---|---|
| Gadget (a reusable instruction sequence) | A latent gadget: an FFN neuron / gate direction that promotes an unsafe behavior |
| ROP chain | An ordered set of gadgets that, when activated, reproduces the behavior |
| Exploit | A demonstration that the chain triggers the behavior (the **red** step) |
| Patch / mitigation | An ablation `.vlp` overlay + a runtime latent monitor (the **blue** step) |

The workflow is **red then blue**:

1. **Discover** - rank gate/FFN vectors by their causal effect on a target-behavior token, then verify causally by ablation.
2. **Demonstrate** - chain the gadgets and confirm via steering that the behavior reproduces.
3. **Mitigate** - compile the unsafe gadgets into a `.vlp` overlay that neutralizes them.
4. **Monitor** - run a lightweight "latent IDS" that flags when inference traverses known gadget directions.
5. **Verify** - prove the mitigation does not degrade the model (perplexity / benchmark delta).

**Valid threat model:** you are auditing a model whose weights you control, to find and patch internal failure circuits *before release*.
**Out of scope:** attacking, manipulating, or extracting data from commercial endpoints or any system you do not own. That is neither supported nor a meaningful use of this tool.

---

## 🏗️ Architecture & Folder Structure

```text
llmdb/
├── external/
│   └── larql/           # [SUBMODULE] vendored fork of LARQL (Apache-2.0)
├── lcp/                 # Latent Control Plane: Python wrapper over LARQL hooks
├── llmdb/               # Interactive GDB-style CLI debugger
└── latent_rop/          # Defensive latent-gadget auditor
```

Unlike interpretability stacks that assume large GPU VRAM arrays, llmdb runs on CPU / Apple Silicon using LARQL's zero-copy memory mapping (`mmap`), so a multi-billion-parameter model's layers load from disk without a GPU.

### Built on existing LARQL primitives

llmdb does not reimplement the heavy machinery.
The following already exist in LARQL (`larql._native.WalkModel`) and llmdb composes them:
`capture_residuals`, `forward_with_capture`, `forward_steer`, `forward_ablate`, `patch_activations`, `logit_lens`, `project_through_unembed`, `gate_knn`, `embedding_neighbors`, `generate_with_hooks`, and `.vlp` patch overlays.

The one primitive LARQL does not expose is mid-forward attention mask surgery ("attention pinning"); it is deferred and, if needed, will be contributed upstream.

---

## 🚀 Installation & Setup

Because this project relies on a deeply integrated Rust submodule, you must clone recursively.

### 1. Clone the Repository

```bash
git clone --recursive git@github.com:chayan-bit/llmdb.git
cd llmdb
```

_(If you already cloned without `--recursive`, run: `git submodule update --init --recursive`)_

### 2. Build the Inference Engine

You need Rust installed to compile the LARQL backend.

```bash
cd external/larql
cargo build --release
pip install -e .
cd ../../
```

### 3. Install Python Dependencies

```bash
pip install -r requirements.txt
```

---

## 📖 Quick Start Examples

### 1. The llmdb Debugger

Launch an interactive session against a local vindex model:

```bash
python -m llmdb.cli --model gemma3-4b.vindex
```

```text
(llmdb) watch "Sydney"
(llmdb) break when prob("Sydney") > 0.50
(llmdb) run "The capital of Australia is"
...
[Breakpoint 1 Hit] at Layer 18!
(llmdb) why "Sydney"                 # decompose which layers/heads/FFNs promoted it
(llmdb) patch insert ("Australia", "capital_of", "Canberra") --layer 18 --boost 100
(llmdb) step
```

### 2. Auditing latent gadgets (defensive)

Find the internal pathways that push an open-weight model toward an unintended tool-call:

```bash
python -m latent_rop.scanner --model gemma3-4b.vindex --target_token '{"' --verify ablation
```

```text
[auditor] 3 candidate gadgets at Layer 33 (gate KNN), 2 confirmed causal by ablation.
Gadget 1 | layer 33 | neuron 8112 | Δlogit('{"')=+4.1 | ablation kills behavior: yes
```

### 3. Mitigate and verify

Compile the confirmed gadgets into a mitigation overlay, then prove it does not degrade the model:

```bash
python -m latent_rop.mitigate --gadgets gadgets/toolcall.json --out harden.vlp
python -m latent_rop.eval --model gemma3-4b.vindex --overlay harden.vlp   # perplexity / benchmark delta
```

---

## 🗺️ Roadmap

Build order is Python-first; Rust is last and optional.

- [x] Integrate LARQL `vindex` architecture via Git submodule.
- [ ] Phase 1: `LCP` wrapper over LARQL hooks (capture / steer / ablate / patch).
- [ ] Phase 2: `latent_rop` scanner - gadget discovery with causal (ablation) verification.
- [ ] Phase 3: Eval harness - perplexity / benchmark delta for provable mitigations.
- [ ] Phase 4: Mitigation overlay generator (`.vlp` emit).
- [ ] Phase 5: Latent IDS - runtime monitor for gadget-direction traversal.
- [ ] Phase 6: `llmdb` REPL - the headline `why` / `patch` debugging loop.
- [ ] Phase 7 (optional): Attention pinning in `larql-inference`, ideally upstreamed.

---

## ⚖️ Licensing

llmdb's own code (`lcp/`, `llmdb/`, `latent_rop/`) is MIT.
The vendored `external/larql/` is **Apache-2.0**: its `LICENSE` and `NOTICE` are retained, attribution is preserved, and modifications are documented per the license.

## ⚠️ Ethical Disclaimer

This framework is built strictly for mechanistic-interpretability research and **white-box auditing of open-weight models you own or ship**.
The latent-gadget analysis is designed to help engineers find and patch internal failure circuits before release.

**Do not use this tool to attack, manipulate, or extract data from commercial LLM endpoints or any system you do not own or have explicit permission to test.**
