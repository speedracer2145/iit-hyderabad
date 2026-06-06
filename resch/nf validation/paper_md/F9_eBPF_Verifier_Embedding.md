# Validating the eBPF Verifier via State Embedding

**Authors:** Hao Sun, Zhendong Su (ETH Zurich)  
**Year:** 2024 | **Venue:** OSDI '24 (18th USENIX Symposium on Operating Systems Design and Implementation)  
**DOI/Link:** https://www.usenix.org/conference/osdi24/presentation/sun-hao  
**PDF:** https://www.usenix.org/system/files/osdi24-sun-hao.pdf

---

## 1. Overview

The Linux eBPF (extended Berkeley Packet Filter) subsystem has become a cornerstone of modern kernel extensibility, enabling user-space programs to inject sandboxed code into the kernel for networking, observability, and security enforcement tasks. The eBPF *verifier* is the critical gatekeeper that statically analyzes every eBPF program before it is allowed to execute in kernel space, ensuring that no unsafe memory accesses, infinite loops, or privilege violations can occur. However, the verifier is a highly complex piece of software — spanning tens of thousands of lines of kernel code — and its own internal logic is susceptible to subtle bugs. A bug in the verifier's reasoning can cause it to wrongly certify an *unsafe* program as safe (a *soundness violation* or *unsoundness*), creating a direct kernel security vulnerability exploitable for privilege escalation.

This paper introduces **state embedding**, a novel validation technique specifically designed to detect logic bugs in the eBPF verifier itself. The key insight is elegant: rather than trying to formally prove the verifier correct or to fuzz the kernel blindly, the authors embed *known, concrete runtime states* directly into eBPF programs as assertion-like correctness checks. When the verifier analyzes such a state-embedded program and deems it safe, it implicitly claims that its own over-approximated abstract state encompasses those embedded concrete states. If this claim fails — i.e., the concrete state is *not* actually contained within the verifier's tracked approximation — then a logic bug in the verifier has been exposed. The technique is self-referential: it uses the verifier's own abstract interpretation mechanism to validate itself.

The authors realize state embedding in a practical tool called **SEV** (State Embedding Verifier) and apply it to multiple versions of the Linux kernel's eBPF verifier. Within only one month of automated testing, SEV discovered **15 previously unknown logic bugs**, 10 of which were fixed by kernel maintainers prior to publication. Critically, at least 2 of these bugs were confirmed exploitable and could be leveraged for **local privilege escalation** on production Linux systems. This demonstrates that even a heavily scrutinized and battle-tested component like the eBPF verifier harbors serious correctness flaws, and that state embedding is a highly effective methodology for surfacing them.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

The eBPF verifier operates as a **static abstract interpreter**: it traverses the program's control-flow graph (CFG) and, at each instruction, maintains an *abstract state* describing every register's type, value range, and pointer provenance. Because the verifier must handle all possible runtime behaviors without actually executing the program, it uses *over-approximation* — the abstract state at any program point must be a superset of all possible concrete states that can arise at that point in any real execution.

**State embedding** exploits this over-approximation invariant as a testable oracle:

1. **Concrete State Capture:** SEV first takes an eBPF program that the verifier accepts (or generates a synthetic one). It then either *executes* this program to capture concrete register states at key program points (basic block boundaries), or generates concrete states from knowledge of register value domains.

2. **State Embedding:** These captured concrete states (specific register values, pointer types, known bounds) are embedded back into the program as eBPF bytecode constructs that *assert* the concrete values. This is done by inserting instructions that force specific register values or perform conditional checks tied to the concrete state. The resulting program is structured so that — if the verifier's abstract state correctly over-approximates reality — the verifier must accept these embedded assertions as satisfying its invariants.

3. **Contradiction Detection:** SEV submits the state-embedded program to the Linux eBPF verifier. If the verifier *rejects* the state-embedded program, this is a false negative (program was safe but verifier erred — a standard rejection). However, if the verifier *accepts* the program but the embedded concrete state is provably *outside* the verifier's claimed abstract bounds (detectable because the embedded program can trigger a concrete memory violation that the verifier failed to predict), then a soundness bug is exposed. Conversely, the embedded checks can also manifest as programs that the verifier wrongly accepts but that would execute unsafely — detectable via concrete execution.

4. **Bug Classification:** Detected inconsistencies are classified into categories: incorrect scalar value range tracking, unsound pointer type inference, incorrect tnum (tristate number) domain computations, faulty state pruning/merging, and others.

The genius of this approach is that it converts the question "is the verifier correct?" into a *concrete differential testing* problem: the concrete execution truth (what values registers actually hold) is compared against the verifier's claimed abstract approximation, with no need for a reference formal model or a separate correct implementation.

### 2.2 Process Steps

1. **eBPF Program Acquisition / Generation:** SEV either takes real-world eBPF programs from system traces (e.g., from BCC tools, Cilium, Falco), generates synthetic programs via a structured program generator, or uses programs from prior testing corpora.

2. **Verifier Acceptance Check:** The target program is submitted to the Linux kernel's eBPF verifier (via `bpf()` syscall). Only programs that the verifier *accepts* proceed — these are programs the verifier believes are safe.

3. **Concrete State Profiling:** SEV executes the accepted program (in a controlled environment) and records the concrete state at each basic block entry: the actual values held in all 10 eBPF general-purpose registers (R0–R9), the frame pointer (R10), pointer types, and any map-read values. This is the "ground truth" concrete state.

4. **State Embedding Code Generation:** For each profiled basic block boundary, SEV generates eBPF bytecode that embeds the concrete register values. The embedding is done by inserting assertion instructions:
   - For **scalar registers**: instructions that check whether the concrete value falls within the verifier's claimed range. If the verifier's tracked range for a register is `[lo, hi]` and the embedded concrete value `v` satisfies `v ∈ [lo, hi]`, then the verifier should accept these checks. If the embedding reveals that the verifier's claimed range *excludes* `v`, the verifier either rejects the check (detectable inconsistency) or accepts unsafe code (soundness bug).
   - For **pointer registers**: type-assertion checks that verify the embedded pointer type matches what the verifier claims.
   - For **BPF maps**: map accesses crafted to reflect the concrete map state at execution time.

5. **State-Embedded Program Verification:** The augmented, state-embedded program is submitted to the verifier. SEV observes the verifier's verdict.

6. **Bug Triggering and Confirmation:** If a soundness bug is present, the state-embedded program may be accepted by the verifier even though the embedded concrete state exposes that the verifier's abstract state was an *under-approximation* (or otherwise incorrect). This is confirmed by actually executing the state-embedded program and observing whether the "impossible" (from verifier's perspective) code path executes — typically manifesting as a kernel memory read/write that the verifier should have blocked.

7. **Minimization and Reporting:** Bug-triggering programs are minimized (delta debugging) and reported to the Linux kernel security team.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in This Paper |
|---|---|
| **Abstract Interpretation** | The theoretical framework underlying the eBPF verifier's analysis; SEV exploits its over-approximation invariant as the testable oracle |
| **eBPF / BPF bytecode** | The input language — state-embedded programs are crafted as eBPF bytecode programs |
| **Linux `bpf()` syscall** | The interface through which SEV submits programs to the kernel verifier for analysis |
| **tnum (Tristate Number) Domain** | The eBPF verifier's bitwise abstract domain; tracks which bits of a register are known-zero, known-one, or unknown; a key source of verifier bugs |
| **Interval / Range Domain** | The verifier's scalar value range tracking (signed 64-bit, unsigned 64-bit, signed 32-bit, unsigned 32-bit bounds per register) |
| **State Pruning / Path Merging** | The verifier's optimization that stops re-analyzing paths whose abstract state is subsumed by a previously verified state; a major source of unsoundness bugs |
| **Concrete Execution (JIT / Interpreter)** | Used to capture ground-truth concrete states for the embedding step |
| **Delta Debugging / Test Case Minimization** | Reduces bug-triggering programs to minimal reproducers for reporting |
| **SEV Tool** | The authors' implementation of the state embedding pipeline (written in C/Rust, interacting with Linux kernel headers) |

### 2.4 Key Data Structures / Models

- **Abstract Register State (`bpf_reg_state`):** The verifier's per-register abstract state structure, holding: `type` (e.g., `NOT_INIT`, `SCALAR_VALUE`, `PTR_TO_CTX`, `PTR_TO_MAP_VALUE`, `PTR_TO_STACK`, `CONST_PTR_TO_MAP`, etc.), `var_off` (a tnum encoding bitwise constraints), `smin_value`, `smax_value`, `umin_value`, `umax_value` (signed/unsigned 64-bit interval bounds), `s32_min_value`, `s32_max_value`, `u32_min_value`, `u32_max_value` (32-bit interval bounds).

- **Verifier State (`bpf_verifier_state`):** Per-path abstract state maintained by the verifier during CFG traversal, consisting of register states for all 11 registers, a stack frame model, and information about active BPF map references.

- **Control-Flow Graph (CFG):** The eBPF program's CFG is what the verifier traverses. SEV inserts state-embedding checks at CFG node boundaries (basic block entry points).

- **State Embedding Assertion Block:** A synthesized basic block of eBPF instructions that encodes: (a) forcing a register to hold a known concrete value, (b) performing a conditional branch whose verifier-visible branch condition contradicts what the verifier believes possible, (c) on the "impossible" branch, performing a kernel memory dereference that the verifier thought was guarded away.

- **BPF Map State:** For programs using BPF maps, SEV profiles the actual key-value pairs in maps at execution time and embeds representative map reads as additional correctness check points.

- **Path Exploration Tree:** The verifier's internal worklist of `(instruction, abstract_state)` pairs to explore. State pruning (`bpf_verifier_state_list`) affects which paths are explored; SEV's embeddings stress-test whether pruning decisions are sound.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

This paper does **not** directly target network functions (NFs) such as NAT, firewalls, load balancers, or intrusion detection systems. Instead, it targets the **Linux eBPF verifier** — the kernel component that gates *all* eBPF programs, including eBPF-based NFs. Its subject of validation is the verifier's *own internal logic*, not the behavior of any specific NF running on top of eBPF.

In the NF context, the paper is relevant because:
- Many modern eBPF-based NFs (e.g., Cilium's XDP-based load balancer, Katran, Falco security filters, Cloudflare's DDoS mitigation programs) rely on the eBPF verifier for correctness guarantees. If the verifier is unsound, these NF guarantees are void.
- An unsound verifier allows an attacker to craft eBPF programs that appear safe but execute arbitrary kernel code, undermining all NF isolation guarantees.

### 3.2 How It Validates NF Behavior

SEV validates the *verifier*, not an NF directly. The validation pipeline is:

1. Take any eBPF program (which could be an NF like a packet filter or load balancer program).
2. Submit to the Linux verifier → verifier accepts (claims program is safe).
3. Execute the program to capture concrete register states.
4. Embed these states as assertion bytecode → create state-embedded program.
5. Submit state-embedded program to verifier.
6. If verifier accepts the state-embedded program but concrete execution reveals the verifier's abstract state was inconsistent (under-approximating), a soundness bug is confirmed.
7. The soundness bug implies: any eBPF NF running under this verifier may be silently certified as safe even when it performs unsafe operations.

### 3.3 What Properties / Invariants Does It Prove?

The paper checks **verifier soundness**: the property that the verifier's abstract state at each program point is a valid *over-approximation* of all reachable concrete states. Formally:

- **Scalar Range Soundness:** For every register `r` at every program point `p`, if the verifier claims `r.val ∈ [umin, umax]` (unsigned) and `r.val ∈ [smin, smax]` (signed), then for every concrete execution reaching `p`, the actual register value must satisfy both constraints.
- **Bitwise (tnum) Soundness:** For every register `r`, if the verifier's `var_off` claims certain bits are known-zero or known-one, the concrete value must respect those bit constraints.
- **Pointer Type Soundness:** If the verifier classifies a register as holding a particular pointer type (e.g., `PTR_TO_MAP_VALUE` with offset `o`), then at runtime the register must hold a pointer of that type with that offset.
- **State Pruning Soundness:** When the verifier prunes a path because it claims the current abstract state is subsumed by a previously verified state, the pruned path's concrete states must also be covered by the previously verified abstract state.
- **Memory Safety (Implicit):** By extension, if all the above hold, then all memory accesses allowed by the verifier must be within valid bounds (no out-of-bounds reads/writes to kernel memory).

The paper does **not** prove these properties formally; rather, it *tests* them empirically using state embedding as the bug oracle. Bugs found represent violations of these invariants.

### 3.4 Input Requirements

- **Input to SEV:** Any eBPF program in bytecode form (`.o` file or raw bytecode) that the Linux kernel verifier accepts.
- **No source code required:** SEV operates at the eBPF bytecode level.
- **No user-provided specification:** The oracle is derived automatically from the verifier's own abstract interpretation semantics — no external spec is needed.
- **Linux kernel access:** SEV requires the ability to load eBPF programs via the `bpf()` syscall and to execute them (root or `CAP_BPF`/`CAP_SYS_ADMIN` privilege during testing).
- **Profiling runtime:** A userspace or kernel execution environment to capture concrete register states during actual program execution.

### 3.5 Guarantees Provided

- **No formal soundness proof:** SEV is a testing tool, not a verifier. It provides no mathematical guarantee that the eBPF verifier is sound.
- **Bug witnesses:** When a bug is found, SEV provides a concrete eBPF program that demonstrates the verifier's unsoundness — a concrete *counterexample* showing a concrete state not contained in the verifier's claimed approximation.
- **High bug-finding efficacy:** Empirical guarantee: within one month, 15 previously unknown bugs found in the heavily-scrutinized Linux eBPF verifier.
- **Exploitability confirmation:** Two bugs confirmed exploitable for local privilege escalation — the state-embedded programs serve as proof-of-concept exploits.
- **Coverage of major abstract domains:** SEV's embedding covers scalar intervals, tnum, pointer types, and map states — the main abstract domains the verifier uses.

---

## 4. NF Chain Verification

This paper does **not** address NF chains (service chains of multiple NFs in sequence). It is focused entirely on validating the correctness of the Linux eBPF verifier as a single monolithic component.

**Single-component scope only.** The paper's unit of analysis is one eBPF program being analyzed by one instance of the verifier. It does not consider:
- Multiple eBPF programs composed in a chain (e.g., via TC ingress/egress hooks in sequence, or XDP + TC in combination).
- Data-flow or state carry-over between chained NF programs.
- Ordering properties or isolation between co-resident eBPF programs.
- The correctness of the eBPF map-sharing interfaces that allow multiple eBPF programs to share state.

**What would be needed to extend to chains:**
1. A model of how multiple eBPF programs interact — including shared BPF maps, tail calls, and prog-prog calls (eBPF subprogram calls).
2. Compositional abstract interpretation that propagates verifier state across program boundaries.
3. State embedding would need to be extended to embed inter-program state (e.g., the state of a shared map after program A runs and before program B runs).
4. Properties like "program B's verifier-certified guarantees remain valid even after program A has modified the shared map" would require cross-program soundness analysis — a significantly harder problem not addressed here.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs static dataflow analysis on raw eBPF bytecode by constructing a Control-Flow Graph with Null Checks (CFG-NC) directly from bytecode — no source code is required. It then runs a Prolog-based query engine over this CFG to check behavioral assertions about the eBPF program's logic: what packets it drops/forwards, how it manipulates headers, whether it violates invariants like state consistency across BPF maps. It is designed to handle multi-NF chains by composing CFG-NCs of successive eBPF programs and checking chain-level properties such as ordering invariants and stateful carry-over correctness.

### 5.2 Key Differences from This Paper

| Dimension | Sun & Su (OSDI '24) | Yaksha-Prashna |
|---|---|---|
| **Target** | The eBPF *verifier* itself (meta-level) | eBPF *programs* implementing NFs (object-level) |
| **Input** | eBPF bytecode + concrete execution traces | Raw eBPF bytecode only (no execution needed) |
| **Technique** | State embedding (concrete state + oracle from verifier semantics) | Static CFG-NC dataflow analysis + Prolog query engine |
| **Formalism** | Abstract interpretation (testing verifier's own AI) | Dataflow analysis + logic programming (Prolog) |
| **Execution Required?** | Yes — concrete execution to profile register states | No — fully static, offline |
| **Properties Checked** | Verifier soundness (unsoundness = security bug) | NF behavioral properties (drop/forward logic, map state, chain invariants) |
| **NF Chain Support** | None — single program, single verifier | Yes — multi-NF chains, inter-program dataflow |
| **Output** | Counterexample eBPF program demonstrating verifier bug | Behavioral assertion verdicts (pass/fail) with Prolog proof traces |
| **Timing** | Offline (post-program loading, pre-deployment testing) | Offline (pre-deployment static analysis) |
| **Kernel Access Required?** | Yes (needs `bpf()` syscall for verification + execution) | No (operates on bytecode files directly) |
| **Bug Type Detected** | Verifier logic bugs (unsoundness CVEs) | NF semantic bugs (wrong forwarding, map corruption, chain violations) |
| **Requires Source Code?** | No | No |
| **Stateful Analysis** | Yes (BPF map states profiled and embedded) | Yes (BPF map state modeled in CFG-NC and tracked across chain) |

### 5.3 How This Paper Is Useful For Us

1. **Motivational Citation:** Sun & Su's paper provides powerful motivation for why eBPF correctness matters at the kernel level. Their discovery of 15 verifier bugs — two enabling LPE — underscores that the eBPF ecosystem (including eBPF-based NFs) is a high-stakes correctness domain. We can cite this paper to argue that even infrastructure components trusted for security can harbor subtle logic bugs, motivating the need for NF-level behavioral analysis on top of (and independent from) the verifier.

2. **Layer Distinction:** This paper validates the *verifier layer* (kernel infrastructure); Yaksha-Prashna validates the *application layer* (NF behavior). These are complementary and non-overlapping. We can explicitly position our work by saying: "Even if the eBPF verifier is sound (cf. Sun & Su 2024), it only guarantees memory safety — not NF behavioral correctness. Yaksha-Prashna addresses the latter."

3. **Demonstrates eBPF-Level Analysis is Feasible:** The paper demonstrates that meaningful semantic analysis of eBPF bytecode (register states, pointer types, map interactions) is tractable and produces useful results. This validates the general approach of bytecode-level analysis that Yaksha-Prashna also takes.

4. **Limitation We Solve:** Sun & Su require *concrete execution* to profile register states — their tool cannot operate purely statically. Yaksha-Prashna achieves fully static analysis without needing to run the program, making it safer (no privileged kernel access needed) and applicable to programs that may not be safely executable in isolation.

5. **No Chain Analysis:** Sun & Su's tool is single-program only. We can cite this as a gap that Yaksha-Prashna addresses: chain-level verification of composed eBPF NFs.

6. **Related Work Positioning:** In a related work section on eBPF verification, this paper should be cited as the leading work on *verifier meta-validation* — distinct from our focus on NF behavior validation.

### 5.4 Positioning Statement

> "While Sun & Su [OSDI'24] validate the eBPF verifier's own soundness using state embedding — confirming that the verifier correctly over-approximates concrete program states — their approach operates at the kernel infrastructure layer and requires concrete execution, leaving NF behavioral semantics and multi-program chain properties unverified. Yaksha-Prashna addresses this complementary gap by performing fully static, source-free dataflow analysis on eBPF bytecode to check NF-level behavioral assertions and chain-level invariants without requiring any program execution or kernel access."

---

## 6. Additional Notes

### 6.1 Bug Categories Found

The 15 bugs found by SEV fall into several categories of verifier unsoundness:

- **Incorrect scalar range propagation:** After arithmetic operations, the verifier computes incorrect bounds (e.g., after `BPF_ALU64_ADD`, the resulting interval was not widened correctly).
- **tnum domain errors:** Bitwise operations (AND, OR, XOR, shifts) caused incorrect tnum updates — bits claimed known-zero were actually unknown.
- **Pointer arithmetic unsoundness:** After adding a scalar to a pointer, the verifier incorrectly tracked the resulting pointer's offset bounds.
- **State pruning unsoundness:** The verifier pruned paths based on a stale abstract state that did not subsume the current state, causing it to skip analysis of code paths that could be unsafe.
- **32-bit/64-bit interaction bugs:** The verifier's handling of sub-register operations (operating on the lower 32 bits of a 64-bit register) produced incorrect extended values.

### 6.2 Security Impact

- **CVEs:** Multiple CVEs were assigned for the exploitable bugs (specific CVE IDs not published in the paper at time of writing, but disclosed through the Linux kernel security process).
- **Local Privilege Escalation:** Two bugs were demonstrated as exploitable for LPE on unpatched Linux kernels — an attacker with the ability to load eBPF programs could gain root.
- **Kernel Versions Affected:** Bugs were present across multiple recent kernel versions (5.x and 6.x), reflecting the longevity and subtlety of these logic errors.

### 6.3 Comparison to Related eBPF Verification Tools

| Tool | Approach | Target | Source Required? |
|---|---|---|---|
| **SEV (this paper)** | State embedding, testing | eBPF verifier soundness | No (bytecode) |
| **BVF (Sun, prior work)** | Structured program generation | eBPF verifier bugs | No (bytecode) |
| **Serval** | Symbolic execution, Rosette | NF verification (Racket) | Yes (source) |
| **Agni** | Pruning symbolic execution | eBPF NF equivalence | No (bytecode) |
| **Jitterbug** | Formal JIT correctness | eBPF JIT compiler | Yes (model) |
| **K2** | Superoptimizer | eBPF program optimization | No (bytecode) |
| **Yaksha-Prashna** | CFG-NC dataflow + Prolog | eBPF NF chain behavior | No (bytecode) |
