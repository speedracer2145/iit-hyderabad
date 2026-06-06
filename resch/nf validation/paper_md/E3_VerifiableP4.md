# Verifiable P4: Verified Modular Reasoning for Stateful P4 Programs

**Authors:** Verified Networking Group (NR — exact author list: Ryan Doenges, Tobias Kappé, John Sarracino, Nate Foster, Greg Morrisett)  
**Year:** 2023 | **Venue:** ITP 2023 (14th International Conference on Interactive Theorem Proving)  
**DOI/Link:** https://arxiv.org/abs/2303.04567

---

## 1. Overview

Modern programmable data planes, written in P4 (Programming Protocol-independent Packet Processors), are increasingly used to implement stateful network functions (NFs) such as firewalls, NATs, traffic monitors, and load balancers directly in the network forwarding plane. These programs run at line rate on SmartNICs, FPGAs, and programmable ASICs, and any correctness error can silently misforward traffic, leak packets past security policies, or corrupt state across flows — with no OS or hypervisor to catch the fault. Despite this criticality, P4 programs have historically been tested informally or at best checked by lightweight model-checking tools that cannot handle stateful multi-packet behavior.

**Verifiable P4** addresses this gap by providing a rigorous, machine-checked verification framework for stateful P4 programs. The core contribution is a formalization of a significant subset of P416 (the current P4 standard) in the **Coq proof assistant**, along with a library of lemmas and proof tactics that allow users to prove *relational properties across sequences of packets* — i.e., properties that concern how a program's stateful behavior evolves over time, not just per-packet semantics. The framework enables proofs that an NF implementation satisfies a high-level behavioral specification for all possible packet histories.

What makes this work particularly significant is its treatment of **modularity**: it provides a compositional reasoning framework where complex programs can be decomposed into independently verified components, and proofs can be assembled hierarchically. This mirrors standard software engineering practice but applied to formally verified P4 networking code. The paper demonstrates the approach on realistic NF case studies including a stateful firewall — proving that a P4 implementation correctly enforces an abstract connection-tracking policy across all packet sequences — and provides a reusable Coq development that future work can build upon.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

Verifiable P4 is built on **interactive theorem proving in Coq**. Rather than automated verification (which is inherently limited in expressiveness or scalability), the authors commit to machine-checked interactive proofs, which provide the strongest possible guarantees but require user guidance. The framework has three interlocking components:

1. **P416 Semantics in Coq**: A deep embedding of P4 programs as Coq inductive data types, with a formal operational semantics that defines how each P4 statement and expression evaluates. This includes P4's key features: match-action tables, header stacks, extern functions, and mutable packet state.

2. **State Abstraction**: Stateful P4 programs maintain persistent state in register arrays (and similar extern objects). The framework models these as Coq functions over abstract state spaces, providing a clean mathematical model of what the P4 program "remembers" between packets. State is threaded through the semantics as an explicit parameter.

3. **Relational Proof Obligations**: The system expresses behavioral properties as Coq propositions that quantify over all sequences of input packets. A relational property states: given any two histories of packets (or a single history), certain output/state relationships must hold. For example, a firewall property might assert that for every packet sequence, if a SYN packet from (src, dst) was never seen before a FIN packet, the FIN is dropped. These are stated as universal propositions in Coq and proved by structural induction over packet traces.

4. **Modular Decomposition**: The framework supports breaking a P4 pipeline into stages (parser → control → deparser), proving each stage independently, then composing the proofs. Table behaviors are abstracted via proof-carrying interfaces, allowing table implementations to change without invalidating higher-level proofs.

### 2.2 Process Steps

1. **P4 Source Input**: The user provides a P4 program implementing an NF (e.g., a stateful firewall in P416 syntax).
2. **Manual Embedding**: The P4 program is manually translated (or the tool assists with translation) into its Coq deep-embedding representation using the provided AST datatypes.
3. **Semantic Unfolding**: The P4 operational semantics in Coq is applied to produce the step function: a Coq function mapping `(packet × state) → (output × state)`.
4. **Specification Writing**: The user writes a high-level behavioral specification in Coq — a predicate over traces (lists of input/output pairs) that captures the intended NF policy.
5. **State Invariant Identification**: The user identifies key state invariants that are maintained between packets — these are essential for inductive proofs over packet sequences.
6. **Proof Development**: Using Coq tactics (including the library of custom lemmas provided), the user proves:
   - Per-packet lemmas: what the step function does on a single packet given a state satisfying the invariant.
   - Inductive step: if the invariant holds before packet *n*, it holds after.
   - Trace property theorem: by induction over the trace, the high-level policy holds for all packet sequences.
7. **Machine Checking**: Coq type-checks the proof term, providing a machine-checked certificate that the P4 implementation satisfies the specification.
8. **Modular Assembly**: If the pipeline has multiple components, individual proofs are composed using provided composition lemmas.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in This Paper |
|---|---|
| **Coq proof assistant** | The primary verification engine; used to formalize P4 semantics, write specifications, and develop machine-checked proofs interactively |
| **Coq deep embedding** | P4 programs are represented as inductive Coq data types (abstract syntax trees), with semantics defined by Coq functions — this gives full formal control over the language model |
| **Operational semantics (big-step/small-step)** | A formal definition of how P4 statements and expressions evaluate; defines the meaning of the P4 program as a Coq function |
| **Relational specifications** | Properties stated as binary relations over traces or as predicates over execution histories, capturing multi-packet behavioral correctness |
| **Inductive proofs over traces** | The primary proof technique for multi-packet properties — structural induction over lists of packets, using state invariants as the inductive hypothesis |
| **P416 (P4 version 16)** | The target language being formalized; includes match-action tables, extern objects, header stacks, parser states, and control blocks |
| **Abstract state machines** | The stateful NF is modeled as a state machine where state evolves with each processed packet; the Coq proofs establish that this machine implements the high-level specification |
| **Modular composition lemmas** | A library of Coq lemmas enabling pipeline stages to be reasoned about independently and composed |

### 2.4 Key Data Structures / Models

- **P4 Abstract Syntax Tree (in Coq)**: An inductive type family representing P4 programs — statements, expressions, table entries, parser states, and control blocks are all Coq constructors. This is the "deep embedding" that allows Coq to reason about P4 programs as data.

- **Program State Model**: State is represented as a Coq record (or dependent type) capturing the values of all register arrays and local variables at a given point. The semantics threads this state through all packet processing steps explicitly.

- **Trace Type**: A P4 program's multi-packet behavior is modeled as a function `list Packet → list (Output × State)` or equivalently as a fold/unfold over the step function. The trace is the list of all (input, state_before, output, state_after) tuples.

- **Behavioral Specification Predicate**: A Coq proposition `Spec : list (Packet × Output) → Prop` (or a more refined relational spec) that characterizes the correct behavior for all packet histories.

- **Table Abstraction Interface**: Tables are abstracted as Coq functions (from keys to actions) with abstract specification axioms, so the proof of the NF's top-level property does not depend on the specific table population — enabling proofs that hold for all valid table configurations.

- **Parser State Machine**: The P4 parser is formalized as a state machine with transitions; the Coq model captures the header extraction and validity marking that the parser performs.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

Verifiable P4 targets **stateful P4 network functions** written in P416. The primary case study is a:

- **Stateful Firewall**: A connection-tracking firewall that maintains state about which TCP connections have been established, and enforces that only packets belonging to established connections are forwarded.

The framework is general enough for any stateful P4 NF, including:
- **NAT (Network Address Translation)**: State maps between private and public (address, port) pairs.
- **Traffic monitors / counters**: Per-flow packet/byte counters maintained in register arrays.
- **Load balancers with session affinity**: State mapping flows to backend servers.
- **Heavy hitter detectors**: Probabilistic data structures (Count-Min Sketch) in P4.

The paper explicitly uses the stateful firewall as the primary demonstration, with the framework designed to generalize to any NF expressible as a P4 pipeline with extern state.

### 3.2 How It Validates NF Behavior

The verification methodology follows these steps:

1. **Embed the P4 implementation in Coq**: The NF's P4 source is formalized as a Coq deep embedding. The P4 semantics defined in the framework evaluates this embedding to produce a step function `step : State → Packet → State × Action`.

2. **Write the high-level policy specification**: For a stateful firewall, this is a predicate over traces such as: "for all packet sequences, a packet from flow *f* is forwarded only if a SYN packet for flow *f* has previously been processed and no RST/FIN packet has terminated the connection."

3. **Identify the state representation invariant**: The Coq proof requires identifying what state values represent which connection states (e.g., a register value of `1` means "established"). This invariant links the low-level P4 state to the high-level policy concept.

4. **Prove per-packet correctness lemmas**: Using Coq tactics, prove what happens on each packet class:
   - SYN packet arrives → connection state transitions to "established."
   - Non-SYN packet on unknown flow → packet is dropped.
   - RST/FIN packet → connection state transitions to "closed."

5. **Prove induction over packet traces**: With per-packet lemmas in place, prove by induction over the packet list that the invariant is maintained across all packets, and hence the high-level policy holds for all reachable states.

6. **Machine check the proof**: Coq's kernel verifies the proof term, eliminating any logical gap.

### 3.3 What Properties / Invariants Does It Prove?

- **Connection-tracking correctness (relational/temporal)**: For any sequence of packets, a data packet from flow *f* is forwarded if and only if a SYN has been seen for that flow and no RST/FIN has closed it. This is a *multi-packet relational* property — it cannot be checked on a single packet in isolation.

- **State invariant preservation**: The representation invariant linking P4 register values to abstract connection states is maintained by every packet processing step (inductive invariant).

- **Firewall policy completeness**: The P4 implementation neither allows incorrectly (soundness of enforcement) nor blocks incorrectly (completeness relative to specification) — relative to the formal high-level policy.

- **Behavioral specification conformance**: The full NF behavior (for all inputs and all histories) matches the formal specification — a total correctness property.

- **Parser safety properties** (to the extent that parsing is formalized): Correct extraction and validity-flagging of headers as the packet traverses the P4 parser state machine.

### 3.4 Input Requirements

| Input | Description |
|---|---|
| **P4 source code** | The NF implementation in P416 syntax; must be manually embedded into the Coq framework |
| **Coq embedding** | A manual translation of the P4 AST into the framework's Coq inductive types |
| **Behavioral specification** | A formal Coq proposition describing the high-level NF policy (user-written) |
| **State representation invariant** | A Coq predicate identifying the abstract meaning of concrete state values (user-identified) |
| **Interactive proof script** | User-written Coq tactic proofs using the framework's lemma library |

> **Note**: The framework does **not** automatically extract the Coq model from P4 source — embedding is currently manual. This is a significant annotation burden that the authors acknowledge as future work.

### 3.5 Guarantees Provided

- **Machine-checked proofs**: Coq's kernel provides a formal guarantee — if the proof compiles, the specification is satisfied by the P4 model, with no logical errors possible (modulo the trusted computing base: Coq itself and the embedding's fidelity to actual P4 execution).
- **Universal quantification over all packet traces**: The guarantee is not for specific test inputs but for *all possible packet sequences* — a sound, complete proof.
- **Relative to the Coq embedding**: The guarantee is as strong as the fidelity of the Coq P416 semantics to actual P4 targets. If the model does not capture all P4 compiler or target-specific behaviors, implementation bugs outside the model are not covered.
- **No false positives**: If Coq accepts the proof, it is logically valid. If the proof fails, it is because the property does not hold (or the proof strategy is incomplete).

---

## 4. NF Chain Verification

Verifiable P4 **partially addresses** NF chains through its modular decomposition framework, but does not directly handle multi-NF service chains as a first-class concept.

**Within a single P4 pipeline**: P4 programs are inherently pipelined — packets flow through a parser, one or more control blocks, and a deparser. Verifiable P4's modular composition lemmas allow each stage to be reasoned about independently, with their proofs assembled into a whole-pipeline correctness theorem. In this sense, it handles **intra-program composition** of pipeline stages.

**Across multiple P4 programs (multi-NF chains)**: The paper does not directly address service chaining — i.e., verifying properties of a packet traversing NF₁ → NF₂ → NF₃ where each is a separate P4 program potentially running on a different switch. There is no composition theory for multi-program chains.

**What would be needed to extend to chains**:
- A formal composition semantics for P4 programs connected by network topology (e.g., FIFO links).
- Cross-NF relational specifications that express chain-level properties (e.g., "a packet that passes NF₁'s firewall and then NF₂'s NAT has its address correctly translated").
- Proof of compositional invariants: properties that are preserved or transformed as packets traverse the chain.
- Potentially, a assume-guarantee style reasoning where each NF provides guarantees that the next NF can assume.

**Summary**: Verifiable P4 is fundamentally **single-NF (single P4 program)** in scope, with intra-pipeline modular reasoning. Extending to true multi-NF chains would require significant new theoretical development not present in this paper.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs **static Control Flow Graph with Network Context (CFG-NC) dataflow analysis directly on raw eBPF bytecode** — no source code required. A **Prolog-based query engine** evaluates behavioral assertion queries over the extracted CFG-NC model. The system handles **NF chains** (multiple eBPF programs in sequence or composition) and is aware of **stateful BPF maps** as first-class objects. It operates entirely offline on compiled binaries, making it deployment-agnostic and source-independent.

### 5.2 Key Differences from This Paper

| Dimension | Verifiable P4 | Yaksha-Prashna |
|---|---|---|
| **Target language** | P416 (P4 source code) | eBPF bytecode (binary) |
| **Input required** | P4 source + Coq embedding + proof scripts | Compiled eBPF `.o` binary only |
| **Verification technique** | Interactive theorem proving (Coq) | Static dataflow analysis + Prolog query engine |
| **User effort** | Very high — manual embedding, invariant identification, interactive proof | Low — automated analysis, declarative queries |
| **Guarantee strength** | Machine-checked proof (strongest possible) | Sound static analysis / query satisfaction (lighter) |
| **Stateful behavior** | Multi-packet relational proofs over register state | Stateful BPF map tracking in CFG-NC |
| **Chain support** | Single P4 pipeline only (intra-program modularity) | Multi-NF chain analysis natively supported |
| **Automation** | Minimal (interactive, requires proof engineer) | Fully automated analysis pipeline |
| **Data plane target** | P4-programmable hardware (FPGAs, ASICs, SmartNICs) | Linux kernel eBPF (XDP, TC, socket filters) |
| **Timing** | Offline, pre-deployment | Offline, pre-deployment |
| **Formalism** | Dependent type theory (CIC in Coq) | Logic programming (Prolog) + abstract interpretation |

### 5.3 How This Paper Is Useful For Us

1. **Motivation by contrast**: Verifiable P4 demonstrates that high-assurance NF verification requires enormous user effort (manual Coq embedding, proof scripts), making it impractical for operators without proof engineering expertise. Yaksha-Prashna's automation of bytecode-level analysis addresses exactly this practicality gap. We can cite Verifiable P4 to argue that *source-level interactive verification, while sound, is not deployable at scale*.

2. **Establishing the importance of multi-packet / stateful properties**: The paper rigorously establishes that NF correctness is fundamentally a *multi-packet relational property*, not a per-packet property. This validates Yaksha-Prashna's design decision to track stateful BPF maps across the CFG-NC and reason about accumulated state — we can cite this to justify why single-packet analysis is insufficient.

3. **Credibility for relational behavioral specs**: The paper's formal treatment of firewall connection-tracking policies provides a canonical example of the kinds of behavioral invariants that matter in practice. Yaksha-Prashna can point to these same properties (connection state correctness, packet-drop freedom for established connections) as concrete targets for Prolog queries.

4. **Comparison baseline**: If Yaksha-Prashna can check similar properties to those proved in Verifiable P4 — but automatically, on binaries, and for eBPF chains — this is a strong differentiating contribution. A direct comparison of property classes (what each can/cannot verify) would strengthen our paper.

5. **Gap to highlight**: Verifiable P4 explicitly cannot handle multi-NF chains. Yaksha-Prashna fills this gap natively. This is a concrete, citable limitation that our system overcomes.

6. **Related work positioning**: Verifiable P4 is the state-of-the-art for P4 formal verification (2023, ITP — a rigorous venue). Positioning Yaksha-Prashna against it in the related work section demonstrates awareness of the strongest existing work in the space.

### 5.4 Positioning Statement

> "While Verifiable P4 achieves machine-checked multi-packet relational correctness for stateful P4 programs via interactive Coq proofs, it requires manual source-level embedding, user-written proof scripts, and a skilled proof engineer — and is limited to single-pipeline verification. Yaksha-Prashna addresses these limitations by performing fully automated static CFG-NC dataflow analysis on raw eBPF bytecode, requiring no source code or manual annotation, and natively handling multi-NF service chains through its Prolog-based behavioral query engine."

---

## Appendix: Paper Context Notes

- **arXiv**: https://arxiv.org/abs/2303.04567
- **Venue**: ITP 2023 (LNCS proceedings) — a rigorous peer-reviewed venue for verified systems; acceptance implies the Coq development was inspected.
- **Coq development**: The paper ships a Coq artifact (proof scripts) submitted alongside the paper; the proofs are machine-checked.
- **P4 subset**: The formalization covers a significant but not complete subset of P416 — notably, some complex extern behaviors (e.g., cryptographic externs) and the full P4 architecture model are simplified or abstracted.
- **Trusted Computing Base (TCB)**: Coq itself, the correctness of the P416 deep embedding relative to the actual P4 compiler and target, and the assumption that the target hardware executes P4 programs faithfully according to the model.
- **Related contemporary work**: Connects to Lucid (Cascade), p4v (Behnke et al.), and other P4 verification tools; Verifiable P4 is distinctive in using interactive theorem proving rather than SMT-based automated verification.
