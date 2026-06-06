# Automated Verification of Network Function Binaries (Klint)

**Authors:** Solal Pirelli, Akvilė Valentukonytė, Katerina Argyraki, George Candea  
**Year:** 2022 | **Venue:** USENIX NSDI '22  
**DOI/Link:** https://dslab.epfl.ch/pubs/klint.pdf | https://www.usenix.org/conference/nsdi22/presentation/pirelli  
**Code:** https://github.com/dslab-epfl/klint

---

## 1. Overview

Formally verifying network functions (NFs) is a well-recognized prerequisite for reliable and trustworthy network infrastructure. However, dominant approaches to NF verification share two critical limitations: they require access to the NF's full source code, and they mandate that the NF developer use only a small, pre-verified set of approved data structures. These constraints make formal verification inaccessible to network operators deploying commercial or proprietary NF binaries, and to developers who need the flexibility to choose efficient, custom-designed data structures.

Klint directly attacks both limitations simultaneously. It introduces an automated technique to verify **NF binaries** — compiled executable code — without source code or debug symbols, for compliance with a user-specified behavioral specification. The key conceptual insight is that NF data structures can universally be *modeled* as abstract maps, regardless of how they are concretely implemented in memory. Klint introduces "ghost maps" as a universal abstraction: a formal map type that serves simultaneously as the specification language for NF behavior and as the contract language for data structure behavior. This decouples verification of the NF's high-level behavior from any specific data structure implementation.

In practice, Klint verifies 7 real NF binaries (spanning C/Rust, DPDK, and BPF frameworks) against Python-written specifications in minutes, checking correctness, memory safety, and crash freedom — all without source code, debug symbols, or restrictions on data structure choice. Klint can even verify an entire NF software stack down to the NIC driver, by running verification within a minimal operating system environment. The combination of binary-level analysis, ghost map abstraction, symbolic execution, and SMT checking makes Klint one of the most practically deployable NF verification tools in the literature.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

Klint's verification rests on three tightly integrated ideas:

**1. Binary-Level Symbolic Execution with Environment Inference**  
Klint does not require source code. Instead, it *lifts* the NF binary into LLVM IR (an intermediate representation that symbolic execution engines can process) and runs symbolic execution over it. The critical challenge when operating without source code is that types and control flow cannot be derived from compiler-annotated ASTs. Klint resolves this by observing the NF's interactions with its *environment* — the network stack, memory allocator, clock, and packet APIs. The types and semantics of these external interfaces are known, and by tracking what functions the NF calls and with what arguments, Klint infers the types and control flow structure of the NF binary.

**2. Ghost Map Abstraction**  
The central innovation is the *ghost map*: a universal, conceptually infinite key-value map (similar to a mathematical partial function) that serves as the abstract data type for all NF state. Every concrete data structure used by an NF — hash tables, LRU caches, ring buffers, flow tables — is replaced during verification by a ghost map that tracks the same logical state. Developers provide *contracts* for their data structures: declarative descriptions of how each data structure operation (lookup, insert, delete, expire) maps to ghost map operations. These contracts are not full proofs; they are *assumed* correct for the purpose of NF verification (though independent verification is encouraged). This design means Klint never reasons about the internal pointer manipulation of a concrete data structure — only about its abstract map semantics.

**3. SMT-Based Specification Checking**  
NF specifications are written in Python using ghost maps. At each symbolic execution path, Klint accumulates a path condition and the corresponding ghost map state trace. It then queries an SMT solver asking: does the symbolic trace satisfy the specification? The SMT solver decides whether the concrete binary behavior, as captured symbolically, conforms to the abstract specification across all possible inputs. The combination of symbolic execution (for path exploration) and SMT (for property checking) gives Klint both scalability and formal rigor.

### 2.2 Process Steps

1. **Input:** An NF binary (ELF, BPF object, or full binary stack) + a Python specification using ghost maps + optional data structure contracts written in terms of ghost map operations.

2. **Binary Lifting:** The NF binary is lifted to LLVM IR using binary analysis tooling. No source code, DWARF symbols, or debug info is required. For eBPF binaries, the eBPF ISA is lifted to LLVM IR via a custom translation layer.

3. **Environment Modeling:** Klint replaces calls to environmental APIs (packet I/O, memory allocation, timers, random number generation) with symbolic stubs. Arguments to these calls are made symbolic; return values are fresh symbolic variables. This models an arbitrary environment without needing code.

4. **Type and Control Flow Inference:** By observing which environmental functions are called and with what argument patterns, Klint infers the types of variables and the overall control flow structure. This replaces the role that compiler type information would normally play.

5. **Ghost Map Substitution:** Data structure library calls (e.g., `map_get`, `vector_borrow`, `dchain_expire_one_index`, BPF map helpers) are replaced by ghost map operations. Each substitution is guided by the data structure contract. The NF's internal state is thus entirely abstracted into ghost maps.

6. **Symbolic Path Exploration:** KLEE (or a Klint-customized symbolic execution engine built atop LLVM IR) explores all feasible execution paths through the NF, accumulating path conditions and ghost map state transitions per path.

7. **SMT Specification Query:** For each explored path, Klint encodes the path condition and ghost map state as SMT formulas and queries Z3 to check whether the observed behavior satisfies the specification. Violations produce counterexamples.

8. **Output:** A verification result: either a proof of compliance for all reachable behaviors, or a concrete counterexample trace demonstrating a violation. For NFs without specifications (e.g., some BPF programs), Klint also automatically checks memory safety and crash freedom.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in Klint |
|---|---|
| **LLVM IR** | Intermediate representation to which NF binaries are lifted; serves as the substrate for symbolic execution. Enables language-agnostic analysis of binaries compiled from C, Rust, or other LLVM-targeting languages. |
| **KLEE** | LLVM-based symbolic execution engine (adapted by Klint) that explores all feasible execution paths of the lifted IR, generating path conditions and symbolic state traces. |
| **Z3 (SMT Solver)** | Used to check whether symbolic path conditions plus ghost map traces satisfy the Python-encoded specification. Z3 decides satisfiability of the encoded logical constraints. |
| **Ghost Maps** | Klint-introduced abstract data type: a universal key-value map formalism used both as a specification language and as the contract language for all NF data structures. Mathematically modeled as partial functions over symbolic domains. |
| **Python (Specification Language)** | NF specifications are written as Python programs using ghost maps. The Python specification is translated into SMT assertions during verification. |
| **Symbolic Execution** | Core path exploration technique. Variables are kept symbolic (not assigned concrete values) to represent all possible inputs simultaneously. Path conditions constrain which values are feasible on each path. |
| **eBPF ISA Lifter** | Custom component that translates eBPF bytecode instructions (including map helper calls) into equivalent LLVM IR operations, enabling Klint to handle BPF object files. |
| **Minimal OS (for full-stack verification)** | When verifying down to the NIC driver, Klint uses a minimal operating system environment to model the full software stack, enabling end-to-end verification without a real OS. |
| **Uninterpreted Functions (in SMT)** | Ghost map operations (lookup, update, remove) are encoded as SMT uninterpreted functions satisfying algebraic axioms, making them tractable for Z3 without concrete implementation reasoning. |

### 2.4 Key Data Structures / Models

- **Ghost Map:** The central data structure of the Klint framework. Formally, a ghost map is a partial function `K → V` (key type to value type), supporting operations `get(k)`, `set(k, v)`, `has(k)`, and set-like predicates. A single NF may maintain several named ghost maps (e.g., one for flow entries, one for port mappings). Ghost maps have no aliasing and no pointer semantics, making them trivially amenable to SMT encoding via the theory of arrays or uninterpreted functions.

- **Data Structure Contracts:** A contract for a data structure (e.g., a hash map library or BPF LRU map) specifies, for each operation, how it updates and queries ghost maps. For instance, a `map_get(key)` contract might say: "if ghost_map.has(key), return ghost_map.get(key); else return error." Contracts are written in terms of ghost maps and bridge the NF's concrete memory operations with the abstract verification world. Contracts are assumed correct, not proven.

- **Symbolic Execution Tree (implicit in KLEE):** KLEE's internal representation of all feasible paths through the lifted LLVM IR, with path conditions accumulated as SMT formulas. Each leaf corresponds to a return from the packet-processing function, associated with a final ghost map state and a path condition.

- **Specification Model:** A Python object/program that, given a ghost map state and a packet (as a symbolic byte sequence), specifies what the NF *should* do. This encodes the NF's behavioral contract as an executable predicate over symbolic maps and packets, which is then translated to SMT for checking.

- **NF Environment Model:** A set of stubs and symbolic wrappers for all external API calls (packet metadata, memory, clock, randomness). Together, these form the "harness" within which the NF binary is symbolically executed, capturing an arbitrary and adversarial environment.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

Klint targets **stateful network function binaries** — compiled code that processes packets using internal, persistent state. The paper explicitly verifies:

- **NAT (Network Address Translator):** Maps (internal IP, port) → (external IP, port), with timeout-based flow expiry.
- **Stateful Firewall:** Tracks connection state to permit or block packets based on connection history.
- **Load Balancer:** Distributes flows across backend servers using consistent hash-based or flow-affinity assignment.
- **Policer / Rate Limiter:** Enforces per-flow rate limits using token buckets.
- **Bridge (MAC Learning):** Learns MAC-to-port mappings and forwards frames accordingly.
- **NAT64:** Translates between IPv6 and IPv4 addresses and packets.
- **BPF-based NFs:** NFs implemented in eBPF/XDP (Linux kernel's fast-path packet processing framework), for which Klint verifies memory safety and crash freedom even when no behavioral specification is provided.

The framework works with NFs written in **C or Rust**, using frameworks such as **DPDK** (userspace I/O), **BPF/XDP** (kernel-bypass), or custom OS environments. It is the first system to verify NF binaries across all of these frameworks without source code.

### 3.2 How It Validates NF Behavior

1. **Specify:** The operator (or developer) writes a Python specification using ghost maps to express the intended NF behavior. For example, the NAT specification says: for each packet, if there is an active flow entry in the ghost map matching the source (IP, port), rewrite the header according to that entry; otherwise, allocate a new entry and forward.

2. **Instrument:** Klint lifts the NF binary to LLVM IR and replaces data structure calls with ghost map contract stubs. Environmental calls are replaced with symbolic stubs.

3. **Explore:** KLEE-based symbolic execution explores all feasible packet-processing paths, producing a set of (path condition, ghost map trace) pairs — one per feasible execution path.

4. **Check:** For each path, Klint encodes the path condition and observed ghost map state changes as SMT formulas and checks them against the specification's SMT encoding using Z3. It asks: "Does the observed ghost map state change, under these input constraints, conform to what the specification mandates?"

5. **Report:** If all paths satisfy the specification, Klint reports verification success. If any path violates the specification (or causes a memory error, null pointer dereference, etc.), it reports a counterexample with the concrete triggering input.

For BPF NFs without explicit specifications, Klint automatically checks memory safety (no out-of-bounds accesses, no use-after-free) and crash freedom (no panics, assertion failures, or BPF verifier bypasses) using the same infrastructure.

### 3.3 What Properties / Invariants Does It Prove?

| Property | Description |
|---|---|
| **Functional / Spec Compliance** | For all possible input packets and history of prior packets, the NF's output matches the behavioral specification on every execution path. |
| **Memory Safety** | No out-of-bounds memory accesses, no use-after-free, no wild pointer dereferences in the NF binary or its data structure libraries. |
| **Crash Freedom** | The NF never panics, triggers an assertion failure, or encounters an unrecoverable error. Checked automatically even for NFs without explicit behavioral specifications. |
| **Stateful Consistency** | The ghost map state evolves correctly per the spec across multiple packet events; stateful invariants (e.g., flow table consistency, timeout correctness) are preserved. |
| **Type Safety (inferred)** | Types inferred from environment interactions; Klint ensures the NF uses all APIs in a type-safe manner as implied by the environmental API contracts. |
| **No Packet Corruption** | Packet header fields are not modified in ways not sanctioned by the specification (implicit in the spec compliance check). |

Properties Klint does **not** claim to prove without additional specifications: liveness (eventual forwarding), fairness, RFC compliance (unless explicitly encoded in the spec), or timing properties.

### 3.4 Input Requirements

| Input | Required? | Notes |
|---|---|---|
| NF binary (ELF, BPF object, etc.) | **Yes** | No source code or debug symbols needed |
| Python specification (ghost map-based) | **Yes for behavioral verification; optional for safety-only** | Operator writes this using Klint's ghost map API |
| Data structure contracts | **Yes if NF uses non-trivial custom data structures** | Developer provides map-based contracts for each DS library; the DS code itself need not be separately verified |
| Minimal OS model (for full-stack) | **Optional** | Required only when verifying down to NIC driver |

### 3.5 Guarantees Provided

- **Soundness:** Klint provides sound guarantees over the paths that symbolic execution covers. If all paths are explored and all pass the SMT checks, the guarantee is unconditional for all possible packet inputs on those paths.
- **Completeness Caveat:** Path explosion is the classical limitation of symbolic execution. Klint relies on the bounded, single-packet nature of NF processing functions to make exploration tractable (NF packet handlers are typically short-lived without unbounded loops). In practice, verification completes in minutes for the evaluated NFs.
- **Contract Trust Boundary:** Verification soundness is conditional on the correctness of the data structure contracts. If a contract is incorrect (i.e., the ghost map semantics do not faithfully capture the concrete data structure behavior), verification may produce false positives. Klint explicitly documents this trust boundary.
- **Result Type:** A formal proof of compliance (for all explored inputs) or a concrete counterexample trace (demonstrating which input causes a violation).

---

## 4. NF Chain Verification

Klint is designed and evaluated as a **single-NF verifier**. It verifies the behavior of one compiled NF binary at a time, against a specification of that NF's intended behavior in isolation.

The paper does not address NF chain composition, where multiple NFs are deployed in sequence and packets pass through them in order. Klint makes no claims about:

- **Ordering properties** (whether NF A must precede NF B in a chain for correctness),
- **Compositional safety** (whether the output of NF A is a safe/valid input to NF B),
- **Cross-NF stateful interactions** (e.g., a NAT modifying a packet in a way that breaks a downstream firewall's stateful tracking),
- **End-to-end chain invariants** (e.g., that no packet is both forwarded and dropped at different chain positions),
- **Shared map state between eBPF programs** in a multi-program pipeline.

**To extend Klint to NF chains**, one would need to:
1. Compose ghost map specifications: define how the ghost map state of one NF feeds into the initial state/input domain of the next.
2. Compose symbolic execution: model the multi-NF pipeline where the symbolic output packet and ghost map state of NF₁ become the input to NF₂.
3. Express chain-level properties (e.g., end-to-end forwarding correctness, header integrity across the chain) as SMT assertions over the composed ghost map evolution.
4. Handle shared state: if two NFs in a chain share a BPF map, their ghost map models must be unified and consistency constraints added.

None of this is architecturally blocked by Klint's design, but none of it is implemented or evaluated in the paper. Klint is **single-NF only** in its current form.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna is a static analysis framework that operates directly on **raw eBPF bytecode** (no source code required). It builds a **Control Flow Graph with Network Context (CFG-NC)** from the bytecode — capturing packet header dataflows, BPF map accesses, helper call sequences, and XDP/TC action decisions — and feeds this into a **Prolog-based query engine** for behavioral assertion checking. Yaksha-Prashna natively handles **NF chains** (multiple eBPF programs linked in sequence) and **stateful BPF maps** (hash maps, array maps, LRU maps), checking chain-level ordering invariants, packet field provenance, and behavioral assertions without requiring source code, runtime execution, or SMT solving.

### 5.2 Key Differences from This Paper

| Dimension | Klint (D6) | Yaksha-Prashna |
|---|---|---|
| **Input format** | Compiled binary (ELF/BPF object) lifted to LLVM IR | Raw eBPF bytecode analyzed directly (no lifting) |
| **Analysis technique** | Symbolic execution (KLEE) + SMT (Z3) | Static dataflow analysis (CFG-NC) + Prolog logic queries |
| **Specification language** | Python programs using ghost maps (must be written by user) | Prolog assertions; chain properties derived from structural CFG-NC analysis |
| **Chain analysis** | ❌ Single NF only | ✅ Multi-NF chain natively supported |
| **BPF map state modeling** | Ghost maps via developer-provided contracts | Structural BPF map access tracking in CFG-NC (no contracts needed) |
| **Solver dependency** | SMT solver (Z3) required for all property checking | Prolog inference engine; no SMT solver needed |
| **BPF specificity** | BPF is one supported target among many; no BPF-specific semantics | eBPF-native; deep semantic understanding of BPF ISA and map types |
| **Scalability mechanism** | Path-bounded by NF handler structure; risk of path explosion on complex NFs | Static analysis; scales to large programs and chains without path explosion |
| **Developer participation** | Requires correct data structure contracts from developer | No developer-provided contracts needed |
| **Specification requirement** | Operator must write behavioral spec | Safety/behavioral queries posed without a full spec; chain properties are structurally inferred |

### 5.3 How This Paper Is Useful For Us

1. **Motivational Citation for Source-Free Analysis:** Klint demonstrates conclusively that binary-level NF verification is feasible and practically valuable. Since Yaksha-Prashna also operates without source code, Klint establishes the broader research context. We cite Klint [ref 70] to justify the source-free design choice and to show that the problem has industrial relevance.

2. **Limitation Contrast — Specification Burden:** Klint requires an operator-written Python specification using ghost maps. Yaksha-Prashna extracts behavioral structure directly from bytecode and supports assertion queries without needing a full external spec. This is a significant ergonomic improvement: network operators should not need to write formal specs to get safety assurances.

3. **Limitation Contrast — Chain Coverage:** Klint explicitly verifies only single NFs. Our ability to handle multi-NF eBPF chains is a direct advance over Klint's scope. This contrast is clean, well-scoped, and directly citable.

4. **Limitation Contrast — SMT Scalability vs. Static Analysis:** Klint's symbolic execution + SMT approach carries path explosion risk on complex NFs and requires SMT solving (which can be expensive). Yaksha-Prashna's static Prolog-based approach avoids both. This is a concrete scalability argument.

5. **Ghost Map Concept Alignment:** Yaksha-Prashna's structural BPF map access tracking is conceptually analogous to Klint's ghost map abstraction — both represent NF state as abstract key-value maps. We can cite Klint's ghost map formalization as theoretical grounding for Yaksha-Prashna's map abstraction design, while noting that Yaksha-Prashna does not require developer-provided contracts to instantiate the abstraction.

6. **NF Taxonomy Reference:** The set of NFs Klint verifies (NAT, firewall, load balancer, bridge, policer) is a useful reference taxonomy for evaluating Yaksha-Prashna on comparable NF types, providing a credible benchmark of NF complexity.

7. **Complementary Technique:** For operators needing deep spec-relative behavioral checking on individual NFs, Klint and Yaksha-Prashna are complementary: Klint for deep single-NF spec compliance, Yaksha-Prashna for chain-level structural properties and eBPF-specific safety.

### 5.4 Positioning Statement

> "While Klint [Pirelli et al., NSDI '22] achieves source-free NF binary verification via symbolic execution and ghost map abstractions, it verifies only single NFs in isolation and requires developer-written data structure contracts as well as an operator-authored behavioral specification. Yaksha-Prashna addresses all three gaps: it directly analyzes raw eBPF bytecode without any developer contracts or external specifications, natively handles multi-NF chains through shared BPF map dependency tracking, and employs static CFG-NC dataflow analysis with Prolog-based assertion queries — eliminating both SMT path explosion and specification authoring burden while enabling chain-level behavioral guarantees that single-NF verifiers cannot provide."

---

## Summary Table

| Attribute | Value |
|---|---|
| **Core method** | Binary lifting → LLVM IR → KLEE symbolic execution → ghost map abstraction → Z3 SMT |
| **Input** | Compiled binary (ELF / BPF object) — no source code, no debug symbols |
| **NF types verified** | NAT, Stateful Firewall, Load Balancer, Policer, MAC Bridge, NAT64, BPF/XDP NFs |
| **Chain support** | ❌ No — single NF only |
| **Spec required** | ✅ Yes — Python trace DSL using ghost maps (for behavioral verification) |
| **Guarantee type** | Sound w.r.t. explored paths; counterexample-generating on violations |
| **Venue** | USENIX NSDI 2022 |
| **Cited by Yaksha-Prashna** | ✅ Yes [ref 70] |
| **Open-source** | ✅ Yes — https://github.com/dslab-epfl/klint |
