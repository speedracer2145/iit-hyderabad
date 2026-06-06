# Automated Verification of Customizable Middlebox Properties with Gravel

**Authors:** Kaiyuan Zhang, Danyang Zhuo, Aditya Akella, Arvind Krishnamurthy, Xi Wang  
**Year:** 2020 | **Venue:** USENIX NSDI 2020  
**DOI/Link:** https://wisr.cs.wisc.edu/papers/nsdi20-gravel.pdf  
**Artifact:** https://github.com/Kaiyuan-Zhang/Gravel-public

> **Note on Paper ID H3 Metadata:** The caller metadata labels this as "Differential Testing of Click Middleboxes (Gravel/USENIX approach)" with a differential-testing framing. While Gravel does compare implementation behavior against specifications, the actual technique is **formal verification via symbolic execution + SMT solving**, not black-box differential testing against a reference implementation (e.g., Linux kernel NAT or iptables). The analysis below reflects the actual published methodology.

---

## 1. Overview

Software middleboxes — NATs, firewalls, load balancers, proxies — are critical components in modern networks, but verifying their correctness is notoriously difficult. They are stateful, written in low-level systems languages (C++), and expected to adhere to complex protocol semantics defined by RFCs. Existing formal verification tools (e.g., Coq-based methods for Verdi/IronFleet) require enormous manual proof effort — often a 10:1 proof-to-code ratio — making them impractical for real-world deployment.

Gravel addresses this by enabling **automated, property-based formal verification of Click modular router middleboxes** with minimal code changes and zero manual proof work. It leverages the modular, element-based architecture of Click: individual elements perform a small, bounded set of operations, making them amenable to symbolic execution. Gravel compiles the C++ implementation of Click elements to LLVM IR, symbolically executes each element to derive a formal behavioral model, and then uses the Z3 SMT solver to automatically check whether element compositions satisfy high-level properties specified by the developer in Python.

The core contribution is a two-stage verification architecture that separates (1) element-level equivalence checking (C++ code vs. element spec) from (2) configuration-level property checking (element composition vs. system-level trace properties). This separation ensures that verification scales without requiring whole-program symbolic execution. The paper demonstrates that 45% of existing Click elements can be verified automatically with no code changes, and another 33% with minor modifications, enabling verification of complete middleboxes including MazuNAT, a stateful firewall, a load balancer, a web proxy, and a learning switch.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

Gravel's core insight is that Click middleboxes, being composed of small, modular elements, are structurally well-suited for symbolic execution. Each element performs a finite, bounded number of operations per packet, and data flow between elements follows a statically defined graph (the Click configuration). By exploiting this structure, Gravel avoids the state explosion that plagues whole-program symbolic execution.

The verification methodology has two decoupled stages:

1. **Element-Level Verification:** Each Click element's C++ implementation is compiled to LLVM IR and symbolically executed. The resulting symbolic expression — capturing the element's behavior over symbolic packet inputs — is compared against a formal **element specification** (written in Gravel's Python-based DSL). Z3 discharges the equivalence condition automatically.

2. **Configuration-Level Verification:** Given that each element has been verified against its spec, Gravel composes element specs according to the Click configuration graph and checks the composed behavior against **high-level, trace-based system properties** (e.g., connection persistency for a load balancer, endpoint-independent mapping for a NAT). This stage operates purely on the abstract specs — not on C++ — making it highly efficient.

The key design principle is that once an element is verified against its spec, the spec (not the code) is used in all subsequent compositional reasoning. This modularity means new middleboxes can be verified by composing already-verified elements without re-running symbolic execution.

### 2.2 Process Steps

1. **Input:** C++ source code of Click elements + a Python property specification describing desired middlebox behavior.
2. **Compilation:** The C++ source of each element is compiled to LLVM IR using Clang.
3. **Symbolic Execution:** Gravel symbolically executes the LLVM IR of each element with symbolic packet inputs, computing a symbolic expression tree representing element behavior (packet output fields, side effects on state structures).
4. **Element Spec Generation / Equivalence Check:** The symbolic expression is compared against the element's formal specification using Z3. If they match, the element is marked verified. If not, Z3 produces a counterexample (a concrete packet input that triggers divergence).
5. **Configuration-Level Composition:** Gravel instantiates the Click configuration graph using verified element specs. Packet flow is symbolically traced through the composed element chain.
6. **Property Checking:** High-level trace properties (specified in Python as logical predicates over packet sequences and middlebox state) are checked against the composed symbolic behavior using Z3.
7. **Output:** Either a proof of correctness (for each specified property), or a counterexample trace identifying which property is violated and under which packet sequence.
8. **Code Modification (if needed):** For elements where symbolic execution finds divergence from spec (due to unsupported patterns like loops, dynamic dispatch, etc.), the developer makes targeted, minimal code changes to bring the element within Gravel's verifiable fragment.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in Gravel |
|---|---|
| **Click Modular Router** | Target framework; provides the modular element + configuration graph structure that Gravel exploits |
| **LLVM IR** | Intermediate representation produced from C++ via Clang; Gravel's symbolic execution operates at this level |
| **Symbolic Execution** | Core technique for deriving formal behavioral models of individual Click elements from their LLVM IR |
| **Z3 SMT Solver** | Automatically discharges verification conditions at both element-level (equivalence) and configuration-level (property checking) |
| **Python DSL (Property Language)** | Developer-facing language for specifying both element-level specs and high-level system-level trace properties |
| **Trace-Based Property Specification** | Formalism for expressing temporal/behavioral properties as sequences of packet processing events and state transitions |
| **Compositional Verification** | Design principle: element specs compose via configuration graph; no whole-program analysis needed |
| **Counterexample-Guided Iteration** | When Z3 finds a violation, it produces a concrete counterexample helping the developer diagnose bugs or refine the implementation |

### 2.4 Key Data Structures / Models

- **Element Symbolic Expression:** An abstract representation of what an element does to a packet — a mapping from symbolic input packet fields + symbolic element state to output packet fields + updated state. This is the unit of compositional reasoning.
- **Element Specification (Spec):** A formal model (Python object) of an element's intended behavior. Gravel verifies that the symbolic expression derived from LLVM IR matches this spec. Pre-verified elements (e.g., `IPRewriter`, `Classifier`, `LookupIPRouteMP`) have library specs.
- **Click Configuration Graph (Data Graph):** A directed graph where nodes are Click elements and edges are packet forwarding paths. Gravel traverses this graph to compose element behaviors.
- **Symbolic Packet Trace:** A sequence of symbolic packet processing events across the element chain, used as the unit for high-level property checking.
- **State Abstraction / Ghost Maps:** For stateful elements (e.g., NAT connection tables, load balancer session maps), Gravel models state using abstract map data structures (HashSet, HashMap, Vector) with verified operations. These serve as the "ghost state" enabling correctness proofs over multiple packets.
- **Verification Conditions (VCs):** Logical formulas expressing the property to be proved; passed to Z3. If Z3 returns `unsat`, the property holds. If it returns `sat` with a model, a counterexample is produced.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

Gravel targets **Click-based software middleboxes** specifically. The paper evaluates the following concrete NF types:

- **MazuNAT** — a production-grade NAT implemented in Click (Network Address Translation)
- **Stateful Firewall** — a connection-tracking firewall built from Click elements
- **Load Balancer** — a consistent-hashing or persistent-connection load balancer
- **Web Proxy** — an HTTP proxy handling persistent connections
- **Learning Switch** — a MAC-learning Layer-2 switch

All target NFs must be implemented in C++ using the Click modular router framework. Non-Click NFs (e.g., DPDK-based, kernel NFs, eBPF NFs) are outside scope.

### 3.2 How It Validates NF Behavior

Gravel performs **formal verification** — not testing — of NF correctness. The process:

1. **Specify Properties:** The developer writes high-level behavioral properties in Python. For example, for MazuNAT: "endpoint-independent mapping" (RFC 4787 §4.1) — a NAT must always use the same external port for packets from the same internal address/port, regardless of destination.
2. **Compile and Symbolically Execute:** Each Click element in the NF is compiled to LLVM IR and symbolically executed by Gravel to derive its symbolic expression.
3. **Verify Element-Level Equivalence:** Each element's symbolic expression is checked against its library or custom spec using Z3. This catches implementation bugs at the element level.
4. **Compose and Check System Properties:** Verified element specs are composed per the Click configuration graph. Z3 checks whether the composed behavior satisfies each property from step 1.
5. **Identify Violations:** If a property is violated, Z3 provides a concrete packet sequence and state that witnesses the violation. The developer either fixes the implementation or amends the property spec.
6. **Iterate:** Minor code modifications (e.g., 133 lines changed in MazuNAT's `IPRewriter`) bring elements into the verifiable fragment and/or fix identified bugs.

The MazuNAT case study is illustrative: the original implementation violated RFC 4787 endpoint-independent mapping because it keyed flow records on the 5-tuple (src IP, src port, dst IP, dst port, protocol) rather than the 2-tuple (src IP, src port). Gravel's verification exposed this semantic bug.

### 3.3 What Properties / Invariants Does It Prove?

Properties are expressed as trace-based logical predicates over packet sequences and middlebox state. The paper verifies:

- **Endpoint-Independent Mapping (RFC 4787 §4.1):** For NAT — same internal (IP, port) always maps to the same external port, regardless of destination. Violation found in original MazuNAT.
- **Connection Persistency (Load Balancer):** Once a flow is assigned to a backend server, all subsequent packets from that flow are forwarded to the same server.
- **Stateful Firewall Correctness:** Only packets belonging to established, permitted connections pass through; unsolicited inbound packets are dropped.
- **Packet Rewriting Correctness:** NAT correctly rewrites source/destination IP and port fields in both directions (forward and reverse paths).
- **State Consistency:** The NF's internal connection tables remain consistent across packets (no stale entries used for active connections, no missing entries for established ones).
- **No Unintended Packet Dropping:** For NFs that should forward all valid packets, no valid packet is incorrectly dropped.
- **Custom/User-Specified Properties:** Gravel's property language is extensible; operators can specify domain-specific invariants beyond those listed above.

### 3.4 Input Requirements

| Input | Description |
|---|---|
| **C++ Source Code** | Full source of all Click elements used in the middlebox — required for LLVM IR compilation and symbolic execution |
| **Click Configuration File** | The `.click` configuration file specifying element composition and data graph topology |
| **Element Specs (Library or Custom)** | Formal specifications for each element; many standard Click elements have pre-written Gravel specs; custom elements require spec authorship |
| **Property Specifications (Python)** | High-level trace-based properties to verify, written in Gravel's Python DSL |
| **Minor Code Annotations/Modifications** | For ~33% of elements, small code changes needed to remain in Gravel's verifiable fragment (e.g., eliminating unsupported loop patterns, replacing dynamic dispatch) |

**No binary-only NFs, no eBPF bytecode, no kernel modules** — Gravel requires C++ source and is tightly coupled to the Click framework.

### 3.5 Guarantees Provided

- **Soundness (for verified elements):** If Z3 returns `unsat`, the verified property is provably correct for all possible packet inputs and all reachable states — not just tested cases. This is a **formal proof**, not a coverage estimate.
- **Counterexample on Violation:** If a property is violated, Z3 produces a concrete counterexample — a specific packet sequence and state configuration that witnesses the failure.
- **Scope Caveat:** The proof applies only to the verifiable fragment of the implementation. Elements requiring code modification are re-verified post-modification. The proof does not cover external libraries, OS interactions, or hardware-level behaviors.
- **No Runtime Overhead:** Gravel is a static verification tool; verified middleboxes run at full Click performance with no instrumentation overhead.
- **Bug Discovery Guarantee:** In practice, Gravel found previously unknown semantic bugs in MazuNAT (RFC 4787 compliance violation) through the verification process.

---

## 4. NF Chain Verification

Gravel has **limited, intra-middlebox chain support** — it does not address multi-middlebox service chain verification in the SDN/NFV sense.

**What Gravel does support (intra-NF element chains):**
- Within a single Click-based middlebox, packets flow through a *directed acyclic graph (DAG) of Click elements* (e.g., Classifier → IPRewriter → RoutingTable → Output). Gravel composes element specs along this internal graph and verifies end-to-end properties over the entire element chain within one middlebox. In this sense, it does handle internal "chaining" of processing elements.
- The two-stage architecture (element-level + configuration-level) is specifically designed to scale compositional verification through this internal element chain without re-running symbolic execution for each element.

**What Gravel does NOT support (multi-NF service chains):**
- Gravel is scoped to a **single Click middlebox instance**. It does not model packet flow across multiple independent NFs in a network service chain (e.g., Firewall → NAT → Load Balancer as separate deployed boxes).
- There is no mechanism for composing the formal models of two separate, independently verified middleboxes to check inter-NF properties (e.g., does the NAT's port-rewriting interfere with the firewall's connection tracking?).
- Stateful carry-over across NF boundaries — where state in one NF affects correctness in another — is not addressed.
- To extend Gravel to multi-NF chains, one would need: (a) a shared inter-NF packet model, (b) a way to compose the symbolic trace models of separate Click instances, and (c) properties that span NF boundaries. None of this is provided.

**Verdict:** Gravel is **single-middlebox only** for the purposes of chain verification. Its internal element graph composition is a useful design pattern, but does not constitute service chain verification.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs **static CFG-level Network-Centric (CFG-NC) dataflow analysis directly on raw eBPF bytecode** — no source code required. It uses a **Prolog-based query engine** to perform behavioral assertion checking over eBPF programs, supporting stateful analysis through BPF map modeling. It is designed to handle **NF chains** (multiple eBPF programs in sequence), checking inter-NF properties, ordering invariants, and stateful carry-over across chain boundaries — without any source code, annotations, or framework dependencies.

### 5.2 Key Differences from This Paper

| Dimension | Gravel (NSDI 2020) | Yaksha-Prashna |
|---|---|---|
| **Input Format** | C++ source code (LLVM IR via Clang) | Raw eBPF bytecode (no source needed) |
| **NF Framework** | Click modular router only | Any eBPF-based NF (XDP, TC, cgroup) |
| **Verification Technique** | Symbolic execution + Z3 SMT solving | CFG-NC dataflow analysis + Prolog query engine |
| **Property Language** | Python trace-based DSL | Prolog-based behavioral assertion queries |
| **NF Chain Support** | Single-middlebox only (internal element graph) | Multi-NF chains; cross-boundary stateful analysis |
| **Stateful Analysis** | Yes (abstract state maps within one NF) | Yes (BPF map modeling across chain) |
| **Code Modification Required** | Yes (~33% of elements) | No modifications — bytecode is the input |
| **Deployment Context** | Design-time / development-time | Test-time / deployment-time (bytecode in hand) |
| **Formalism** | SMT-based proof (Z3) | Prolog-based logical inference |
| **Guarantees** | Sound formal proof for verified elements | Behavioral assertion checking over all bytecode paths |
| **Annotation Required** | Yes (property specs, element specs in Python) | Minimal (query in Prolog) |
| **Target Technology** | C++ Click (userspace software router) | eBPF (Linux kernel data plane) |

### 5.3 How This Paper Is Useful For Us

1. **Motivating the Source-Code Requirement Gap:** Gravel's primary limitation is the hard requirement for C++ source code. Our work directly addresses this by operating on compiled bytecode. We can cite Gravel to motivate why source-code-dependent verification is insufficient for eBPF NFs deployed in production (where operators have bytecode, not source).

2. **Two-Stage Compositional Verification as Design Inspiration:** Gravel's separation of element-level and configuration-level verification is a principled compositional approach. We can draw conceptual parallels with Yaksha-Prashna's approach of analyzing individual eBPF programs and then reasoning about chain-level properties — framing our work as achieving similar compositional goals but without source code.

3. **Baseline for Property Types:** The RFC-based properties Gravel verifies (endpoint-independent mapping, connection persistency, stateful firewall correctness) are directly relevant to the kinds of behavioral assertions Yaksha-Prashna checks. We can use these as a shared vocabulary of NF correctness properties.

4. **Click → eBPF Gap:** Gravel targets Click, which is increasingly less common than eBPF for high-performance NFs. This gap in coverage directly motivates our work. Gravel's limitation to the Click ecosystem, while thorough, leaves the vast and growing eBPF NF ecosystem unaddressed.

5. **Counterexample / Bug Discovery Comparison:** Gravel found a real RFC 4787 compliance bug in MazuNAT. If Yaksha-Prashna can similarly identify bugs in eBPF NATs/firewalls, citing Gravel establishes precedent for the value of formal verification in this space.

6. **SMT vs. Prolog Tradeoff:** Gravel's use of Z3 enables strong formal guarantees but requires expressing properties as SMT formulae (via the Python DSL). Yaksha-Prashna's Prolog engine makes behavioral assertions more declarative and inspectable. This is a useful point of contrast in a related work section.

### 5.4 Positioning Statement

> "While Gravel achieves formal SMT-based verification of Click middlebox properties with strong soundness guarantees, it requires full C++ source code and is architecturally restricted to the Click modular router ecosystem, leaving the rapidly growing eBPF-based NF data plane entirely unaddressed. Yaksha-Prashna bridges this gap by operating directly on raw eBPF bytecode — requiring no source code, no annotations, and no framework coupling — while extending verification scope to multi-NF service chains through its CFG-NC dataflow analysis and Prolog-based assertion engine."

---

## Appendix: Summary of Gravel's Evaluation Results

| Middlebox | Elements Modified | LOC Changed | Properties Verified | Bugs Found |
|---|---|---|---|---|
| MazuNAT | `IPRewriter` + 2 others | ~133 of 1,687 | Endpoint-independent mapping, packet rewriting correctness | 1 (RFC 4787 §4.1 violation) |
| Load Balancer | Minimal | ~50 of ~800 | Connection persistency | None (property holds) |
| Stateful Firewall | Minimal | Small | Stateful connection filtering | None reported |
| Web Proxy | Several | ~50 of 953 | Connection tracking correctness | None reported |
| Learning Switch | Minimal | Minimal | MAC learning correctness | None reported |

**Click element coverage:** 45% verifiable automatically, 33% verifiable with minor changes, ~22% require more significant restructuring or are outside Gravel's verifiable fragment (unbounded loops, heavy dynamic dispatch).

---

*Analysis written for the Yaksha-Prashna NF validation research survey. Paper: "Automated Verification of Customizable Middlebox Properties with Gravel," Zhang et al., USENIX NSDI 2020.*
