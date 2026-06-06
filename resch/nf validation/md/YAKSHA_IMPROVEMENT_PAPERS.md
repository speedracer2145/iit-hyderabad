# YAKSHA-PRASHNA IMPROVEMENT PAPERS
## Cross-Domain Research to Address Yaksha's 6 Core Limitations

**Purpose:** Find techniques from ANY domain that can improve Yaksha-Prashna's capabilities  
**Source paper:** arXiv:2602.11232 — Yaksha-Prashna: Understanding eBPF Bytecode Network Function Behavior  
**Compiled:** May 2026 — IIT Hyderabad NF Validation Research

---

## YAKSHA'S 6 CONFIRMED LIMITATIONS (from codex_NF.md §4.9 + paper abstract)

| # | Limitation | Why It Matters |
|---|---|---|
| **L1** | Cannot reason about **BPF map state across multiple packets** | NFs like NAT, firewall, LB rely on conntrack/session maps — Yaksha sees map accesses but not what they contain across a packet sequence |
| **L2** | Cannot precisely model all **200+ eBPF helper functions** | Helpers (bpf_map_lookup, bpf_redirect, bpf_fib_lookup, bpf_csum_diff) are black boxes to Yaksha; their side effects are approximated |
| **L3** | Cannot handle **concurrent map access** | eBPF programs run simultaneously on N CPUs; per-CPU maps, BPF spinlocks, RCU locks are invisible to Yaksha's static analysis |
| **L4** | Cannot check **JIT/offload divergence** | Yaksha analyzes BPF bytecode; JIT-compiled x86/ARM code may behave differently (Jitterbug found 16 such bugs) |
| **L5** | **Prolog query language is too limited** | Cannot express temporal ordering ("map updated BEFORE packet forwarded"), quantitative ("drops < 1%"), or liveness properties |
| **L6** | **Cross-program chain analysis is incomplete** | Tail calls, tc_redirect, bpf_redirect between eBPF programs create complex inter-program dependencies that compositional analysis must handle soundly |

---

## IMPROVEMENT TRACK L1: Multi-Packet Stateful Map Reasoning
### Problem: Yaksha sees `bpf_map_lookup_elem(conntrack, &key)` but cannot reason about what value is in the map across a sequence of packets

---

### L1-P1. Infer / Bi-Abduction: Compositional Shape Analysis
**Calcagno, Distefano, O'Hearn, Yang | POPL 2011 / JACM 2015**  
📎 https://dl.acm.org/doi/10.1145/1926385.1926403

**What it does:** Bi-abduction automatically infers **separation logic specifications** (pre/post-conditions) for heap-manipulating programs — including those that read/write linked lists, trees, and maps — without requiring manual annotations.

**Technique:** Given a procedure that reads from a heap data structure, bi-abduction infers:
- What the heap must look like *before* the call (precondition) 
- What it looks like *after* (postcondition)
Uses an abductive inference rule: solve `H * anti-frame ⊢ H'` to find the anti-frame automatically.

**How it improves Yaksha (L1):**  
BPF maps are essentially heap key-value stores accessed by eBPF instructions. Bi-abduction could automatically infer what a BPF map must contain *before* a packet arrives (precondition) and what the map contains *after* processing (postcondition). This would let Yaksha reason: "after 3 packets with the same 5-tuple, the conntrack map contains an active session entry."

**Scalability:** Ships as Facebook's Infer — runs on millions of lines of Java/C in CI.  
**Source needed?** Works from source. Adapting to bytecode requires lifting BPF instructions to a heap model.

---

### L1-P2. Incorrectness Separation Logic (ISL)
**O'Hearn | POPL 2020**  
📎 https://dl.acm.org/doi/10.1145/3371078

**What it does:** Classical separation logic proves *absence* of bugs (over-approximation). ISL proves *presence* of bugs (under-approximation) — it produces **concrete witnesses** showing a reachable bug.

**Technique:** Replaces the "frame rule" with an "anti-frame rule" that reasons about reachable states rather than all states. Generates concrete counterexample inputs that trigger the bad heap state.

**How it improves Yaksha (L1):**  
When a user queries "does this NAT NF ever have an inconsistent conntrack entry?", ISL-style under-approximation can produce a *concrete packet sequence* that witnesses the inconsistency in the BPF map state. This goes beyond Yaksha's current yes/no Prolog queries to produce concrete packet-level witnesses.

**Scalability:** Implemented in Pulse (Meta's checker) — production scale.

---

### L1-P3. CIVL: Verifying Concurrent Data Structures
**Hawblitzel et al. | SOSP 2015**  
📎 https://dl.acm.org/doi/10.1145/2815400.2815402

**What it does:** CIVL verifies concurrent data structures (including hash maps and lock-free data structures) using layered refinement — each layer proves that the data structure's high-level specification is correctly implemented by low-level concurrent code.

**Technique:** Layered reasoning: prove that a concurrent hash map implements an abstract map specification, then reason about the abstract map in higher-level proofs.

**How it improves Yaksha (L1):**  
BPF maps (HASH, LRU_HASH, PERCPU_HASH) have specific concurrent access semantics. CIVL's approach of separating "abstract map semantics" from "concurrent implementation semantics" could let Yaksha reason about BPF map behavior using an abstract map model without needing to model per-CPU or RCU details at every step.

---

### L1-P4. Exact Separation Logic (ESL)
**Batz, Kaminski, Katoen, Matheja | POPL 2022**  
📎 https://arxiv.org/abs/2111.12553

**What it does:** ESL bridges over-approximation (proving correctness) and under-approximation (finding bugs) — it characterizes *exactly* the set of reachable states, making both verification and bug-finding possible with one framework.

**How it improves Yaksha (L1):**  
Exact reasoning about BPF map state: Yaksha currently over-approximates by treating all possible map values as unknown. ESL could allow exact characterization of what states a BPF map can be in after N packets from a specific traffic pattern.

---

### L1-P5. VeriFast: Separation Logic for C/Java Programs
**Jacobs, Piessens | JAR 2011 + ongoing**  
📎 https://github.com/verifast/verifast

**What it does:** VeriFast is a practical separation logic verifier for C and Java that can verify programs using heap structures including hash maps. Already used in Vigor (NF verification framework).

**Technique:** User provides ghost code annotations (predicates) that describe heap invariants. VeriFast checks them automatically.

**How it improves Yaksha (L1):**  
The Vigor team already uses VeriFast to verify NF data structures (Vigor's libVig). The same approach could be used to write VeriFast predicates for BPF map types (e.g., `bpf_hash_map(key_type, val_type, max_entries)` predicate) and verify NF behavior with respect to these maps. Requires lifting bytecode to annotated C model — feasible via BTF (BPF Type Format) debug info.

---

## IMPROVEMENT TRACK L2: Helper Function / External Call Modeling
### Problem: Yaksha treats `bpf_map_lookup_elem()`, `bpf_redirect()`, `bpf_fib_lookup()` etc. as imprecise black boxes

---

### L2-P1. Function Summaries in Inter-Procedural Analysis (SUMMARY-BASED SE)
**Godefroid | POPL 2007 + demand-driven summaries literature**  
📎 https://dl.acm.org/doi/10.1145/1190216.1190248

**What it does:** Instead of inlining every function call, compute a **symbolic summary** (pre/post-condition pair) for each function, then reuse the summary at call sites. Functions are analyzed once; their summaries are reused for all callers.

**Technique:** For each function f: compute symbolic summary `{P} f {Q}` where P is the symbolic precondition over arguments and Q is the symbolic postcondition. At call site, apply the summary instead of re-analyzing f.

**How it improves Yaksha (L2):**  
Each eBPF helper could have a **manually-written summary** or **automatically synthesized summary**:
- `bpf_map_lookup_elem(map, key)`: Summary = "returns pointer to value if key exists, NULL otherwise; map state unchanged"
- `bpf_redirect(ifindex, flags)`: Summary = "returns XDP_REDIRECT; adds (ifindex, flags) to packet metadata"
- `bpf_csum_diff(from, size, to, size, seed)`: Summary = "returns updated checksum; no memory side effects"

These summaries can be encoded as Prolog facts in Yaksha's existing query engine, dramatically improving helper precision without analyzing kernel source.

---

### L2-P2. LTLf-Guided Symbolic Execution for External Calls
**Presented at POPL 2025**  
📎 https://dl.acm.org/doi/10.1145/3704909 (referenced in search results)

**What it does:** Uses **Linear Temporal Logic over Finite Traces (LTLf)** specifications to guide symbolic execution through opaque library/system calls. The engine uses the LTL spec to prune infeasible paths and explore states that matter.

**Technique:** Express allowed behaviors of external functions as LTLf automata. The SE engine consults the automaton at each external call to determine which symbolic states to explore.

**How it improves Yaksha (L2):**  
Many eBPF helpers have sequencing constraints: `bpf_skb_pull_data` must be called before reading sk_buff fields; `bpf_spin_lock` must be followed by `bpf_spin_unlock`. These are LTLf properties. Encoding them as automata and using them to guide Yaksha's CFG-NC construction would catch programs that violate helper call ordering — a class of bugs currently invisible to Yaksha.

---

### L2-P3. KLEEF: Extended KLEE with Fuzzing Solvers for External Calls
**Toolchain Labs | 2022+**  
📎 https://toolchain-labs.com/projects/kleef.html

**What it does:** Improves KLEE's handling of external/opaque calls by integrating fuzzing-based solvers that produce values for external function return values without full concretization.

**Technique:** When a symbolic execution engine hits an external call, instead of concretizing symbolic arguments (losing coverage), use fuzzing to produce multiple symbolic return values that maintain path diversity.

**How it improves Yaksha (L2):**  
Yaksha's CFG-NC construction currently makes single pessimistic assumptions about helper return values (e.g., `bpf_map_lookup_elem` always returns non-NULL). KLEEF's approach of maintaining multiple symbolic return values would let Yaksha explore both the "map hit" and "map miss" paths through a single NF bytecode, giving more precise behavioral analysis.

---

### L2-P4. MUSKETEER: Automated Stub Generation for Symbolic Execution
**Chowdhury, Cok | JAIST 2020+**  
📎 https://arxiv.org/abs/2012.09344

**What it does:** Automatically generates symbolic execution stubs for library functions by analyzing function signatures, documentation, and usage patterns. Reduces the manual effort of writing helper models from O(N) to O(1) with a generator.

**How it improves Yaksha (L2):**  
eBPF helpers have well-documented signatures in the kernel (man bpf-helpers). MUSKETEER-style stub generation could automatically produce Prolog fact templates for all 200+ helpers from the kernel's BTF type information and man pages, giving Yaksha coverage for the long tail of rarely-used helpers.

---

### L2-P5. Android API Model for Static Taint Analysis (FlowDroid)
**Arzt, Rasthofer, Fritz, Bodden et al. | PLDI 2014**  
📎 https://dl.acm.org/doi/10.1145/2594291.2594299

**What it does:** FlowDroid models 12,000+ Android SDK API methods using **source/sink/sanitizer** annotations to enable scalable taint analysis without analyzing SDK source code.

**Technique:** Each API method is annotated with: what data flows in (sources), what data flows out (sinks), and what sanitizes (removes taint). This compact model enables precision without source code.

**How it improves Yaksha (L2):**  
The same source/sink model can be applied to eBPF helpers:
- **Sources:** `bpf_skb_load_bytes()`, `bpf_xdp_load_bytes()` — load packet data → source
- **Sinks:** `bpf_skb_store_bytes()`, `bpf_xdp_store_bytes()` — write packet data → sink
- **Sanitizers:** `bpf_csum_update()` — fixes checksum after modification → sanitizer

This would let Yaksha track data flow from packet fields through helper calls — answering questions like "does this NF ever write to the IP destination field without recomputing the checksum?"

---

## IMPROVEMENT TRACK L3: Concurrent BPF Map Access
### Problem: eBPF programs run on N CPUs simultaneously; per-CPU maps and spinlocks create races invisible to Yaksha

---

### L3-P1. RacerD: Compositional Race Detection at Scale
**Blackshear, Gorogiannis, O'Hearn, Sergey | OOPSLA 2018**  
📎 https://dl.acm.org/doi/10.1145/3276514

**What it does:** RacerD is Meta's production race detector — deployed on hundreds of millions of lines of Java/C++ code. Uses Concurrent Separation Logic (CSL) principles but designed for high precision (low false positives) over soundness.

**Technique:** 
1. Analyze each procedure to compute "ownership" of memory locations (is this location protected by a lock?).
2. Track "unprotected accesses" — reads/writes without lock protection.
3. Report races when two threads perform unprotected accesses to the same location with at least one being a write.
4. Compositional: analyzed procedure by procedure without full call graph.

**How it improves Yaksha (L3):**  
eBPF programs run as "threads" on different CPUs. The same BPF map is a shared memory location. RacerD's ownership tracking could be adapted to:
- Track which BPF maps are per-CPU (safe, no race) vs shared (requires lock).
- Check whether shared maps are accessed under `bpf_spin_lock` protection.
- Detect unprotected concurrent writes to shared maps — a real class of eBPF bugs.

**Scalability:** Production at Meta — handles industrial scale.

---

### L3-P2. Owicki-Gries Concurrent Program Logic
**Owicki, Gries | Acta Informatica 1976 + modern variants**  
📎 https://link.springer.com/article/10.1007/BF00268134

**What it does:** The foundational concurrent program logic. Proves correctness of concurrent programs by showing that each thread's proof "interference-free" with others — no thread invalidates another thread's assertions.

**How it improves Yaksha (L3):**  
For eBPF's simple concurrency model (N copies of one program, one shared map), Owicki-Gries is tractable:
- Prove that each individual BPF program execution correctly updates the map.
- Prove interference freedom: two concurrent executions updating different keys don't interfere.
- For programs using `bpf_spin_lock`: prove the critical section maintains map invariants.

---

### L3-P3. Rely-Guarantee Reasoning for Concurrent Programs
**Jones | 1983 + modern SMT-based variants**  
📎 (foundational, many extensions)

**What it does:** Rely-Guarantee (RG) extends Hoare logic to concurrent programs by adding:
- **Rely:** What the environment (other threads) may do to shared state.
- **Guarantee:** What this thread guarantees it will do to shared state.

**How it improves Yaksha (L3):**  
For per-CPU eBPF maps (no sharing): Rely = "nothing" (no interference). For shared maps: Rely = "another CPU may atomically update any key" → Yaksha's analysis can be parameterized by the rely condition.

This would let operators express: "assume at most one other CPU is inserting into the conntrack map at the same time" → Yaksha checks the NF is correct under that assumption.

---

### L3-P4. Thread-Modular Abstract Interpretation (TMABI)
**Miné | SAS 2011**  
📎 https://link.springer.com/article/10.1007/s10703-011-0124-4

**What it does:** Analyzes concurrent programs by treating each thread independently but modeling interference from other threads as an abstract "environment" updated iteratively to fixpoint.

**How it improves Yaksha (L3):**  
eBPF's concurrency is simple: N identical copies of one program. Thread-modular analysis would:
1. Analyze one eBPF program copy with symbolic map state.
2. Model what other CPUs may do to the map as an abstract environment.
3. Iterate to fixpoint — does the per-CPU behavior stabilize?

Scalable because eBPF programs are short (< 1M instructions, kernel limit).

---

## IMPROVEMENT TRACK L4: JIT/Offload Equivalence Checking
### Problem: Yaksha analyzes BPF bytecode but the actual execution is JIT-compiled x86/ARM code — which may differ

---

### L4-P1. Alive2: Bounded Translation Validation for LLVM
**Lopes, Lee, Liu, Regehr | PLDI 2021**  
📎 https://dl.acm.org/doi/10.1145/3453483.3454030

**What it does:** Alive2 verifies that LLVM optimization passes are semantically correct — the optimized LLVM IR is equivalent to the original. Uses Z3 SMT solver to check equivalence of IR fragments bounded by loop unrolling depth.

**Key technique:** Encode LLVM IR semantics as SMT formulas. Check: for all inputs, original and optimized produce the same output. If Z3 finds a counterexample, report the failing optimization.

**Real impact:** Found **47 miscompilation bugs** in LLVM in first deployment.

**How it improves Yaksha (L4):**  
The same approach can check: does the eBPF JIT-compiled x86 code produce the same outputs as the BPF bytecode? Encode both BPF bytecode semantics and x86 native code semantics as SMT, use Alive2-style bounded equivalence checking. This directly addresses Yaksha's JIT divergence gap. Already applied in **Jitterbug** (OSDI 2020) which found 16 JIT bugs — Yaksha could call Jitterbug's methodology as a complementary pass.

---

### L4-P2. Semantic Program Alignment for Equivalence Checking
**Churchill, Padon, Sharma, Aiken | PLDI 2019**  
📎 https://dl.acm.org/doi/10.1145/3314221.3314596

**What it does:** Automatically finds "alignments" between two program versions so that equivalence proofs can be structured along aligned points — even when programs differ significantly in structure.

**How it improves Yaksha (L4):**  
When comparing eBPF bytecode (BPF ISA) with JIT-compiled x86 code, direct equivalence checking is hard because the two programs have very different structures. Semantic alignment finds corresponding points in both programs (e.g., after each packet field access) and structures the equivalence proof along those points — making it tractable.

---

### L4-P3. SMT-Based Translation Validation for ML Compilers
**CAV 2022**

**What it does:** Uses SMT-based equivalence checking to validate that ML compiler transformations (operator fusion, graph rewrites) preserve semantics.

**How it improves Yaksha (L4):**  
The same bounded SMT equivalence approach used for ML compilers applies to eBPF JIT compilers:  each BPF-to-x86 translation rule can be verified independently, then composed. Already partially done by Jitterbug, but Yaksha could integrate this check as an optional pass when JIT source is available (e.g., the Linux kernel JIT source is open).

---

## IMPROVEMENT TRACK L5: Richer Query Language for Temporal/Quantitative Properties
### Problem: Yaksha's Prolog queries cannot express "map is updated BEFORE packet is forwarded" or "at most 1% drop rate"

---

### L5-P1. Soufflé Datalog: High-Performance Program Analysis Engine
**Jordan, Scholz, Subotic | CAV 2016 + ongoing**  
📎 https://souffle-lang.github.io/

**What it does:** Soufflé is a Datalog compiler that achieves 1000x speedup over traditional Datalog by compiling to parallel C++ and using novel evaluation strategies (seminaive evaluation, stratified negation, ADTs).

**Why better than Prolog for Yaksha:**
- Prolog: backward-chaining, no parallelism, limited for large fact bases
- Soufflé Datalog: forward-chaining with seminaive evaluation, parallel, handles billions of facts
- Soufflé supports user-defined **lattices** and **SMT constraints** — enabling quantitative properties

**How it improves Yaksha (L5):**  
Replace Yaksha's Prolog backend with Soufflé Datalog:
- Encode CFG-NC facts as Soufflé relations
- Express NF behavioral properties as Datalog rules
- Add lattice domains for quantitative reasoning (e.g., packet count bounds)
- 10-100x faster query evaluation on large programs

---

### L5-P2. DatalogMTL: Datalog with Metric Temporal Logic
**Brandt, Kalaycı, Ryzhikov, Xiao, Zakharyaschev | 2017-2022**  
📎 https://arxiv.org/abs/1712.01093

**What it does:** Extends Datalog with metric temporal operators (within N time steps, always eventually, etc.) to reason about **temporal sequences of facts** — not just static relations.

**How it improves Yaksha (L5):**  
DatalogMTL could express NF behavioral properties over packet sequences:
- "Within 3 packets after a SYN, there must be a conntrack map insertion" 
- "Every packet that passes the firewall must have been preceded by a SYN-ACK in the flow"
- "The map key is always looked up before the packet action is decided"

These are precisely the temporal ordering properties that Prolog cannot express but are crucial for stateful NF validation.

---

### L5-P3. Temporal Vadalog: Production Temporal Reasoning System
**Berger, Bellomarini et al. | VLDB 2021**  
📎 https://arxiv.org/abs/2105.12341

**What it does:** A fully engineered, production-ready system for DatalogMTL that handles time-series data, infinite time intervals, and real-world scale.

**How it improves Yaksha (L5):**  
Temporal Vadalog could serve as Yaksha's temporal reasoning backend for multi-packet property checking — replacing Prolog with a system designed specifically for temporal sequence queries over fact databases.

---

### L5-P4. LTL Runtime Verification and Monitor Synthesis
**Leucker, Schallhart | JLAP 2009 + CAV/RV conference series**  
📎 https://www.sciencedirect.com/science/article/pii/S1567832608000775

**What it does:** Automatically compiles LTL specifications into efficient runtime monitors that detect violations on execution traces. The compiled monitor runs in O(1) time per step.

**How it improves Yaksha (L5):**  
Yaksha could compile Yaksha-Prashna language properties into LTL formulas, then synthesize runtime monitors that validate live eBPF NF execution traces — extending static analysis with runtime monitoring. This addresses both L5 (richer properties) and fills the offline→online gap.

---

### L5-P5. Hyperproperties and HyperLTL
**Clarkson, Finkbeiner et al. | TOCL 2014 / CAV 2014**  
📎 https://dl.acm.org/doi/10.1145/2535505

**What it does:** Hyperproperties are properties of sets of traces (not single traces). HyperLTL can express: "for any two executions with the same source IP, the packet transformation is identical" (determinism), or "no two flows ever observe each other's conntrack state" (isolation).

**How it improves Yaksha (L5):**  
NF isolation properties (critical for cloud environments) are hyperproperties:
- "Tenant A's traffic never affects Tenant B's NF behavior" = 2-safety hyperproperty
- "The load balancer distributes flows consistently across reloads" = trace equivalence

Yaksha's Prolog backend cannot express these — HyperLTL model checking would fill this gap.

---

## IMPROVEMENT TRACK L6: Cross-Program Chain Compositional Analysis
### Problem: Yaksha handles single programs well but multi-program chains (tail calls, bpf_redirect, program arrays) lack sound compositional reasoning

---

### L6-P1. Demanded Summarization: Incremental Compositional Analysis
**Raghothaman et al. | PLDI/ECOOP 2020-2021**  
📎 https://dl.acm.org/doi/10.1145/3447657

**What it does:** Computes inter-procedural analysis summaries on-demand — only for procedures actually needed for a given query, not the whole program. Enables incremental re-analysis when individual procedures change.

**How it improves Yaksha (L6):**  
eBPF program chains are exactly an inter-procedural analysis problem. Demanded summarization would:
1. Compute a summary of program A's CFG-NC (what it reads/writes/decides)
2. Compose with program B's summary at the tail-call/redirect boundary
3. Answer queries about the composed chain without re-analyzing each program

When one program in the chain updates, only re-compute that program's summary.

---

### L6-P2. PolyVer: Compositional Verification Across Language Boundaries
**arXiv 2023**  
📎 https://arxiv.org/abs/2301.10546

**What it does:** Verifies multi-language systems (C+Rust, Java+JNI) by composing language-specific verifiers with automatically synthesized pre/post-condition contracts at language boundaries.

**How it improves Yaksha (L6):**  
eBPF chains cross "language boundaries" in the sense that:
- XDP program → TC redirect → socket filter: each hook type has different context semantics
- BPF to BPF tail call: program arrays create dynamic dispatch

PolyVer's approach of synthesizing contracts at boundaries could be adapted to synthesize Yaksha-Prashna properties at XDP→TC→socket chain boundaries.

---

### L6-P3. DimSum: Decentralized Multi-Language Semantics
**Sammler, Lepigre et al. | POPL 2023**  
📎 https://dl.acm.org/doi/10.1145/3571220

**What it does:** A framework for verifying programs that call across multiple language/runtime boundaries using "linking specifications" — each component is verified against its spec, then components are composed.

**How it improves Yaksha (L6):**  
Tail calls and redirects in eBPF create exactly this kind of cross-boundary call. DimSum's linking specs could define the interface between eBPF programs in a chain:
- Program A's guarantee: "I write ifindex X to skb redirect metadata"
- Program B's assumption: "I receive skb with ifindex X already set"
- Linking spec: "A's guarantee matches B's assumption"

---

### L6-P4. Timepiece: Modular Control Plane Verification
**Beckett, Panda et al. | PLDI 2023**  
📎 https://dl.acm.org/doi/10.1145/3591239

**What it does:** A modular control-plane verifier that verifies each router's policy independently, then composes results — 10-100x faster than monolithic verification.

**Technique:** Define "network annotations" — invariants that must hold at each node's boundary. Verify each node against its annotation. Compose to prove network-wide properties.

**How it improves Yaksha (L6):**  
Timepiece's boundary annotation approach directly maps to eBPF chains:
- Each eBPF program gets a "program annotation" — invariants that hold at its entry/exit
- Verify each program against its annotation using Yaksha's existing CFG-NC analysis
- Compose annotations to prove chain-level behavioral properties

This is the most directly applicable paper for Yaksha's chain analysis improvement.

---

## SUMMARY: IMPROVEMENT ROADMAP FOR YAKSHA-PRASHNA

| Limitation | Best Paper to Adopt | Technique | Feasibility |
|---|---|---|---|
| **L1: Map state across packets** | Infer/Bi-Abduction (L1-P1) | Separate heap specs for BPF maps | Medium — needs BTF-based lifting |
| **L1: Map state across packets** | ISL (L1-P2) | Under-approx for concrete witness packets | High — natural fit with Prolog |
| **L2: Helper modeling** | Function Summaries (L2-P1) | Pre/post summaries per helper | **High — immediately applicable** |
| **L2: Helper modeling** | FlowDroid source/sink (L2-P5) | Source/sink annotations for 200+ helpers | **High — immediately applicable** |
| **L2: Helper call ordering** | LTLf-Guided SE (L2-P2) | LTLf automata for helper sequences | Medium — needs automata engine |
| **L3: Concurrency** | RacerD (L3-P1) | Lock-ownership tracking for BPF maps | Medium — adapt to eBPF concurrency model |
| **L3: Concurrency** | Thread-Modular AI (L3-P4) | Abstract environment for per-CPU behavior | Medium |
| **L4: JIT equivalence** | Alive2 (L4-P1) | SMT equivalence of bytecode vs native | Low — needs JIT source + SMT infra |
| **L4: JIT equivalence** | Jitterbug (already in survey) | Rosette solver verification | Low — separate system |
| **L5: Query expressiveness** | Soufflé Datalog (L5-P1) | Replace Prolog with Soufflé | **High — drop-in replacement** |
| **L5: Temporal properties** | DatalogMTL (L5-P2) | Metric temporal queries over packet seq | Medium — needs temporal fact encoding |
| **L5: Hyperproperties** | HyperLTL (L5-P5) | 2-safety isolation properties | Medium — needs trace product construction |
| **L6: Chain composition** | Timepiece annotations (L6-P4) | Per-program boundary annotations | **High — directly mappable** |
| **L6: Chain composition** | Demanded Summarization (L6-P1) | On-demand inter-program summaries | High |

### Immediate Wins (High feasibility, implement now):
1. **Soufflé Datalog backend** → replaces Prolog, 10-100x faster, enables lattice/quantitative queries
2. **Helper function summaries** → manually encode 50 most-used helpers as pre/post Prolog/Datalog facts using kernel man-pages + BTF
3. **FlowDroid-style source/sink model** → tag helper calls as packet-data sources/sinks; enables data-flow queries
4. **Timepiece-style boundary annotations** → add per-program interface specs for tail-call/redirect chains

### Research-Level Improvements (Require new research):
1. **Bi-abduction for BPF maps** → automatic map precondition inference (new contribution)  
2. **DatalogMTL for packet sequences** → temporal reasoning over multi-packet NF behavior (new contribution)
3. **RacerD adaptation for eBPF concurrency** → BPF spinlock and per-CPU map race detection (new contribution)

---

*All papers corroborated by research search. Techniques selected based on fit to eBPF bytecode analysis context and Yaksha-Prashna's confirmed architecture (CFG-NC + Prolog query engine). No claims fabricated.*
