# NetSMC: A Custom Symbolic Model Checker for Stateful Network Verification

**Authors:** Yifei Yuan, Soo-Jin Moon, Sahil Uppal, Limin Jia, Vyas Sekar  
**Year:** 2020 | **Venue:** USENIX NSDI 2020 (17th USENIX Symposium on Networked Systems Design and Implementation)  
**DOI/Link:** https://www.usenix.org/conference/nsdi20/presentation/yuan

---

## 1. Overview

Modern enterprise and cloud networks increasingly deploy **stateful network functions (NFs)** — firewalls, load balancers, NATs, intrusion detection systems — that modify their internal state based on packet history. Verifying that such networks correctly enforce security and SLA policies is fundamentally hard: even simple isolation properties (e.g., "host A cannot reach host B") become **undecidable** once stateful middleboxes are in the picture, and the tractable fragments known before this paper were either EXPSPACE-hard or required unrealistic behavioral assumptions (e.g., VMN's out-of-order packet buffering model).

NetSMC is a custom symbolic model checker built specifically for stateful network verification. Rather than reducing the problem to a general SMT or BDD-based model checking instance, the authors exploit three domain-specific insights simultaneously: (1) a **compact yet practically equivalent semantic model** for stateful NFs based on state tables and update rules; (2) a **restricted but expressive subset of LTL** tailored to realistic network policies including dynamic service chaining and path pinning; and (3) a **custom symbolic model checking algorithm** that encodes network states as Existential First-Order (EFO) logic formulas and links containment checking to the well-studied **conjunctive query containment** problem from database theory.

The result is a verifier that scales to networks of hundreds of stateful NFs and runs **28×–200×+ faster** than the prior state-of-the-art tool VMN on fat-tree topologies. For a well-defined class of policies (notably isolation), NetSMC is sound and complete; for more complex interleaved-packet policies it serves as a sound-but-incomplete bug finder. The paper was presented at NSDI 2020, and NetSMC represents one of the most sophisticated purpose-built verifiers for stateful service-chain policies published to date.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

NetSMC's core is a **custom symbolic model checker** for the class of stateful networks it can model. Rather than delegating to a general-purpose backend (Z3, NuSMV, etc.), the authors design every component — the state representation, the image computation, and the containment test — to exploit the algebraic structure of NF behavior.

**Semantic Model (State Tables + Rules):**  
Each stateful NF is formalized as a pair: (i) a collection of *state tables* indexed by packet header fields (e.g., the connection-tracking table of a stateful firewall maps `(src_ip, dst_ip, proto)` → connection state), and (ii) a set of *processing rules* that match on packet headers and current table contents, then update table entries and determine packet forwarding decisions. This two-layer abstraction captures the essential behavior of firewalls (connection tracking), load balancers (server-selection tables), and IDSs (alert state) without requiring source-code-level detail.

**Compact Network Model (One-Packet-at-a-Time):**  
The global network state is the Cartesian product of all individual NF state tables. Packet processing is modeled as a single-packet step function: one packet travels through a service chain, each NF in sequence updates its state table and forwards or drops the packet. The authors prove this "one-packet-at-a-time" semantic is **equivalent** to the general model for the class of policies they verify — crucially, it allows the tool to sidestep the full complexity of concurrent multi-packet interleavings for the decidable subset.

**Symbolic State Representation (EFO Logic):**  
Instead of enumerating individual concrete states, NetSMC represents *sets of network states* as formulas in **Existential First-Order (EFO) logic** — specifically, conjunctions of existentially quantified membership tests and inequality constraints over packet header variables and table contents. EFO is closed under the pre-image operator for this NF model, meaning the result of a pre-image computation stays expressible in the same fragment, which is essential for termination.

**Containment via Conjunctive Query Containment:**  
The central bottleneck of symbolic model checking is the **containment check**: given two EFO state sets S₁ and S₂, is S₁ ⊆ S₂? This check is needed for fixpoint termination. NetSMC identifies that this EFO containment problem is equivalent to **conjunctive query containment (CQC)** in database theory, a well-studied problem for which efficient polynomial-time algorithms exist under the homomorphism characterization. This insight eliminates the need for expensive general SMT calls and is the primary source of NetSMC's speedup.

**Policy Language (LTL Subset → CTL):**  
Policies are expressed in a restricted subset of **Linear Temporal Logic (LTL)** over sequences of packet events. The key restriction is that this subset can be mechanically translated into **Computation Tree Logic (CTL)**, enabling backward symbolic model checking (pre-image fixpoint over a Kripke structure) rather than full LTL automaton product construction. The subset is carefully chosen to remain intuitive for network operators while covering a wide range of real policies.

### 2.2 Process Steps

1. **Input collection:** The operator provides (a) stateful NF models expressed as state-table + rule pairs for each NF in the chain; (b) the network topology (how NFs are chained); (c) a policy expressed in NetSMC's LTL-subset language.

2. **Policy translation:** The LTL-subset formula is translated into a CTL formula. For properties like isolation, this is a straightforward syntactic transformation. For more complex policies, the CTL translation approximates the LTL formula (sound but incomplete).

3. **NF chain Kripke structure construction:** The NFs and network topology are composed into a Kripke structure whose states are EFO-encoded global network states and whose transitions encode one-packet-at-a-time execution.

4. **Backward symbolic model checking (fixpoint):**  
   - Start from the set of "bad" states encoded as an EFO formula (e.g., states where packet from A reaches B).  
   - Iteratively apply `COMPUTEPREIMAGE` to compute the set of predecessors.  
   - At each iteration, check EFO-containment (via CQC) to test if the newly computed set is a subset of the union of previously seen sets (fixpoint convergence test).  
   - If fixpoint is reached without intersecting the initial state set: policy verified (no counterexample reachable).  
   - If initial states are intersected: a concrete counterexample (violating packet trace) is returned.

5. **Pre-image computation (`COMPUTEPREIMAGE`):** For each NF rule in the chain, symbolically "reverse" the transition: given a target EFO state formula, compute the EFO formula representing states that transition to it under that rule. The output is a disjunction of EFO conjunctions, which remains in the EFO fragment.

6. **Containment check (CQC):** Map the EFO containment S₁ ⊆ S₂ to CQC by treating each EFO conjunction as a conjunctive query. Use the homomorphism-based polynomial-time algorithm to decide containment without calling Z3.

7. **Output:** A verification verdict (safe / counterexample trace) along with timing and state-space statistics.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in NetSMC |
|---|---|
| **Existential First-Order Logic (EFO)** | Symbolic representation of sets of network states (global table contents); closed under NF transition pre-image |
| **Linear Temporal Logic (LTL subset)** | Policy specification language for multi-packet temporal properties over packet event sequences |
| **Computation Tree Logic (CTL)** | Intermediate target after LTL-subset translation; enables backward fixpoint model checking |
| **Conjunctive Query Containment (CQC)** | Database-theory algorithm (homomorphism test) used for efficient EFO set-containment checking; replaces SMT solver calls |
| **Symbolic Model Checking (custom)** | Purpose-built backward pre-image fixpoint loop; not based on NuSMV, CBMC, or similar general tools |
| **Cloudlab testbed** | Evaluation platform for running real pfSense and HAProxy instances to extract NF models |
| **VMN** (baseline comparison) | Prior stateful network verifier used as performance/expressiveness baseline |

No external SMT solvers (e.g., Z3) or BDD libraries are used in the core algorithm; the custom CQC engine replaces them.

### 2.4 Key Data Structures / Models

- **State Table Formula:** An EFO conjunction encoding the contents of NF connection/session tables as sets of tuples, e.g., `∃x,y. (x,y) ∈ ConnTable_FW ∧ x.src = A` represents "firewall has seen a packet from A."
- **EFO Formula (Symbolic State Set):** A disjunction of EFO conjunctions representing potentially exponentially many concrete global states compactly.
- **NF Rule Schema:** A pair `(match-predicate, update-action)` over packet header fields and table predicates; the basis of pre-image computation.
- **Kripke Structure (implicit):** States = EFO-formulae; transitions = NF rule applications; initial states = empty-table formula; property = CTL formula over Kripke states.
- **Conjunctive Query (CQ):** Each EFO conjunction is isomorphically a CQ; containment S₁ ⊆ S₂ becomes "does every CQ in S₁ map via homomorphism into some CQ in S₂?"
- **Fat-Tree Topology Model:** Used in evaluation; a k-ary fat-tree with stateful NFs at each node — the concrete network model used to benchmark scalability.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

NetSMC targets **stateful network functions** deployed in enterprise/datacenter service chains, specifically:

- **Stateful Firewalls** (e.g., pfSense): Connection-tracking firewalls that admit reply packets only after seeing the initiating SYN; model = connection table + allow/deny rules.
- **Load Balancers** (e.g., HAProxy): Stateful LBs that pin flows to backend servers; model = flow-to-server mapping table + distribution rules.
- **Intrusion Detection Systems (IDS):** Alert-state NFs that accumulate evidence across multiple packets to trigger blocks; model = alert/session table + multi-packet matching rules.
- **NATs** (implied): Connection-rewriting NFs with address-mapping state; readily expressible as state-table + rewrite rules.

The NFs must be expressible as state-table + rule pairs. This is a model-level abstraction rather than a source-code or binary analysis.

### 3.2 How It Validates NF Behavior

Validation is entirely **model-based and offline**:

1. The operator (or a separate extraction step) creates a **formal NF model**: a state-table schema (what fields are tracked) and a rule set (how packets update the table and where they are forwarded or dropped).
2. The model is encoded as the EFO transition relation for that NF.
3. The NF model is composed into a chain with other NF models to form the global transition system.
4. The model checker runs backward fixpoint search over the composed EFO state space to determine whether a given policy (expressed in LTL) is violated.
5. If a violation is found, a concrete packet sequence (counterexample trace) is extracted and returned.

There is **no direct analysis of NF source code or binary**; correctness of the model relative to the real NF implementation is the operator's responsibility (though the paper validates models against real pfSense and HAProxy deployments experimentally).

### 3.3 What Properties / Invariants Does It Prove?

NetSMC's policy language covers a range of multi-packet temporal properties expressible in its LTL subset:

| Property Class | Example | Completeness |
|---|---|---|
| **Isolation** | "No packet from host A ever reaches host B" | Sound & complete |
| **Reachability** | "Host C can always reach service S via the web tier" | Sound & complete (for this fragment) |
| **Service-chain compliance** | "All traffic from external hosts must traverse FW → IDS before reaching DB servers" | Sound & complete |
| **Dynamic service chaining** | "Traffic flagged by IDS must be redirected to deep-inspection FW on subsequent packets" | Sound-but-incomplete (interleaving) |
| **Path pinning / symmetry** | "Forward and reverse flows for the same connection must traverse the same firewall instance" | Supported in policy language |
| **SLA / waypoint enforcement** | "Any HTTP request to the datacenter must pass through the load balancer" | Sound & complete |
| **Multi-packet temporal properties** | "After three failed login attempts, subsequent packets from that IP are dropped" | Sound-but-incomplete |

For properties involving arbitrary **packet interleaving** (ordering among flows), NetSMC is sound but incomplete: it may fail to find some bugs, but any counterexample it produces is real.

### 3.4 Input Requirements

| Input | Format | Provider |
|---|---|---|
| **NF models** | State-table schema + rule sets in NetSMC's model language | Human operator or automated extractor |
| **Network topology** | Chain ordering + routing specification | Operator |
| **Policy** | LTL-subset formula over packet predicates | Operator / policy author |
| **Initial state** | Implicitly: all NF state tables empty (cold-start assumption) | Fixed by tool |

NetSMC does **not** accept source code, binaries, P4 programs, or eBPF bytecode directly. It operates exclusively on the formal NF models, which must be written or synthesized separately.

### 3.5 Guarantees Provided

- **For isolation / reachability / waypoint policies:** A **sound and complete proof** — if the tool says "safe," no violating packet sequence exists under the model; if it says "violated," the counterexample is real.
- **For policies with packet interleaving:** A **sound-but-incomplete** guarantee — any counterexample found is real (no false positives), but the tool may miss some true violations (potential false negatives).
- **Performance guarantee:** Termination is guaranteed for the decidable subset because EFO is closed under pre-image and the CQC-based containment check ensures the fixpoint converges.
- **Scalability claim:** Verified experimentally at hundreds of NFs in fat-tree topologies; not proven as a complexity bound.

---

## 4. NF Chain Verification

NetSMC is **explicitly and centrally designed for service chain verification** — this is one of its primary contributions over prior single-NF or stateless verifiers.

### Chain Composition Model
The tool composes individual NF models into a chain by threading packet transitions through NFs in sequence. The global Kripke structure's transition relation is the sequential composition of each NF's rule-based transition: a packet enters NF₁, updates NF₁'s state table, is forwarded (or dropped), then enters NF₂, updates NF₂'s state table, and so on. The EFO global state is the conjunction of all NF state-table formulas.

### Chain Properties Verified

| Chain Property | Support |
|---|---|
| **Ordering enforcement** | ✅ — LTL subset can mandate that packets traverse NFs in a specific order (e.g., FW before IDS before DB) |
| **Dynamic service chaining** | ✅ — Traffic steering decisions made by one NF (e.g., IDS flagging a flow) can redirect the flow to additional NFs on subsequent packets |
| **Path pinning / symmetry** | ✅ — Policy language supports symmetric path constraints: forward flow through FW₁ implies return flow through FW₁ |
| **Stateful carry-over across NFs** | ✅ — Each NF's state table persists and can be queried by subsequent rules; EFO encodes cross-NF state correlations |
| **Isolation across tenants in a shared chain** | ✅ — Tenant-level isolation ("traffic from tenant A never reaches tenant B's segment even through shared LB") expressible and verifiable |
| **Concurrent flow interleaving across chains** | ⚠️ — Sound-but-incomplete; arbitrary interleaving of multiple flows is approximated |

### What Would Be Needed for Broader Chain Coverage
- Full interleaving semantics for concurrent flows would require moving beyond the one-packet-at-a-time relaxation, reintroducing undecidability.
- Chains involving **programmable NFs** (P4 pipelines, eBPF-based NFs) would require a model extraction step or a new NF modeling formalism.
- **Runtime chain reconfiguration** (live policy updates mid-flow) is not addressed.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna is a static analysis framework that operates directly on **raw eBPF bytecode** — with no source code, no annotations, and no operator-supplied NF models required. It builds a **Control Flow Graph with Network Context (CFG-NC)** from the bytecode, performs dataflow analysis over it, and uses a **Prolog query engine** to check behavioral assertions about individual eBPF-based NFs and NF chains. It handles stateful BPF maps (hash maps, array maps, LRU maps) as first-class objects in its analysis, and is designed to verify chains of eBPF NFs without requiring any manual model construction.

### 5.2 Key Differences from This Paper

| Dimension | NetSMC | Yaksha-Prashna |
|---|---|---|
| **Input format** | Formal NF model (state-table + rules) — operator-supplied | Raw eBPF bytecode — no models or annotations needed |
| **NF representation** | Abstract state-table abstraction | Actual bytecode CFG with BPF map semantics |
| **Modeling burden** | High — operator must write correct NF model | None — automatic extraction from binary |
| **Property language** | LTL subset → CTL (temporal over packet sequences) | Prolog assertions (behavioral queries over CFG-NC dataflow facts) |
| **Verification engine** | Custom EFO symbolic model checker + CQC containment | Prolog inference engine over CFG-NC dataflow lattice |
| **Completeness** | Sound+complete for isolation/reachability; incomplete for interleaving | Sound (for the properties expressible as Prolog queries over CFG-NC) |
| **Chain support** | ✅ First-class design goal | ✅ First-class design goal |
| **Stateful analysis** | ✅ Via EFO-encoded state tables | ✅ Via BPF map value tracking in dataflow |
| **Runtime model** | Offline, model-based | Offline, bytecode-based |
| **Target NF ecosystem** | Traditional middleboxes (pfSense, HAProxy) modeled abstractly | eBPF kernel programs (XDP, TC hooks) analyzed concretely |
| **Scalability** | Hundreds of NFs in fat-tree; formal scalability depends on CQC complexity | Limited by dataflow analysis complexity over bytecode; chain depth |
| **Counterexample** | Concrete packet trace (model-level) | Prolog query failure / dataflow witness |
| **Undecidability handling** | Practical relaxation (one-packet-at-a-time model) | Not applicable — works at program analysis level, not semantics level |

### 5.3 How This Paper Is Useful For Us

1. **Motivation for chain-level verification:** NetSMC is one of the strongest existence proofs that verifying stateful NF *chains* (not just individual NFs) is both necessary and achievable, motivating the chain-level scope of Yaksha-Prashna. We can cite it directly to establish that the community considers chain verification a first-class problem.

2. **Baseline comparison anchor:** NetSMC defines the state of the art for model-based stateful chain verification as of 2020. Yaksha-Prashna can be positioned as complementary (bytecode-level vs. model-level) — or, if we also check isolation/reachability style properties, we can compare expressiveness of our Prolog assertions against NetSMC's LTL subset.

3. **Limitation to exploit:** NetSMC's most significant limitation from our perspective is its **dependency on manually constructed NF models**. An operator must correctly abstract a pfSense or HAProxy instance into a state-table + rule formalism. Yaksha-Prashna avoids this entirely by analyzing the actual eBPF bytecode, eliminating the model gap. This is a clean, citable limitation that directly motivates our approach.

4. **Policy language contrast:** NetSMC's LTL-subset policies are expressive but require temporal formula expertise from the operator. Our Prolog-based query interface is arguably more accessible for network engineers who think in terms of "what flows should be blocked/allowed" rather than LTL formulae — a potential UX argument for Yaksha-Prashna.

5. **Reference for undecidability context:** The paper's formal treatment of undecidability of stateful network verification provides theoretical grounding we can cite to justify why a sound-but-incomplete static analysis (like Yaksha-Prashna) is a reasonable and principled design choice.

6. **NF type coverage:** NetSMC's evaluation on FW + LB + IDS chains exactly matches the NF types Yaksha-Prashna targets in eBPF form, allowing direct comparison of scope.

### 5.4 Positioning Statement

> "While NetSMC achieves scalable, sound verification of stateful NF chain policies using a custom EFO symbolic model checker, it requires operators to manually construct formal NF models — creating a gap between verified models and deployed implementations. Yaksha-Prashna addresses this by operating directly on raw eBPF bytecode, automatically extracting the CFG-NC and BPF map semantics without any operator-supplied models, and enabling behavioral assertion checking over real deployed NF chains through its Prolog query engine."

---

## Appendix: Quick Reference Summary

| Attribute | Value |
|---|---|
| **Paper ID** | G4 |
| **Core technique** | Custom EFO symbolic model checking + CQC containment |
| **Policy formalism** | LTL subset → CTL |
| **NF types** | Stateful FW, LB, IDS (modeled abstractly) |
| **Chain support** | Yes — first-class |
| **Stateful** | Yes — via EFO-encoded state tables |
| **Input required** | Formal NF models (state-table + rules) |
| **Timing** | Offline |
| **Completeness** | Sound+complete for isolation; sound-incomplete for interleaving |
| **Speedup vs. VMN** | 28×–200×+ on fat-tree topologies |
| **Cited by Yaksha-Prashna** | Yes (ref 89) |
| **Key limitation for us** | Model-level abstraction requires manual NF model construction |
