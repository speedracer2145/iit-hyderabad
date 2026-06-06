# Network Context Extraction from Bytecode/Binary: Full Literature Survey
## What every relevant paper extracts, how they extract it, what features they use, and how they detect violations

---

## The Core Question we are tryign to address

"How do you extract MORE network context from bytecode than Yaksha-Prashna does?"

Before answering, you need to understand the precise taxonomy:

LEVEL 1 — Structural context (what YP does well):
  Which fields are read/written, which protocols are checked, which helpers are called,
  which actions are taken. Extracted via dataflow analysis.

LEVEL 2 — Semantic context (where YP is limited):
  What a map IS (flow table vs rate limiter vs routing table).
  What a field ACCESS MEANS (this is a flow key, not just "src_ip is read").
  What the program's BEHAVIORAL CONTRACT is (it preserves affinity, it is conntrack-aware).
  Extracted via: AI over map domains, protocol RE, argument slicing.

LEVEL 3 — Stateful context (what YP entirely lacks):
  How map state EVOLVES across invocations.
  What state a chain of NFs collectively maintains.
  Whether state transitions across NFs are consistent.
  Extracted via: packet effect semantics, symbolic model checking, abstract middlebox modeling.

---

## DOMAIN 1: eBPF-Specific Context Extraction

### Paper 1: Yaksha-Prashna (your baseline)
**Citation:** Singh et al., arXiv:2602.11232, 2026.

**What they extract:**
  - updatesField(NF, field), readsField(NF, field) — packet header field R/W
  - accessesProtocol(NF, protocol) — from next-protocol checks
  - mapLookup(NF, map, field), mapWrite(NF, map, field) — map access
  - correlatedMaps(NF, [(map_a, map_b)]) — when lookup result is used as key
  - drops(NF, hook, [(field, value)]), passes(NF, ...) — packet actions with conditions
  - redirects(NF, ...) — redirect targets
  - invokedHelpers(NF, helper) — BPF helper calls

**How they extract it:**
  Dataflow analysis on the eBPF CFG. Rule-based extraction table (Table 1 in paper).
  Protocol identification via: next-protocol field check + bounds check before header access.
  Map field tracking via: R7 rule propagates header field annotations through registers.

**Violation detection:**
  Prolog queries over the KB. RAW/WAR/WAW dependencies via successorNF + updatesField/readsField.
  24 assertion predicates. Query examples: "NF2 writes field X; NF3 reads field X" → RAW dep.

**Known limitations (directly from paper §7):**
  - No map KEY SCHEMA extraction (knows map is accessed, not WHAT key type)
  - No map VALUE TYPE extraction (knows map is written, not WHAT value means)
  - Protocol check assumption: NFs that skip explicit next-protocol checks are not identified
    (NF12 UDP example — only TCP is identified because UDP has no explicit check)
  - No runtime state queries (map contents are runtime-dependent)
  - No broadcast packet predicate
  - No algorithm identification (hash function type, DDoS pattern, encryption)
  - XDP only: TC, socket filter, cgroup hooks not supported

---

### Paper 2: PREVAIL — Simple and Precise Static Analysis of Untrusted Linux Kernel Extensions
**Citation:** Gershuni, Amit, Gurfinkel, Narodytska, Navas, Rinetzky, Ryzhyk, Sagiv.
*PLDI 2019.* doi:10.1145/3314221.3314590

**What they extract:**
  - Memory region typing: stack | context (sk_buff/xdp_md) | packet data | map values
  - Pointer type tracking: PTR_TO_MAP_VALUE, PTR_TO_PACKET, PTR_TO_STACK
  - Integer range intervals: [lo, hi] for every register at every program point
  - Memory offset bounds: is this access within the safe region of its type?
  - Loop bounds: abstract fixpoints for bounded loops

**How they extract it:**
  Abstract interpretation with a CUSTOM abstract domain for eBPF memory:
  - "Offset_value" domain: tracks pointer + offset as a pair
  - "Interval" domain: integer value ranges (from Crab AI framework)
  - Mixed precision: precise (zones/intervals) for packet-offset registers,
    coarser for general-purpose integers
  Key insight: eBPF programs access memory in a DISCIPLINED way — fixed regions known at
  compile time. This lets you use tight abstract domains instead of generic shape analysis.

**Violation detection:**
  Safety properties only: out-of-bounds access, illegal pointer arithmetic, type errors.
  NOT functional (does not check NF behavior, only safety).
  But: the abstract domain for memory regions is directly transferable to functional analysis.

**What YP can take from this:**
  The MEMORY REGION abstract domain for eBPF gives you a way to distinguish
  map value pointers (PTR_TO_MAP_VALUE + offset) from packet pointers
  (PTR_TO_PACKET + offset). This distinction is exactly what you need to extract
  map VALUE STRUCTURE — what fields within the map value does the NF access?

---

### Paper 3: Automatic Bit-and Memory-Precise Verification of eBPF Code
**Citation:** Bromberger, Schwarz, Weidenbach. *LPAR 2024.*
(Referenced in YP's related work section)

**What they extract:**
  - Full bitvector-precise semantics of eBPF instructions
  - Memory cell typing and access patterns
  - Formal specification of the eBPF ISA as SMT theories

**How they extract it:**
  Translation to first-order logic with bit-vector arithmetic. Mechanized in Isabelle/HOL.

**Violation detection:**
  Memory safety, functional correctness against formal spec.
  Key contribution: SOUNDNESS PROOF for the translation — no false negatives in memory safety.

**What YP can take from this:**
  The bitvector-precise eBPF ISA formalization. You can use their ISA spec
  to make your dataflow rules sound — currently YP's rules are sound by construction
  but not formally proven.

---

## DOMAIN 2: NF Binary Verification (source-agnostic)

### Paper 4: Klint — Automated Verification of Network Function Binaries
**Citation:** Pirelli, Valentukonytė, Argyraki, Candea. *NSDI 2022.*
doi:10.usenix.org/conference/nsdi22/presentation/pirelli

**What they extract:**
  The KEY INSIGHT is the "ghost map" universal abstraction:
  - All NF data structures (linked lists, hash tables, LRU maps) are modeled as MAPS
    (abstract: key → value, with containment and size operations)
  - Ghost maps are SPECIFICATIONS, not implementations
  - NF behavior is expressed as: sequence of map operations (insert, lookup, remove)
    across all possible execution paths

**How they extract it:**
  Symbolic execution of the NF binary (without source code, without debug symbols).
  KLEE-style path exploration + custom memory model for NF data structures.
  The symbolic execution traces are converted to sequences of map operations.
  Abstract invariants over map operations are proven using VeriFast (separation logic).

**Features extracted as map operations:**
  - lookup(map, key) → value: flow table check
  - insert(map, key, value): new connection tracking entry
  - remove(map, key): eviction, timeout
  - size(map): capacity-based drop decision
  - contains(map, key): membership test

**Violation detection:**
  Specification comparison: user writes a Python spec of what the NF SHOULD do.
  Klint checks: does the symbolic execution trace match the spec's map operation sequence?
  Violation = there exists an execution path where the map operation sequence deviates from spec.

**What YP can take from this:**
  The GHOST MAP abstraction. YP knows THAT a map is accessed. Klint's ghost map
  tells you WHAT OPERATION on the map (lookup, insert, remove) and WHAT THE
  SEMANTICS OF THAT OPERATION IS. This is exactly the missing "map semantic schema"
  in YP's limitations.

  Concretely: extend YP's mapLookup/mapWrite facts to:
    mapOp(NF, map, LOOKUP, key_fields, [null_implies_drop | null_implies_pass])
    mapOp(NF, map, INSERT, key_fields, value_schema)
    mapOp(NF, map, REMOVE, key_fields, trigger_condition)

  These three operations cover 95% of NF stateful behavior and are extractable
  from bytecode via the same dataflow rules YP already has.

**Limitation vs YP:**
  Klint requires specifying the expected behavior in Python. YP is query-driven.
  Klint takes 2.7s–1.5min per NF. YP takes 13–300ms. YP is 200–1000× faster.

---

### Paper 5: Vigor — Verifying Software NFs with No Verification Expertise
**Citation:** Zaostrovnykh et al. *SOSP 2019.* vignat.github.io/vigor-paper.pdf

**What they extract:**
  - All execution PATHS through the stateless NF code (via exhaustive symbolic execution)
  - For each path: the sequence of STATEFUL LIBRARY CALLS made
    (these are the Vigor library operations — equivalents of map operations)
  - Formal invariants over data structures (proven once, amortized across all NFs)

**How they extract it:**
  Split NF code into:
    (1) Stateless code (NF logic): symbolically executed via KLEE → path traces
    (2) Stateful code (data structures): formally verified via VeriFast + separation logic

  The symbolic execution reports: for all possible input packets,
  the sequence of stateful calls made. This is the "behavioral summary" of the NF.

**Features used:**
  Not features in the ML sense. The behavioral summary IS the semantic context:
  sequence of (data_structure_op, key, value, result) tuples for each path.

**Violation detection:**
  Specification-based: Python spec says "if flow is in table, forward; else drop."
  Violation = path through NF code where the stateful call sequence doesn't match spec.
  Proven correct = all paths match spec.

**What YP can take from this:**
  The BEHAVIORAL SUMMARY concept. YP extracts per-fact context.
  Vigor's contribution: express NF behavior as a SEQUENCE OF OPERATIONS, not individual facts.
  For YP, this means: instead of separate mapLookup + drops facts,
  represent the NF's core behavior as:
    behaviorPattern(NF, [lookup(flow_table, 5-tuple, result),
                         if(result == NULL, drop, forward)])
  This is a higher-level behavioral abstraction that captures INTENT.

---

## DOMAIN 3: Stateful Network Verification

### Paper 6: NetSMC — Stateful Network Verification via Symbolic Model Checking
**Citation:** Yuan, Moon, Uppal, Jia, Sekar. *NSDI 2020.*
doi:usenix.org/conference/nsdi20/presentation/yuan

**What they extract (per NF):**
  - Packet fields: 5-tuple fields + any user-defined header fields
  - NF state tables: (flow_key → state_value) models
  - NF transition rules: (current_state, packet_predicate) → (next_state, packet_action)
  - Packet location: which physical network point the packet is at

**How they extract it:**
  NOT from bytecode — from a CUSTOM POLICY LANGUAGE.
  NF behavior is SPECIFIED, not extracted from binaries.
  The specification captures state tables and transition rules.

**Features used:**
  Existential first-order logic formulas over:
  - NF state variables: state_i ∈ {BLOCKED, ALLOWED, TRACKED}
  - Packet field variables: src_ip, dst_ip, proto, etc.
  - Location variables: which network point the packet is at

**Violation detection:**
  Backward reachability: given a "bad state" (e.g., blocked host reaches server),
  compute the pre-image iteratively until reaching the initial state or a fixpoint.
  If pre-image intersects initial state: violation found with a witness trace.
  Custom algorithm for stateful networks: O(states × NFs) instead of exponential BDD.

**What YP can take from this:**
  The NF TRANSITION RULE model. NetSMC models each NF as a state machine:
  (current_state, packet_guard) → (next_state, packet_action).
  YP already extracts packet_action (drops/passes/redirects with conditions).
  What YP is missing: the STATE VARIABLE and the STATE TRANSITION.
  For a stateful firewall: state ∈ {NEW, ESTABLISHED, CLOSED}.
  For a load balancer: state = backend_assignment[flow_key].
  These states are tracked via eBPF maps — connecting the dots from YP's mapWrite
  to NetSMC's state transition is the key missing link.

**Key result:**
  Stateful network verification is UNDECIDABLE in general (proven in companion paper).
  NetSMC's tractability comes from relaxations:
  (1) Ignore packet arrival ORDER (unordered channel abstraction)
  (2) Ignore CORRELATIONS between different NFs' states
  (3) Represent state by packet EFFECT, not full state space

---

### Paper 7: Abstract Interpretation of Stateful Networks
**Citation:** Alpernas, Manevich, Panda, Sagiv, Shenker, Shoham, Velner.
*SAS 2018.* arxiv.org/pdf/1708.05904

**What they extract (per middlebox/NF):**
  - "Packet effect semantics": for each packet type p, what does this middlebox
    DO to p? (This is NOT the full state space — just the effect on each packet TYPE)
  - Packet type = equivalence class of packets with the same behavior
  - State queries: how many times does the middlebox READ its own state per packet?
    (This is the key parameter k for complexity: O(n^k) where n = network size)

**How they extract it:**
  Manual specification of middlebox behavior in their framework.
  NOT automated extraction from binaries.

**The critical theoretical result:**
  Safety verification of stateful networks is:
  - UNDECIDABLE in general (unbounded ordered channels between middleboxes)
  - EXPSPACE-complete when ignoring channel ordering
  - POLYNOMIAL when: (a) middleboxes can "reset" (timeout/failure recovery)
                      (b) state queries per packet k is small (which is true for real NFs!)

  This last result is the key: REAL NFs make at most 2–5 map lookups per packet.
  So k ≤ 5 in practice, and the algorithm is O(n^5) = polynomial in network size.

**What YP can take from this:**
  The PACKET EFFECT abstraction. Instead of extracting all possible behaviors,
  extract: for each distinguishable PACKET TYPE (IPv4/IPv6/TCP/UDP/ARP/...),
  what is the NF's EFFECT on that packet type?
  This is EXACTLY what YP's drops(NF, hook, [(field, value)]) fact begins to capture —
  but packet effect semantics is more complete: it includes PASS, DROP, REDIRECT,
  and STATE MUTATION for each packet type.

  The complexity result means: if you model NFs as packet-effect automata,
  chain verification scales polynomially with network size,
  even though it's undecidable for the full model.

---

### Paper 8: Some Complexity Results for Stateful Network Verification
**Citation:** Alpernas, Panda, Rabinovich, Sagiv, Shenker, Shoham, Velner.
Formal Methods in Computer Science, 2021. arxiv.org/pdf/2106.01030

**The key theoretical backbone:**
  Classifies middlebox types by verification complexity:
  - Class 1 (FIREWALLS): state = accept/deny decision. Polynomial verification.
  - Class 2 (CACHES/LEARNING SWITCHES): state = content table. coNP-complete.
  - Class 3 (GENERAL STATEFUL): EXPSPACE-complete.
  - Class GENERAL WITH ORDERING: UNDECIDABLE.

**What this means for YP:**
  When you extract map semantics from YP's bytecode analysis,
  you are implicitly CLASSIFYING the NF:
  - NF whose map is used for DROP decisions → Class 1 (tractable)
  - NF whose map stores cached content → Class 2 (coNP)
  - NF whose map has complex multi-field dependencies → Class 3 (expensive)
  
  This classification tells you WHICH QUERIES ARE TRACTABLE without trying them.

---

## DOMAIN 4: Protocol Reverse Engineering from Binary (PRE)

### Paper 9: BinPRE — Enhancing Field Inference in Binary Analysis Based Protocol RE
**Citation:** Jiang, Zhang, Wan, Chen, Sun, Su. *CCS 2024.*
doi:10.1145/3658644.3690299

**What they extract:**
  - Field FORMAT: boundaries between protocol fields (where does field A end, field B begin?)
  - Field SEMANTICS: what does each field MEAN? (length field, type field, checksum, payload)
  - From binary implementations, no source code, no traces required

**How they extract it:**
  Three-stage pipeline:
  (1) Instruction-based SEMANTIC SIMILARITY: group instructions by what they DO
      (comparison instructions that look like protocol type checks cluster together)
  (2) Atomic SEMANTIC DETECTORS: small pattern matchers for known field types:
      - length_field_detector: finds "if read_value > remaining_bytes then drop"
      - type_field_detector: finds "if field == CONST then branch"
      - checksum_field_detector: finds XOR/add-loop patterns over packet bytes
      - sequence_num_detector: finds fields compared to stored previous values
  (3) Cluster-and-refine: cluster similar detections, refine with cross-field validation

**Features:**
  - Instruction opcode sequences around field accesses
  - Branch condition patterns (what constant is the field compared to?)
  - Data flow from field access to subsequent operations
  - Memory access patterns (field size, alignment)

**Violation detection:**
  Not about NF chain violations — about PRE accuracy.
  F1-score 0.74 on semantic inference of field types.
  But: the atomic semantic detectors are DIRECTLY APPLICABLE to YP's branch extraction.

**What YP can take from this:**
  YP's branch extraction currently produces:
    BranchSemantic(field=ethernet.ethertype, value=0x0800, meaning=ETHERTYPE_CHECK)
  BinPRE's atomic detectors add:
    FieldRole(ethernet.ethertype, TYPE_FIELD)   ← this field discriminates packet types
    FieldRole(ipv4.total_length, LENGTH_FIELD)  ← this field controls parsing bounds
    FieldRole(tcp.seq_num, SEQUENCE_FIELD)      ← this field enables ordered delivery
  
  These FIELD ROLES are exactly the additional network context YP needs.
  They are deterministically extractable from the same instruction patterns YP already analyzes.

---

### Paper 10: Protocol RE Survey
**Citation:** Kleber, Kopp, Kargl. Computer Communications, 2021.
doi:10.1016/j.comcom.2021.11.009

**The PRE phase taxonomy (directly applicable to YP):**
  Phase 1: DATA COLLECTION (network traces OR binary execution)
  Phase 2: FEATURE EXTRACTION — what information to use
  Phase 3: MESSAGE TYPE IDENTIFICATION — clustering packet types
  Phase 4: MESSAGE FORMAT INFERENCE — field boundaries
  Phase 5: SEMANTIC DEDUCTION — field meaning
  Phase 6: BEHAVIOR MODEL RECONSTRUCTION — state machine over protocol exchanges

**What features are used in PRE:**
  - Byte position frequency distributions (which offsets are consistently read?)
  - Comparison constants (what values are fields compared to?)
  - Branch coverage per field value (does reading offset 12 == 0x0800 produce different behavior?)
  - Data flow length (how many instructions between field read and branch?)
  - Register reuse patterns (is a field value stored and reused → it's a key field)

**Critical insight for YP:**
  Phase 6 BEHAVIOR MODEL RECONSTRUCTION is exactly what YP is missing.
  A protocol state machine captures: "this NF tracks TCP connection states.
  It reads tcp.flags to detect SYN/ACK/FIN and transitions its state table."
  YP knows tcp.flags is read. PRE phase 6 would tell you: it's used in a STATE MACHINE.

---

## DOMAIN 5: Production-Scale Data Plane Verification

### Paper 11: Aquila — Verification for Production-Scale Programmable Data Planes
**Citation:** Tian et al. *SIGCOMM 2021.* doi:10.1145/3452296.3472889

**What they extract (per P4 program):**
  - Packet header field reads/writes (P4 is typed, so this is easier than eBPF)
  - Table lookups: (match_key, action) pairs for all tables
  - Control flow: which tables execute in which order for which packet types
  - Pipeline stage ordering: which processing steps are in which pipeline stage
  - Inter-pipeline value passing: which fields are set in stage 1 and read in stage 2

**How they extract it:**
  P4 programs are compiled → abstract syntax tree → symbolic encoding.
  Sequential encoding algorithm: instead of enumerating all 2^N paths,
  encode each TABLE LOOKUP as a SMT existential quantifier over possible entries.
  This avoids path explosion for P4 pipelines.

**Violation detection:**
  Specification = Python-like assertions over packet fields.
  Violations found: ARP packets hitting IPv4-only table (undefined behavior).
  Method: SAT/UNSAT for "∃ packet such that spec is violated."
  Bug localization: narrow down to specific table/action via slice analysis.

**What YP can take from this:**
  SEQUENTIAL ENCODING to avoid path explosion.
  Instead of: enumerate all paths through NF bytecode
  Do: encode each MAP LOOKUP as "∃ entry in map such that..."
  This is the correct way to handle eBPF maps symbolically —
  treat map contents as existentially quantified, not as unconstrained symbolic arrays.

---

### Paper 12: Abstract Interpretation of Stateful Networks (Revisited for features)
(see Paper 7 above for full citation)

**Additional feature: the Cartesian abstraction for NF chains:**
  Instead of tracking the JOINT state of all NFs in a chain simultaneously,
  track each NF's state INDEPENDENTLY and compose.
  This is called Cartesian abstraction — known to be sound for safety properties
  in "reverting" networks (where NFs can reset state).

  Key result for your work:
  If you model each NF as a PACKET-EFFECT AUTOMATON:
    State: the NF's map contents (abstracted)
    Transitions: triggered by packet fields
    Output: packet action (drop/pass/redirect)
  
  Then chain verification = composition of packet-effect automata.
  Polynomial complexity in network size. Sound for reachability/isolation properties.

---

## DOMAIN 6: Binary Code Understanding (Cross-Domain Insights)

### Paper 13: Bin2Summary (your project file)
**What they extract:**
  Functionality-relevant blocks via ARGUMENT DATA FLOW SLICING.
  The key idea: slice from the function's input arguments forward.
  Blocks that process the input directly are "functionality-relevant."

**Applied to eBPF:**
  r1 = pointer to xdp_md (the input argument).
  Forward slice from r1 identifies: packet header access blocks (functionality-relevant)
  vs. map bookkeeping blocks (not directly processing the packet).
  This gives you a cleaner separation of packet-processing logic from state management.

---

### Paper 14: CLAP (your project file)
**What they extract:**
  Contrastive binary-NL alignment via:
  - Basic block feature vectors (opcode histogram, control flow features)
  - Lifted IR (architecture-independent intermediate representation)
  - Natural language description alignment via contrastive learning

**Applied to eBPF:**
  Each eBPF basic block → 64-dim embedding.
  Query: "does this block implement a 5-tuple hash?" → cosine similarity search.
  This gives semantic labels WITHOUT the rigid rule-based extraction of YP.

---

## SYNTHESIS: What YP Is Missing and How to Extract It

| Missing context | What is it | How to extract | Paper backing |
|---|---|---|---|
| **Map operation type** | Is this lookup-then-drop or lookup-then-redirect? | Trace result of map_lookup through SSA to branch condition | Klint (NSDI'22): ghost map ops |
| **Map key schema** | Which packet fields constitute the map key? | Backward SSA slice from r2 at map helper call site | BinPRE (CCS'24): field role detection |
| **Map value structure** | What does the stored value MEAN? | Forward trace of map_lookup result + value usage pattern | Klint ghost map semantics |
| **Field semantic role** | Is this a TYPE field, LENGTH field, SEQ field? | BinPRE atomic detectors on branch patterns | BinPRE (CCS'24) |
| **NF state machine** | What states does the NF track? What triggers transitions? | Map key → value → branch pattern forms a state transition | NetSMC (NSDI'20): NF modeling |
| **Behavioral pattern** | Is this NF a firewall, LB, router? What CONTRACT does it have? | Behavioral summary from map operations + packet actions | Vigor (SOSP'19): behavioral summary |
| **Packet effect semantics** | For packet type P, what does this NF do? | Per-packet-type path enumeration on the CFG | Alpernas et al. (SAS'18) |
| **Protocol skip behavior** | NFs that handle protocols without explicit checks | Use field ROLE not just EQUALITY CHECK to identify protocol handling | BinPRE phase 2 |
| **Chain state evolution** | How does shared map state change across NF invocations? | Packet-effect composition of per-NF state machine models | Alpernas et al. (SAS'18) + NetSMC |
| **Runtime assumptions** | What does the NF assume about its environment? | Pattern match: redirect without bounds check → CPUMAP_POPULATED assumption | Your SemanticIR design (Part 1) |

---

## The Three Problems Re-Stated as Feature Extraction Problems

Your professors are right. The three "problems" are actually FEATURE GAPS:

PROBLEM 1 (path explosion) IS ACTUALLY:
  Lack of NF SUMMARY FEATURES that can be composed without re-enumerating paths.
  Fix: extract transfer function features (field read/write lattice per NF) + map operation type.
  Papers: Klint (ghost map ops), PREVAIL (abstract domain per memory region).

PROBLEM 2 (stateful queries) IS ACTUALLY:
  Lack of MAP SEMANTIC FEATURES: key schema, value type, operation type, state transition.
  Fix: add map operation extraction (LOOKUP/INSERT/REMOVE + key_fields + null_behavior).
  Papers: Klint (ghost maps), NetSMC (state tables), Alpernas et al. (packet effect).

PROBLEM 3 (complex chain queries) IS ACTUALLY:
  Lack of BEHAVIORAL CONTRACT FEATURES: NF role, packet effect per type, state machine.
  Fix: extract packet-effect-per-type facts + behavioral pattern from map ops + packet actions.
  Papers: Vigor (behavioral summary), Alpernas et al. (packet effect automaton),
          BinPRE (field semantic roles).

---

## Full Reference List

1. Singh et al. "Yaksha-Prashna." arXiv:2602.11232, 2026.
2. Gershuni et al. "Simple and Precise Static Analysis of Untrusted Linux Kernel Extensions (PREVAIL)." PLDI 2019.
3. Bromberger, Schwarz, Weidenbach. "Automatic Bit-and Memory-Precise Verification of eBPF Code." LPAR 2024.
4. Pirelli, Valentukonytė, Argyraki, Candea. "Automated Verification of Network Function Binaries (Klint)." NSDI 2022.
5. Zaostrovnykh et al. "Verifying Software NFs with No Verification Expertise (Vigor)." SOSP 2019.
6. Yuan, Moon, Uppal, Jia, Sekar. "NetSMC: A Custom Symbolic Model Checker for Stateful Network Verification." NSDI 2020.
7. Alpernas, Manevich, Panda, Sagiv, Shenker, Shoham, Velner. "Abstract Interpretation of Stateful Networks." SAS 2018.
8. Alpernas et al. "Some Complexity Results for Stateful Network Verification." FMCS 2021.
9. Jiang et al. "BinPRE: Enhancing Field Inference in Binary Analysis Based Protocol Reverse Engineering." CCS 2024.
10. Kleber, Kopp, Kargl. "Protocol Reverse-Engineering Methods and Tools: A Survey." Computer Communications, 2021.
11. Tian et al. "Aquila: A Practically Usable Verification System for Production-Scale Programmable Data Planes." SIGCOMM 2021.
12. Panda et al. "Verifying Isolation Properties in the Presence of Middleboxes." NSDI 2015.
13. (Bin2Summary — your project file)
14. (CLAP — your project file)
