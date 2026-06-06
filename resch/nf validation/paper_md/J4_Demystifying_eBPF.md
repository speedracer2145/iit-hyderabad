# Demystifying Performance of eBPF Network Applications

**Authors:** Farbod Shahinfar, Sebastiano Miano, Aurojit Panda, Gianni Antichi  
**Year:** 2025 | **Venue:** ACM CoNEXT (Proceedings of the ACM on Networking, Vol. 3, Issue CoNEXT3, Article 16)  
**DOI/Link:** https://dl.acm.org/doi/10.1145/3700898  
**Artifact:** https://github.com/bpf-endeavor/bpf-app-offload-measurement

---

## 1. Overview

eBPF (extended Berkeley Packet Filter) has become the dominant technology for accelerating network data-plane logic inside the Linux kernel, enabling developers to attach custom packet-processing programs at hooks such as XDP (eXpress Data Path), TC (Traffic Control), and SK_SKB without requiring kernel recompilation or module loading. Systems like Cilium, Katran, and Cloudflare's firewall leverage eBPF to achieve wire-speed performance. Beyond traditional network functions (NFs), eBPF is increasingly being applied to accelerate broader networked applications such as in-kernel key-value caches (BMC/Memcached), custom TCP stacks, and consensus protocol helpers. The implicit assumption driving this adoption is that eBPF offloading necessarily yields better performance.

This paper challenges that assumption through rigorous empirical analysis. Shahinfar et al. conduct a systematic benchmarking study across the full eBPF execution stack — hooks (XDP, TC, SK_SKB), map APIs (Array, Hash, Ring), compilation pipeline (Clang/LLVM JIT), and system-level interference patterns — and construct an explanatory performance model. Their central finding is that eBPF does not uniformly accelerate network applications: the benefit depends critically on how much application logic can be offloaded, the nature of CPU operations required (presence of SIMD, cryptography, floating-point), and whether the application's working set fits the constraints of the eBPF runtime. In certain configurations, eBPF offloading actively hurts performance, both for the accelerated application and for co-located workloads.

The paper's contribution is threefold: (1) a principled characterisation of eBPF runtime overheads across each major hook and API surface; (2) a formal performance model parameterised by the fraction of offloaded requests (ω) and the relative cost of the eBPF program (ρ) that predicts when offloading benefits or harms throughput; and (3) identification of concrete eBPF architectural gaps — JIT code quality, verifier-imposed loop bound-checks, instruction set limitations, and performance isolation failures — that limit applicability to broader classes of networked applications. The work is accompanied by a fully open-source artifact enabling reproduction on Cloudlab infrastructure.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

The paper adopts a **systematic micro- and macro-benchmarking methodology** combined with **analytical performance modelling**. Rather than evaluating a single end-to-end system, the authors decompose the eBPF execution stack into individual cost components and measure each independently, then validate a compositional performance model against observed system behaviour.

The analytical model frames eBPF offloading as a partial computation accelerator. Let:
- **ω** = fraction of requests handled entirely by the eBPF "fast path" (never reaching user space)
- **ρ** = ratio of per-packet cost with the eBPF program active versus without it (overhead factor)

Then eBPF offloading is beneficial only when: **ω > ρ − 1**. If too few requests can be satisfied in the kernel (low ω), or the eBPF program itself is expensive (high ρ), offloading degrades throughput. This model is validated against empirical measurements of BMC (eBPF-based Memcached accelerator) and an AF_XDP-based key-value store (xsk_cache over a Seastar-based server).

The benchmarking framework measures latency at nanosecond resolution using hardware timestamping (Mellanox ConnectX-6 packet timestamping), `bpf_ktime_get_ns()` in eBPF programs, and kernel tracepoints. End-to-end throughput is measured using iperf3 and a custom client workload generator against a back-to-back 100 Gbps testbed.

### 2.2 Process Steps

1. **Testbed configuration:** Two servers connected back-to-back over 100 Gbps Mellanox ConnectX-6 NICs. Intel Xeon Silver 4310 (2.10 GHz, 24 cores), 128 GB DDR4 RAM. Linux kernel 6.8.0-rc7. eBPF programs compiled with Clang 14, targeting eBPF ISA v3 (`-mcpu=v3`). NUMA-aware pinning: all programs run on the NUMA node connected to the NIC.

2. **Hook overhead measurement (Table 1):** A minimal "pass-through" eBPF program (performing no packet modification) is attached to XDP, TC, and SK_SKB hooks in turn. iperf3 traffic is generated and per-packet hook entry/exit latency is logged via `dmesg` timestamps. This characterises the irreducible overhead of entering/exiting each hook regardless of logic complexity.

3. **Time-to-hook measurement:** Timestamps the packet arrival at the NIC driver (XDP) versus when the same packet becomes visible at downstream hooks (TC, SK_SKB). Reveals the latency cost incurred solely by traversing kernel subsystems before application logic executes.

4. **Round-trip latency at each hook (Figure 3):** An echo server (`server_bounce`) reflects packets back at XDP, TC, SK_SKB, or user-space socket level. A custom client measures per-packet RTT over 100 seconds. Compares latency profiles across the stack to quantify the latency benefit of early-path processing.

5. **Map and API overhead benchmarking:** Measures the per-access latency of Array maps, Hash maps, and Ring Buffers under varying key/value sizes and concurrent access patterns. Compares kernel-side eBPF map lookups against equivalent user-space hash table operations as a baseline.

6. **Compilation and verifier overhead:** Inserts synthetic loop-bound-checks (required by the verifier for backward branches) into tight loops and measures the throughput impact. Compares JIT-compiled eBPF code performance against equivalent hardcoded kernel driver code.

7. **System interference measurement:** Co-locates an eBPF-accelerated flow with a non-accelerated flow on shared CPU cores. Measures degradation of the non-accelerated flow's latency and throughput as the eBPF program's computational cost increases — quantifying "noisy neighbour" effects.

8. **Packet trimming benefit:** Measures the throughput improvement from reducing packet payload size via `bpf_xdp_adjust_tail` before passing to user space via AF_XDP, illustrating the cost of DMA and cache misses from large payloads.

9. **Macro-case study validation:** Applies the performance model to BMC (eBPF cache for Memcached over UDP/XDP) and xsk_cache (AF_XDP-based key-value store on Seastar). Varies ω by adjusting cache hit rate and measures whether observed throughput matches model predictions.

10. **Gap analysis and recommendations:** Synthesises findings into architectural recommendations: improved JIT register allocation, SIMD/crypto instruction support in eBPF ISA, verifier-side performance annotations, and OS-level performance isolation primitives for eBPF tenants.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in This Paper |
|---|---|
| **eBPF / Linux BPF subsystem** | The primary runtime under study; programs attached at XDP, TC (tc_cls_act), and SK_SKB hooks |
| **libbpf** | User-space library for loading, verifying, and attaching eBPF programs; standard for modern eBPF development |
| **Clang/LLVM (v14, -mcpu=v3)** | Compiles C-subset eBPF programs to eBPF bytecode; the paper analyses JIT code quality output |
| **eBPF JIT compiler (x86-64 Linux)** | Translates eBPF bytecode to native instructions at load time; analysed for instruction quality vs. native code |
| **eBPF Verifier** | Kernel static analyser enforcing safety (type safety, bounded loops, no null dereferences); studied for loop bound-check overhead |
| **XDP (eXpress Data Path)** | Earliest-path hook at NIC driver layer, before SKB allocation; provides lowest-latency processing |
| **TC (Traffic Control / cls_act)** | Hook after SKB allocation; supports ingress and egress; higher overhead than XDP but richer context |
| **SK_SKB** | Socket-level hook for stream parsing and verdict; highest latency but enables per-socket eBPF programs |
| **AF_XDP** | Zero-copy kernel-bypass socket; eBPF program at XDP steers matched packets to user space via XSK rings |
| **eBPF Maps (Array, Hash, Ring Buffer)** | In-kernel data structures for state storage and kernel-to-user communication; overhead measured per type |
| **iperf3** | Generates high-speed TCP/UDP traffic for hook overhead and throughput benchmarks |
| **Mellanox ConnectX-6 hardware timestamping** | NIC-level packet timestamps used for sub-microsecond latency measurement accuracy |
| **BMC (BPF Memcached Cache)** | Real-world eBPF application serving as case study: in-kernel UDP/XDP cache for Memcached GET requests |
| **xsk_cache / Seastar** | AF_XDP-based user-space key-value store; case study contrasting pure kernel-bypass performance vs. eBPF offloading |
| **ω/ρ Analytical Performance Model** | Mathematical framework derived to predict when eBPF offloading is net-positive; parameterised from empirical measurements |

### 2.4 Key Data Structures / Models

- **eBPF Array Map:** Fixed-size, integer-indexed kernel array; O(1) lookup; used for per-CPU counters and small lookup tables in NFs. Overhead: approximately 20–50 ns per access depending on value size.
- **eBPF Hash Map:** Kernel hash table with per-bucket locking; used for connection tracking and flow tables. Higher latency (~100–200 ns under load) due to hashing and lock contention.
- **eBPF Ring Buffer:** Single-producer, single-consumer ring for kernel→user data export; latency dominated by memory bandwidth and cache line sharing across NUMA boundaries.
- **XSK (XDP Socket) Descriptor Rings:** Shared-memory SPSC rings (FILL, COMPLETION, RX, TX) used by AF_XDP for zero-copy packet steering to user space. Packet trimming via `bpf_xdp_adjust_tail` reduces DMA footprint before enqueue.
- **ω/ρ Performance Model:** A two-parameter analytical model. ω is the hit rate on the eBPF fast path (fraction of packets fully served in-kernel). ρ is the overhead multiplier of the eBPF program (measured cycle cost with program vs. without). Net throughput ratio with eBPF: `T_ebpf / T_baseline = 1 / (ρ − ω)` when ω < 1. Offloading is net-beneficial only when ω > ρ − 1.
- **SKB (Socket Buffer):** Kernel per-packet metadata structure allocated at TC and SK_SKB hooks but absent at XDP; its allocation cost is a measurable component of hook-entry overhead analysis.
- **eBPF Program CFG (implicit):** The verifier traverses the program's control flow graph; the paper discusses how verifier-inserted bound-checks in loop back-edges add redundant instructions to JIT-compiled output, degrading throughput by 5–15% in tight loops.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

The paper evaluates eBPF-based NFs across two categories:

**Traditional NF types (studied as context and benchmark baseline):**
- **L4 Firewall / Packet Filter:** Stateless ACL implemented at XDP; classifies packets against a rule table stored in an Array or Hash map and drops non-matching packets.
- **Network Address Translator (NAT):** Stateful DNAT/SNAT implemented at XDP/TC; uses eBPF Hash maps for connection state (5-tuple → translated endpoint).
- **Layer-4 Load Balancer:** XDP-based L4 load balancer performing consistent-hash-based ECMP; maintains backend state in per-CPU Array maps.

**Broader networked applications (primary experimental targets):**
- **In-kernel Key-Value Cache (BMC):** eBPF program at XDP hook intercepting Memcached GET requests over UDP; serves cache hits directly from eBPF maps without reaching user-space Memcached.
- **AF_XDP Key-Value Store (xsk_cache):** User-space key-value server built on the Seastar framework using AF_XDP sockets; case study for where eBPF partial offloading offers diminishing returns.
- **Generic echo server:** Used as a benchmarking primitive across all hooks (XDP, TC, SK_SKB, user-socket) to measure pure hook overhead without application semantics.

### 3.2 How It Validates NF Behavior

This paper does **not** perform functional correctness validation of NF behaviour (i.e., it does not check whether packets are forwarded, dropped, or translated correctly per policy). Instead, it performs **performance validation**: verifying that the observed runtime performance of eBPF NFs matches what the analytical performance model predicts, and characterising deviations.

The empirical performance validation procedure is:

1. **Establish baseline:** Measure throughput and latency of the workload with no eBPF program attached (pure kernel stack or pure user space).
2. **Attach eBPF program:** Load the NF's eBPF component at the target hook using libbpf.
3. **Apply traffic workload:** Generate realistic traffic (varying packet sizes: 64B, 128B, 1500B MTU; varying request rates; varying flow counts) using iperf3 and a custom load generator.
4. **Measure performance metrics:** Collect throughput (Mpps or Gbps), per-packet latency (ns, median and interquartile range), CPU utilisation per core.
5. **Compare against model:** Use the ω/ρ model to predict expected performance; compare predicted vs. measured values to determine whether the model explains observed behaviour or whether additional effects (cache misses, JIT suboptimality, interference) account for deviations.
6. **Identify isolation violations:** Co-locate an eBPF NF with a benign unrelated workload on shared CPU cores and measure degradation suffered by the unrelated workload.

### 3.3 What Properties / Invariants Does It Prove?

The paper empirically validates the following **performance properties and invariants** (not formal proofs):

- **Hook overhead bounds:** Measured median latency bounds for entering/exiting XDP, TC, and SK_SKB hooks on the testbed hardware, establishing baseline costs for any eBPF NF.
- **Throughput monotonicity violation:** Demonstrates that adding an eBPF program at XDP can reduce aggregate system throughput for traffic that does not interact with the program — a performance isolation violation property.
- **JIT code quality regression:** Shows Clang/LLVM generates sub-optimal eBPF bytecode for bulk memory operations, and the x86-64 JIT backend fails to exploit SIMD instructions, making eBPF implementations 2–5× slower than equivalent user-space code for the same operation.
- **Verifier overhead quantification:** Loop bound-checks inserted by the verifier in tight packet-processing loops reduce throughput by 5–15%, constituting unnecessary overhead for programs whose loops are already verified safe.
- **Map access cost model:** Validates that eBPF Hash map lookup cost is approximately 3–4× higher than Array map lookup at scale, and that Ring Buffer write cost is dominated by cache-line invalidation across NUMA boundaries.
- **Offloading benefit condition:** Empirically validates that the ω/ρ model correctly predicts the crossover point at which eBPF offloading becomes harmful, with model predictions matching measurements within ~10% error on the BMC case study.

### 3.4 Input Requirements

The benchmarking framework requires:

- **eBPF program binary (.o file):** Compiled eBPF object files (via Clang) for the NF under test; loaded via libbpf. Source code is required to compile, though the benchmark runtime only loads the compiled binary.
- **Traffic workload specification:** iperf3 parameters (packet size, rate, duration, protocol) or custom client parameters for the echo and KV-store workloads.
- **Hook and attachment configuration:** CLI arguments specifying which hook (XDP, TC, SK_SKB) to attach to and the target network interface.
- **Hardware testbed:** Two servers with 100 Gbps NICs connected back-to-back; results are hardware-dependent (documented for Mellanox ConnectX-6 and Intel Xeon Silver 4310).
- **Kernel version:** Linux 6.8.0-rc7; some measurements depend on kernel-internal timestamping APIs present in this specific version.

The operator does **not** need to provide NF specifications, formal models, or annotations. The framework is pure black-box empirical performance measurement.

### 3.5 Guarantees Provided

The paper provides **empirical measurement-based characterisations**, not formal proofs:

- **Performance bounds:** Measured median latency and IQR for hook overheads on specific hardware/kernel, reproducible via the open artifact.
- **Model predictive validity:** The ω/ρ model predicts performance within ~10% on studied workloads; generality to arbitrary eBPF programs is not formally proven.
- **Full reproducibility:** Complete artifact with scripts, patches, and a step-by-step ARTIFACT.md for reproducing all figures and tables on Cloudlab infrastructure (sm110p machines, Ubuntu 22).
- **No functional correctness guarantees:** The paper explicitly does not check whether NFs forward or drop packets correctly per policy; it measures only performance.

---

## 4. NF Chain Verification

This paper does **not** address NF chains (service function chaining / sequenced composition of multiple NFs). All experiments study individual eBPF programs attached at a single hook on a single NIC interface, processing traffic in isolation. There is no multi-NF composition, no ordering semantics analysis, and no chain-level property checking.

**What the paper does address regarding multi-program composition:**  
The artifact benchmarks do investigate the overhead of "chaining" multiple eBPF programs at the same hook using the `bpf_tail_call` mechanism, which is commonly used to work around the eBPF program complexity limit (maximum ~1M verified instructions). However, this is studied exclusively as an eBPF runtime cost issue — the additional latency incurred by each tail-call invocation (~50–100 ns per call) — not as a correctness or semantic composition problem. The paper does not reason about whether a sequence of tail-called programs implements a correct NF policy, nor does it handle map-based state sharing between compositionally chained programs.

**What would be needed to extend to NF chains:**  
1. A compositional semantics capturing how eBPF programs interact via shared maps, tail calls, BPF-to-BPF function calls, and TC classifier chains.
2. End-to-end performance models accounting for additive hook latencies, map contention across co-resident programs, and inter-program state dependencies.
3. Functional correctness analysis of whether the composition implements the desired chain policy (e.g., firewall before NAT before load balancer) — entirely outside the scope of this work.

**Conclusion:** This paper is **single-NF / single-hook scope only**. It provides no mechanism for chain-level reasoning.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna is a static analysis and query framework that operates directly on **raw eBPF bytecode** — no source code required. It constructs a **Control Flow Graph with Network Context (CFG-NC)** that captures packet field accesses, BPF map read/write operations, and helper call semantics across the program's CFG. A **Prolog-based query engine** then evaluates behavioural assertions against this representation (e.g., "does this program always drop packets matching predicate P?", "does this NF ever modify the destination port?"). Yaksha-Prashna handles **eBPF NF chains** by composing CFG-NC representations across multiple programs and reasoning about stateful carry-over through BPF map dependency analysis. It operates at **static load-time** — no live traffic, no hardware testbed required.

### 5.2 Key Differences from This Paper

| Dimension | Demystifying eBPF (J4) | Yaksha-Prashna |
|---|---|---|
| **Primary goal** | Performance measurement and modelling | Functional correctness and behavioural verification |
| **Analysis phase** | Runtime (dynamic benchmarking, live traffic) | Static (load-time, pre-deployment) |
| **Input required** | eBPF binary + live traffic + testbed hardware | eBPF bytecode only; no traffic, no hardware needed |
| **Technique** | Empirical benchmarking + ω/ρ analytical model | CFG-NC dataflow analysis + Prolog query engine |
| **Properties checked** | Throughput, latency, CPU utilisation, performance isolation | Packet dropping, field mutation, map access patterns, policy compliance |
| **NF chain support** | No (single-hook, single-NF only) | Yes (multi-program composition via CFG-NC) |
| **Stateful analysis** | Observes state effects via map access overhead measurement | Statically models BPF map read/write dataflow dependencies |
| **Guarantees** | Empirical measurements with reproducible artifact | Static soundness guarantees over verified bytecode CFG |
| **Source code required** | Yes (to compile eBPF programs for testbed) | No (operates on already-compiled bytecode) |

### 5.3 How This Paper Is Useful For Us

**1. Motivates the problem space authoritatively.** This paper is a top-tier CoNEXT 2025 confirmation that eBPF-based NFs are widely deployed, operationally critical, and exhibit complex behaviours that are not fully understood — even by their authors. We can cite it to establish why rigorous static analysis of eBPF NFs is necessary and timely.

**2. Highlights performance isolation violations as a correctness-adjacent concern.** The paper empirically shows that eBPF programs can be "noisy neighbours," violating performance isolation for co-located workloads. This is a behavioural property currently undetectable by any static tool. Yaksha-Prashna could be cited as a complementary static approach capable of reasoning about shared-resource access patterns (e.g., per-CPU vs. global maps) before deployment, using this paper's empirical harm evidence as motivation.

**3. Precisely documents the eBPF hook ecosystem.** The characterisation of XDP, TC, and SK_SKB hooks — entry/exit costs, context available, ordering constraints — provides accurate technical grounding for Yaksha-Prashna's CFG-NC construction. The hook latency data also informs which hooks are the most operationally critical analysis targets (XDP and TC).

**4. Provides realistic NF case studies for evaluation.** The BMC and xsk_cache applications (with full source code in the open artifact at `bpf-endeavor/bpf-app-offload-measurement`) are realistic, non-trivial eBPF programs that could serve as ground-truth test cases for Yaksha-Prashna. The ARTIFACT.md guide enables us to compile and deploy these NFs for our own analysis experiments.

**5. Identifies JIT and verifier as correctness-opaque layers.** The paper notes the eBPF verifier focuses on safety only (not performance isolation or functional policy) and that the JIT produces semantically equivalent but suboptimal code. Yaksha-Prashna operates at the verified bytecode level — the correct level to reason about NF behaviour as the kernel's verifier sees it — and can be positioned as addressing the gap: they measure what the program *does* at runtime; we statically determine what it *must* do by construction.

**6. Provides overhead budget reference numbers.** If Yaksha-Prashna introduces any load-time instrumentation overhead, this paper's measurements of XDP/TC hook entry costs (tens to hundreds of nanoseconds) establish the performance budget that any verification overhead must respect to remain non-intrusive.

### 5.4 Positioning Statement

While Shahinfar et al. rigorously characterise *when* eBPF network applications deliver performance benefits through systematic empirical benchmarking and an analytical performance model, their framework is purely dynamic — requiring live traffic, testbed hardware, and providing no mechanism to detect functional policy violations before deployment. Yaksha-Prashna complements this by performing static behavioural verification directly on eBPF bytecode at load time, enabling operators to assert correctness properties of individual NFs and composed NF chains without executing a single packet — addressing the functional guarantees dimension that runtime benchmarking, by design, cannot provide.

---

## References / Further Reading

- **BMC:** Ghigoff et al., "BMC: Accelerating Memcached using Safe In-kernel Caching and Pre-stack Processing," NSDI 2021.
- **Morpheus:** Miano et al., "Morpheus: Bringing Data Plane Abstraction for Software Data Planes," 2024 (JIT optimisation companion work from the same research group).
- **AF_XDP Performance:** Molè et al., "Performance Implications at the Intersection of AF_XDP and Programmable NICs," 2025 (companion paper from same group).
- **AF_XDP:** Björn Töpel and Magnus Karlsson, "AF_XDP," Linux Kernel Documentation, 2018.
- **eBPF ISA v3:** Linux BPF instruction set architecture specification, supporting 32-bit sub-registers, bounded loops, and extended map types.
- **Artifact Repository:** https://github.com/bpf-endeavor/bpf-app-offload-measurement
