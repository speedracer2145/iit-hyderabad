# FlowMage: LLM-Based Static Analysis for Stateful NF Chain Optimization

**Authors:** Hamid Ghasemi, Jonatan Langlet, Andreas Kassler et al.  
**Year:** 2024 | **Venue:** EuroMLSys (ACM Workshop on Machine Learning for Systems)  
**DOI/Link:** https://dl.acm.org/doi/10.1145/3642970.3655824

---

## 1. Overview

Deploying high-performance stateful NF chains (e.g., FastClick or VPP pipelines) requires careful configuration of the network I/O layer — specifically **RSS (Receive Side Scaling)**, which distributes incoming flows across CPU cores. If RSS is misconfigured, stateful NFs that need to see all packets of a flow on the same CPU core (for correct state machine tracking) will receive flow packets on different cores, causing correctness failures (dropped connections, wrong firewall decisions, incorrect NAT translations).

Manually configuring RSS for complex NF chains is error-prone and expert-intensive. FlowMage proposes a radically different approach: use **GPT-4** to read NF source code and automatically extract the semantic properties needed to compute a correct RSS configuration. It then validates the generated configuration against these extracted properties.

FlowMage is significant for our research because it represents one of the first applications of Large Language Models to the problem of NF behavioral analysis — not just code generation but semantic property extraction from real NF implementations.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

FlowMage's pipeline has two phases:

**Phase 1 — LLM-based property extraction:** GPT-4 is prompted to read each NF's C++ source code (FastClick element or VPP plugin) and answer structured questions: Which packet header fields does this NF read? Which does it write? What stateful operations does it perform? Does it require all packets of a flow to be processed on the same CPU core (affinity constraint)?

**Phase 2 — RSS configuration validation:** Given the extracted per-NF properties, FlowMage computes the required RSS hash key — which header fields must be hashed consistently for the NF chain to be correct. It then validates whether the planned RSS configuration satisfies all NF affinity requirements.

### 2.2 Process Steps

1. **NF Source Ingestion:** Read C++ source of each FastClick element / VPP plugin in the chain.

2. **Structured LLM Prompting:** For each NF, send a structured prompt to GPT-4:
   - "What packet header fields does this NF read?"
   - "Does this NF maintain per-flow state? Which fields define a flow?"
   - "Would processing two packets of the same flow on different CPU cores cause correctness issues?"

3. **Property Extraction:** Parse GPT-4's output (structured JSON or natural language) into formal per-NF property objects:
   - `reads_fields`: set of header fields accessed
   - `writes_fields`: set of header fields modified
   - `affinity_fields`: set of fields that define a "flow" for this NF (must be on same CPU)
   - `is_stateful`: boolean

4. **Affinity Constraint Computation:** Compute the union of all `affinity_fields` across all NFs in the chain. This is the minimal RSS key that satisfies all NFs.

5. **RSS Configuration Validation:** Check whether the proposed RSS configuration (hash key, queue count) satisfies the computed affinity requirements. If not, report the violation and suggest a corrected configuration.

6. **Output:** Validated RSS configuration or error report with explanation.

### 2.3 Tools & Formalisms Used

| Tool/Formalism | Role |
|---|---|
| **GPT-4** | LLM backbone for source code semantic analysis and property extraction |
| **Structured Prompting** | Chain-of-thought + structured output to elicit specific property answers |
| **FastClick/VPP** | NF frameworks whose elements are analyzed |
| **RSS (Receive Side Scaling)** | The network I/O distribution mechanism being configured |
| **Affinity Constraint Algebra** | Set-union computation over per-NF `affinity_fields` |

### 2.4 Key Data Structures / Models

- **Per-NF Property Object:** `{nf_name, reads_fields: set, writes_fields: set, affinity_fields: set, is_stateful: bool, complexity: enum{O(1), O(n), O(log n)}}`
- **Chain Affinity Requirements:** Union of all `affinity_fields` across the NF chain
- **RSS Configuration:** `{hash_key: tuple of header fields, queue_count: int, queue_assignment: policy}`

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

- **FastClick elements:** Stateful click elements (NAT, stateful firewall, DPI, connection tracker, TCP reassembler)
- **VPP plugins:** VPP graph nodes implementing stateful functions
- Any C/C++ NF whose source is readable by the LLM

**NOT:** eBPF/XDP NFs, kernel modules, or compiled binaries without source.

### 3.2 How It Validates NF Behavior

FlowMage does not validate correctness of the NF's packet processing logic. It validates **configuration correctness** — specifically: does the RSS configuration satisfy the affinity requirements extracted from the NF source code?

Validation steps:
1. Extract affinity requirements via LLM (which fields must be co-located per flow?)
2. Compare against the actual RSS hash key configuration
3. Flag any mismatch: if RSS hashes on fields that don't cover all NF affinity requirements, that's a correctness violation

### 3.3 What Properties / Invariants Does It Prove?

- **Flow affinity correctness:** All packets of a flow that a stateful NF needs to process together are guaranteed to land on the same CPU core
- **RSS configuration soundness:** The hash key is a superset of the union of all NF affinity fields
- **Processing complexity extraction:** Per-NF computational complexity (for performance planning)

**Does NOT prove:** NF behavioral correctness (whether the NAT rule is right, whether firewall rules are correct), memory safety, or any formal property of the NF's logic itself.

### 3.4 Input Requirements

| Input | What's Needed |
|---|---|
| NF source code | C++ source of each FastClick element or VPP plugin |
| NF chain specification | The ordered sequence of NFs in the chain |
| RSS configuration | The proposed or current RSS hash key and queue config |
| GPT-4 API access | LLM backend |

### 3.5 Guarantees Provided

- **LLM-based (probabilistic):** Properties are extracted by GPT-4, not formally proved — risk of hallucination or misclassification
- **Validation is sound** given correct property extraction: if extracted affinity fields are correct, the RSS check is deterministic
- **Best-effort:** Authors validate GPT-4 accuracy on a benchmark of FastClick elements (reported: high accuracy, low false negative rate)

---

## 4. NF Chain Verification

FlowMage is **chain-aware** in a specific, limited way:
- It analyzes each NF individually and then composes the requirements across the chain
- The chain-level property checked: "Does the RSS configuration ensure flow coherence for all NFs in the chain?"
- It does NOT check: inter-NF semantic correctness (what NF_1 outputs matters to NF_2), stateful carry-over behavior, or chain-level invariants beyond flow affinity

Chain verification scope:
- ✅ Flow affinity composition across chain
- ✅ RSS configuration correctness for the full chain
- ❌ No inter-NF behavioral semantics
- ❌ No stateful carry-over analysis
- ❌ No chain-level packet processing correctness

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna takes raw eBPF bytecode, builds a CFG-NC (packet-header accesses, BPF map R/W, helper calls, action decisions), and answers behavioral assertion queries via Prolog. It handles chains through shared BPF map tracking and inter-program dependency analysis.

### 5.2 Key Differences from This Paper

| Dimension | FlowMage | Yaksha-Prashna |
|---|---|---|
| **Input** | C++ source code | eBPF bytecode |
| **NF framework** | FastClick / VPP | eBPF XDP/TC (Linux kernel) |
| **Analysis technique** | LLM semantic extraction | Static dataflow analysis |
| **Property type** | Configuration correctness (RSS) | Behavioral assertion (packet processing) |
| **Guarantee** | Probabilistic (LLM-based) | Deterministic (static analysis) |
| **Chain analysis** | Flow affinity only | Full behavioral dependencies |

### 5.3 How This Paper Is Useful For Us

1. **LLM-as-analyzer precedent:** FlowMage validates using LLM for NF behavioral property extraction — this supports the general direction of LLM-assisted NF analysis and provides a comparison point for accuracy vs. formal methods.
2. **Configuration validation angle:** FlowMage checks configuration properties that YP currently does not — this is a gap YP could address in future work (e.g., XDP program placement and CPU affinity for eBPF NF chains).
3. **NF semantic property extraction:** FlowMage's structured prompting for "which header fields does this NF read/write?" is directly parallel to what YP's CFG-NC extraction does for eBPF — YP does this formally, FlowMage does it with LLMs. Cite to contrast formal vs. LLM-based property extraction.
4. **Workshop paper — low citation weight:** EuroMLSys is a workshop; cite selectively, primarily to motivate LLM directions or configuration validation.

### 5.4 Positioning Statement

> "While FlowMage demonstrates that LLMs can extract semantic NF properties (such as flow affinity constraints) from source code to validate deployment configurations, its approach is probabilistic and limited to C++ NF frameworks. Yaksha-Prashna extracts equivalent behavioral properties — packet field accesses, map operations, action decisions — deterministically from compiled eBPF bytecode, providing formal guarantees independent of the LLM's accuracy and applicable to the rapidly growing ecosystem of eBPF-based NFs."

---

## Summary Table

| Attribute | Value |
|---|---|
| **Core method** | GPT-4 property extraction + RSS affinity validation |
| **Input** | C++ NF source code + RSS config |
| **NF types** | FastClick / VPP stateful NF chains |
| **Chain support** | ✅ Flow affinity across chain only |
| **Spec required** | No — properties extracted by LLM |
| **Guarantee** | Probabilistic (LLM-based extraction) |
| **Venue** | EuroMLSys 2024 (ACM workshop) |
| **Cited by YP** | No |
