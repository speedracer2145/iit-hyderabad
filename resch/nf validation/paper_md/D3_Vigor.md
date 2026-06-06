# Vigor: Verifying Software Network Functions with No Verification Expertise

**Authors:** Arseniy Zaostrovnykh, Solal Pirelli, Rishabh Iyer, Matteo Rizzo, Luis Pedrosa, Katerina Argyraki, George Candea  
**Year:** 2019 | **Venue:** ACM SOSP (Symposium on Operating Systems Principles)  
**DOI/Link:** https://vigor-nf.github.io/vigor-paper.pdf | ACM: https://dl.acm.org/doi/10.1145/3341301.3359647

---

## 1. Overview

Software-based network functions (NFs) — middleboxes such as NATs, firewalls, load balancers, and traffic policers — are critical components of modern network infrastructure. Despite their importance, these NFs are routinely deployed without formal correctness guarantees. They are complex stateful programs that interact with low-level packet-processing frameworks (like DPDK), and bugs in them can cause security vulnerabilities, service disruptions, and protocol non-compliance. Prior formal verification approaches either required deep expertise in theorem proving/separation logic, or were too limited in scope to handle real-world NFs built on top of sophisticated I/O frameworks.

**Vigor** addresses this by providing a *push-button, full-stack, pay-as-you-go* verification framework. Network function developers write their NF in standard C against a purpose-built library (`libVig`), provide a Python behavioral specification, and then simply run the Vigor toolchain — which automatically produces a formal proof that the implementation matches the specification. Crucially, no formal verification expertise is required of the NF developer. The verification experts' work is done once (when building and verifying `libVig`), and thereafter amortized across all NFs built using the library.

The key contribution is a three-pronged design philosophy that makes verification tractable: (1) **architectural discipline** — NF code is structured so that all stateful operations go through verified library APIs, making the NF logic essentially stateless and amenable to symbolic execution; (2) **lazy proofs** — a novel compositional technique that glues together exhaustive symbolic execution (KLEE) over the NF logic with separation-logic proofs (VeriFast) over the library; and (3) **full-stack modeling** — Vigor models not just the NF itself but also the DPDK framework, the OS interfaces, and the NIC driver. Vigor demonstrates its approach on five representative stateful NFs and achieves performance competitive with unverified production-grade middleboxes.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

Vigor's verification methodology is based on a *divide-and-verify* strategy that exploits the architectural separation between stateless NF logic and stateful library data structures:

**Part A — Stateless NF Logic (verified via KLEE symbolic execution):**  
The NF developer writes packet-processing logic in C. All persistent state (connection tables, flow maps, etc.) is accessed exclusively through `libVig` API calls. From the symbolic execution engine's perspective, these API calls have known pre- and post-conditions (contracts), so the NF logic becomes a program over symbolic packet data with no complex memory state to track. KLEE exhaustively explores all feasible execution paths through this logic, checking each path against the NF's Python behavioral specification.

**Part B — Stateful Library (verified via VeriFast theorem prover):**  
`libVig` provides verified data structures: hash maps, vectors, double-ended queues (dchains for managing connection expiry), and indexed data structures. Verification experts (the Vigor authors) have manually proved, using **VeriFast** (a separation logic verifier for C), that these implementations satisfy their functional contracts: they are memory-safe, do not overflow, do not alias dangerously, and correctly implement their abstract map/set semantics. These proofs are done once and hold for all NFs.

**Part C — Lazy Proofs (gluing A and B together):**  
The key insight is that, instead of verifying a single monolithic NF, Vigor separates the reasoning. The symbolic execution of Part A generates *symbolic traces* — one per feasible execution path. Each trace is a sequence of `libVig` API calls with symbolic arguments. The *Vigor Validator* then checks whether there exists an interpretation of those API calls (using the formal contracts from Part B) such that the overall trace satisfies the Python specification. This deferred or "lazy" approach avoids the combinatorial explosion that would result from trying to prove properties of models universally: instead, the API contracts are validated only for the specific paths that actually arise during symbolic execution of the target NF.

**Full-Stack Scope:**  
Vigor replaces the real DPDK calls with verified symbolic models. The developer writes against the real DPDK API; Vigor substitutes stub implementations that faithfully model the semantics (e.g., `rte_pktmbuf_alloc` returns a symbolic packet buffer). This means the proof covers the entire execution stack — not just the application logic.

---

### 2.2 Process Steps

1. **NF Implementation:** Developer writes the NF in C, using only `libVig`-provided data structures for stateful storage. The NF must follow the Vigor programming model: a single `nf_process(packet)` function, bounded loops, and no raw pointer arithmetic over unverified state.

2. **Python Specification:** Developer (or verification engineer) writes a Python behavioral model of the NF. This model is a high-level, executable description of what the NF *should* do on each packet (e.g., for a NAT: which packets get translated, which get dropped, what state changes result). The Python model operates over abstract symbolic packet fields and NF state.

3. **DPDK Symbolic Modeling:** Vigor replaces real DPDK library calls with verified symbolic stubs. The modified KLEE environment loads both the NF C code and these stubs.

4. **Exhaustive Symbolic Execution (KLEE):** Vigor's modified KLEE runs the `nf_process` function with fully symbolic packet input (all header fields symbolic). KLEE explores *all* feasible control-flow paths through the NF. For each path, it records the sequence of `libVig` API calls made (with their symbolic arguments and return values), the decisions taken, and the packet output action (forward/drop/transform).

5. **Path Trace Generation:** Each completed KLEE path produces a *symbolic trace* — a linear sequence of operations with symbolic constraints on the path conditions. These traces are exported as a structured intermediate representation.

6. **Vigor Validator — Contract Instantiation:** The Validator takes each trace and the `libVig` API contracts (from the VeriFast proofs). For each `libVig` call in the trace, it instantiates the formal pre/post-condition with the actual symbolic arguments from the trace. This is the "lazy" step: contracts are checked in context, not universally.

7. **Specification Matching:** The Validator checks that, for each trace, if the pre-conditions hold, then the trace output (packet action + state update) matches what the Python specification would require. This check is discharged by a constraint solver (Z3 is used under the hood by KLEE for path constraints).

8. **Proof Output / Counterexample:** If all traces pass, Vigor declares the NF correct with respect to the specification. If any trace fails, Vigor emits a concrete counterexample — the specific symbolic input that causes the violation.

---

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in Vigor |
|---|---|
| **KLEE** (modified) | Exhaustive symbolic execution engine for C. Explores all feasible paths through the stateless NF logic by treating packet input as symbolic. Vigor extends KLEE with custom DPDK stubs and trace export. |
| **VeriFast** | Separation-logic-based deductive verifier for C programs. Used by Vigor's expert authors to prove memory safety and functional correctness of `libVig` data structures. Proofs are done once, not per-NF. |
| **Z3 SMT Solver** | Used by KLEE internally to discharge path feasibility constraints and by the Validator to check specification conformance on each symbolic trace. |
| **Python** | Language in which developers write the behavioral specification (the "oracle") for their NF. Python serves as the high-level reference model; Vigor compares symbolic execution traces against it. |
| **DPDK (Data Plane Development Kit)** | The packet-processing framework that real NFs are built on. Vigor builds verified symbolic stubs for DPDK APIs, extending the proof boundary to the full I/O stack. |
| **Separation Logic** | The logical formalism underpinning VeriFast proofs. Used to express heap ownership, aliasing absence, and pointer safety invariants for `libVig` data structures. |
| **Symbolic Execution** | Core analysis technique — treating input bytes as symbolic variables and systematically exploring all execution paths through the NF's `nf_process` function. |
| **Lazy Proof Composition** | Novel Vigor-specific technique that defers the instantiation of `libVig` API contracts to the point of each specific trace, rather than requiring universal validity proofs. |

---

### 2.4 Key Data Structures / Models

- **`libVig` Data Structures:** A formally verified library of stateful primitives:
  - **`Map`** (hash map): key→value association for connection tables. Verified for correctness and memory safety via VeriFast.
  - **`Vector`** (dynamic array): fixed-size typed array with bounds-checked access.
  - **`DoubleChain` (dchain)**: an index allocator supporting expiry-based reclamation. Used to implement timeouts (e.g., NAT session expiry). Verified via VeriFast.
  - **`MapVector`** composite: combines Map + Vector for common NF patterns (mapping flow keys to flow metadata).
  
- **Symbolic Packet Model:** A symbolic struct representing an Ethernet/IP/TCP-UDP packet where every header field is a fresh symbolic variable. KLEE treats these as unconstrained inputs, forcing exploration of all reachable paths.

- **Symbolic Trace:** The path-level intermediate representation — a sequence of `(libVig_api_call, symbolic_args, return_value, path_constraint)` tuples generated by KLEE per execution path. This is the interface between Part A (SE) and Part C (Validator).

- **API Contracts (VeriFast pre/post-conditions):** Formal specifications of each `libVig` function: precondition (heap shape + argument validity) and postcondition (heap modification + return value meaning). Written in VeriFast's assertion language using separation logic predicates.

- **Python NF Specification Model:** A high-level state machine / function that takes (packet, abstract_NF_state) → (action, new_NF_state). Vigor compares each symbolic trace against this model's output.

- **DPDK Stub Models:** C implementations of DPDK APIs (packet allocation, freeing, queuing) that behave correctly with respect to their contracts but use symbolic memory, enabling KLEE to reason over them without path explosion.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

Vigor targets **stateful software NFs** running on commodity servers using DPDK for packet I/O. The five NFs implemented and fully verified in the paper are:

1. **NAT (Network Address Translator):** Translates between internal (private) and external (public) IP:port pairs. Maintains a flow table of active sessions with timeout-based expiry. Verified against an RFC-derived specification.
2. **Maglev Load Balancer:** A consistent-hashing load balancer (inspired by Google's Maglev). Maps flow keys to backend servers such that flow distribution is stable across server changes.
3. **MAC-Learning Bridge (Ethernet Switch):** Learns source MAC addresses and their associated ports; forwards unknown destinations out all ports (flooding). Verified for correct learning and forwarding behavior.
4. **Stateful Firewall:** Tracks connection state to permit reply traffic while blocking unsolicited inbound flows. Maintains connection tracking table.
5. **Traffic Policer:** Enforces rate limits using a token-bucket algorithm. Drops packets that exceed the configured rate for a flow.

All five are **stateful** (they maintain per-flow or per-MAC persistent state across packets), and all use `libVig` data structures for that state.

---

### 3.2 How It Validates NF Behavior

1. **Write NF in C + libVig:** The NF is structured so that `libVig` handles all persistent state. The developer annotates any loops with loop bounds (needed for KLEE termination).

2. **Write Python Specification:** A Python function describes the expected behavior: given any input packet and any abstract NF state, what should the NF output be (forwarded/modified packet, dropped, state delta)?

3. **Run Symbolic Execution:** Vigor's modified KLEE is invoked on the NF code with all packet fields symbolized. It explores every feasible path, recording traces.

4. **Lazy Proof Instantiation:** For each trace, the Validator applies the VeriFast-proved API contracts for each `libVig` call encountered, instantiated with the actual symbolic arguments in that trace. This validates that the sequence of API calls is legal and that the contracts hold in context.

5. **Specification Check:** The Validator then checks that the trace's observable behavior (output packet + state mutation) matches what the Python model specifies for the same input. This is discharged via SMT (Z3).

6. **Report:** If all paths match, the NF is certified correct. If any path fails, a concrete violation witness is produced.

**Properties Checked:**
- Does every packet processed produce the output mandated by the spec?
- Does state evolve correctly (correct insertions/deletions/lookups)?
- Is memory accessed safely throughout?
- Do DPDK-level resources (mbufs) get correctly managed (no leaks, no double-frees)?

---

### 3.3 What Properties / Invariants Does It Prove?

| Property | How Proven |
|---|---|
| **Memory Safety** | VeriFast separation-logic proofs over `libVig`; KLEE checks no out-of-bounds accesses in NF logic. |
| **Crash Freedom** | No null dereferences, no division by zero, no assertion failures — KLEE explores all paths. |
| **Hang Freedom** | All loops have developer-provided bounds; KLEE verifies termination under symbolic input. |
| **Functional Correctness** | Every KLEE path is checked against the Python behavioral specification. |
| **RFC Compliance** | Python specifications are derived from RFC requirements (e.g., RFC 3022 for NAT); correctness implies RFC conformance. |
| **No Packet Buffer Leaks** | DPDK mbuf lifecycle (alloc/free) is modeled and checked. |
| **State Invariants** | `libVig` data structures maintain their internal invariants across all operations (proved via VeriFast). |

---

### 3.4 Input Requirements

| Input | Provided By | Format |
|---|---|---|
| NF implementation | NF developer | C source code, using `libVig` APIs |
| Behavioral specification | NF developer (or spec engineer) | Python function / model |
| Loop bounds | NF developer | Code annotations (integer constants in loop headers) |
| `libVig` library | Vigor framework (pre-verified) | C source + VeriFast proof files |
| DPDK stubs | Vigor framework (pre-built) | C source (symbolic models) |

**No binary-only input is accepted** — Vigor requires C source to feed into KLEE and VeriFast. The developer must also structure their code to conform to the Vigor programming model (no raw pointer arithmetic over unverified external state, DPDK I/O only through the provided stubs).

---

### 3.5 Guarantees Provided

- **Sound Formal Proof:** If Vigor completes without counterexample, the NF is provably correct with respect to the Python specification for *all possible inputs* — not just tested cases. The guarantee is machine-checkable.
- **Full-Stack Scope:** The proof covers the NF code, `libVig` data structures, DPDK-layer interactions, and packet buffer management — not just the application logic in isolation.
- **Counterexample on Failure:** If a property violation exists, Vigor produces a concrete failing input (specific packet bytes + NF state), enabling debugging.
- **Soundness Boundary:** The guarantee holds under the assumption that (a) the DPDK stubs faithfully model real DPDK behavior, (b) the Python specification is itself correct, and (c) the hardware behaves as modeled. The stubs are manually written and argued correct but not themselves machine-verified against the real DPDK source.

---

## 4. NF Chain Verification

**Vigor does not directly address NF chain (service chain) verification.** The paper targets individual, standalone NFs. Each verified NF is treated as an isolated program that processes packets in one `nf_process` call; there is no mechanism in Vigor to compose the verification of two or more NFs in sequence and prove properties of the composed chain.

**What would be needed to extend Vigor to chains:**

1. **Compositional Specification:** A Python specification would need to describe the end-to-end behavior of the chain (e.g., firewall → NAT → load balancer) as a composed function. This is theoretically possible since Python specs are executable, but the state of each NF is siloed — Vigor does not model inter-NF state transfer.

2. **Symbolic Interface Between NFs:** The output packet (and output packet metadata) of one NF would need to be fed as symbolic input to the next NF's symbolic execution. This requires a multi-NF symbolic execution harness, which Vigor does not provide.

3. **Cross-NF State Reasoning:** If NF A modifies packet header fields that NF B relies on (e.g., A does SNAT before B does firewall lookup), the chain's correctness depends on the composed semantics. Vigor would need to track how NF A's state and packet transforms interact with NF B's state and logic.

4. **Chain-Level Properties:** Properties like "no packet is dropped by the chain unless the firewall rule says so" or "the effective NAT + firewall policy is consistent" require reasoning over the composition — not just individual NF correctness.

The Vigor paper acknowledges this as future work. Theoretically, since each NF's behavior is characterized by a formal Python specification, these specifications could be composed. But the automated toolchain — KLEE-based symbolic execution — would need significant extension to handle chained programs without state explosion across NF boundaries.

**In summary:** Vigor is strictly single-NF. Chain verification would require compositional extensions that the paper does not provide.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs **static CFG-NC (Control Flow Graph with Network Context) dataflow analysis** directly on **raw eBPF bytecode** — no source code required. It extracts behavioral summaries from the bytecode-level CFG, models BPF map operations as typed state transitions, and feeds these into a **Prolog-based query engine** for checking behavioral assertions (e.g., packet classification correctness, stateful forwarding consistency, chain ordering constraints). It is designed to work on **deployed, binary-level eBPF programs**, handles **NF chains** (multiple eBPF programs in sequence in the kernel data path), and reasons over **stateful BPF map interactions** between chain members — all without requiring source code or developer-provided specifications.

---

### 5.2 Key Differences from This Paper

| Dimension | Vigor | Yaksha-Prashna |
|---|---|---|
| **Input Format** | C source code (mandatory) | Raw eBPF bytecode (no source needed) |
| **Developer Effort** | Must write Python spec + loop annotations + use libVig | Zero developer effort; analysis is fully automated on deployed binary |
| **Technique** | Symbolic execution (KLEE) + separation logic (VeriFast) + lazy proof composition | Static CFG dataflow analysis + Prolog query engine |
| **NF Framework** | DPDK-based userspace NFs (dpdk NFs in C) | Kernel-native eBPF programs (XDP, TC hooks) |
| **Statefulness** | Handled via libVig (manually verified data structures) | BPF maps analyzed as typed state structures directly from bytecode |
| **NF Chain Support** | Not supported — single NF only | First-class support — inter-NF state and ordering analyzed |
| **Guarantee Type** | Sound formal proof (if Python spec is correct) | Behavioral assertion checking (decidable Prolog queries over extracted CFG model) |
| **Specification Source** | Developer writes Python spec | No spec required; behavioral properties expressed as Prolog queries |
| **Full Stack Scope** | DPDK + OS stubs + NF logic | eBPF verifier-checked bytecode + BPF map semantics |
| **Timing** | Offline, pre-deployment | Offline, post-deployment (binary analysis) |
| **Expertise Required** | NF developer: moderate (must use libVig, write spec) | Operator/analyst: Prolog query formulation only |

---

### 5.3 How This Paper Is Useful For Us

1. **Motivating the Source-Code Limitation:** Vigor's requirement for C source code, `libVig` API compliance, Python specification authorship, and loop annotations makes it inapplicable to existing deployed eBPF NFs (which are often proprietary, pre-compiled, or embedded in kernel modules). We can cite Vigor to show that even the most rigorous existing NF verification approaches require significant source-level artifacts — a limitation our bytecode-level approach directly addresses.

2. **Establishing the "Gold Standard" for Individual NF Correctness:** Vigor represents the strongest formal guarantee achievable for a single NF. We can use it as a baseline to calibrate what Yaksha-Prashna provides (behavioral assertion checking vs. full correctness proof) and to argue that our weaker but more broadly applicable guarantee is the right trade-off for the eBPF deployment scenario.

3. **Demonstrating the Single-NF Gap:** Vigor's lack of chain support is a concrete, documented limitation that Yaksha-Prashna overcomes. Citing Vigor explicitly and noting its single-NF scope makes the case that chain-level verification is an unaddressed problem that we solve.

4. **Complementary Work Angle:** Vigor and Yaksha-Prashna operate in non-overlapping domains (DPDK userspace NFs vs. kernel eBPF NFs). We can position Yaksha-Prashna as a complement — bringing verifiability to the eBPF ecosystem the way Vigor brought it to the DPDK ecosystem, while extending coverage to chains and binary inputs.

5. **The "Push-Button" Aspiration:** Vigor's push-button claim (no verification expertise for the developer) is a design goal we share. We can cite Vigor as the precedent for this aspiration, then show that Yaksha-Prashna extends it further — not even a Python specification is required from the operator.

6. **Technical Contrast for the Related Work Section:** Vigor's lazy proof composition, separation logic contracts, and symbolic execution form a rich technical contrast to our CFG-NC + Prolog approach. The comparison makes Yaksha-Prashna's design choices (why Prolog? why static analysis rather than SE?) easier to justify in a related-work discussion.

---

### 5.4 Positioning Statement

> "While Vigor achieves full formal correctness proofs for stateful DPDK-based network functions using symbolic execution and separation logic, it requires C source code, a developer-authored Python specification, and compliance with the `libVig` programming model — making it inapplicable to pre-compiled, binary-deployed eBPF network functions and unable to reason across multi-NF service chains. Yaksha-Prashna addresses both limitations: by operating directly on raw eBPF bytecode without any source-level artifacts, and by natively modeling inter-NF dataflow and state interactions across eBPF program chains via a CFG-NC dataflow engine and Prolog behavioral query system."

---

## 6. Additional Notes

### Vigor's Impact and Follow-on Work

Vigor has been highly influential in the NF verification community and has spawned several follow-on projects from the same EPFL/ETH group:

- **Klint (OSDI 2021):** Applies similar push-button verification to eBPF-based NFs using KLEE over LLVM IR, bridging the gap toward the eBPF domain. However, it still requires annotated source code.
- **Bolt (NSDI 2021):** Extends the Vigor approach to *performance verification* — proving CPU cycle bounds and throughput guarantees for NFs.
- **PIX:** Explores binary-level analysis of NFs, though with different goals than Yaksha-Prashna.

### Verification Time

The paper reports verification times of approximately **60–90 minutes** per NF (symbolic execution is the bottleneck). This is reasonable for pre-deployment certification but unsuitable for runtime or dynamic reconfiguration scenarios.

### Completeness Caveat

Vigor's soundness relies on the Python specification being accurate. If the specification itself contains errors (under-specification, wrong RFC interpretation), Vigor will prove an incorrect property. The framework has no mechanism for specification validation — this is a fundamental meta-problem in specification-based verification.

### libVig Library Scope

The `libVig` library is intentionally narrow: developers must use only its provided data structures for stateful storage. NFs that require data structures not in `libVig` (e.g., Bloom filters, PATRICIA tries, specialized caches) either cannot be verified with Vigor or require new VeriFast proofs — which requires expert effort. This constrains Vigor's generality relative to arbitrary NF implementations.
