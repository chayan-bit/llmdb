# 🧠 llmdb: Latent Control Flow & Exploit Framework for LLMs

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Rust](https://img.shields.io/badge/rust-nightly-orange.svg)](https://www.rust-lang.org/)

**llmdb** is a unified toolkit bridging the gap between traditional binary exploitation (like GDB and ROP chains) and Large Language Models (LLMs).

By treating LLM weights as a compiled, queryable graph database (via a custom fork of [LARQL](https://github.com/chrishayuk/larql)), this framework allows researchers and red teamers to pause inference, write directly to the residual stream, mathematically pin attention heads, and execute deterministic Latent-Space exploits on consumer hardware.

---

## 🛠️ The Tool Suite

This monorepo contains three highly coupled tools:

1. **`LCP` (Latent Control Plane):** The backend engine. A Python wrapper over our modified LARQL Rust inference engine. It enables direct write-access to context boundary residuals and introduces "Attention Pinning" to enforce deterministic execution paths.

2. **`llmdb` (The Debugger):** An interactive, GDB-style CLI for mechanistic interpretability. Step through transformer layers, set breakpoints on token probabilities, and live-patch model knowledge using `.vlp` overlays.

3. **`LatentROP` (The Exploit Orchestrator):** An offensive security tool. It scans the model's K-Nearest Neighbor (KNN) gate vectors to automatically discover "gadgets" (tool-execution pathways) and compiles them into undetectable latent-space payloads.

---

## 🏗️ Architecture & Folder Structure

```text
llmdb/
├── external/
│   └── larql/           # [SUBMODULE] Our custom fork of the LARQL inference engine
├── lcp/                 # The Latent Control Plane API
├── llmdb/               # Interactive CLI debugger (GDB for LLMs)
└── latent_rop/          # Gadget discovery and payload compiler
```

Unlike traditional mechanistic interpretability tools (e.g., TransformerLens) that require massive GPU VRAM arrays, llmdb operates on CPU/Apple Silicon using zero-copy memory mapping (`mmap`). A 7B parameter model's layers are loaded instantly from disk.

---

## 🚀 Installation & Setup

Because this project relies on a deeply integrated Rust submodule, you must clone recursively.

### 1. Clone the Repository

```bash
git clone --recursive git@github.com:chayan-bit/llmdb.git
cd llmdb
```

_(If you already cloned it without `--recursive`, run: `git submodule update --init --recursive`)_

---

### 2. Build the Custom Inference Engine

You need Rust installed to compile the modified LARQL backend (which includes Attention Pinning).

```bash
cd external/larql
cargo build --release
pip install -e .
cd ../../
```

---

### 3. Install Python Dependencies

```bash
pip install -r requirements.txt
```

---

## 📖 Quick Start Examples

### 1. The llmdb Debugger

Launch an interactive debugging session against a local vindex model database:

```bash
python -m llmdb.cli --model gemma3-4b.vindex
```

#### Example Session

```text
(llmdb) watch "Sydney"
(llmdb) break when prob("Sydney") > 0.50
(llmdb) run "The capital of Australia is"
...
[Breakpoint 1 Hit] at Layer 18!
(llmdb) trace --decompose target="Sydney"
(llmdb) patch insert ("Australia", "capital_of", "Canberra") --layer 18 --boost 100
(llmdb) step
```

---

### 2. Finding Gadgets with LatentROP

Scan the model to find the exact multi-dimensional vectors required to force the model to output a specific JSON tool-call schema:

```bash
python -m latent_rop.scanner --model gemma3-4b.vindex --target_token '{"'
```

**Output:**

```text
[LatentROP] Discovered 3 FFN Gadgets at Layer 33.
Gadget 1 Trigger Vector: [-0.012, 0.441, ... ] (Saves to gadgets/tool_call.npy)
```

---

### 3. Executing a Latent Exploit

Compile the discovered gadgets into a Boundary Context file and bypass standard text guardrails:

```bash
python -m latent_rop.compiler --chain "tool_call -> exfiltrate" --out exploit.bndx
python -m lcp.engine --inject exploit.bndx
```

---

## 🗺️ Roadmap

- [x] Integrate LARQL Vindex architecture via Git Submodules.
- [ ] Phase 1: Implement BoundaryWriter latent payload injection in LCP.
- [ ] Phase 2: Modify larql-inference Rust crate to support Attention Pinning masks.
- [ ] Phase 3: Build out the cmd REPL loop for llmdb.
- [ ] Phase 4: Develop the Reverse-KNN gadget scanner for LatentROP.

---

## ⚠️ Ethical Disclaimer

This framework is built strictly for educational purposes, mechanistic interpretability research, and authorized red-teaming. The latent-space vulnerabilities and exploit chains (LatentROP) researched here are designed to help AI safety engineers understand the fundamental vulnerabilities in Transformer architectures.

**Do not use this tool to attack, manipulate, or extract data from commercial LLM endpoints or systems you do not own or have explicit permission to test.**
