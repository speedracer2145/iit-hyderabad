# A Formally Verified NAT (VigNAT)

**Authors:** Arseniy Zaostrovnykh, Solal Pirelli, Luis Pedrosa, Katerina Argyraki, George Candea  
**Year:** 2017 | **Venue:** ACM SIGCOMM 2017  
**DOI/Link:** https://vignat.github.io/vignat-paper.pdf | https://dl.acm.org/doi/10.1145/3098822.3098825

---

## 1. Overview

VigNAT is a **formally verified Network Address Translator (NAT)** implemented in C and proven — directly against its C source — to be (1) semantically correct per RFC 3022, (2) memory-safe, and (3) crash-free. The paper attacks a fundamental credibility gap in software network functions (NFs): NFs are increasingly deployed in production (as virtualized middleboxes on commodity hardware), yet they are written in low-level C and remain almost entirely unverified. Prior network verification research (e.g., Header Space Analysis, Minesweeper) typically models NFs as abstract transition functions and verifies *network-level* properties such as reachability or loop-freedom — never the implementation itself. VigNAT closes this gap by applying formal proof to the actual running C code.

The central research challenge is **scalability**: stateful NFs like NATs maintain per-flow tables, and verifying their C code with conventional symbolic execution leads to catastrophic path explosion. VigNAT's key insight is to restructure the NF codebase into two separable layers: (a) a library of formally verified stateful data structures (`libVig`), proved correct once using separation logic and VeriFast; and (b) the remaining stateless NAT logic, which can then be efficiently verified by KLEE symbolic execution because all state operations now reduce to opaque, pre-proved contracts. This decomposition allows the two techniques to work in concert, each applied where it is most effective.

The paper further demonstrates that formal verification need not sacrifice performance. VigNAT runs on Intel DPDK, achieves approximately **1.8 Mpps** throughput, and imposes only a ~0.38 μs additional latency per packet over a Linux/NetFilter NAT — outperforming the Linux kernel NAT significantly (~0.6 Mpps) while also being formally proved correct. This refutes the conventional assumption that verification is incompatible with high-performance packet processing.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

VigNAT employs a **hybrid verification strategy** that partitions the codebase along a stateful/stateless boundary, then applies the most appropriate formal method to each partition.

**Part 1 — Separation Logic + VeriFast (for stateful data structures):**  
The NAT's persistent state — particularly the flow table (session table) — is encapsulated in `libVig`, a purpose-built library of formally specified C data structures. Each data structure in `libVig` is annotated with VeriFast **separation logic contracts**: function pre-conditions and post-conditions expressed as heap predicates, plus custom separation logic predicates that describe how data structures own and lay out memory on the heap. VeriFast statically discharges these proof obligations at the module level, producing a **machine-checked proof** that:
- The data structure's internal heap invariants hold at all times.
- Every function call transfers heap ownership safely (no aliasing, no use-after-free, no buffer overrun).
- The abstract functional behavior (e.g., "map insert adds this key-value pair") matches the C implementation.

This one-time library verification amortizes the separation-logic proof effort across any NF that uses `libVig`.

**Part 2 — Symbolic Execution (for stateless NAT logic):**  
With `libVig` verified and its functions abstracted to their contracts, the core NAT decision logic is effectively stateless from the perspective of any symbolic executor. KLEE is run on this "thin" NAT layer, exhaustively exploring all symbolic packet inputs (source IP, destination IP, ports, protocol, ICMP type, etc.) and all possible return values from `libVig` calls. Because KLEE never needs to concretely explore the internals of hash maps or ring buffers (only their proved contracts), path explosion is avoided. KLEE generates a proof certificate that the NAT logic crashes on no input and that it always forwards/drops packets consistent with the formal NAT specification.

**Bridge between the two parts:**  
The two proofs are composed via verified API boundaries. `libVig` functions export contracts that KLEE treats as axioms during symbolic execution. Soundness of the overall composition follows from the modular soundness of VeriFast proofs and the exhaustiveness of KLEE's path exploration within the bounded model.

### 2.2 Process Steps

1. **Formalize RFC 3022:** The RFC 3022 "Traditional NAT" standard is encoded as a formal mathematical specification (abstract state machine). This defines, for each incoming packet type (TCP/UDP/ICMP), the expected output (forwarded or dropped), the expected state change (new session, expired session, existing session lookup), and the translated address/port.

2. **Implement NAT in C with libVig:** The NAT is written in C, using DPDK for packet I/O. The stateful parts (session table, port allocator) are implemented using `libVig` data structures (Map, Vector, DoubleMap, Ring buffer). The stateless NAT logic (5-tuple lookup, IP/port rewriting, checksum update, ICMP translation) is kept separate.

3. **Annotate libVig with VeriFast contracts:** Each `libVig` data structure and its member functions are annotated with:
   - **Predicate definitions:** Formal heap predicates describing valid internal state (e.g., a `Map` predicate capturing the invariant that every key in the backing array matches its hash).
   - **Function contracts:** `requires`/`ensures` clauses specifying pre- and post-conditions in terms of the predicates and functional models.

4. **Run VeriFast to verify libVig:** VeriFast mechanically checks each function body against its contract using separation logic inference rules. All proof obligations are discharged, yielding a machine-verified certificate.

5. **Run KLEE on NAT logic:** KLEE symbolically executes the NAT packet-processing loop with symbolic inputs. `libVig` calls are replaced by their contracts (as KLEE models). KLEE explores all paths, checking for assertion violations, memory errors, division-by-zero, and consistency with the formal RFC 3022 spec (encoded as assertions).

6. **Verify RFC 3022 properties:** For every symbolically explored execution path, KLEE asserts that the output packet and resulting state match the RFC 3022 formal model. Any counterexample would indicate a bug; none were found.

7. **Performance evaluation:** VigNAT is benchmarked on a commodity server running DPDK, compared against Linux/NetFilter NAT and an unverified DPDK NAT. Throughput, latency, and flow table scalability are measured.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in VigNAT |
|---|---|
| **VeriFast** | Separation-logic-based deductive verifier for C. Checks `libVig` data structure correctness against annotated heap contracts. Produces machine-checked proofs of memory safety and functional correctness of each data structure. |
| **Separation Logic** | The logical formalism underlying VeriFast. Used to express ownership of heap regions, absence of aliasing, and functional invariants of pointer-linked data structures. Contracts written as `requires`/`ensures` clauses using custom heap predicates. |
| **KLEE** | LLVM-based symbolic execution engine. Runs the stateless NAT logic with symbolic packet inputs, exploring all execution paths up to loop bounds. Checks assertions, memory safety, and RFC-compliance assertions. |
| **LLVM / Clang** | Compilation infrastructure. KLEE operates on LLVM bitcode; VigNAT's C source is compiled to LLVM IR for symbolic execution. |
| **DPDK** (Data Plane Development Kit) | High-performance user-space packet I/O framework. VigNAT is implemented on DPDK for production-grade throughput. DPDK itself is part of the TCB (trusted, not verified) in this paper. |
| **RFC 3022 Formal Spec** | The paper encodes RFC 3022 (Traditional NAT) as a formal mathematical model (abstract state machine over NAT mappings, session states, packet fields). This serves as the ground-truth specification against which VigNAT is verified. |
| **Z3 (implicit via KLEE)** | SMT solver used internally by KLEE to decide path feasibility and generate symbolic counterexamples. |

### 2.4 Key Data Structures / Models

**`libVig` Data Structures:**

- **Map:** A hash-map from keys (e.g., 5-tuples) to integer indices. Used to look up existing NAT sessions by flow key. Annotated with a separation-logic predicate capturing internal hash-table invariant, size bounds, and key-value correspondence.

- **Vector:** A heap-allocated array with verified bounds. Stores per-flow state (translated address, port, timestamps). Indexed by the integer handle returned by Map.

- **DoubleMap:** A two-keyed associative structure allowing lookup by either of two keys (e.g., external 5-tuple or internal 5-tuple). Used to locate sessions from either direction (inbound or outbound packet). Formally proved to keep both key views consistent.

- **Ring Buffer / Port Allocator:** A circular buffer tracking free port numbers for outbound NAT port assignment. Proved to never allocate a port beyond its configured range and to correctly cycle through available ports.

**Formal NAT State Model (RFC 3022 Abstract Machine):**

- **Mapping table:** A function `M : (internal_addr, internal_port, proto) → (external_port)` capturing the current set of established NAT mappings.
- **Session state:** Each session carries timestamps (for timeout), protocol type, and bidirectional 5-tuple information.
- **Transition relation:** Defines, for each input packet and current state `M`, whether to forward with translated fields, drop, or create/expire a mapping.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

VigNAT targets a single, specific NF type:

- **Stateful NAT (Traditional NAT per RFC 3022):** A NAT that rewrites source IP addresses and TCP/UDP/ICMP ports for outbound sessions and the reverse for inbound packets. This is the most common form of NAT found in home gateways, enterprise edge routers, and cloud infrastructure.

The paper focuses exclusively on this NF. The `libVig` library and verification methodology are designed to generalize (and were later extended in the Vigor framework), but VigNAT itself verifies only the NAT.

### 3.2 How It Validates NF Behavior

The validation pipeline is as follows:

1. **Formalize the intended NF behavior:** RFC 3022 is transcribed into a formal specification — a mathematical abstract state machine. The specification captures: session creation (outbound SYN / first UDP datagram), session timeout, port-rewriting rules for TCP/UDP, and ICMP translation rules.

2. **Verify the data structure library (VeriFast):** VeriFast is run offline on `libVig` source files annotated with separation-logic contracts. It mechanically verifies that each C function body satisfies its contract. Output: a VeriFast proof certificate.

3. **Verify the stateless NAT logic (KLEE):** KLEE is invoked on the NAT's packet-processing C function with:
   - All packet header fields made symbolic (source/destination IP, port, protocol, ICMP type/code).
   - `libVig` function calls replaced by their verified contracts (summary functions).
   - RFC 3022 specification assertions inserted at each decision point.
   
   KLEE explores all feasible paths exhaustively, checking every assertion on every path.

4. **Discharge RFC assertions:** For every path, KLEE checks that the output packet (post-rewriting) and the libVig state updates match the RFC 3022 formal model. If any path violates an assertion, it is a bug.

5. **Report:** If KLEE terminates without a counterexample (and VeriFast succeeds), the entire NAT is declared proved correct.

### 3.3 What Properties / Invariants Does It Prove?

VigNAT proves the following properties:

| Property | Description |
|---|---|
| **RFC 3022 Semantic Correctness** | For every possible input packet, the NAT's forwarding decision (forward vs. drop) and field rewriting (IP rewrite, port rewrite) exactly match the RFC 3022 abstract specification. |
| **Session Table Consistency** | The flow table always maintains a bijective mapping between internal and external 5-tuples; no two flows share an external port; lookups are always consistent with insertions. |
| **Port Allocation Correctness** | The NAT never allocates a port outside its configured range, never double-allocates a port to two concurrent sessions. |
| **ICMP Translation Correctness** | ICMP error messages (type 3/11) embed original IP/port headers; the NAT correctly translates embedded fields in outbound ICMP error packets. |
| **Session Timeout** | After the configured timeout period, sessions are expunged from the table, and stale entries are not matched. |
| **Memory Safety** | No buffer overflows, no null-pointer dereferences, no out-of-bounds reads/writes, no use-after-free anywhere in the codebase. |
| **Crash Freedom** | The NAT does not abort, divide-by-zero, or raise any undefined behavior on any possible packet input. |

### 3.4 Input Requirements

| Requirement | Detail |
|---|---|
| **C Source Code** | Full C source of the NAT (packet processing logic + `libVig` data structures) is required. The approach is white-box and source-code-level. |
| **VeriFast Annotations** | `libVig` functions must be manually annotated with separation-logic contracts (`requires`, `ensures`, predicate definitions). This is a significant one-time annotation effort (approx. ~3,000 lines of VeriFast proof annotations for libVig). |
| **Formal Specification** | RFC 3022 must be manually transcribed into a formal mathematical model (done once by the paper authors). |
| **KLEE Assertions** | Assertions encoding the formal spec must be manually inserted into the NAT packet-processing code at appropriate check points. |
| **KLEE Configuration** | Loop/recursion bounds must be set (KLEE requires bounded exploration; the paper bounds are derived from the finite NAT table size). |
| **DPDK Environment** | The NAT runs on DPDK; the verification assumes DPDK's API is correct (DPDK is not verified in VigNAT, only in later Vigor work). |

### 3.5 Guarantees Provided

- **Sound formal proof** (for the verified scope): The combination of VeriFast separation-logic proof + KLEE exhaustive symbolic execution yields a **mathematically sound guarantee** — not merely testing or model checking of a model. Every execution path is covered; there are no unsound approximations in the combined proof for the C code.
- **Machine-checked:** VeriFast's proof obligations are checked by its automated proof engine; KLEE's exhaustive exploration is mechanized.
- **Bounded by TCB:** The guarantee is contingent on the correctness of: the Clang/LLVM compiler, VeriFast itself, KLEE itself, the DPDK framework (unverified in VigNAT), and the hardware. These form the Trusted Computing Base.
- **Not a runtime guarantee:** Verification is entirely **offline/static**. There is no runtime monitoring or enforcement.
- **Not a network-level guarantee:** The proof ensures only that VigNAT's *implementation* is correct; it does not prove properties of a network in which VigNAT is deployed (e.g., routing correctness, policy compliance across a chain of NFs).

---

## 4. NF Chain Verification

**VigNAT is single-NF only.** The paper does not address verification of NF chains (service function chaining). The scope is strictly limited to one NAT instance in isolation.

**What is not handled:**

- **Composition:** VigNAT makes no guarantees about what happens when NAT is chained with a firewall, load balancer, or IDS. Semantics of composed NFs (e.g., whether NAT-then-firewall produces the intended combined policy) are not considered.
- **Ordering invariants:** The paper does not check whether swapping NAT with another NF in a chain would violate policy.
- **Stateful carry-over:** There is no analysis of how NAT state interacts with state maintained by adjacent NFs in a chain (e.g., a connection tracker that also maintains flow tables).
- **Inter-NF isolation:** Whether one NF's memory or state can corrupt another is not addressed.

**What would be needed to extend to chains:**

1. A **compositional specification framework** that can compose individual NF formal specifications into a chain-level specification (e.g., using assume-guarantee reasoning or sequential composition of abstract state machines).
2. **Cross-NF state interaction modeling** — particularly important when multiple NFs maintain flow tables over the same packet flows (e.g., NAT + connection-tracking firewall).
3. A **chain-level correctness criterion** (analogous to RFC 3022 for the individual NAT), defining what the end-to-end behavior of the chain should be for each packet.

The successor Vigor framework (NSDI 2019+) generalizes `libVig` and the methodology to other NF types, but still verifies each NF individually, not chains.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs **static CFG-NC (Control Flow Graph with Network Constraints) dataflow analysis** directly on **raw eBPF bytecode** — no source code required. It uses a **Prolog-based query engine** to check behavioral assertions about NF behavior, including support for **stateful BPF maps** (eBPF's primary state-keeping mechanism). It can analyze **NF chains** (sequences of eBPF programs sharing maps), checking cross-NF ordering, composition, and stateful carry-over properties. The analysis is purely offline and operates at the binary (bytecode) level.

### 5.2 Key Differences from This Paper

| Dimension | VigNAT | Yaksha-Prashna |
|---|---|---|
| **Input format** | C source code (annotated with VeriFast contracts) | Raw eBPF bytecode (binary) |
| **Source code required?** | Yes — mandatory | No — binary-only analysis |
| **Annotation burden** | High — thousands of lines of separation-logic annotations + KLEE assertions | None — fully automated |
| **Target NF type** | NAT (single specific NF) | Any eBPF-based NF |
| **Verification technique** | Separation logic (VeriFast) + symbolic execution (KLEE) | CFG-NC dataflow analysis + Prolog assertion engine |
| **Stateful analysis** | Yes — via `libVig` verified data structures | Yes — via eBPF map modeling in CFG-NC |
| **NF chain support** | No — single NF only | Yes — multi-NF chains |
| **Guarantee type** | Machine-checked formal proof (sound) | Static assertion checking (soundness depends on model precision) |
| **Runtime overhead** | None (offline) | None (offline) |
| **Infrastructure modeled** | DPDK (trusted, not verified) | eBPF verifier + kernel (partially modeled) |
| **Generalizability** | Requires re-annotation and re-proof per new NF | Works on any eBPF bytecode without modification |
| **Performance** | Verifies that performance-optimized C code is correct | Analyzes production eBPF programs already deployed |

### 5.3 How This Paper Is Useful For Us

1. **Motivating citation for source-code dependence as a limitation:** VigNAT explicitly requires C source code and thousands of lines of manual VeriFast annotations. We can cite this as a concrete example of how existing deep-verification approaches are impractical to deploy at scale or on third-party/closed-source NFs — a gap that Yaksha-Prashna addresses by working directly on eBPF bytecode.

2. **Establishing the gold-standard baseline for formal NF verification:** VigNAT is the seminal paper in formally verifying NF implementations (rather than models). Positioning our work relative to it establishes academic seriousness. We can cite VigNAT as "the state of the art for single-NF C source verification" and then contrast with Yaksha-Prashna's bytecode-level, chain-aware approach.

3. **RFC-level behavioral specification as a reference:** VigNAT's methodology of encoding RFC 3022 as a formal spec against which the implementation is checked is directly analogous to what Yaksha-Prashna does by encoding behavioral assertions in Prolog. The difference is our assertions are checked on bytecode without needing to embed them in source. This parallel strengthens our framing.

4. **Illustrating the annotation burden:** The ~3,000 lines of VeriFast annotations for libVig alone, plus KLEE instrumentation, is a compelling data point showing the engineering cost of existing approaches. Yaksha-Prashna eliminates this entirely.

5. **NF chain gap:** VigNAT's explicit limitation to single-NF verification is an opportunity for us to position Yaksha-Prashna's chain-level analysis as extending beyond the state of the art.

6. **Performance data as a reference point:** VigNAT's performance evaluation (1.8 Mpps, ~0.38 μs latency) establishes that formally verified NFs can match unverified ones in performance. We can reference this to argue that verification overhead is not a concern for offline static methods like ours.

### 5.4 Positioning Statement

> "While VigNAT achieves sound formal proof of NAT correctness using separation logic and symbolic execution, it requires full C source code, thousands of lines of manual VeriFast annotations, and is limited to a single NF in isolation. Yaksha-Prashna addresses these barriers by performing behavioral assertion checking directly on raw eBPF bytecode — requiring no source code, no annotations, and supporting multi-NF chains with stateful BPF map modeling — making formal-style NF verification practical for production eBPF deployments."

---

*Analysis prepared for the NF Validation Research Project — Paper ID: D2*
