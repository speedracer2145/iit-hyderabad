# Heimdall: Formally Verified Automated Migration of Legacy eBPF Programs to Rust

**Authors:** Vasu Dasu, Monika Sharma, Rafiur Rashid, Aziz Khan, Saeid Tizpaz-Niari, Gang Tan  
**Institutions:** Pennsylvania State University; University of Illinois Chicago  
**Year:** 2026 | **Venue:** arXiv preprint (arXiv:2605.25411)  
**DOI/Link:** https://arxiv.org/abs/2605.25411  
**License:** CC BY 4.0

---

## 1. Overview

eBPF programs are small kernel extensions that enable high-performance networking, observability, and security enforcement in the Linux kernel. Traditionally written in C using the `libbpf` framework, these programs pass through the kernel's in-kernel eBPF verifier, which enforces low-level memory safety and bounded execution. However, the verifier was not designed to catch source-level bugs: struct fields left uninitialized before emission to userspace, helper return values silently discarded, buffer-size mismatches, hook-context type confusion, map schema inconsistencies, and signed/unsigned integer confusion all compile cleanly and pass the verifier without any complaint. The paper discovers and demonstrates six such bug classes in production tools, including previously unreported KASLR-defeat information leaks and cross-event ring-buffer residue leaks in ten widely deployed open-source eBPF programs (e.g., `bashreadline`, `mountsnoop`, `opensnoop`).

The paper presents **Heimdall**, an automated pipeline that tackles this problem by translating legacy `libbpf` C eBPF programs into memory-safe Rust using the `Aya` framework, while providing a **formal, per-program guarantee** that the translated program preserves the observable behavior of the original. The core insight is that Rust's type system and Aya's typed API surface directly close many of these bug classes at compile time (e.g., `error[E0381]` for uninitialized fields, `error[E0308]` for hook/context mismatches), while a custom symbolic execution + Z3 equivalence checking backend provides the rigorous behavioral guarantee that the LLM-generated translation did not introduce semantic drift.

Heimdall is the **first system to automate memory-safe-language migration of production eBPF programs with per-program formal guarantees** that the migration preserves observable behavior. In an evaluation on 102 real-world eBPF programs, it produces 96 (94.1%) formally proven-equivalent translations, and discovers and closes all six bug classes in every affected program in the verified corpus.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

Heimdall is a five-stage iterative pipeline that combines LLM-driven code translation with formal verification. The key insight is to operate on **compiled eBPF bytecode** rather than at the C or Rust source level for equivalence checking, eliminating source-level semantic discrepancies (ownership moves, integer promotion, trait resolution, panic semantics) that do not survive compilation.

The pipeline can be instantiated in two modes:
- **Deterministic**: An automaton-style controller explicitly drives stage transitions; the LLM acts as a stateless translator that receives structured feedback from the failing stage.
- **Agentic**: The same five-stage logic is followed implicitly by a code agent (Claude Code, Codex, Gemini-CLI) with full tool-mediated reasoning — file browsing, bytecode inspection, subagent delegation, in-context learning from prior verified translations.

The agentic mode dramatically outperforms the deterministic mode (73–98% vs. 47–63% triple-pass success on the benchmark set), and both substantially outperform a raw code-agent baseline (4–12%).

### 2.2 Process Steps

**Stage 1 — LLM Translation:**
- Input: C eBPF source file + entry point symbol name + map type annotations.
- An LLM (e.g., Claude claude-opus-4.6) translates the C source into idiomatic Aya Rust, guided by a prompt containing Aya API mappings, hook macro tables, map access patterns, and a complete working example.
- Output: A candidate Rust source file.

**Stage 2 — Compile and Kernel Verify:**
- The Rust source is compiled to eBPF bytecode with strict lint guards: `#![deny(unused_unsafe)]`, `#![deny(undocumented_unsafe_blocks)]`, `#![deny(multiple_unsafe_ops_per_block)]`, `#![deny(unused_must_use)]`.
- Compiler errors are fed back to the LLM in an inner retry loop.
- On successful compilation, the binary is loaded into the actual Linux kernel eBPF verifier (bounded loops, memory safety, restricted helpers).
- Kernel verifier rejections are also fed back to the LLM.
- Output: A compiled, kernel-verifier-accepted `.o` ELF binary.

**Stage 3 — Static Analysis Safety Engine:**
- A custom source-level safety analyzer applies pattern checks, regex bans, and file-level invariants on the Rust source to reject unsafe patterns that both the Rust compiler and the kernel verifier would miss.
- Key rules driven by the two dominant bug classes:
  - **Uninitialized state**: Bans untyped `RingBuf::reserve::<[u8; N]>` and `RingBufEntry<[u8; N]>` (forcing typed reservations with mandatory zero-fill), catches missing struct fields via `error[E0063]`.
  - **Unchecked helper returns**: Rejects `let _ = ...` and `.ok()` discards on failable Aya helpers; forces `?`-propagation or explicit `Ok/Err` matching.
- Output: Safety-policy-compliant Rust source.

**Stage 4 — Symbolic Execution:**
- Both the original C binary and the compiled Rust binary are loaded into a **custom eBPF backend for `angr`** (built from scratch, as no prior tool supported realistic eBPF bytecode analysis).
- The backend performs fully symbolic exploration of all feasible execution paths.
- For each terminated path, a **formula generator** emits a structured summary: path predicate (conjunction of branch conditions), return value (register R0), and final map state (including mutable ELF globals).
- Output: Sets of symbolic path summaries for C and Rust binaries.

**Stage 5 — Equivalence Checking:**
- The equivalence checker submits the symbolic formulae to Z3 to determine if the C and Rust programs are equivalent for all inputs.
- Map state is encoded using an **ITE-chain encoding** (If-Then-Else chains over bitvectors) that captures last-write-wins semantics under symbolic key aliasing.
- A **conditional equivalence** strategy handles safety-improving translations: equivalence is required only on inputs satisfying declared safety conditions (helper-success paths; output-sink bytes excluded from comparison).
- If Z3 returns UNSAT → translation is verified. If SAT → a concrete counterexample is formatted, divergence is classified (sign extension, truncation, missing/extra write, etc.), and the counterexample is fed back to the LLM for targeted repair, re-entering the pipeline at Stage 2.
- Output: Formal equivalence proof or actionable counterexample.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in Heimdall |
|---|---|
| **Large Language Models** (Claude claude-opus-4.6/claude-sonnet-4.6, GPT-5.4, Gemini-3-flash) | Translate C eBPF source to Rust Aya; iteratively repair based on structured feedback from each pipeline stage |
| **Aya** (Rust eBPF framework) | Target language and typed API surface; its generic map types and typed hook macros enforce compile-time schema correctness and hook-context matching |
| **Rust compiler** (rustc 1.93.1) | Enforces ownership, initialization (E0381, E0063), type safety, and lint guards on the generated Rust source |
| **Linux eBPF kernel verifier** | Validates bytecode-level memory safety, bounded loops, and restricted helper usage; rejects non-conforming translations |
| **angr** (binary analysis platform, v9.2) | Symbolic execution framework; Heimdall extends it with a full eBPF backend across 5 layers (CLE loader, archinfo, VEX lifter, SimOS dispatch, formula generator) |
| **Z3 SMT solver** (v4.13) | Checks unsatisfiability of the negated equivalence formula; provides concrete counterexamples when SAT |
| **CEGIS** (Counter-Example-Guided Inductive Synthesis) | Conceptual framework: LLM as synthesizer, Z3 as verifier producing counterexamples that guide the next candidate |
| **ITE-chain encoding** | Custom bitvector encoding of map state (instead of Z3 Theory of Arrays, which is incompatible with angr/Claripy's bitvector model) |
| **Conditional semantic equivalence** | Formal equivalence definition that scopes comparison to safety-valid inputs, allowing safety-improving Rust translations to coexist with bytecode-level equivalence proofs |
| **tree-sitter-rust** | Static count of unsafe operations in generated Rust source (raw pointer dereferences, calls inside `unsafe`, inline assembly, mutable-static reads) |
| **clang 14.0.0 / bpftool v7.4.0 / libbpf v1.4** | Compilation toolchain for the original C libbpf programs |
| **Python 3.12** | Pipeline orchestration and formula post-processing |

### 2.4 Key Data Structures / Models

- **eBPF ELF backend (angr CLE extension):** Resolves eBPF-specific relocations — map references become runtime handles, BPF-to-BPF call targets become reachable addresses, read-only constants and mutable globals are bound to memory regions or stubs. Auto-detects program type (kprobe, tracepoint, XDP, TC, socket-filter, LSM, fentry/fexit, uprobe) from ELF section names.

- **eBPF Architecture Definition:** A 64-bit eBPF architecture registered with angr (`archinfo`), declaring 10 general-purpose registers (R0–R9), stack pointer (R10), and auxiliary state for instruction flow and BPF-to-BPF call depth tracking.

- **VEX IR Lifter for eBPF:** Decodes the eBPF instruction format and lifts to angr's VEX IR. Covers the full eBPF ISA: 32-bit and 64-bit ALU/bitwise operations, conditional/unconditional control flow (including helper calls and BPF-to-BPF calls), load/store at all standard widths (8/16/32/64-bit), and atomic memory operations (`__sync_fetch_and_add`, etc.).

- **Helper Stubs (SimOS models):** Models for every BPF helper a program may call. Helpers that read kernel state return symbolic bitvectors under a stable naming convention shared between C and Rust binaries (e.g., `input_pid_tgid`, `input_ktime`), ensuring both programs are evaluated under identical symbolic kernel state during equivalence checking.

- **Symbolic Path Summary:** Per-terminated-path triple `(φᵢ, rᵢ, Mᵢ')` where `φᵢ` is the path predicate (conjunction of branch guards), `rᵢ` is the return value (R0 register contents, 64-bit bitvector), and `Mᵢ'` is the final map state (sequence of map-entry snapshots including mutable ELF globals).

- **ITE-chain Map Model:** Two-level encoding for each BPF map:
  - *Inner chain* (per path): Walk the ordered write trace newest-first; return the first write whose key matches and whose post-state is present (not deleted). Falls through to the shared initial map value.
  - *Outer chain* (per program): Selects the inner chain corresponding to the path predicate `φⱼ` that is satisfied by the current input.
  - *Presence chain* `Pₘ(kq)`: Parallel structure that returns the presence bit (1 for insert, 0 for delete), supporting `delete_elem` semantics.
  - Supports 20 BPF map types across four categories: hash-like (8 types, including `lru_hash`, `lpm_trie`, `sockhash`), array-like (8 types, integer-keyed), output sinks (`perf_event_array`, `ringbuf`), and map-of-maps (2 types).

- **Variable Unification Namespace:** Shared variables (kernel state, initial map contents, key-existence conditions) are mapped to identical Z3 variables for both C and Rust binaries. Distinct output variables carry per-program suffixes (`output_r0_c` vs. `output_r0_rust`).

- **Conditional Equivalence Predicate `Φ_safe`:** Conjunction of two safety conditions:
  - `Φ_safe^(i)` (Helper-success): Equivalence required only on inputs where every helper call succeeds.
  - `Φ_safe^(ii)` (Output-sink opacity): Per-event bytes emitted to `perf_event_array`/`ringbuf` sinks are excluded from comparison (allowing zero-fill safety improvements to coexist with C originals that emit stale bytes).

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

Heimdall targets **eBPF programs** broadly, including those used as network functions. The evaluation dataset of 102 programs spans:
- **Network-path programs (XDP / TC):** XDP packet processors (DAE transparent proxy, Fluvia IPFIX exporter, Hercules high-speed bulk transfer, CRAB, XDP-FW firewall, hXDP FPGA NIC offload, BMC in-kernel Memcached cache), TC filters (Suricata XDP filter, Suricata XDP load balancer, BMC tx_filter, update_cache), socket-filter programs.
- **Tracing / observability programs (kprobe / tracepoint / uprobe / fentry-fexit):** libbpf-tools collection (bashreadline, mountsnoop, opensnoop, execsnoop, tcplife, etc.), libbpf-bootstrap tutorial programs, KEN tracing dataset.
- **Security enforcement (LSM):** LSM hook programs.

For network-function purposes, the primary NF types targeted are **XDP/TC packet processors** (stateless forwarding, load balancing, caching, IPFIX export) and **network IDS/IPS filters** (Suricata eBPF offload).

### 3.2 How It Validates NF Behavior

1. **Input:** C eBPF source file + entry-point symbol + map type annotations.
2. **LLM translation:** Produces candidate Rust Aya source.
3. **Compilation + kernel verifier:** Rejects syntactically or structurally invalid translations.
4. **Static safety engine:** Rejects unsafe patterns that escape the Rust type system.
5. **Symbolic execution of both binaries:** The C ELF and Rust ELF are each symbolically executed under a fully symbolic initial context (packet buffer, program context, initial map state, helper return values). All feasible paths are explored exhaustively.
6. **Z3 equivalence check:** Z3 asserts unsatisfiability of the conjunction of the safety condition and the negation of the equivalence formula. If UNSAT, the translation is formally verified. If SAT, the satisfying assignment is a concrete input that causes observable divergence.
7. **Counterexample-guided repair:** Concrete diverging inputs are classified and fed back to the LLM for targeted repair; the loop repeats until equivalence is proved or resource limits are exceeded.

### 3.3 What Properties / Invariants Does It Prove?

The paper proves **conditional semantic equivalence** between C and Rust programs. Concretely:

- **Return value equivalence:** For every input satisfying `Φ_safe`, the return value of the C program (register R0) equals the return value of the Rust program. For XDP programs, this means the packet verdict (XDP_PASS / XDP_DROP / XDP_TX / XDP_REDIRECT) is identical.
- **Map state equivalence:** For every input satisfying `Φ_safe` and for every map and query key, the final map state (presence + value) is identical between C and Rust. This captures all write-side effects: `bpf_map_update_elem`, `bpf_map_delete_elem`, pointer-level mutations via `bpf_map_lookup_elem`.
- **Global state equivalence:** Mutable ELF globals are tracked and included in the equivalence query alongside maps.
- **Entry-point type compatibility:** BPF program type is verified to match between C and Rust ELF section names before symbolic execution.
- **Atomic operation preservation:** A supplementary static bytecode scan verifies that atomic operations (`__sync_fetch_and_add`, etc.) are preserved in count.
- **Safety improvements (closed bug classes, separately verified):**
  - Uninitialized struct fields → Rust `error[E0381]`/`error[E0063]` prevents emission.
  - Unchecked helper returns → Aya `Result<T, E>` forces explicit handling.
  - Buffer/size mismatches → Aya typed APIs (`PerfEventArray<T>::output`) derive size from type parameter.
  - Hook/context mismatches → Aya typed hook macros produce `error[E0308]`.
  - Map schema confusion → Aya generics (`HashMap<K,V>`, `Array<A>`) enforce key/value types.
  - Signed/unsigned confusion → `StackTrace::get_stackid()` returns `Result<i64, i64>`.

### 3.4 Input Requirements

| Requirement | Details |
|---|---|
| **C eBPF source code** | The `.c` libbpf source file for the program to be migrated; must be compilable with clang |
| **Entry point symbol name** | The name of the BPF program entry function (e.g., `xdp_prog`, `kprobe__sys_read`) |
| **Map type annotations** | List of BPF map names with their map types (used to guide the LLM and configure the angr symbolic execution backend) |
| **Running Linux kernel** | Required for Stage 2 kernel verifier validation (Linux kernel 6.8.0 in evaluation) |
| **No annotations / specifications required** | The system does not require user-provided invariants, loop bounds, or behavioral specifications |

### 3.5 Guarantees Provided

- **Formal equivalence proof (SMT-based):** When Z3 reports UNSAT, the guarantee is that for *all* possible inputs satisfying `Φ_safe`, the C and Rust programs produce identical return values and identical map/global state. This is a **sound mathematical proof** over the symbolic execution model of the eBPF bytecode.
- **Conditional scope:** The guarantee is conditional — it applies only on helper-success inputs and excludes per-event output-sink bytes. On excluded paths, safety is separately validated (the Rust translation is verified to behave safely).
- **Scalability caveat:** For 3/102 programs, solver resource limits prevented reaching a verdict (partial verification for 3 more). The guarantee therefore does not hold for these 6 programs.
- **Runtime validation:** An independent 1,000-trial runtime validation on 10 programs confirms that formally equivalent programs also exhibit identical observable behavior in execution.

---

## 4. NF Chain Verification

**Heimdall does not address NF chains.** It is a **single-program** verification framework: it takes one C eBPF source file, produces one Rust eBPF program, and proves equivalence between that pair. The concept of multiple eBPF programs composing an NF service chain — e.g., an XDP firewall followed by a load balancer — is not addressed.

Extending Heimdall to chains would require:
1. **Multi-program composition semantics:** Modeling how the output of one eBPF program (packet verdict, modified packet buffer, shared map state) becomes the input context for the next program in the chain.
2. **Cross-program map consistency:** BPF maps are often shared between programs in a chain; equivalence checking would need to account for the interleaved map writes of multiple programs under a shared global map state.
3. **Chain-level behavioral properties:** Properties such as end-to-end packet-drop correctness, forwarding invariants, or ordering guarantees that span the entire chain rather than individual programs.
4. **TC classifier chaining:** For TC programs that use `TC_ACT_PIPE` to invoke the next action, the chain composition is explicit in the BPF program structure, but Heimdall does not model this.

None of these extensions are discussed in the paper.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs **static control-flow graph (CFG) and network-context (NC) dataflow analysis directly on raw eBPF bytecode** — no source code required. It extracts a Prolog-compatible representation of the eBPF program's behavioral semantics (packet field accesses, BPF map read/write patterns, helper call sequences, branch structures) and provides a **Prolog query engine** for asserting and checking behavioral invariants over the extracted representation. It handles **NF chains** (sequences of eBPF programs) and is aware of **stateful BPF map** usage across programs in the chain.

### 5.2 Key Differences from This Paper

| Dimension | Heimdall | Yaksha-Prashna |
|---|---|---|
| **Primary goal** | C-to-Rust migration correctness; behavioral equivalence between two implementations | Behavioral assertion checking on deployed eBPF NF programs; NF chain property verification |
| **Input required** | C source code + entry-point symbol + map type annotations | Raw eBPF bytecode (ELF) — no source code needed |
| **Verification technique** | Symbolic execution (angr) + Z3 SMT (equivalence checking) | Static CFG/NC dataflow analysis + Prolog query engine |
| **Property checked** | Semantic equivalence between C and Rust versions (same return value + map state for all inputs) | Behavioral assertions (packet-drop freedom, access control correctness, forwarding invariants, BPF map usage patterns, chain-level properties) |
| **NF chains** | Single program only | Multi-program chains natively supported |
| **State modeling** | ITE-chain encoding of map state under symbolic keys; full symbolic execution across all paths | Dataflow analysis of BPF map accesses (key patterns, value types, read/write sequences); Prolog-encodable map usage facts |
| **LLM involvement** | Central: LLM performs code translation and is in the CEGIS repair loop | None |
| **Timing** | Offline (pre-deployment, during migration) | Offline (pre/post deployment, without source) |
| **Guarantee type** | Sound SMT proof of behavioral equivalence (for programs that finish within solver limits) | Assertion violations or proofs-by-query over the extracted behavioral model |
| **Bug classes discovered** | Source-level safety gaps (uninitialized state, unchecked helpers, etc.) | Behavioral correctness violations across the NF chain (forwarding errors, access control gaps, policy violations) |

### 5.3 How This Paper Is Useful For Us

1. **Motivational citation for eBPF source-level analysis limitations:** Heimdall's §2 catalogs six concrete bug classes that pass the kernel eBPF verifier without complaint. This directly motivates Yaksha-Prashna's approach of going beyond the kernel verifier to perform higher-level behavioral analysis. We can cite Heimdall to establish that "the kernel verifier is not sufficient for behavioral correctness" with concrete, published evidence.

2. **Evidence that source code is not needed for eBPF analysis:** Heimdall itself concludes that source-level comparison of C and Rust is problematic ("comparing two translations at their respective source levels would force reconciliation of Rust and C idiosyncrasies") and chooses to operate at the bytecode level. This directly supports our design choice to operate on raw eBPF bytecode.

3. **Novel angr eBPF backend:** Heimdall builds the first full eBPF backend for `angr` that handles maps, relocations, helpers, atomic operations, and BPF-to-BPF calls. This is a potential infrastructure contribution that Yaksha-Prashna might leverage or compare against for the symbolic analysis component. The ITE-chain map encoding in particular is a technically sound approach to BPF map modeling that we can reference.

4. **Orthogonal complementary technique:** Heimdall ensures a migrated Rust program is equivalent to its C source. Yaksha-Prashna ensures the deployed eBPF program (C or Rust, source-agnostic) satisfies behavioral chain-level properties. These are orthogonal guarantees and the two systems could be composed: Heimdall verifies migration correctness, Yaksha-Prashna verifies operational correctness.

5. **Baseline comparison point:** For any evaluation component of Yaksha-Prashna that touches single-program behavioral properties of XDP/TC programs, Heimdall's dataset of 102 programs (publicly released) and its symbolic execution infrastructure could serve as a comparison baseline.

6. **Identified scalability limitations:** Heimdall fails to verify 6/102 programs due to symbolic execution and solver resource limits. This highlights the scalability limitation of path-enumeration + SMT approaches for eBPF programs — a limitation that Yaksha-Prashna's static dataflow + Prolog approach is designed to avoid (trading completeness for scalability).

### 5.4 Positioning Statement

> "While Heimdall provides per-program behavioral equivalence proofs for eBPF C-to-Rust migration using symbolic execution and Z3 SMT solving, it requires C source code as input, is limited to single-program verification, and faces scalability constraints for complex programs with large path spaces. Yaksha-Prashna addresses these limitations by operating directly on raw eBPF bytecode without requiring any source code, scaling to NF chains of multiple programs through static CFG-NC dataflow analysis, and using a Prolog query engine to flexibly assert and check a broader class of behavioral invariants — including chain-level forwarding correctness, access control consistency, and stateful BPF map usage patterns — across the full NF pipeline."

---

## 6. Additional Notes

### Security Findings (Novel Contributions)

The paper reports **newly disclosed** information leaks in production eBPF tools:

- **bashreadline** (libbpf-tools): Stack-resident `struct str_t` (84 bytes) emitted via `bpf_perf_event_output` without zero-initialization. Trailing bytes contain per-CPU BPF stack residue including kernel-text return addresses. 100% of events in testing exposed KASLR-defeating kernel pointers; both `do_fault+0xf0` and `mtree_load+0x271` yielded the same KASLR slide of `0x0e600000` across independent test runs.

- **mountsnoop** and 9 other programs: Ring-buffer cross-event content leaks. Reserved slots submitted with only a subset of fields written; trailing kilobytes carry the previous traced record verbatim. In `mountsnoop`, UMOUNT-class events leak prior PID, command name, filesystem type, and full source/destination paths — including per-user systemd credential paths exposing which users triggered prior mount operations.

### Dataset Statistics

| Dataset | Programs | LoC range |
|---|---|---|
| libbpf-tools (BCC) | ~50 programs | Medium-high |
| libbpf-bootstrap | ~8 programs | Low-medium |
| KEN tracing | varied | Low-medium |
| Network NFs (DAE, Fluvia, Hercules, CRAB, XDP-FW, hXDP, BMC, Suricata) | ~12 programs | Medium-high |
| **Total evaluated** | **102 programs** | — |
| **Formally verified** | **96 (94.1%)** | — |

### Evaluation Results Summary

| Approach | Compile | KV Pass | Safety Pass | Equivalence | Triple-Pass |
|---|---|---|---|---|---|
| Baseline (code agents, no Heimdall) | 51/51 | 36–49/51 | N/A | N/A | 2–6/51 (4–12%) |
| Heimdall Deterministic | 51/51 | similar | similar | similar | 24–32/51 (47–63%) |
| Heimdall Agentic | 51/51 | similar | similar | similar | 37–50/51 (73–98%) |
| **Full-scale (Agentic, opus-4.6, 102 programs)** | **102/102** | — | — | — | **96/102 (94.1%)** |

> [!NOTE]
> Heimdall's agentic mode averages ~27.3 min/program and ~$2.94/program (using claude-opus-4.6), with a 2.89× bytecode overhead (Rust vs. C `.text` sections) and 18.3 unsafe operations per translation on average.
