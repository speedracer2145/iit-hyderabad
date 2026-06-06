# eNetSTL: Towards an In-Kernel Library for High-Performance eBPF-based Network Functions

**Authors:** Bin Yang, Dian Shen, Junxue Zhang, Hanlin Yang, Lunqi Zhao, Beilun Wang, Guyue Liu, Kai Chen  
**Year:** 2025 | **Venue:** EuroSys '25 (Twentieth European Conference on Computer Systems, Rotterdam, Netherlands)  
**DOI/Link:** https://doi.org/10.1145/3689031.3696094  
**Code:** https://github.com/chonepieceyb/eNetSTL

---

## 1. Overview

eBPF has become the dominant mechanism for deploying in-kernel network functions (NFs) in production systems. It allows user-written code to run safely inside the Linux kernel without modifying kernel source, providing both performance and programmability. However, as NF logic grows more complex — covering use cases like load balancing, packet classification, counting, and sketching — developers repeatedly run into a hard wall: the eBPF execution environment is too restricted to implement the high-performance data structures and algorithms that modern NFs require. Pure eBPF programs suffer from verifier rejections, loop limitations, lack of SIMD access, and inability to safely manage non-contiguous memory. The result is either incomplete NF functionality or severe performance degradation (up to 49.2% compared to equivalent native kernel implementations).

eNetSTL proposes a systematic answer: the first *in-kernel standard template library* for eBPF-based network functions. Drawing conceptual inspiration from C++'s STL, eNetSTL provides a curated set of reusable, performance-critical building blocks — a memory wrapper, three algorithms, and two data structures — as a kernel module that exposes its interface to eBPF programs via the `kfunc`/`kptr` mechanism. NF developers write eBPF programs in C (or Rust) that call into eNetSTL's APIs, while the library itself is compiled in Rust and lives in a loadable kernel module (`eNetSTL.ko`).

The key insight of the paper is that many disparate NFs — load balancers, firewalls, sketch-based traffic monitors, Cuckoo filters, etc. — share a small set of underlying computational patterns: parallel hashing, SIMD key comparison, bitwise count operations, and non-linear memory access. By abstracting these into a stable, composable library, eNetSTL lets NF developers reuse well-tested, optimized code and sidestep verifier complexity. Experimentally, eNetSTL-enabled NFs achieve **14.6%–75.4% higher packet processing rates** compared to pure eBPF implementations, and come within **3.42% (average)** of native kernel performance — closing almost the entire performance gap between eBPF and kernel-native code.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

eNetSTL's architecture rests on two complementary pillars: (1) a **kernel module library** that implements high-performance primitives using hardware features inaccessible to eBPF, and (2) a **metadata-assisted eBPF verifier** interface that allows these primitives to be safely called from eBPF programs without requiring kernel source modifications.

The library is implemented in **Rust**, which provides memory safety guarantees at the kernel module level. The kfunc mechanism (introduced in Linux 5.13+) allows kernel functions to be explicitly exported to the eBPF verifier's whitelist, enabling eBPF programs to call into them. kptrs allow eBPF programs to hold typed pointers to kernel-allocated objects. Together, these mechanisms let eNetSTL expose powerful, performance-critical operations to eBPF programs while keeping the safety invariants the verifier requires.

The "metadata-assisted verifier" approach is the safety bridge. When eNetSTL kfuncs are declared, they carry verifier-readable annotations (`__sz`, `__szk`, `__k`, `__nullable`, `__uninit`) that convey semantic information:
- `__sz` / `__szk`: Associates a memory pointer argument with a corresponding size argument so the verifier can enforce bounds.
- `__k`: Marks scalar arguments that must be compile-time constants (e.g., BTF type IDs for type-safe allocation).
- `__nullable`: Signals that a pointer may be null, requiring null-check before dereference in the eBPF program.
- `__uninit`: Tells the verifier that the kfunc initializes the pointed-to memory, suppressing uninitialized-read errors.

These annotations allow the eBPF verifier to reason about the safety of cross-boundary calls into eNetSTL without understanding the full implementation of each function — a minimal, non-intrusive extension to the existing verifier infrastructure.

### 2.2 Process Steps

1. **Survey and characterization**: The authors survey seven categories of eBPF-based NFs (key-value query, membership test, packet classification, load balancing, counting, sketching, queuing) and identify which performance bottlenecks are caused by eBPF limitations vs. algorithm complexity.

2. **Abstraction identification**: Common cross-cutting performance patterns are isolated: non-contiguous memory management, SIMD-based parallel hashing, parallel key comparison, bitwise population count, hardware CRC hashing, and random number generation. These form the eNetSTL API surface.

3. **Library design (kernel module, Rust)**: eNetSTL is implemented as a loadable Linux kernel module (`eNetSTL.ko`) written in Rust. It exposes exactly one **memory wrapper**, three **algorithms**, and two **data structures** as kfunc-callable APIs.

4. **Metadata annotation for verifier**: Each kfunc is annotated with verifier hints (type annotations, size annotations, nullability flags). The kptr type system is used for objects that eBPF programs reference across calls.

5. **NF program development**: NF developers write eBPF programs (using Clang/libbpf or the Coolbpf SDK) that include `coolbpf.h` and call eNetSTL APIs. At load time, the eBPF verifier validates each program, using the metadata to accept calls that would otherwise fail safety checks.

6. **Module load + eBPF program load**: At deployment, `eNetSTL.ko` is loaded first, registering its kfuncs with the kernel. The eBPF program is then loaded; the verifier checks all kfunc calls against the registered function signatures and annotations.

7. **Runtime execution**: During packet processing (e.g., XDP or TC hook), the eBPF program calls eNetSTL kfuncs, which execute in the kernel module context with access to SIMD instructions, unrestricted loops, and kernel allocators.

8. **Evaluation**: The authors implement 7+ NF categories using eNetSTL and compare against (a) pure eBPF implementations and (b) native kernel implementations, measuring packet processing rates (Mpps).

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in This Paper |
|---|---|
| **eBPF / Linux kernel** | Execution substrate; eBPF programs attach to XDP/TC hooks for packet processing |
| **kfunc mechanism** | Allows kernel functions to be explicitly whitelisted for eBPF calls, with type-level verification |
| **kptr (kernel pointer)** | Typed pointer to kernel-allocated objects; enables eBPF programs to hold references to eNetSTL data structures across packet events |
| **Rust (for kernel modules)** | Implementation language for eNetSTL; provides memory safety at the kernel module level without GC overhead |
| **eBPF verifier (extended with metadata)** | Static analysis engine that validates eNetSTL API calls using kfunc annotations; enforces memory bounds, type safety, null-safety |
| **SIMD / Intel AVX / SSE** | Used within kfuncs for parallel key hashing and parallel key comparison (not accessible in native eBPF) |
| **POPCNT instruction** | Hardware population-count instruction used in membership tests (Bloom filters, Cuckoo filters) |
| **CRC32 hardware hash** | Hardware-accelerated CRC used in hashing algorithms for load balancing and classification |
| **BTF (BPF Type Format)** | Type system used to associate kfunc arguments with kernel types, enabling type-safe kptr operations |
| **Coolbpf / OpenAnolis** | Production eBPF toolchain into which eNetSTL is integrated; provides CO-RE compilation and kernel backporting |
| **libbpf / Clang** | Standard eBPF program compilation and loading toolchain used for NF programs that call eNetSTL |

### 2.4 Key Data Structures / Models

**eNetSTL provides the following library primitives:**

- **Memory Wrapper (`eNetSTL_mem`)**: Manages non-contiguous kernel memory allocations. Standard eBPF programs can only work with contiguous stack/map memory; eNetSTL's memory wrapper wraps kernel-allocated pages and exposes them to eBPF via kptr, enabling large data structures (e.g., hash tables with many buckets) that would overflow eBPF's 512-byte stack limit.

- **Algorithm 1 — Bitwise Operations**: Hardware-assisted bitwise operations (e.g., POPCNT-based popcount) unavailable in vanilla eBPF. Used in membership test NFs (Bloom filters, Cuckoo filters).

- **Algorithm 2 — SIMD Parallel Hashing**: Uses Intel SIMD instructions to compute multiple hash values simultaneously. Critical for consistent hashing in load balancers and multi-hash membership structures.

- **Algorithm 3 — SIMD Parallel Key Comparison**: Compares multiple keys simultaneously using SIMD vector registers. Used in Cuckoo-style hash tables (CuckooSwitch, Cuckoo filter) where multiple candidate buckets must be probed per lookup.

- **Data Structure 1 — List Bucket (`eNetSTL_list_bucket`)**: A kernel-managed linked list of hash buckets, enabling hash tables with variable-length chains that exceed eBPF's memory model. Exposed via kptr; eBPF programs reference and traverse buckets through kfunc calls.

- **Data Structure 2 — Random Number Pool (`eNetSTL_rng_pool`)**: A pre-seeded pool of random values for load balancing algorithms (e.g., power-of-two-choices) that require true randomness rather than eBPF's limited pseudo-random helpers.

**Internal model**: The system relies on the **BTF type graph** to propagate type information from the Rust kernel module to the eBPF verifier. Each kptr type is registered with its BTF descriptor so the verifier can enforce correct usage (e.g., a `list_bucket*` kptr cannot be passed where a `rng_pool*` is expected).

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

eNetSTL targets **eBPF-based in-kernel network functions** across seven functional categories:

1. **Key-Value Query**: NFs that perform packet-keyed lookups (e.g., flow tables). Representative implementation: CuckooSwitch (a Cuckoo-hashing based forwarding table).
2. **Membership Test**: NFs that test set membership (e.g., ACLs, blocklists). Representative: Bloom filter, Cuckoo filter.
3. **Packet Classification**: Multi-field rule matching for firewall/policy enforcement. Representative: tuple-space search classifiers.
4. **Load Balancing**: Per-flow or per-packet traffic distribution (e.g., ECMP, consistent hashing). Representative: Katran (Meta's production load balancer).
5. **Counting**: Per-flow or per-packet traffic counting. Representative: exact flow counters.
6. **Sketching**: Approximate traffic measurement (e.g., heavy hitter detection). Representative: Count-Min Sketch, Nitro Sketch.
7. **Queuing**: Per-flow queuing and scheduling. Representative: token bucket rate limiters (RakeLimit).

Real-world projects evaluated: **PolyCube**, **Katran**, **RakeLimit**, and open-source sketch implementations.

### 3.2 How It Validates NF Behavior

eNetSTL does not perform behavioral verification of NF *logic* in the formal sense. Instead, it validates NF *safety* at the API boundary using the eBPF verifier augmented with kfunc metadata annotations. Specifically:

1. **Pre-load static analysis (eBPF verifier)**: When an NF program using eNetSTL APIs is loaded, the standard eBPF verifier performs symbolic execution across all paths. For each kfunc call site, the verifier consults the annotation metadata to verify:
   - That pointer arguments are non-null (unless `__nullable`).
   - That memory regions are within declared bounds (using `__sz` size annotations).
   - That kptr-typed arguments match the declared BTF type.
   - That constant-required arguments (`__k`) are compile-time known.

2. **Type-safe kptr lifetime tracking**: The verifier tracks kptr acquisition (from kfunc allocators) and release (via kfunc free calls), ensuring no use-after-free or double-free at the eBPF program level.

3. **Functional correctness of library**: The library implementations themselves are validated by correctness testing across all seven NF categories, comparing output against reference implementations.

4. **Integration testing**: Real NF programs (PolyCube, Katran, RakeLimit, sketches) are run under traffic workloads and their packet processing rates and functional correctness (e.g., load distribution uniformity, sketch error rates) are measured.

### 3.3 What Properties / Invariants Does It Prove?

| Property | Mechanism |
|---|---|
| **Memory safety of kfunc calls** | eBPF verifier + `__sz`/`__szk` size annotations; verifier enforces bounds before each kfunc call |
| **Null-pointer safety** | `__nullable` annotation forces null-check in eBPF program before kfunc call; verifier rejects programs without the check |
| **Type safety of kernel pointer usage** | kptr + BTF type system; verifier rejects mistyped pointer dereferences or wrong-type kfunc arguments |
| **No eBPF stack overflow** | Memory wrapper externalizes large allocations to kernel heap; programs remain within 512-byte eBPF stack limit |
| **Constant argument safety** | `__k` annotation ensures BTF IDs and size constants are resolved at verify-time, not runtime |
| **Correctness of library algorithms** | Functional testing: algorithm outputs validated against reference implementations for all NF categories |
| **Performance correctness** | Throughput benchmarks confirm 14.6%–75.4% improvement over pure eBPF; within 8.54% of native kernel |

**What is NOT proven**: eNetSTL does not prove NF-level behavioral properties such as correctness of packet forwarding logic, absence of forwarding loops, stateful protocol compliance, or end-to-end safety of the NF as a whole. It proves API-level safety at the library boundary.

### 3.4 Input Requirements

| Input | Description |
|---|---|
| **eBPF source program (C or Rust)** | The NF program written using the eNetSTL API; must include `coolbpf.h` or equivalent header |
| **`eNetSTL.ko` kernel module** | Must be loaded prior to eBPF program loading; registers kfuncs with the verifier |
| **Standard Linux kernel (5.13+)** | Required for kfunc support; eNetSTL targets recent kernels (or uses Coolbpf backporting for older kernels) |
| **libbpf + Clang toolchain** | For compiling and loading the eBPF NF program |
| **No formal specifications required** | No user-supplied contracts, invariants, or annotations are needed beyond correct API usage |

The user/NF developer does not need to annotate their program with any safety specifications. They simply call eNetSTL APIs and the verifier enforces safety automatically via the pre-registered kfunc metadata.

### 3.5 Guarantees Provided

- **Sound safety guarantee at the API boundary**: Any eBPF program that passes the verifier with eNetSTL kfuncs is guaranteed to: (1) not access out-of-bounds memory through eNetSTL APIs, (2) use kptrs only with the correct types, (3) not pass null where non-null is required.
- **No kernel modification required**: The safety guarantees hold without patching kernel source, only requiring a loadable module and standard verifier annotations.
- **Functional performance guarantee**: Empirical guarantees only — eNetSTL achieves 14.6–75.4% throughput improvement over pure eBPF; within 3.42% average of native kernel implementations.
- **No crash guarantee for kernel module**: The Rust-based kernel module provides compile-time memory safety, reducing risk of kernel panics from eNetSTL itself.
- **No formal proof**: The guarantees are enforcement-based (verifier-checked) and empirical, not formally proven with a theorem prover.

---

## 4. NF Chain Verification

eNetSTL is **single-NF only** in scope. The paper does not address NF service chains (sequences of NFs processing the same packet), chain-level composition properties, inter-NF state sharing, or ordering invariants.

**What the paper does**: It provides a library that makes individual eBPF NFs faster and safer. Each NF is an independent eBPF program attached to an XDP or TC hook. There is no model of how multiple eNetSTL-using NFs interact when chained.

**What would be needed to extend to chains**: 
- A composition model for eBPF NF chains (e.g., how XDP programs hand off to TC programs, or how TC programs chain via `bpf_tail_call`).
- A shared state model specifying how BPF maps modified by one NF are read by the next, and what invariants must hold at each handoff point.
- Chain-level properties such as: all packets accepted by NF-A are correctly classified by NF-B; stateful NAT rewriting by NF-A is undone by NF-C; etc.
- eNetSTL's kptr model is per-program (kptrs are local to a single eBPF program call and cannot be passed across programs via maps), so inter-NF state sharing would require extension to map-based object sharing.

In summary, eNetSTL is fundamentally a **within-NF performance and safety library**, not a chain-level verification or composition framework.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs **static control-flow and network-condition (CFG-NC) dataflow analysis directly on raw eBPF bytecode** — no source code required. It pairs this with a **Prolog-based query engine** that allows researchers and operators to assert behavioral properties about NF programs: packet processing policies, map access patterns, stateful behavior, and inter-program dependencies. It handles **eBPF NF chains** and can reason about **stateful BPF maps** across program boundaries without any developer annotation.

### 5.2 Key Differences from This Paper

| Dimension | eNetSTL | Yaksha-Prashna |
|---|---|---|
| **Primary goal** | Performance + library-level safety | Behavioral verification + policy assertion |
| **Input** | eBPF C source using eNetSTL API | Raw eBPF bytecode (no source needed) |
| **Technique** | kfunc/kptr metadata + eBPF verifier static analysis | CFG-NC dataflow analysis + Prolog query engine |
| **Safety scope** | API boundary: memory bounds, type safety, null-safety | Program behavior: packet paths, map access semantics, chain-level properties |
| **NF chain support** | None — single NF programs only | Yes — multi-NF chains, inter-map dependencies |
| **Stateful reasoning** | Partial — kptr lifetime tracking within one program | Full — BPF map read/write semantics across chain |
| **Formal guarantees** | Verifier-enforced (sound for safety properties at API boundary) | Dataflow + Prolog-derived (sound for asserted behavioral properties) |
| **Performance improvement** | Yes (14.6–75.4% over pure eBPF) | Not a performance tool |
| **Deployment requirement** | Requires specific library + kernel module | Works on any compiled eBPF binary |
| **Developer burden** | Must rewrite NF using eNetSTL API | Zero — post-deployment / binary analysis |

### 5.3 How This Paper Is Useful For Us

1. **Motivational contrast**: eNetSTL demonstrates that even the best-engineered eBPF library can only provide API-level safety (bounds, types, null) at the individual program level. It explicitly does not address *behavioral* correctness of NF logic, chain composition, or stateful policy enforcement — all of which Yaksha-Prashna targets. We can cite eNetSTL to establish that the state of the art in eBPF safety tools focuses on low-level memory safety, and argue that higher-level behavioral verification is still an open problem.

2. **Baseline for "what eBPF verifier knows"**: eNetSTL carefully documents what the standard eBPF verifier can and cannot check (kfunc annotation boundaries, kptr lifetimes, bounds). Yaksha-Prashna operates *above* this layer on compiled bytecode, so understanding the verifier's limits helps frame what our analysis adds that the verifier cannot provide.

3. **NF taxonomy**: The seven NF categories (key-value query, membership test, classification, load balancing, counting, sketching, queuing) are an excellent, empirically grounded reference taxonomy. We should adopt or reference this categorization when describing the range of NFs that Yaksha-Prashna can handle.

4. **Real-world NF benchmarks**: eNetSTL evaluates Katran, PolyCube, RakeLimit, and open-source sketches. These are legitimate, publicly available eBPF NF programs that Yaksha-Prashna could use as evaluation targets. Showing that our tool can analyze programs from this ecosystem (even when written without eNetSTL) would strengthen our evaluation.

5. **"What we don't require"**: eNetSTL requires developers to use a specific API, load a kernel module, and use a specific toolchain. Yaksha-Prashna requires none of this — it operates on the compiled binary. This is a compelling ease-of-use and universality argument.

### 5.4 Positioning Statement

"While eNetSTL advances eBPF NF safety by providing a metadata-annotated in-kernel library that enforces memory and type safety at the API boundary, it is constrained to individual NF programs, requires source-level adoption of its API, and cannot reason about behavioral correctness of NF logic, stateful policy semantics, or multi-NF chain composition. Yaksha-Prashna addresses this orthogonal and deeper verification need by performing CFG-NC dataflow analysis on raw eBPF bytecode and enabling Prolog-based behavioral assertion queries that span full NF chains and stateful map accesses — without any source code, API adoption, or kernel module dependency."

---

## References and Further Reading

- Bin Yang, Dian Shen, Junxue Zhang, Hanlin Yang, Lunqi Zhao, Beilun Wang, Guyue Liu, Kai Chen. *eNetSTL: Towards an In-kernel Library for High-Performance eBPF-based Network Functions.* EuroSys '25. ACM. DOI: 10.1145/3689031.3696094.
- Source code: https://github.com/chonepieceyb/eNetSTL
- Coolbpf / OpenAnolis integration: eNetSTL is shipped as part of the Coolbpf eBPF development platform.
- Related: Katran (Meta's eBPF load balancer), PolyCube (eBPF NF framework), RakeLimit (eBPF rate limiter), Nitro Sketch / Count-Min Sketch (sketch-based traffic measurement).
