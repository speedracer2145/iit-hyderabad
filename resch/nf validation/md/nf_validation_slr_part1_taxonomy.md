# Network Function Validation — Systematic Literature Review
## Part 1: Taxonomy & Classification Framework

> **Survey Scope**: This SLR covers all significant research on validating network functions (NFs) including firewalls, NATs, load balancers, IDS/IPS, routing functions, service chains, SDN data planes, programmable data planes (P4, eBPF/XDP), and virtual/cloud NFs. Both single-NF and network-wide validation are in scope. Both pre-deployment and runtime approaches are included.

---

## 1. Taxonomy of NF Validation Approaches

The taxonomy below was constructed from first principles and iteratively refined against the discovered literature.

### 1.1 Validation Timing Dimension

| Timing Class | Definition | Examples |
|---|---|---|
| **Offline / Pre-deployment** | Validation runs before NF is deployed; no live traffic involved | Formal model checking, static analysis, symbolic execution |
| **Online / Runtime** | Validation runs while NF processes live traffic | Runtime monitoring, invariant checking on live state |
| **Incremental / Update-triggered** | Validation runs only when configuration changes | Delta-net, VeriFlow, Minesweeper for diffs |
| **Continuous** | Always-on background validation | Streaming telemetry + invariant engines |
| **Postmortem** | Validation after the fact from logs/traces | Trace-based replay, log analysis |
| **Hybrid** | Combines offline model + online monitoring | Offline policy model + runtime enforcement |

---

### 1.2 Validation Methodology Dimension

| Methodology | Description | Key Techniques |
|---|---|---|
| **Rule-based Validation** | Pattern matching on rules/configs; no execution | Firewall rule conflict detection, ACL analysis |
| **Static Analysis** | Analysis of code/config without execution | Data-flow analysis, taint tracking, abstract interpretation |
| **Symbolic Execution** | Explores all code paths via symbolic inputs | SMT-based path exploration, path constraints |
| **Model Checking** | Exhaustive state-space exploration | BFS/DFS over state graph, CTL/LTL formulas |
| **Formal Verification** | Mathematical proof of correctness | Theorem proving (Coq, Isabelle), deductive verification |
| **Dataplane Verification** | Header-space or BDD-based forwarding analysis | HSA, BDD reachability, ternary matching |
| **SMT/SAT-based Verification** | Constraint solving for property checking | Z3, CVC5, Boolector backends |
| **Runtime Monitoring** | Observing live NF behavior for invariant violations | Stream processing, eBPF monitoring hooks |
| **Trace Validation** | Replaying/analyzing packet traces | Differential replay, trace-based conformance |
| **Fuzzing / Testing** | Generating inputs to find bugs | Grammar-based, mutation-based, coverage-guided fuzzing |
| **Behavioral Validation** | Comparing actual vs. expected behavior | Reference model comparison, oracle-based testing |
| **Learning-based / ML** | Using ML to learn and check NF behavior | Anomaly detection, learned invariants |
| **Abstract Interpretation** | Sound over-approximation of program semantics | Domain-based AI for eBPF, NF code |
| **Hybrid** | Combines two or more methodologies | Offline formal + online monitoring |

---

### 1.3 Validation Target Dimension

#### 1.3.1 Safety / Reachability Properties
- **Reachability**: Can packet X reach destination Y?
- **Isolation**: Can packet from tenant A reach tenant B's resources?
- **Loop freedom**: Does forwarding contain no cycles?
- **Black-hole freedom**: Does every packet have a valid forwarding path?
- **Consistency**: Do all paths agree on routing decisions?

#### 1.3.2 Correctness Properties
- **Firewall correctness**: Does firewall implement intended ACL policy?
- **NAT correctness**: Are source/destination translations correct and reversible?
- **Load balancing correctness**: Are flows distributed per policy?
- **Packet transformation correctness**: Are header rewrites correct?
- **Service chain correctness**: Does traffic traverse NFs in order?
- **State consistency**: Does NF state converge to expected values?
- **Connection tracking correctness**: Are stateful session tables accurate?
- **Protocol compliance**: Does NF follow RFC specifications?

#### 1.3.3 Security Properties
- **Policy compliance**: Does NF enforce intended security policy?
- **Information isolation**: Do tenant networks remain isolated?
- **ACL correctness**: Are access control lists correctly applied?
- **Invariant preservation**: Are security invariants maintained?

#### 1.3.4 Safety / Program Safety Properties
- **Memory safety**: No buffer overflows, use-after-free, OOB access
- **Termination**: Program always terminates (no infinite loops)
- **Helper safety**: Only safe kernel helpers called
- **Type safety**: Correct use of types
- **Resource safety**: Bounded resource usage

#### 1.3.5 Performance Properties
- **Latency correctness**: Latency within bounds
- **Throughput correctness**: NF maintains required throughput
- **Scalability correctness**: Performance scales as expected

---

### 1.4 NF Type Dimension

| NF Category | Specific NF Types |
|---|---|
| **Packet Filters** | Firewalls, ACLs, iptables/nftables rules |
| **Address/Port Translators** | NAT, NAPT, PAT, DNAT, SNAT |
| **Traffic Managers** | Load balancers (L3/L4/L7), traffic shapers, QoS |
| **Intrusion Detection/Prevention** | IDS, IPS, DPI |
| **Routing Functions** | IP routing, BGP, OSPF, MPLS, segment routing |
| **Service Chains** | Sequential NF chains, SFC (RFC 7665) |
| **SDN Data Planes** | OpenFlow-based forwarding |
| **P4 Programs** | P4₁₄ / P4₁₆ programs on software/hardware |
| **eBPF/XDP/TC Programs** | BPF bytecode loaded into kernel |
| **DPDK/Click Programs** | Userspace packet processors |
| **Virtual NFs** | VNFs in NFV/VM environments |
| **Cloud NFs** | AWS Security Groups, Azure NSG, GCP VPC rules |
| **SmartNIC Programs** | Programs on Mellanox/Marvell/Intel SmartNICs |
| **FPGA Programs** | Network processing on FPGAs |

---

### 1.5 Abstraction Level Dimension

| Level | Description | Examples |
|---|---|---|
| **Source Code** | High-level language: C, P4, Python | Click elements, P4 programs, DPDK NFs |
| **Intermediate Representation** | LLVM IR, BPF IR | Compiled eBPF before optimization |
| **Bytecode** | eBPF bytecode (64-bit RISC ISA) | `.o` files loaded by bpf() syscall |
| **Binary / Machine Code** | Native x86-64, ARM binaries | Compiled router software |
| **Control Flow Graph** | CFG extracted from code | CFG of eBPF program |
| **Forwarding Tables** | FIB/RIB entries, OpenFlow tables | Switch forwarding tables |
| **Packet Traces** | PCAP files, flow records | tcpdump captures |
| **Runtime Telemetry** | Live counters, spans, events | eBPF perf events, INT data |
| **Network Topology** | Graph of nodes/links/policies | Config-level analysis |
| **Control Plane Rules** | BGP routes, OSPF LSAs, config files | BGP policy, Batfish configs |
| **State Tables/Maps** | Connection track tables, NAT tables | conntrack, BPF maps |
| **Formal Model** | Petri nets, state machines, process algebra | Model extracted for verification |

---

### 1.6 Statefulness Dimension

| Category | Description |
|---|---|
| **Stateless NF Validation** | NF has no per-flow state; validation is purely structural |
| **Stateful NF Validation** | NF maintains per-flow/per-connection state; state explosion challenge |
| **Stateful NF w/ Bounded State** | State bounded by symbolic summaries or finite abstractions |
| **Cross-NF State Validation** | State consistency across NF replicas/instances |

---

### 1.7 Scope Dimension

| Scope | Description |
|---|---|
| **Single NF** | Validates one NF in isolation |
| **NF Chain / Service Graph** | Validates ordered composition of NFs |
| **Network-wide** | Validates entire data plane |
| **Multi-tenant** | Validates across tenant boundaries |
| **Cross-domain** | Validates across administrative domains |

---

## 2. Classification Taxonomy Summary (Multi-Dimensional)

Each paper in this SLR will be classified along:

1. **T** = Timing (Offline / Online / Incremental / Hybrid / Postmortem)
2. **M** = Methodology (Rule / Static / Symbolic / ModelCheck / Formal / Dataplane / SMT / Runtime / Trace / Fuzz / Behavioral / Learning / AbsInt / Hybrid)
3. **P** = Properties Validated (Reachability / Isolation / LoopFree / Firewall / NAT / LB / ServiceChain / Protocol / Memory / Termination / etc.)
4. **N** = NF Type (Firewall / NAT / LB / IDS / Router / SFC / SDN / P4 / eBPF / DPDK / VNF / Cloud / SmartNIC)
5. **S** = Statefulness (Stateless / Stateful / BoundedStateful)
6. **G** = Granularity (SingleNF / Chain / NetworkWide / MultiTenant)
7. **A** = Abstraction Level (Source / IR / Bytecode / Binary / CFG / ForwardingTable / Trace / Telemetry / Topology / FormalModel)
8. **R** = eBPF/Runtime Relevance (Direct / Partial / Conceptual / None)

---

*This taxonomy document will be cross-referenced in the main paper catalog. See Part 2 for paper extractions.*
