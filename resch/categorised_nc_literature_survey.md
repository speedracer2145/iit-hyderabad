# Categorised Literature Survey: Extracting Network Context from Bytecode/Binary
## Every Paper | Every NC Category | Every Feature | How Violations Are Found

---

## TAXONOMY: The 6 Categories of Network Context

Before the survey, here is the precise taxonomy.
Every paper in existence on this topic extracts context from one or more of these six categories:

```
NC-1  HEADER FIELD CONTEXT        — which packet fields are read, written, and what values
NC-2  PROTOCOL STRUCTURE CONTEXT  — which protocols are parsed, in what order, with what checks
NC-3  MAP / STATE CONTEXT         — what eBPF maps / data structures store, how they are used
NC-4  PACKET ACTION CONTEXT       — what happens to the packet (drop/pass/redirect/modify)
NC-5  BEHAVIORAL CONTRACT CONTEXT — what the NF promises to do, what it assumes (Rely/Guarantee)
NC-6  BINARY SEMANTIC CONTEXT     — function purpose, instruction semantics, code similarity
```

Yaksha-Prashna covers NC-1 (partial), NC-2 (partial), NC-3 (name only), NC-4 (yes), NC-5 (none), NC-6 (none).
The survey below shows exactly what each paper adds.

---

## CATEGORY NC-1: HEADER FIELD CONTEXT

### What it is
Which packet header fields (eth.src, ipv4.dst, tcp.sport, sk_buff.mark, etc.) are read, written, compared, or used as keys.
Includes: field offset resolution, field type (TYPE, LENGTH, SEQ, CHECKSUM, PAYLOAD), field usage pattern.

---

### Paper 1.1 — Yaksha-Prashna
**Citation:** Singh, A., et al. "Yaksha-Prashna: Understanding eBPF Bytecode Network Function Behavior."
arXiv:2602.11232, 2026.
**Venue:** arXiv (under review)

**NC extracted:** NC-1, NC-2, NC-3 (partial), NC-4
**Features:**
- `readsField(NF, field)` — field is read on at least one path
- `updatesField(NF, field)` — field is written on at least one path
- Field name resolved from byte offset using protocol layout spec (e.g., offset 12 from Ethernet = ethertype)
- Register-level SSA propagation: field value annotated through registers R0–R10
**How violations found:**
- Prolog rule: `updatesField(NF_i, F), successorNF(NF_i, NF_j), readsField(NF_j, F)` → RAW dependency
- Prolog rule for WAW, WAR analogously
**Limitation for NC-1:**
- No field ROLE extraction (does not know if ethertype is a TYPE field or a LENGTH field)
- No field VALUE range extraction (only knows it is read, not what values it takes)
- No conditional field extraction (cannot say "field is written only on IPv4 paths")

---

### Paper 1.2 — BinPRE
**Citation:** Jiang, J., Zhang, X., Wan, C., Chen, H., Sun, H., Su, T., Luo, B., Liao, X., Xu, J., Kirda, E., Lie, D.
"BinPRE: Enhancing Field Inference in Binary Analysis Based Protocol Reverse Engineering."
*CCS 2024.* doi:10.1145/3658644.3690299

**NC extracted:** NC-1, NC-2
**Features:**
- **Format extraction:** field BOUNDARIES — where does field A end and field B begin in the packet
  - Method: instruction-based semantic similarity — group instructions that process the same input bytes
  - Feature: "which byte offsets are always accessed together by a single basic block?"
- **Semantic inference:** field ROLE classification
  - TYPE field: byte compared to a constant → `if (field == 0x0800) branch` pattern
  - LENGTH field: byte used in arithmetic to bound later access → `if (offset + field > limit) drop`
  - CHECKSUM field: byte involved in XOR/add loop over other bytes
  - SEQUENCE field: byte compared to a stored previous value
  - PAYLOAD field: byte passed to another function without comparison
  - Method: Atomic semantic detectors — small pattern classifiers for each type
  - Feature: opcode context around field access + data flow from field to branch condition
- Cluster-and-refine: group similar semantic detections, resolve conflicts across implementations
**Accuracy:** Format: Perfection 0.73; Semantic (type): F1=0.74; Semantic (function): F1=0.81
**How violations found:**
- Not violation detection — PRE accuracy evaluated against ground truth protocol specs
- Applied value: field role annotations enable schema-level compatibility checks between NFs
**What YP gains:** Add field role to every `readsField` and `updatesField` fact:
  `readsField(NF, field, role(TYPE_FIELD))` → enables "does NF_j expect field F to be a TYPE field
   that NF_i wrote as a SEQUENCE field?" mismatch detection.

---

### Paper 1.3 — NetLifter (Lifting Network Protocol Implementation to Precise Format Specification)
**Citation:** Shi, Q., Shao, J., Ye, Y., Zheng, M., Zhang, X.
"Lifting Network Protocol Implementation to Precise Format Specification with Security Applications."
*CCS 2023.* doi:10.1145/3576915.3616614

**NC extracted:** NC-1, NC-2
**Features:**
- Produces a BNF-style **Abstract Format Graph (AFG)** from source/binary of protocol parsers
- AFG nodes: protocol fields (with type annotations)
- AFG edges: sequential parsing order, conditional dependencies, loop structure
- Field types extracted: INTEGER, ARRAY, STRING, FLAG, ENUM, LENGTH-DELIMITED
- Cross-field dependencies: field A's length determines where field B starts
- Conditional field presence: field C exists only if field B == VALUE
**Method:** Abstract interpretation on control flow graph — derive AFG through inference rules.
Precision without path explosion: merges subpaths irrelevant to format structure.
**How violations found (security application):**
- Two implementations of same protocol → compare AFGs → discrepancy = hidden vulnerability
- Applied: found CVE in SSH implementation by comparing OpenSSH AFG to Dropbear AFG
**What YP gains:** AFG is a richer NC-1 representation than YP's flat field list.
For eBPF NFs: the AFG captures "NF processes fields in this ORDER with these CONDITIONALS"
— enabling protocol conformance checking (does NF_i follow the same parsing structure as NF_j?).

---

### Paper 1.4 — AIFORE
**Citation:** Shi, J., Wang, Z., Feng, Z., Lan, Y., Qin, S., You, W., Zou, W., Payer, M., Zhang, C.
"AIFORE: Smart Fuzzing Based on Automatic Input Format Reverse Engineering."
*USENIX Security 2023.* Pages 4967–4984.

**NC extracted:** NC-1, NC-2
**Features:**
- Byte-level taint analysis: map input bytes → basic blocks that process them
- Indivisible field identification: bytes always processed together form one field
  - Method: minimum clustering on taint sets — bytes with identical BB processing set = same field
- Field type classification via neural network on BB behavior:
  - Features fed to NN: opcode histogram of BB, number of comparisons, constant values compared,
    memory access patterns (byte/word/dword), branch count
  - Types: INTEGER, STRING, ENUM, SIZE, MAGIC, CHECKSUM
- Field boundary accuracy: 84.06% vs AFL-Analyze's 23.73%
- Field type accuracy: 84.26% vs ProFuzzer's 56.60%
**How violations found:**
- Fuzzer-focused: use format knowledge to generate valid test inputs
- Applied to network protocol analyzers with high branch coverage gain
**What YP gains:** The BB behavior features (opcode histogram + comparison constants + branch count)
are directly applicable to eBPF basic block analysis — these same features characterize
what each block DOES to a packet field.

---

### Paper 1.5 — Header Space Analysis (HSA)
**Citation:** Kazemian, P., Varghese, G., McKeown, N.
"Header Space Analysis: Static Checking for Networks."
*NSDI 2012.* Pages 113–126.

**NC extracted:** NC-1, NC-4
**Features:**
- Packet header as a point in {0,1}^L space (L = maximum header length in bits)
- Box model: each network device is a TRANSFER FUNCTION on header space
  - Input: a set of header bit vectors
  - Output: transformed set of header bit vectors
- Composition: chain of boxes = composition of transfer functions
- Fields: ALL header bits treated uniformly — no per-field semantics
**How violations found:**
- Reachability failure: ∃ header h such that h reaches wrong port → HEADER SPACE INTERSECTION
- Forwarding loop: network's composed transfer function has a fixed point (h → same output)
- Traffic isolation: two slices share header space → SLICE LEAK
**What YP gains:** The transfer function model for box composition.
YP can adopt this: each NF has a transfer function over FIELD SPACE (not bit space).
Chain composition = composition of field-level transfer functions.
This is the formal basis for the compositional approach to path explosion.

---

### Paper 1.6 — Real Time Network Policy Checking (NetPlumber)
**Citation:** Kazemian, P., Chang, M., Zeng, H., Varghese, G., McKeown, N., Whyte, S.
"Real Time Network Policy Checking Using Header Space Analysis."
*NSDI 2013.* Pages 99–112.

**NC extracted:** NC-1, NC-4
**Features:**
- Extends HSA to dynamic networks: incremental updates to transfer functions
- Dependency detection: which rule depends on which other rule's header space
- Rule-level conflict detection: two rules overlap in header space but have different actions
**How violations found:**
- Rule conflict: rules R1 and R2 match overlapping packets but take different actions
- Dependency loop: R1's output is R2's input and R2's output feeds back to R1's match space
**What YP gains:** The incremental update model — when NF_k is added to the chain,
only recompute affected transfer function compositions, not the whole chain.

---

## CATEGORY NC-2: PROTOCOL STRUCTURE CONTEXT

### What it is
Which protocols an NF parses, in what order, with what conditional checks.
Includes: protocol parse tree, next-protocol field checks, bounds checks before header access,
implicit protocol handling (no explicit check).

---

### Paper 2.1 — Yaksha-Prashna
(same citation as 1.1)
**NC-2 features:**
- `accessesProtocol(NF, protocol)` — protocol identified if:
  (a) NF reads next-protocol field (eth.ethertype, ipv4.protocol)
  (b) AND NF performs bounds check before accessing that protocol's header
- Parse order inferred from instruction sequence
**Limitation:** "Strict protocol check assumption" — NFs skipping explicit next-protocol checks
(e.g., treating all non-TCP as UDP) are NOT identified.

---

### Paper 2.2 — PREVAIL
**Citation:** Gershuni, E., Amit, N., Gurfinkel, A., Narodytska, N., Navas, J.A., Rinetzky, N., Ryzhyk, L., Sagiv, M.
"Simple and Precise Static Analysis of Untrusted Linux Kernel Extensions."
*PLDI 2019.* doi:10.1145/3314221.3314590

**NC extracted:** NC-2 (as memory regions), NC-3 (map pointer types)
**Features:**
- Memory region typing: PTR_TO_PACKET, PTR_TO_PACKET_END, PTR_TO_MAP_VALUE,
  PTR_TO_STACK, PTR_TO_CTX (xdp_md/sk_buff)
- Packet access region: [data, data_end) — precisely tracked offsets into packet
- Context access: sk_buff.mark, xdp_md.rx_queue_index — specific struct fields accessed
- Protocol-agnostic: does NOT identify which protocol a field belongs to — only memory bounds
**How violations found:**
- Out-of-bounds: access to PTR_TO_PACKET at offset > data_end → SAFETY violation
- Type error: arithmetic on PTR_TO_MAP_VALUE → ILLEGAL POINTER ARITHMETIC
**What YP gains:** The MEMORY REGION TYPE SYSTEM. eBPF programs can access:
stack, ctx, packet, map_value, map_key — each is a distinct region with distinct semantics.
YP's NC-2 can be enriched with: "which MEMORY REGION TYPE is the accessed field in?"
This resolves the implicit protocol handling limitation (packet access without explicit proto check
still touches a specific region of PTR_TO_PACKET that corresponds to a protocol field).

---

### Paper 2.3 — NetLifter
(same citation as 1.3)
**NC-2 features:**
- Full protocol PARSE TREE extracted (not just field list)
- Conditional parsing: "field C is only present if field B == FLAG_VALUE"
- Nested protocols: "TCP payload is parsed as HTTP if port 80"
- Length-delimited parsing: "TLV structure where tag → length → value"
**What YP gains:** The conditional protocol presence model.
YP currently says "NF accesses IPv4." NetLifter says "NF accesses IPv4 only if eth.ethertype == 0x0800,
AND accesses TCP only if ipv4.proto == 6, AND accesses HTTP only if tcp.dport == 80."
This conditional chain IS the protocol parse tree — and it's extractable from the branch conditions
YP already partially captures.

---

### Paper 2.4 — BinaryInferno
**Citation:** Chandler, J. "BinaryInferno: A Semantic-Driven Approach to Field Inference for Binary Message Formats."
*Usenix WOOT 2023 / IEEE CNS 2023.*

**NC extracted:** NC-1, NC-2
**Features:**
- Semantic-driven field inference: detect semantically meaningful fields vs. padding
- Delimiter detection: specific bytes that mark field boundaries
- Compression/encoding field detection: fields that are entropy-transformed
- Correlation-based field grouping: fields whose values correlate with each other
**What YP gains:** Delimiter and correlation detection applied to eBPF branch constants:
the constants an NF compares packet bytes to ARE the protocol's delimiters.
Extracting all comparison constants from branch instructions gives the full protocol discriminant set.

---

### Paper 2.5 — Protocol RE Survey
**Citation:** Kleber, S., Kopp, H., Kargl, F.
"Protocol Reverse-Engineering Methods and Tools: A Survey."
*Computer Communications* 182, 2021. doi:10.1016/j.comcom.2021.11.009

**NC extracted:** NC-2 (survey / taxonomy)
**Features (taxonomy of what PRE extracts):**
- Phase 1: Message segmentation (field boundaries)
- Phase 2: Message type identification (clustering packet types by behavior)
- Phase 3: Field semantic inference (type, role, value domain)
- Phase 4: State machine reconstruction (protocol state = sequence of message types)
**Key insight for NC-2:** Phase 4 (state machine) is EXACTLY the stateful NC missing from YP.
A protocol state machine captures: "NF is in state S1; receives SYN → transitions to S2;
in state S2, expects ACK; if not received, sends RST."
This is extractable from: map key = flow state, branch on map result = state transition check.

---

## CATEGORY NC-3: MAP / STATE CONTEXT

### What it is
What eBPF maps (or equivalent data structures) store, how they are keyed, what operation is performed,
what the stored value means, how map state evolves across NF invocations.

---

### Paper 3.1 — Yaksha-Prashna
(same citation as 1.1)
**NC-3 features (what exists):**
- `mapLookup(NF, map_id, field)` — map M is looked up using field F as (part of) key
- `mapWrite(NF, map_id, field)` — field F is written into map M
- `correlatedMaps(NF, [(map_a, map_b)])` — lookup result from map_a used as key in map_b
**NC-3 GAPS (from paper §7):**
- NO map KEY SCHEMA extraction (doesn't know IF 5-tuple or just dst_ip is the key)
- NO map VALUE TYPE extraction (doesn't know if value is a state machine enum or a backend ID)
- NO cross-NF map state tracking (can't check if NF_j sees a consistent state after NF_i writes)
- NO map type extraction (doesn't distinguish HASH_MAP from LPM_TRIE from PERCPU_ARRAY)

---

### Paper 3.2 — Klint
**Citation:** Pirelli, S., Valentukonytė, A., Argyraki, K., Candea, G.
"Automated Verification of Network Function Binaries."
*NSDI 2022.* Pages 585–600.

**NC extracted:** NC-3, NC-4, NC-5
**Key contribution: THE GHOST MAP ABSTRACTION**
**Features:**
- All NF data structures (linked lists, hash tables, LRU caches) modeled as MAPS: key → value
- Ghost map operations extracted from binary:
  - `lookup(map, key) → option[value]` — membership test / retrieval
  - `insert(map, key, value)` — new entry creation
  - `remove(map, key)` — eviction or expiration
  - `size(map)` — capacity check
  - `contains(map, key)` — membership test without retrieval
- For each operation: key TYPE and value TYPE inferred from usage patterns
- Null-branch semantics: what does the NF do when lookup returns NULL?
  - `null → drop` = membership enforcement (firewall)
  - `null → pass` = negative check (block-list)
  - `null → insert → pass` = stateful tracking (conntrack)
**Method:** Symbolic execution (KLEE) on binary without source, without debug symbols.
Invariants over data structures proven with VeriFast (separation logic).
**How violations found:**
- User writes Python specification: "if flow in table → forward; else → drop"
- Klint checks: does the symbolic execution trace match the ghost map operation sequence?
- Violation = execution path where map operation sequence deviates from spec
**What YP gains:**
- Extend `mapLookup` → `mapOp(NF, M, LOOKUP, key_fields, null_behavior)`
  where null_behavior ∈ {DROP, PASS, INSERT_THEN_PASS, REDIRECT}
- Extend `mapWrite` → `mapOp(NF, M, INSERT, key_fields, value_schema)`
- These three annotations cover 95% of NF stateful behavior

---

### Paper 3.3 — Vigor
**Citation:** Zaostrovnykh, A., Pirelli, S., Iyer, R., Rizzo, M., Pedrosa, L., Argyraki, K., Candea, G.
"Verifying Software Network Functions with No Verification Expertise."
*SOSP 2019.* Pages 275–290.

**NC extracted:** NC-3, NC-5
**Features:**
- Behavioral summary = complete sequence of stateful library calls for each execution path
- Each call: (LibVig_function, key, value, result) — maps to ghost map operations
- For each path: the SEQUENCE of (op, key, value) tuples is the NF's behavioral fingerprint
- Stateful data structure contracts: "LRU map invariant: all flows in LRU are also in the map"
  — formally proven once, amortized across all NFs using the same structure
**Method:** Split NF code:
- Stateless code → KLEE symbolic execution → path enumeration → call sequences
- Stateful code (data structures) → VeriFast theorem proving → invariants
**How violations found:**
- Specification written in Python against LibVig abstractions
- Violation = call sequence on some path does not match Python spec
- Proved correct = ALL paths match spec
**What YP gains:** The BEHAVIORAL SUMMARY concept: instead of per-fact extraction,
represent NF as "a sequence of map operations for each packet type."
This is the direct bridge between NC-3 (map context) and NC-5 (behavioral contract).

---

### Paper 3.4 — NetSMC
**Citation:** Yuan, Y., Moon, S.J., Uppal, S., Jia, L., Sekar, V.
"NetSMC: A Custom Symbolic Model Checker for Stateful Network Verification."
*NSDI 2020.* Pages 181–200.

**NC extracted:** NC-3, NC-4
**Features:**
- NF state table model: `state_table[flow_key] = state_value`
  - flow_key: which packet fields constitute the flow identifier
  - state_value: what the NF tracks (connection state, backend assignment, counter)
- NF transition rules: `(current_state, packet_predicate) → (next_state, packet_action)`
  - packet_predicate: field condition (e.g., tcp.flags has SYN set)
  - packet_action: DROP, FORWARD, REDIRECT_TO_BACKEND[X]
- Location model: which physical network point each packet is at
**Method:** Policy language for NF specification (NOT extracted from binary — specified).
Backward reachability from "bad state" using symbolic model checking.
**How violations found:**
- Policy violation: ∃ execution trace reaching a bad state (e.g., blocked host reaches server)
- Method: compute pre-image iteratively until initial state or fixpoint
- Produces: witness trace showing exactly how violation occurs
**What YP gains:** The STATE TABLE model. YP's `mapLookup` knows the map is accessed.
NetSMC's model says: the map IS the state table, and the NF's branches on map result
ARE the state transitions. Connecting YP's facts to NetSMC's model is the missing link.

---

### Paper 3.5 — Abstract Interpretation of Stateful Networks
**Citation:** Alpernas, K., Manevich, R., Panda, A., Sagiv, M., Shenker, S., Shoham, S., Velner, Y.
"Abstract Interpretation of Stateful Networks."
*SAS 2018 / Static Analysis Conference.* doi:10.1007/978-3-319-99725-4_3

**NC extracted:** NC-3, NC-4
**Features:**
- **Packet effect semantics**: for each distinguishable PACKET TYPE P, what does this NF do?
  - For a firewall: (SYN from new_ip, state=EMPTY) → INSERT entry → FORWARD
  - For a firewall: (ACK from established_ip, state=ESTABLISHED) → FORWARD
  - For a firewall: (SYN from blocked_ip, state=any) → DROP
  - This is NOT the full state space — only the PER-PACKET-TYPE effect
- Abstract map domain: represents map state as an ABSTRACT OBJECT, not a concrete table
  - key_type: which equivalence class of keys are in the map
  - value_type: what abstract values are stored
- **Cartesian abstraction**: track each NF's state INDEPENDENTLY, compose
**Theoretical result:**
- Full stateful network verification: UNDECIDABLE (ordered channels)
- With unordered channel abstraction: EXPSPACE-complete
- With resetting middleboxes (NFs that timeout/reset): POLYNOMIAL in network size
- Key parameter: k = number of map queries per packet (typically k ≤ 5 for real NFs)
- Complexity: O(n^k) where n = network size → polynomial for small k
**How violations found:**
- Isolation property: can host A reach host B through the chain?
- Method: abstract interpretation fixpoint on the Cartesian product of NF abstract states
- Sound: never misses a violation; may produce false positives
**What YP gains:** The PACKET EFFECT abstraction gives you:
- Per-packet-type NC-4 (not just "drops IPv4" but "drops IPv4 SYN when src not in whitelist")
- The theoretical justification that polynomial complexity is achievable for real NFs
- The Cartesian abstraction principle: verify each NF independently, compose results

---

### Paper 3.6 — Some Complexity Results for Stateful Network Verification
**Citation:** Alpernas, K., Panda, A., Rabinovich, A., Sagiv, M., Shenker, S., Shoham, S., Velner, Y.
"Some Complexity Results for Stateful Network Verification."
*Formal Methods in Computer Science, 2021.* arXiv:2106.01030

**NC extracted:** NC-3 (theoretical)
**Key classification (directly applicable to map semantic extraction):**
| NF Class | Map behavior | Verification complexity | Examples |
|----------|-------------|------------------------|---------|
| Class 1  | DROP/FORWARD based on membership | POLYNOMIAL | Firewall, ACL |
| Class 2  | Cache/content-based forwarding | coNP-complete | Cache, learning switch |
| Class 3  | Complex multi-field state | EXPSPACE | General stateful NF |
**What YP gains:** When YP extracts map semantics, the null_behavior annotation (DROP vs PASS vs INSERT)
CLASSIFIES the NF into Class 1/2/3. This classification tells you which queries are tractable
BEFORE attempting verification. A pipeline of 50 NFs can be classified upfront:
"these 40 are Class 1 (polynomial), these 8 are Class 2 (coNP), these 2 are Class 3 (expensive)."

---

### Paper 3.7 — Understanding Performance of eBPF Maps (SIGCOMM eBPF Workshop 2024)
**Citation:** ACM SIGCOMM 2024 Workshop on eBPF and Kernel Extensions.
"Understanding Performance of eBPF Maps."

**NC extracted:** NC-3
**Features:**
- eBPF map type taxonomy: HASH, ARRAY, LRU_HASH, PERCPU_ARRAY, LPM_TRIE, RINGBUF, SOCKHASH
- Access pattern per map type: O(1) for HASH, O(log n) for LPM_TRIE, lock-free for PERCPU
- Sharing semantics: per-CPU maps have NO cross-core sharing; global maps DO
**What YP gains:** The MAP TYPE is present in the ELF section header of every eBPF program.
Extracting map type (currently missing from YP's NC-3) gives:
- LPM_TRIE → this NF does longest-prefix-match routing
- LRU_HASH → this NF evicts old entries (conntrack with timeout)
- PERCPU → this NF has NO cross-packet state sharing (safely parallelisable)
These are SEMANTICALLY RICH facts about the NF's behavior that cost zero additional analysis.

---

## CATEGORY NC-4: PACKET ACTION CONTEXT

### What it is
What the NF does to the packet: drop, pass, redirect, modify, clone. With what conditions.
Includes: per-path actions, per-packet-type actions, action conditions (field predicates).

---

### Paper 4.1 — Yaksha-Prashna
(same citation as 1.1)
**NC-4 features:**
- `drops(NF, hook, [(field, value)])` — drops packets matching predicate list
- `passes(NF, hook, [(field, value)])` — passes packets matching predicate list
- `redirects(NF, hook, port)` — redirects to specific port/interface
- Per-path extraction: each CFG path gives a separate (predicate, action) pair
**Limitation:** Does not extract CONDITIONAL action composition:
"passes on path P1 AND drops on path P2 where P1 ∪ P2 is not the full packet space"
→ cannot tell if there are UNHANDLED PACKET TYPES.

---

### Paper 4.2 — Aquila
**Citation:** Tian, B., Gao, J., Liu, M., Zhai, E., Chen, Y., Zhou, Y., Dai, L., Yan, F., Ma, M., Tang, M., Lu, J., Wei, X., Liu, H., Zhang, M., Tian, C., Yu, M.
"Aquila: A Practically Usable Verification System for Production-Scale Programmable Data Planes."
*SIGCOMM 2021.* doi:10.1145/3452296.3472889

**NC extracted:** NC-4 (as P4 pipeline context)
**Features:**
- Table lookup context: (match_key, action_name, action_parameters) for all tables
- Control flow context: which tables execute in which order for which packet types
- Inter-pipeline value passing: field F set in ingress pipeline → read in egress pipeline
- Undefined behavior detection: packet type with no matching table entry
- Sequential encoding: encode each table lookup as SMT existential (avoids path explosion)
  - "∃ entry in table T such that packet matches and action A is applied"
  - This avoids enumerating all 2^N paths through the pipeline
**How violations found:**
- Undefined behavior: ARP packet hits IPv4-only table → no match → undefined action
- Pipeline correctness: field set in stage 1 has correct value range when read in stage 3
- Method: SMT SAT/UNSAT for "∃ packet violating assertion X"
- Bug localization: backward slice from violation to identify culprit table/action
**What YP gains:** The SEQUENTIAL ENCODING principle for map lookups:
Instead of branching on "map lookup returns value V → path P1; returns NULL → path P2",
encode as "∃ entry in map M such that..."
This is the CORRECT way to handle eBPF map lookups symbolically — treating map contents
as existentially quantified prevents path explosion from map-dependent branching.

---

### Paper 4.3 — PIX (Performance Interfaces for Network Functions)
**Citation:** Iyer, R., Argyraki, K., Candea, G.
"Performance Interfaces for Network Functions."
*NSDI 2022.* Pages 567–584.

**NC extracted:** NC-4, NC-5
**Features (performance-focused but directly relevant):**
- Extracts: for each packet TYPE and WORKLOAD, what is the execution PATH LENGTH?
  - Features: branch conditions, loop iteration counts, memory access counts
  - Method: symbolic program analysis on eBPF/DPDK NF implementations
- Path coverage: which packet types trigger which execution paths
- Memory access patterns: which cache lines are accessed for which packet types
- Evaluated on: Katran (Facebook LB), Natasha (NAT), Cilium XDP filter — all eBPF NFs
**How violations found (as performance contract):**
- Performance contract: "for packets of type P, latency ≤ X ns at percentile Y"
- Violation: actual latency exceeds contract for specific packet type
**What YP gains:** The PATH COVERAGE extraction. PIX knows which packet types trigger which
paths through an NF. This is exactly the "per-packet-type packet effect" that Alpernas et al.
define abstractly — PIX extracts it CONCRETELY from eBPF bytecode.

---

### Paper 4.4 — BOLT (Performance Contracts for NFs)
**Citation:** Iyer, R., Pedrosa, L., Zaostrovnykh, A., Pirelli, S., Argyraki, K., Candea, G.
"Performance Contracts for Software Network Functions."
*NSDI 2019.* Pages 517–530.

**NC extracted:** NC-4, NC-5
**Features:**
- Performance critical variables: variables that determine execution path length
  - For a firewall: `num_flows` (number of entries in flow table)
  - For a LB: `num_backends` (active backend count)
  - These are the STATE VARIABLES that control NF behavior
- Contract: performance as a function of critical variables
- Full software stack analysis: NF logic + DPDK framework + NIC driver
**What YP gains:** The CRITICAL VARIABLE concept.
The variables that determine performance are the SAME variables that determine behavior:
`num_flows` is the conntrack table size; behavior changes when it's full (LRU eviction vs. drop).
Identifying critical variables identifies exactly which map operations matter for NC-3 extraction.

---

## CATEGORY NC-5: BEHAVIORAL CONTRACT CONTEXT

### What it is
What the NF promises about its output given its input (Guarantee).
What the NF assumes about its input (Rely).
NF role classification: firewall, load balancer, router, NAT, monitor, etc.
Behavioral patterns: conntrack, flow affinity, rate limiting, etc.

---

### Paper 5.1 — Vigor
(same citation as 3.3)
**NC-5 features:**
- NF behavioral summary = Python specification of expected map operation sequences
- Specification structure: "if flow exists in table → forward; else → if src in allowed → insert and forward; else → drop"
- Verification: does the NF's implementation match this contract?

---

### Paper 5.2 — Klint
(same citation as 3.2)
**NC-5 features:**
- Ghost map specification: user writes expected map operation sequence
- Klint checks: binary matches spec → NO VIOLATION; else → VIOLATION with counterexample
- No annotation: operator writes Python spec, Klint automates everything else
**What YP gains:** The spec-based contract verification model.
YP can be extended with: "expected_behavior(NF, python_spec)" as a new predicate type.
Verification becomes: does YP's extracted NC-3 + NC-4 match the expected_behavior spec?

---

### Paper 5.3 — Abstract Interpretation of Stateful Networks (Alpernas et al. SAS 2018)
(same citation as 3.5)
**NC-5 features:**
- Packet effect automaton: a finite automaton encoding the NF's per-packet-type behavior
- States: abstract map content (equivalence classes of map states)
- Transitions: (packet_type, current_state) → (action, next_state)
- Guarantee: for each input (packet_type, state), the NF produces (action, new_state)
- Rely: the NF assumes arriving packets satisfy certain predicates (e.g., valid IP header)

---

### Paper 5.4 — Rely-Guarantee for Interfering Programs (Foundational Reference)
**Citation:** Jones, C.B.
"Tentative Steps Toward a Development Method for Interfering Programs."
*ACM TOPLAS* 5(4), 1983.

**NC extracted:** NC-5 (formal framework)
**The formal definition:**
- Rely(P): predicate on system state that P assumes BEFORE executing
- Guarantee(P): predicate on system state that P establishes AFTER executing
- Chain correctness: for sequential processes P_1,...,P_n:
  Guarantee(P_i) must IMPLY Rely(P_{i+1}) for all adjacent pairs
**Applied to NF chains:**
- Rely(NF_3) = {sk_buff.mark is a valid cluster_id}
- Guarantee(NF_2) = {sk_buff.mark is set to routing_value}
- Check: routing_value ≠ cluster_id → VIOLATION
- This is the exact Cilium+AWS outage in formal notation

---

### Paper 5.5 — Local Rely-Guarantee Reasoning
**Citation:** Feng, X., Ferreira, R., Shao, Z.
"On the Relationship Between Concurrent Separation Logic and Assume-Guarantee Reasoning."
*POPL 2007.*

**NC extracted:** NC-5
**Key contribution:** AUTOMATED EXTRACTION of Rely conditions from code:
- If NF reads field F and branches on it → NF implicitly relies on F being valid
- If NF reads map M with key K and drops on NULL → NF relies on M containing K
- These implicit relies can be inferred via BACKWARD ANALYSIS from decision points
**What YP gains:** The automated Rely extraction algorithm:
1. Find all branches in NF_i's CFG that depend on a packet field or map result
2. For each such branch: the field/map_result IS the implicit Rely condition
3. Extract Rely(NF_i) = union of all implicit rely conditions
This is implementable using YP's existing dataflow analysis with a backward pass.

---

### Paper 5.6 — VEP (Two-Stage Verification for Full eBPF Programmability)
**Citation:** Wu, X., Feng, Y., Huang, T., Lu, X., Lin, S., Xie, L., Zhao, S., Cao, Q.
"VEP: A Two-Stage Verification Toolchain for Full eBPF Programmability."
*NSDI 2025.*

**NC extracted:** NC-5, NC-3
**Features:**
- Proof-carrying code: C source annotations → formal proofs for eBPF
- Verification of: memory safety + functional correctness
- Cross-program invariants: what one eBPF program promises, another can rely on
**What YP gains:** The CROSS-PROGRAM INVARIANT model — directly applicable to NF chains
where NF_i's postcondition is NF_{i+1}'s precondition.

---

## CATEGORY NC-6: BINARY SEMANTIC CONTEXT

### What it is
Function purpose, instruction-level semantic similarity, code embedding, NF role classification.
Includes: binary-NL alignment, function name prediction, behavioral similarity across architectures.

---

### Paper 6.1 — CLAP (your project file)
**Citation:** (From your project CLAP_.pdf)
**NC extracted:** NC-6
**Features:**
- Contrastive binary-NL alignment
- Basic block embeddings: opcode histogram, CFG topology, data flow features
- Natural language description alignment via contrastive learning (CLIP-style)
- Zero-shot classification: 83% Recall@1 on NF role queries
**What YP gains:** Semantic labels for what basic blocks DO:
"this block implements a 5-tuple hash" → NC-3 enrichment
"this NF is a stateful firewall" → NC-5 classification

---

### Paper 6.2 — Bin2Summary (your project file)
**Citation:** (From your project bin2summary.pdf)
**NC extracted:** NC-6
**Features:**
- Functionality-relevant block identification via ARGUMENT DATA FLOW SLICING
  - r1 = packet context (xdp_md) → forward slice → identifies packet-processing blocks
  - Separates: packet logic blocks vs. bookkeeping blocks (counters, logging)
- Natural language summaries of binary function behavior
- "Decode a public key" / "Forward packet if flow exists in table"
**What YP gains:** The argument slicing technique:
- Slice from r1 (packet context) → packet processing blocks → NC-1, NC-2, NC-4 extraction
- Slice from r6 (map pointer) → state management blocks → NC-3 extraction
- Clean separation reduces noise in YP's dataflow rules

---

### Paper 6.3 — Neural Reverse Engineering of Stripped Binaries
**Citation:** David, Y., Alon, U., Yahav, E.
"Neural Reverse Engineering of Stripped Binaries using Augmented Control Flow Graphs."
*OOPSLA 2020.* arXiv:1902.09122

**NC extracted:** NC-6
**Features:**
- Predicts procedure names from stripped executables
- Input: sequences of API call sites (enriched with argument types)
- Model: set-of-sequences encoder → attention decoder → token-by-token name prediction
- Precision 81.70% / Recall 80.12% on GNU packages
**What YP gains:** The API call site enrichment:
For eBPF, API calls = BPF helper calls (bpf_map_lookup_elem, bpf_redirect, bpf_ktime_get_ns).
Enriching each helper call with argument types (which register contains the map pointer,
what packet field is being passed as key) gives richer NC-6 labels for function purpose.

---

### Paper 6.4 — Gemini / Neural Graph Embedding for Binary Similarity (CCS 2017)
**Citation:** Xu, X., Liu, C., Feng, Q., Yin, H., Song, L., Song, D.
"Neural Network-Based Graph Embedding for Cross-Platform Binary Code Similarity Detection."
*CCS 2017.*

**NC extracted:** NC-6
**Features:**
- Attributed Control Flow Graph (ACFG): each basic block annotated with:
  - Number of instructions
  - Number of arithmetic instructions
  - Number of calls (to external functions / helpers)
  - Number of transfer instructions (branches)
  - Number of memory accesses
  - Constants used in comparisons
- GNN embedding: ACFG → 64-dim vector
- Similarity: cosine distance between two ACFG vectors
**What YP gains:** The ACFG feature set for eBPF basic blocks:
These features (instruction counts, comparison constants, call counts) are extractable
from eBPF bytecode and form a structural fingerprint of each basic block.
Two eBPF NFs with similar ACFG fingerprints have similar behavioral patterns
→ useful for NF clustering (bisimulation-based approach from earlier discussions).

---

### Paper 6.5 — jTrans: Jump-Aware Transformer for Binary Code Similarity
**Citation:** Wang, H., Qu, W., Katz, G., Zhu, W., Gao, Z., Qiu, H., Zhuge, J., Zhang, C.
"jTrans: Jump-Aware Transformer for Binary Code Similarity Detection."
*ISSTA 2022.*

**NC extracted:** NC-6
**Features:**
- BERT-style pre-training on assembly instruction sequences
- Jump-aware: jump targets encoded as positional tokens so transformer attends across branches
- Captures long-range dependencies between branches and their targets
**What YP gains:** Jump-aware encoding for eBPF:
eBPF programs are linear bytecode with conditional jumps. jTrans-style encoding captures
"what happened several instructions before this conditional jump" — useful for extracting
the CONTEXT of a map lookup result usage (what instructions led to this branch?).

---

### Paper 6.6 — Gigahorse (Bytecode Decompilation via Declarative Analysis)
**Citation:** Grech, N., Brent, L., Scholz, B., Smaragdakis, Y.
"Gigahorse: Thorough, Declarative Decompilation of Smart Contracts."
*ICSE 2019.* doi:10.1145/3338906.3338977

**NC extracted:** NC-6, NC-3
**Features:**
- Decompiles EVM bytecode → 3-address intermediate representation (IR)
- Makes implicit data/control flow explicit
- Datalog-based analysis: decompilation rules expressed in Soufflé Datalog
- Stack variable resolution: EVM is a stack machine; Gigahorse resolves stack positions to variables
- Context sensitivity: different calling contexts produce different analysis results
**Relevance to eBPF:**
eBPF is a register machine (not stack), but the DATALOG-BASED ANALYSIS PATTERN is directly applicable.
Gigahorse's approach: express all analysis rules in Soufflé Datalog → compile to C++ → fast.
YP uses SWI-Prolog (top-down, stack-based). Migrating to Soufflé = 10–100x speedup + provenance.

---

## SUMMARY TABLE: What Each Paper Extracts vs What YP Has

| Paper | NC-1 | NC-2 | NC-3 | NC-4 | NC-5 | NC-6 |
|-------|------|------|------|------|------|------|
| Yaksha-Prashna (baseline) | field R/W | protocol check | map name only | drop/pass/redirect | NONE | NONE |
| BinPRE (CCS'24) | field ROLE | field order | — | — | — | — |
| NetLifter (CCS'23) | field type + cond. | parse tree | — | — | — | — |
| AIFORE (Security'23) | field boundaries | parse order | — | — | — | NN features |
| HSA (NSDI'12) | bit-level R/W | — | — | reachability | — | — |
| NetPlumber (NSDI'13) | incremental R/W | — | — | rule conflicts | — | — |
| PREVAIL (PLDI'19) | memory regions | bounds checks | pointer types | safety | — | — |
| BinaryInferno ('23) | field semantics | delimiters | — | — | — | — |
| PRE Survey ('21) | taxonomy | state machine | — | — | — | — |
| Klint (NSDI'22) | — | — | ghost map ops + null-behavior | action per op | ghost map spec | — |
| Vigor (SOSP'19) | — | — | behavioral summary | path actions | Python spec | — |
| NetSMC (NSDI'20) | — | — | state table model | transitions | policy language | — |
| Alpernas SAS'18 | — | — | packet effect automaton | per-type actions | Cartesian contract | — |
| Alpernas '21 complexity | — | — | complexity class | — | — | — |
| eBPF Maps SIGCOMM'24 | — | — | map TYPE taxonomy | — | — | — |
| Aquila (SIGCOMM'21) | — | parse tree (P4) | — | table actions | — | — |
| PIX (NSDI'22) | — | — | critical variables | path coverage | perf contract | — |
| BOLT (NSDI'19) | — | — | state variables | per-type latency | perf contract | — |
| Jones RG (TOPLAS'83) | — | — | — | — | Rely/Guarantee | — |
| Feng et al. (POPL'07) | — | — | — | — | automated Rely | — |
| VEP (NSDI'25) | — | — | — | — | cross-program inv | — |
| CLAP (your file) | — | — | — | — | role classification | embeddings |
| Bin2Summary (your file) | — | — | — | — | NL summary | arg slicing |
| Neural RE (OOPSLA'20) | — | — | — | — | function naming | API sequences |
| Gemini (CCS'17) | — | — | — | — | — | ACFG features |
| jTrans (ISSTA'22) | — | — | — | — | — | jump-aware embed |
| Gigahorse (ICSE'19) | — | — | Datalog IR | — | — | decompilation |

---

## WHAT YP NEEDS TO ADD: GAP MAP

Based on the full survey, here is the precise gap map:

**NC-1 GAPS → Add from BinPRE + AIFORE:**
- Field role extraction (TYPE/LENGTH/SEQ/CHECKSUM/PAYLOAD) via atomic semantic detectors
- Field boundary detection for implicit protocol handling (NFs without explicit checks)
- Conditional field extraction ("field F is only written on IPv4 paths")

**NC-2 GAPS → Add from NetLifter + Protocol RE Survey:**
- Full protocol PARSE TREE (not just "accessesProtocol")
- Conditional protocol presence ("processes TCP only if ipv4.proto == 6")
- State machine reconstruction (the protocol processing graph across invocations)

**NC-3 GAPS → Add from Klint + NetSMC + Alpernas + eBPF Maps Workshop:**
- Map operation TYPE: LOOKUP, INSERT, REMOVE (with null behavior)
- Map KEY SCHEMA: which packet fields constitute the key
- Map VALUE TYPE: STATE_RECORD, COUNTER, FLAG, ID
- Map TYPE from ELF: HASH, LPM_TRIE, LRU_HASH, PERCPU_ARRAY
- Per-packet-type map behavior (Alpernas packet effect semantics)

**NC-4 GAPS → Add from Aquila + Alpernas + PIX:**
- Unhandled packet types: are ALL packet types covered by the NF's action space?
- Sequential encoding: map lookups as existential quantifiers (not path branches)
- Per-packet-type action extraction (packet effect automaton)

**NC-5 GAPS → Add from Vigor + Klint + Jones RG + Feng et al.:**
- Automated Rely extraction: backward analysis from decision points
- Guarantee extraction: what fields/maps does NF_i write as part of its postcondition
- Cross-NF compatibility check: Guarantee(NF_i) ⊇ Rely(NF_{i+1})
- NF role classification: firewall/LB/router/NAT via map operation + action pattern

**NC-6 GAPS → Add from CLAP + Bin2Summary + Neural RE:**
- Argument data flow slicing to separate packet logic from bookkeeping
- Semantic labels from NL alignment for map operation purpose
- ACFG features for NF similarity/clustering (bisimulation)

---

## FULL REFERENCE LIST (27 papers)

1. Singh et al. "Yaksha-Prashna." arXiv:2602.11232, 2026.
2. Jiang et al. "BinPRE: Enhancing Field Inference in Binary Analysis Based Protocol RE." CCS 2024.
3. Shi et al. "NetLifter: Lifting Network Protocol Implementation to Precise Format Specification." CCS 2023.
4. Shi et al. "AIFORE: Smart Fuzzing Based on Automatic Input Format RE." USENIX Security 2023.
5. Kazemian, Varghese, McKeown. "Header Space Analysis: Static Checking for Networks." NSDI 2012.
6. Kazemian et al. "Real Time Network Policy Checking Using HSA." NSDI 2013.
7. Gershuni et al. "Simple and Precise Static Analysis of Untrusted Linux Kernel Extensions (PREVAIL)." PLDI 2019.
8. Chandler. "BinaryInferno: A Semantic-Driven Approach to Field Inference." WOOT/CNS 2023.
9. Kleber, Kopp, Kargl. "Protocol Reverse-Engineering Methods and Tools: A Survey." Computer Communications, 2021.
10. Pirelli et al. "Automated Verification of Network Function Binaries (Klint)." NSDI 2022.
11. Zaostrovnykh et al. "Verifying Software NFs with No Verification Expertise (Vigor)." SOSP 2019.
12. Yuan et al. "NetSMC: A Custom Symbolic Model Checker for Stateful Network Verification." NSDI 2020.
13. Alpernas et al. "Abstract Interpretation of Stateful Networks." SAS 2018.
14. Alpernas et al. "Some Complexity Results for Stateful Network Verification." FMCS 2021.
15. "Understanding Performance of eBPF Maps." SIGCOMM eBPF Workshop 2024.
16. Tian et al. "Aquila: Verification for Production-Scale Programmable Data Planes." SIGCOMM 2021.
17. Iyer et al. "Performance Interfaces for Network Functions (PIX)." NSDI 2022.
18. Iyer et al. "Performance Contracts for Software Network Functions (BOLT)." NSDI 2019.
19. Jones, C.B. "Tentative Steps Toward a Development Method for Interfering Programs." TOPLAS 1983.
20. Feng et al. "On the Relationship Between Concurrent Separation Logic and Assume-Guarantee." POPL 2007.
21. Wu et al. "VEP: A Two-Stage Verification Toolchain for Full eBPF Programmability." NSDI 2025.
22. (CLAP — project file)
23. (Bin2Summary — project file)
24. David, Alon, Yahav. "Neural Reverse Engineering of Stripped Binaries." OOPSLA 2020.
25. Xu et al. "Neural Network-Based Graph Embedding for Cross-Platform Binary Code Similarity (Gemini)." CCS 2017.
26. Wang et al. "jTrans: Jump-Aware Transformer for Binary Code Similarity." ISSTA 2022.
27. Grech et al. "Gigahorse: Thorough, Declarative Decompilation of Smart Contracts." ICSE 2019.
