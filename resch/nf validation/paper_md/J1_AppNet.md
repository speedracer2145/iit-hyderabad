# AppNet: High-Level Programming for Application Networks

**Authors:** Xiangfeng Zhu, Yuyao Wang, Banruo Liu, Yongtong Wu, Nikola Bojanic, Jingrong Chen, Gilbert Louis Bernstein, Arvind Krishnamurthy, Sam Kumar, Ratul Mahajan, Danyang Zhuo  
**Year:** 2025 | **Venue:** USENIX NSDI 2025 (22nd USENIX Symposium on Networked Systems Design and Implementation)  
**DOI/Link:** https://www.usenix.org/conference/nsdi25/presentation/zhu-xiangfeng  
**Affiliations:** University of Washington, Duke University, UCLA

---

## 1. Overview

Modern microservice architectures rely heavily on **application networks** — layers of network functions that sit between services and handle cross-cutting concerns such as authentication, rate limiting, load balancing, circuit breaking, distributed tracing, and fault injection. These are commonly deployed as **service meshes** (e.g., Istio/Envoy), where each microservice is paired with a sidecar proxy that intercepts all inbound and outbound RPC traffic. While functionally powerful, this approach imposes severe performance overhead: service mesh proxies can double RPC latency and consume significant CPU resources, even for simple processing tasks.

The root problem is that existing service mesh implementations force developers to specify network functions at a low level — writing Envoy filters, WebAssembly modules, or eBPF programs that operate on raw bytes. This "black box" approach makes it impossible for the system to reason about the *semantic meaning* of what different network functions do, and therefore impossible to automatically optimize their placement or execution. Worse, the proxy-only deployment model is inflexible: some functions would run more efficiently inside the RPC library (in-process, zero-hop), while others legitimately need a sidecar, and still others could run on the sender or receiver side — but without semantic understanding, the system cannot make these tradeoffs automatically.

AppNet addresses this by introducing a **high-level domain-specific language (DSL) for Application Network Functions (ANFs)**, paired with an **optimizing compiler** that uses symbolic abstraction and Z3-based SMT equivalence checking to reason about whether different deployment configurations of a stateful ANF chain are semantically equivalent. If they are, the compiler can safely choose the most performance-efficient configuration without changing observable behavior. The result is a system that expresses common ANFs in 7–28 lines of high-level code, while achieving up to 82% reduction in RPC latency and 75% reduction in CPU usage compared to naive deployments — all with a formal semantic-equivalence guarantee.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

AppNet's core contribution is a **semantic-aware compiler** that transforms high-level, platform-agnostic ANF specifications into optimized, platform-specific deployments. The compilation pipeline has three major concerns:

1. **Expressiveness:** The DSL must be rich enough to capture stateful, field-aware RPC processing (matching on header values, payload fields, and accumulated state) while remaining analyzable by automated tools.

2. **Placement Optimization:** Given a chain of processing elements, the compiler must decide *where* each element runs — inside the RPC library (in-process, highest performance), in a co-located sidecar proxy (medium overhead), or in some dedicated middlebox — and *on which side* of the RPC (client or server).

3. **Semantic Equivalence Verification:** Before any placement transformation is applied, the compiler must formally verify that the original chain and the optimized chain are *semantically equivalent* — i.e., they produce identical outputs for all possible RPC streams. AppNet uses **symbolic abstraction** to represent element behavior as symbolic state-transition functions, then encodes the equivalence question as an SMT formula and queries **Z3** to determine satisfiability. If Z3 cannot find a stream on which the two configurations differ, equivalence is established.

The language is built around **match-action rules**, where each element specifies conditions on RPC fields and internal state, and associated actions that modify fields or update state. Elements are composed into an **element graph** — a directed acyclic graph representing the chain of ANFs applied to each RPC request/response.

### 2.2 Process Steps

1. **Developer writes ANF specification:** The operator defines each processing element using AppNet's high-level DSL. Each element contains match-action rules referencing RPC header fields, payload fields, and typed state variables (counters, maps, queues). A chain of elements is defined by composing them into an element graph.

2. **Parsing and type-checking:** The AppNet compiler parses the element graph, type-checks field references and state variable types, and resolves dependencies between elements (i.e., which elements read or write shared state).

3. **Symbolic abstraction of elements:** Each element is converted into a **symbolic transition function** — a mathematical description of how it transforms the symbolic RPC state (fields, state variables) when a request or response passes through. This abstraction captures the element's effect on *any* possible RPC stream, not just concrete examples.

4. **Configuration space enumeration:** The compiler enumerates candidate deployment configurations. A configuration specifies, for each element: (a) whether it runs client-side or server-side, (b) whether it runs in the RPC library or in a proxy, and (c) whether stateful elements requiring shared state are co-located.

5. **SMT equivalence checking via Z3:** For each candidate configuration, the compiler encodes the question "do the original configuration and this candidate produce identical outputs on all possible RPC streams?" as an SMT formula. It queries Z3. If Z3 returns UNSAT (no counterexample), the candidate is declared semantically equivalent to the original.

6. **Cost-based selection:** Among all equivalent configurations, the compiler selects the lowest-cost one using a heuristic cost model. The cost model penalizes: extra network hops (proxy sidecars add two hops per RPC), remote state access, and cross-process communication. In-library execution is cheapest; cross-node middlebox placement is most expensive.

7. **Code generation:** The compiler emits platform-specific implementation code for the selected configuration. AppNet supports multiple backends: gRPC interceptors (for in-library), Envoy native filters, Envoy WebAssembly (Wasm) filters, and eBPF programs (for sender-side, low-level network processing).

8. **Runtime controller deployment:** A centralized controller deploys the generated code to the appropriate locations (gRPC interceptor JAR, Envoy filter binary, eBPF object file loaded into the kernel), and reconfigures the data plane.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in AppNet |
|---|---|
| **Z3 SMT Solver** | Core equivalence checker — encodes the "are these two ANF configurations observationally equivalent on all RPC streams?" question as a satisfiability problem; UNSAT means equivalent |
| **Symbolic Abstraction / Symbolic Execution** | Converts each match-action element's behavior into a symbolic transfer function over symbolic RPC field values and state variables, enabling formal reasoning without concrete inputs |
| **Match-Action Rules (AppNet DSL)** | High-level language in which operators define ANFs; supports conditional matching on RPC metadata and payload fields, with typed state (maps, counters, queues) |
| **Element Graph** | DAG-based internal IR representing the composed chain of ANFs; nodes are elements, edges represent data/control flow between them |
| **Cost Model (Heuristic Optimization)** | Assigns a numeric cost to each candidate placement configuration; factors include number of network hops, proxy overhead, state locality, and platform-specific overhead |
| **gRPC Interceptors** | In-process backend for generated code — zero-hop, lowest latency deployment option |
| **Envoy Native Filters / Envoy Wasm** | Proxy-based backends — used when in-library execution is semantically infeasible (e.g., requires mutual TLS termination, cross-service state aggregation) |
| **eBPF (BPF Maps)** | Kernel-level backend for sender-side ANFs — generated when packet-level interception is needed; BPF maps used to maintain stateful variables across packet boundaries |

### 2.4 Key Data Structures / Models

- **Element Graph (DAG):** Each node represents one ANF element (e.g., "RateLimiter", "AuthCheck", "Logger"). Edges represent the RPC stream flowing through them in sequence. Elements may read/write shared state variables.
- **Symbolic RPC State:** A vector of symbolic bitvector values representing all named fields of an RPC message (headers + payload fields), plus symbolic state variable values. This is the input to the symbolic transfer function of each element.
- **Symbolic Transfer Function:** A mathematical function `f: (RPC_state, State_vars) → (RPC_state', State_vars', Action)` capturing how an element transforms an RPC. Action may be: `FORWARD`, `DROP`, `REPLY`. Composed symbolically to produce the chain-level transfer function.
- **Placement Configuration:** A mapping `element → (side: {client, server}, platform: {library, proxy, eBPF})`. The full space is exponential in the number of elements; Z3 is used to prune only semantically valid configurations.
- **SMT Equivalence Formula:** Given two placement configurations C1, C2, the formula `∃ stream S such that output(C1, S) ≠ output(C2, S)` is checked for satisfiability. UNSAT → equivalence proven.
- **BPF Maps:** For eBPF-backend elements, stateful variables (e.g., per-client request counters, rate limit windows) are stored in BPF hash maps and array maps, keyed by RPC source/destination identifiers.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

AppNet targets **Application-Level Network Functions (ANFs)** — a class of L7 middlebox functions that operate on structured RPC requests and responses rather than raw packets. Specifically:

- **Access Control / Authentication:** Match on RPC caller identity headers, JWT tokens, or service account fields; allow or deny RPCs based on policy rules.
- **Rate Limiting:** Track per-client or per-service RPC counts using stateful counters; enforce rate thresholds by dropping excess RPCs.
- **Load Balancing:** Inspect RPC payload fields or headers to route requests to specific backend replicas (content-aware load balancing).
- **Circuit Breaking:** Track error-rate state per backend replica; stop forwarding RPCs to failing backends when error thresholds are exceeded.
- **Distributed Tracing / Logging:** Inject trace headers into outgoing RPCs; log request/response fields to external collectors.
- **Fault Injection:** Probabilistically inject delays or errors into RPC streams for resilience testing.
- **Header Manipulation:** Modify, add, or strip RPC metadata fields in transit.

These correspond precisely to **sidecar proxy functions** in service mesh architectures (Istio/Envoy), **gRPC interceptors**, and **L7 gateway middleware**.

### 3.2 How It Validates NF Behavior

AppNet's validation mechanism is a **compile-time semantic equivalence check** rather than a behavioral conformance test. The process works as follows:

1. **User provides the original element graph:** The operator writes the ANF specification in AppNet's DSL. This is the *reference semantics* — what the chain must compute.
2. **Compiler generates candidate optimized configurations:** By exploring different placements (client vs. server side, in-library vs. proxy vs. eBPF) for each element.
3. **Symbolic abstraction of both configurations:** Both the reference chain and the candidate optimized chain are converted into symbolic transfer functions over abstract RPC stream values.
4. **Z3 SMT query for equivalence:** The compiler encodes `∃ RPC stream S: output(reference, S) ≠ output(candidate, S)` as an SMT formula. If Z3 returns UNSAT, no such stream exists — the two configurations are semantically equivalent.
5. **Safe transformation applied:** Only if equivalence is proven does the compiler proceed to generate optimized code for that configuration.

This validates that moving, reordering, or re-platforming elements does not alter observable behavior for *any* possible input — a universal quantification over all RPC streams, not just tested examples.

### 3.3 What Properties / Invariants Does It Prove?

- **Semantic equivalence of placement configurations:** For any sequence of RPC requests and responses, two configurations produce identical output (same forwarded/dropped decisions, same modified field values, same state updates visible to the application).
- **Output equivalence under reordering:** When two elements in a chain are commutative (their effects do not depend on order), this is formally verified via the SMT check, allowing the compiler to reorder them for performance.
- **State consistency under location change:** When a stateful element is moved from server-side to client-side (or vice versa), the symbolic check ensures the state evolution and its effects on RPC outcomes remain identical.
- **Drop/forward equivalence:** The compiler verifies that RPC drop decisions (e.g., access control denials, rate limit enforcements) are preserved exactly across configurations.

Note: AppNet does *not* prove correctness of the ANF specification itself with respect to an external behavioral specification or RFC — it proves only that optimization preserves the operator-specified semantics.

### 3.4 Input Requirements

| Input | Description |
|---|---|
| **ANF Element Definitions** | Written in AppNet's match-action DSL by the operator. Each element specifies match conditions (on RPC fields, state), actions (forward, drop, modify, update state), and state variable declarations (type, scope, key). |
| **Element Graph Composition** | Operator specifies which elements are chained together and in which order. |
| **State Scope Annotations** | Operator optionally annotates whether state is per-client, per-server, per-connection, or global, which constrains valid placements. |
| **Consistency Preference** | Operator may specify strong vs. eventual consistency for shared state, affecting which placement options are valid. |

No binary, pre-existing NF implementation, or external specification is required. The DSL itself is the complete input. The approach is **source-level** and requires operators to write (or translate) their ANF logic in AppNet's language.

### 3.5 Guarantees Provided

- **Sound formal guarantee:** If Z3 returns UNSAT for the equivalence query, the optimized configuration is provably semantically equivalent to the original for *all* possible RPC streams — this is a sound, machine-checked guarantee, not a test-based approximation.
- **Completeness caveat:** Soundness holds within AppNet's symbolic model. The model assumes match-action rules with decidable predicates (bitvector arithmetic, linear arithmetic on counters). Arbitrary opaque functions in element bodies would not be representable symbolically.
- **Performance guarantee:** The selected configuration is the lowest-cost semantically-equivalent configuration found, per the cost model.
- **No runtime overhead:** All verification is performed at compile time; the generated code carries no verification instrumentation at runtime.

---

## 4. NF Chain Verification

AppNet **explicitly handles chains of multiple ANFs in sequence** — this is, in fact, the central use case the paper is designed for. The element graph is a directed composition of multiple elements, and semantic equivalence checking is performed over the *composed* chain, not individual elements in isolation.

**Chain-level properties checked:**

- **Composed equivalence:** The full chain's symbolic transfer function (computed by sequentially composing each element's transfer function) is compared between the original and the optimized configuration. This accounts for interdependencies: if element A modifies a field that element B reads, this data dependency is captured in the composed function.
- **Ordering/commutativity:** If the compiler wishes to reorder elements (e.g., move rate limiting before authentication, or vice versa), it checks whether the two orderings are equivalent via the SMT query. If they are commutative with respect to arbitrary RPC streams, reordering is safe.
- **Stateful carry-over:** State updated by one element and read by a later element in the chain is modeled symbolically. The compiler tracks which state variables are shared and ensures that placement decisions respect data dependencies — e.g., two elements sharing a counter cannot be split to client and server sides without synchronization.
- **Isolation between chains:** Elements in different ANF chains (e.g., for different service pairs) are treated independently. AppNet does not verify cross-chain interactions.

**Chain composition limitations:**

- The approach handles **linear or DAG-shaped chains** of elements. Cyclic dependencies (if any arose) would require additional fixpoint reasoning.
- The SMT decidability assumption constrains the richness of per-element logic. Elements with complex loops or recursive functions cannot be fully symbolically abstracted.
- The paper focuses on **single-hop RPC chains** — the chain of ANFs applied to a *single* RPC call between two services. Multi-hop chains (across multiple service-to-service RPCs in a call graph) are not explicitly addressed as a composition unit.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs **static CFG-NC (Control Flow Graph with Network Context) dataflow analysis on raw eBPF bytecode**, without requiring source code. It uses a **Prolog-based query engine** to check behavioral assertions over the dataflow facts extracted from the bytecode. It handles **chains of eBPF-based NFs** attached to the Linux networking stack, and is aware of **stateful BPF maps** as inter-NF communication channels. The system can verify properties such as packet forwarding invariants, access control correctness, and map consistency across NF chains.

### 5.2 Key Differences from This Paper

| Dimension | AppNet | Yaksha-Prashna |
|---|---|---|
| **Input format** | High-level DSL source code (AppNet language) | Raw compiled eBPF bytecode (no source needed) |
| **NF abstraction level** | Application-layer (L7 RPC fields, headers, payloads) | Network layer (packets, BPF map operations, kernel hooks) |
| **Verification technique** | Z3 SMT semantic equivalence checking | Prolog-driven dataflow analysis + behavioral assertions |
| **What is verified** | Equivalence between original and optimized NF chain configurations | Behavioral correctness properties of eBPF NF chains (safety, invariants) |
| **NF target** | RPC proxies, sidecars, L7 service mesh elements | eBPF XDP/TC programs, kernel-attached packet processors |
| **Chain handling** | Yes — element graph composed symbolically, multi-element equivalence | Yes — multi-NF chains via BPF map dataflow tracking |
| **Stateful reasoning** | Symbolic (abstract state in SMT formulas) | Concrete BPF map types and key schemas from bytecode analysis |
| **Guarantee type** | Provably sound equivalence (Z3 UNSAT certificate) | Static analysis soundness (conservative over-approximation) |
| **Runtime model** | gRPC interceptors, Envoy, WebAssembly, eBPF backends | Kernel eBPF runtime only |
| **Source code requirement** | **Yes** — requires AppNet DSL specification | **No** — operates on compiled bytecode directly |
| **Optimization vs. verification** | Primarily an optimization system (with verification as enabler) | Primarily a verification system |

### 5.3 How This Paper Is Useful For Us

1. **Motivating the eBPF NF verification problem:** AppNet demonstrates that even sophisticated systems that generate NF code from high-level specs still need semantic verification (equivalence checking) as a critical safety net. This motivates our work: NFs deployed as raw eBPF programs (not generated by a trusted compiler) have *no* such semantic guarantees and require post-hoc verification.

2. **Establishing the importance of stateful chain verification:** AppNet's entire equivalence-checking problem is driven by stateful elements — elements that share counters, maps, and queues across RPCs. This directly parallels Yaksha-Prashna's core challenge of tracking BPF map state across XDP/TC program chains. We can cite AppNet to argue that stateful chain composition is an inherently hard problem that requires formal methods.

3. **Baseline for L7 vs. L3/L4 NF verification scope:** AppNet operates at L7 (RPC fields, application headers) while Yaksha-Prashna operates at L3/L4 (network packets, BPF instructions). This allows us to position our work as addressing a *complementary and lower-level* verification layer — one that AppNet itself cannot reach, since its eBPF backend is only a code generation target, not a verification target.

4. **Contrast: source-required vs. binary-level analysis:** AppNet requires the operator to rewrite their NF in its proprietary DSL. Yaksha-Prashna requires no such rewriting — it works on the compiled artifact. This is a significant practical advantage for real-world eBPF deployments where source code is unavailable, proprietary, or produced by third parties.

5. **Citation for service mesh / sidecar overhead motivation:** AppNet's performance numbers (82% latency reduction, 75% CPU savings) concretely quantify the cost of unoptimized NF deployment. This can be used in our introduction to motivate why NF chains are being pushed into the kernel via eBPF (to avoid proxy overhead), which in turn creates the verification gap Yaksha-Prashna addresses.

### 5.4 Positioning Statement

> While AppNet achieves semantic-safe optimization of application-layer NF chains using Z3-based equivalence checking, it requires operators to rewrite all network functions in its proprietary DSL and cannot analyze pre-existing eBPF programs operating at the network layer. Yaksha-Prashna addresses this gap by performing static behavioral verification directly on raw eBPF bytecode — requiring no source code, no recompilation, and no trust in the NF toolchain — making it applicable to deployed kernel-level NF chains where AppNet's source-level approach cannot reach.
