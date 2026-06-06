# Neurosymbolic Programming in Yaksha-Prashna: Full Technical Study

> **Research Context:** IIT Hyderabad · eBPF Network Function Verification  
> **Core Question:** How can neurosymbolic programming overcome Yaksha-Prashna's limitations to handle complex queries beyond its current 6-field, simple-query capability?

---

## 1. What Yaksha-Prashna Does Today

Yaksha-Prashna (arXiv:2602.11232, IIT Hyderabad/CMU/IBM/Walmart, 2026) solves a critical problem: **cloud operators receive third-party eBPF network functions as opaque bytecodes** and have no way to verify correctness without source code.

### Architecture (Pure Symbolic)

```
eBPF Bytecode → [Dataflow Analyzer (R1-R15)] → CFG-NC → [Prolog QE] → Query Results
                      PURE SYMBOLIC                    PURE SYMBOLIC
```

| Component | What It Does |
|---|---|
| **Analyzer** | Dataflow analysis on bytecode. Builds CFG-NC (Control Flow Graph + Network Context). Tracks register (R), memory (M), and context (C) state via 15 transfer function rules |
| **Query Engine** | Translates CFG-NC → Prolog facts. Users write queries in a Prolog-inspired DSL |

### What It Can Extract (The 6 Fields)

| # | Field | Example |
|---|---|---|
| 1 | Packet header reads | `reads(eth.type)`, `reads(ipv4.dst)` |
| 2 | Packet header writes | `writes(ipv4.ttl)` |
| 3 | Protocols processed | `accessesProtocol(ipv4)`, `accessesProtocol(tcp)` |
| 4 | eBPF maps accessed | `accessesMap(lpm_trie_map)` |
| 5 | Helper functions invoked | `callsHelper(bpf_redirect)` |
| 6 | Packet actions | `DROP`, `PASS`, `REDIRECT` |

### Performance Strengths

- **200–1000× faster** than Klint (symbolic execution baseline)
- 4K-path NF analyzed in **<300ms / <50MB RAM**
- 24 properties, 2 retrieval queries across 6 NF classes (Router, Firewall, LB, NAT, Bridge, Policer)
- Tested on production NFs: Katran, Suricata, CRAB, Fluvia

---

## 2. Yaksha-Prashna's Hard Limitations

> [!CAUTION]
> These are the specific gaps that neurosymbolic programming must address.

### L1: Only Simple Queries
The DSL supports only **assertion** and **retrieval** queries over the 6 extracted fields. Cannot express:
- Temporal properties ("does X happen before Y?")
- Quantitative queries ("how many paths lead to DROP?")
- Cross-NF behavioral comparison ("is NF-A similar to NF-B?")
- Conditional chains ("if protocol is TCP AND port is 80, what happens?")

### L2: Only 6 Fields
The analyzer extracts exactly 6 categories of network context. It **cannot** capture:
- Arithmetic operations on packet data (checksums, counters)
- Complex map interactions (read-modify-write patterns)
- Stateful behavior across multiple packet invocations
- Application-layer (L7) protocol parsing
- BPF-to-BPF function call semantics beyond simple inlining

### L3: Static Analysis Only
- No runtime properties (latency, throughput, memory pressure)
- Cannot detect bugs that manifest only under specific input sequences
- No dynamic taint tracking

### L4: Expert-Only Interface
- Users must know **both eBPF internals AND Prolog syntax**
- No natural language interface
- Writing correct DSL queries requires understanding CFG-NC structure

### L5: No Property Discovery
- System **verifies given assertions** but cannot **discover** what to check
- Operators must manually read RFCs to know what properties matter for a new NF class

### L6: No Semantic Understanding
- Cannot answer "what does this NF do?" in plain English
- No similarity search, clustering, or behavioral embedding
- No cross-NF pattern recognition

### L7: Brittle to Unseen Patterns
- Rules R1-R15 assume structured protocol parsing (next-protocol check + memory-bound check)
- Fails on non-standard patterns (e.g., NF12 UDP case in the paper)
- Cannot generalize to new eBPF constructs without manual rule additions

### L8: XDP Hook Only
- Currently supports only XDP programs
- TC, socket filters, cgroup, tracing programs are out of scope

---

## 3. The eBPF Verification Landscape (2024-2026)

### Current Tools and Their Limits

| Tool | Approach | Limitation |
|---|---|---|
| **Linux kernel verifier** | Abstract interpretation + path simulation | Path explosion; 1M instruction limit; false rejections of safe programs |
| **Klint** | Symbolic execution (SMT/Z3) | 200-1000× slower than Yaksha; doesn't scale to complex NFs |
| **Rex** | Language-based safety (Rust) | Requires rewriting in safe Rust; doesn't work on existing bytecodes |
| **Prevail** | Abstract interpretation | Limited to safety properties; no behavioral queries |
| **Yaksha-Prashna** | Dataflow analysis + Prolog | Fast but limited expressiveness (this study) |

### The Language-Verifier Gap (USENIX 2024-2025)
As eBPF programs grow more complex (storage controllers, schedulers, service meshes), there's a growing gap between what high-level languages express and what verifiers can understand. Key issues:
- Complex branching causes path explosion
- Imprecise register dependency tracking in pruning
- Artificial instruction limits (1M per path)
- No support for L7 protocol parsing or stateful reasoning

### What's Missing Industry-Wide
- **No tool** currently combines formal verification with natural language accessibility
- **No tool** provides semantic similarity search across eBPF programs
- **No tool** automatically discovers behavioral specifications from bytecode corpora

---

## 4. The Three Foundational Papers

### 4.1 Yaksha-Prashna → The Symbolic Foundation

**Key contribution:** CFG-NC representation + Prolog knowledge base

The CFG-NC is actually a **goldmine** for neurosymbolic integration because:
1. It's a structured, formally correct intermediate representation
2. It can serve as ground-truth labels for neural training (no hallucination risk)
3. The Prolog KB is already in a logic programming format compatible with DeepProbLog

### 4.2 Bin2Summary (FSE 2024, CUHK) → The Neural Summarization Blueprint

**Key contribution:** Functionality-Specific Embedding (FSE)

| Aspect | Bin2Summary | Yaksha-Prashna Analog |
|---|---|---|
| Input | Stripped binary functions | eBPF bytecode NFs |
| Key insight | Argument dataflow = functionality | Packet dataflow = network behavior |
| Embedding | CFG → FCFG → weighted instruction embeddings | CFG-NC already extracts this! |
| Output | NL function summaries | NL behavioral descriptions |
| Results | 0.728 precision, 0.729 recall | Target: similar or better |

**Critical realization:** Yaksha-Prashna's CFG-NC **already extracts** the functionality-specific information that bin2summary had to learn from scratch. This means the neural component gets a massive head start.

### 4.3 CLAP (ISSTA 2024, Tsinghua) → The Contrastive Learning Blueprint

**Key contribution:** CLIP-style contrastive alignment between assembly code and NL explanations

| Aspect | CLAP | eBPF-CLAP (Proposed) |
|---|---|---|
| Code encoder | RoBERTa-base (110M params) | RoBERTa adapted for eBPF ISA |
| Text encoder | Pre-trained sentence-transformers | Same |
| Training pairs | 195M (assembly, NL) pairs; NL from GPT-3.5 (hallucination risk) | (eBPF bytecode, NL) pairs; NL from CFG-NC facts (zero hallucination) |
| Pre-training | MLM + Jump Target Prediction | MLM + eBPF-specific objectives |
| Contrastive loss | InfoNCE, batch=65536 | Same |
| Zero-shot BCSD | Recall@1 = 83.6% | Target: higher (cleaner supervision) |

**Key advantage over CLAP:** Our NL descriptions come from **formally verified CFG-NC facts**, not from LLM hallucinations. This is a fundamentally cleaner supervision signal.

---

## 5. Neurosymbolic AI: Taxonomy and Relevance

### Three Integration Patterns

```
Pattern 1: Neural → Symbolic
  LLM perceives/embeds raw input → Symbolic engine reasons/verifies
  Example: LLM translates NL to Prolog query → QE verifies

Pattern 2: Symbolic → Neural  
  Symbolic system provides structured scaffolding → Neural uses it as context
  Example: CFG-NC facts → LLM context for behavioral explanation

Pattern 3: Neural ↔ Symbolic (Tight/Differentiable)
  End-to-end differentiable logic; gradients flow through symbolic rules
  Example: DeepProbLog neural predicates inside Prolog
```

### Key Frameworks (2024-2025 State of the Art)

| Framework | What It Does | Relevance |
|---|---|---|
| **DeepProbLog** (KU Leuven) | Neural predicates in ProbLog; differentiable probabilistic logic | Can extend Yaksha's Prolog predicates with neural scoring |
| **DeepSeaProbLog** | Extends to continuous distributions | Handles mixed discrete-continuous eBPF metrics |
| **DeepSoftLog** | Soft-unification in embedding space | Fuzzy matching for similar-but-not-identical NF behaviors |
| **NS-CL** (MIT) | Neuro-symbolic concept learner | Maps to: embedding → query predicate matching |
| **SymBa** (2024) | Symbolic top-down solver + LLM for fact generation | Directly applicable: Prolog solver + LLM for property specification |
| **SYNVER/LLMLift** | LLM generates code → formal verification confirms | Matches our CEGIS loop pattern |
| **GraphRAG** (Microsoft 2024) | Knowledge graphs for retrieval | CFG-NC as structured retrieval corpus |

### The CEGIS Pattern (Counter-Example Guided Inductive Synthesis)

This is the **most critical pattern** for Yaksha-Prashna neurosymbolic integration:

```
User NL Query
    ↓
LLM generates candidate Prolog DSL query
    ↓
Yaksha-Prashna QE evaluates against CFG-NC
    ↓
PASS → return verified result
FAIL → return counterexample (specific path/field that violated)
    ↓
LLM receives counterexample, refines query
    ↓
Loop until convergence
```

This pattern is already demonstrated in the workspace's `neuro_symbolic_poc.py` — the mock CEGIS loop where the neural agent hallucinates `assert_action(udp, DROP)`, gets corrected by the symbolic engine ("UDP is never parsed"), and converges to the correct query.

---

## 6. Five Strategies to Overcome Yaksha's Limitations

### Strategy 1: Neural Query Generation (NL → DSL)
**Fixes:** L1 (simple queries), L4 (expert-only)

**What:** Train an LLM to translate natural language questions into Yaksha-Prashna DSL queries with grammar-constrained decoding.

**How:**
1. Take the 24 existing properties (Table 3) → generate NL paraphrases via GPT-4
2. Fine-tune LLaMA/Mistral on (NL, DSL) pairs
3. Use Yaksha-Prashna's grammar (Figure 1) to constrain the decoder
4. Evaluate via execution accuracy (does generated query produce correct results?)

**Why it works for complex queries:** The LLM can compose multiple predicates, chain conditions, and express queries that would require deep Prolog expertise to write manually.

**Novelty:** No existing work does NL → Prolog predicate generation for eBPF.

---

### Strategy 2: eBPF-CLAP (Contrastive Behavioral Embedding)
**Fixes:** L2 (6 fields), L6 (no semantics), L7 (brittle to unseen)

**What:** Extend CLAP to eBPF. Learn embeddings that capture behavioral semantics beyond the 6 extracted fields.

**Dataset engine:**
1. Compile 100+ open-source XDP/TC programs (Katran, Suricata, Cilium, L3AF) with multiple LLVM opt levels
2. Run Yaksha-Prashna Analyzer → extract CFG-NC facts
3. Auto-template facts → NL descriptions (zero hallucination)
4. Augment with LLM paraphrases

**Model:** RoBERTa-base adapted for eBPF ISA (11 registers, BPF opcodes, helper IDs). InfoNCE contrastive loss, batch ≥16K.

**Downstream tasks:**
- Zero-shot NF classification (firewall/LB/router/NAT)
- Behavioral similarity search
- Cross-NF clustering and drift detection

---

### Strategy 3: Neural Property Discovery
**Fixes:** L5 (no property discovery)

**What:** Automatically discover behavioral specifications for new NF classes using neural ILP (Inductive Logic Programming) over the Prolog KB.

**How:**
1. Run Yaksha-Prashna on corpus of known NFs per class
2. Train property induction model: input = set of CFG-NC facts → output = candidate Prolog assertions consistently true for positive examples
3. Verify induced properties on held-out NFs using the QE

**Connection to DeepProbLog:** Neural predicates assign soft probabilities to candidate properties; Prolog engine verifies with hard logic.

---

### Strategy 4: LLM-Guided CEGIS Refinement Loop
**Fixes:** L1 (simple queries), L4 (expert-only), L5 (no discovery)

**What:** LLM proposes assertions → Yaksha verifies → counterexample feedback → LLM refines. Already prototyped in `neuro_symbolic_poc.py`.

**Implementation:**
- Prompt includes: grammar (Figure 1), predicate semantics (Table 2), counterexample context
- Few-shot: 24 properties from Table 3 as demonstrations
- Termination: max iterations or assertion proven correct

---

### Strategy 5: RAG-based NF Understanding (GraphRAG over CFG-NC)
**Fixes:** L4 (expert-only), L6 (no semantics)

**What:** Use CFG-NC facts as structured retrieval corpus. eBPF-CLAP embeddings for semantic search + LLM for NL generation.

**Use cases:**
- "Explain what this NF does" → auto-generated documentation
- "Why did assertion A3 fail?" → LLM explains using retrieved facts
- "Generate a spec for this NF class" → structured NL spec from inferred properties
- "Is this NF safe to deploy before my Cilium chain?" → cross-NF RAG

---

## 7. How Complex Queries Become Possible

### Current State: What Yaksha Can Query

```prolog
% Simple assertion (single predicate)
passes(nf_id, xdp, [(var, var)]).

% Simple retrieval (single field)
accessesProtocol(nf_id, "ipv4").

% Chain interaction (predefined pattern)
raw_dependency(nf1, nf2, field).
```

### Neurosymbolic Extension: Complex Query Examples

| Query Type | Natural Language | How NSP Handles It |
|---|---|---|
| **Multi-predicate** | "Does this NF read IPv4 source AND write to a map AND then drop?" | Strategy 1: LLM composes conjunction of predicates |
| **Temporal** | "Is TTL decremented before the routing decision?" | Strategy 2: eBPF-CLAP captures sequential patterns in embedding space |
| **Similarity** | "Find all NFs that behave like Katran" | Strategy 2: Cosine similarity in embedding space |
| **Discovery** | "What properties define a compliant firewall?" | Strategy 3: Neural ILP mines invariants from firewall corpus |
| **Explanatory** | "Explain why this NF chain has a WAR dependency" | Strategy 5: RAG retrieves relevant CFG-NC facts → LLM explains |
| **Quantitative** | "How many paths lead to packet DROP?" | Strategy 1: LLM generates aggregation query over QE results |
| **Cross-architecture** | "Does this NF behave differently under -O2 vs -O0?" | Strategy 2: Compare embeddings across optimization levels |
| **Specification** | "Generate RFC-compliant spec for this NAT" | Strategy 3+5: Property discovery + RAG generation |

---

## 8. Comparison Table

| Dimension | Yaksha-Prashna (Current) | Pure Neural (LLM only) | eBPF-NeuroSym (Proposed) |
|---|---|---|---|
| **Input** | eBPF bytecode | eBPF bytecode | eBPF bytecode |
| **Analysis** | Dataflow (symbolic) | Neural embedding | Dataflow + neural embedding |
| **Query interface** | Prolog DSL (expert) | Natural language (unreliable) | Natural language (verified) |
| **Correctness** | 100% (formal) | ~70-85% (hallucinations) | 100% (CEGIS-verified) |
| **Fields extracted** | 6 fixed categories | Unbounded but noisy | 6 verified + semantic extensions |
| **Complex queries** | ❌ Simple only | ✅ But unverified | ✅ Verified via CEGIS |
| **Similarity search** | ❌ | ✅ | ✅ |
| **Property discovery** | ❌ Manual | ❌ | ✅ Neural ILP |
| **Spec generation** | ❌ | Hallucination risk | ✅ RAG + verified |
| **Speed** | <300ms | ~1-5s (LLM inference) | <300ms (QE) + ~2s (NL layer) |
| **Generalization** | Fixed rules | Zero-shot | Zero/few-shot + formal fallback |

---

## 9. The eBPF-NeuroSym Architecture

```
                ┌──────────────────────────────────────────────┐
                │           eBPF-NeuroSym Pipeline             │
                └──────────────────────────────────────────────┘

eBPF Bytecode ──┐
                │
                ▼
    ┌───────────────────────┐
    │  Yaksha-Prashna       │  ← EXISTING SYMBOLIC
    │  Analyzer (R1-R15)    │
    │  → CFG-NC Facts       │
    └───────────┬───────────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
  ┌──────────┐   ┌──────────────────┐
  │ Prolog   │   │ eBPF-CLAP        │  ← NEW NEURAL (S2)
  │ KB       │   │ Encoder          │
  └────┬─────┘   └────────┬─────────┘
       │                   │
       │                   ▼
       │          ┌─────────────────┐
       │          │ FAISS/pgvector  │
       │          │ Embedding Store │
       │          └────────┬────────┘
       │                   │
       ▼                   ▼
  ┌────────────────────────────────────┐
  │     NL Interface Layer             │
  │                                    │
  │  NL → DSL Translation (S1)        │
  │  CEGIS Refinement Loop (S4)       │
  │  RAG-based Explanation (S5)       │
  └──────────────┬─────────────────────┘
                 │
                 ▼
  ┌────────────────────────┐
  │  Property Discovery    │  ← NEURAL+SYMBOLIC (S3)
  │  (Neural ILP over KB)  │
  └────────────────────────┘
```

---

## 10. Research Questions

| RQ | Question | Strategy |
|---|---|---|
| **RQ1** | Can LLMs fine-tuned on 24 properties generalize to novel NL queries with correct DSL? | S1 |
| **RQ2** | Do CFG-NC-derived NL descriptions provide cleaner supervision than LLM-generated ones? | S2 |
| **RQ3** | Can neural ILP over Prolog KB discover useful NF-class properties matching RFC specs? | S3 |
| **RQ4** | Does CEGIS convergence outperform expert-written assertions in time-to-correct? | S4 |
| **RQ5** | Does RAG over CFG-NC produce more accurate behavioral explanations than pure LLM? | S5 |

---

## 11. Implementation Roadmap

### Phase 1: Data Collection (Weeks 1-4)
- Collect 100+ open-source XDP/TC eBPF programs
- Run Yaksha-Prashna Analyzer on all → extract CFG-NC facts
- Auto-generate NL descriptions from facts
- Build (bytecode, CFG-NC, NL description) triple dataset

### Phase 2: eBPF-CLAP Prototype (Weeks 5-10)
- Adapt CLAP-ASM tokenizer for eBPF ISA
- Pre-train with MLM + Jump Target Prediction on eBPF corpus
- Contrastive fine-tuning with CFG-NC-derived NL descriptions
- Evaluate: zero-shot NF classification

### Phase 3: NL Query Interface (Weeks 8-12)
- Fine-tune LLaMA-3/Mistral on (NL, DSL) pairs
- Grammar-constrained decoding using Yaksha grammar
- CEGIS loop integration
- Human evaluation with network operators

### Phase 4: Integration & Evaluation (Weeks 13-16)
- Full pipeline integration
- End-to-end: NL query → verified Yaksha-Prashna result
- Compare with baseline (expert writing DSL manually)
- Paper writing

---

## 12. Key Findings from This Study

> [!IMPORTANT]
> **Finding 1:** Yaksha-Prashna's CFG-NC is simultaneously its greatest strength AND the key enabler for neurosymbolic extension. It provides formally verified ground truth that eliminates the hallucination problem that plagues pure neural approaches (like CLAP's reliance on GPT-3.5 for training labels).

> [!IMPORTANT]
> **Finding 2:** The "6 fields, simple queries" limitation is NOT a fundamental architectural flaw — it's a **interface limitation**. The CFG-NC contains far richer information than the 6 categories expose. Neurosymbolic programming can unlock this latent information through embeddings (Strategy 2) and learned query composition (Strategy 1).

> [!IMPORTANT]
> **Finding 3:** The CEGIS loop is the critical architectural pattern that makes this safe for production. Unlike pure LLM approaches where hallucinations are catastrophic (a "99% accurate hallucination is a breach"), the symbolic engine acts as a mathematical gatekeeper that physically prevents incorrect conclusions.

> [!TIP]
> **Finding 4:** This work occupies a unique position — no existing system combines (a) formal eBPF dataflow verification, (b) contrastive bytecode embeddings, (c) natural language query interface, and (d) automated property discovery. Each individual component has precedent, but the integration is novel.

> [!NOTE]
> **Finding 5:** The eBPF ecosystem (Cilium, Katran, Suricata, BCC, L3AF) provides a rich enough corpus for contrastive pre-training. Combined with multi-optimization-level compilation, 100+ source programs can yield thousands of training pairs.

---

## 13. References

| Paper | Venue | Key Contribution |
|---|---|---|
| Yaksha-Prashna (Singh et al.) | arXiv 2026 (IIT-H/CMU/IBM/Walmart) | CFG-NC dataflow analysis + Prolog QE for eBPF |
| Bin2Summary (Song et al.) | FSE 2024 (CUHK) | Functionality-specific embedding for binary summarization |
| CLAP (Wang et al.) | ISSTA 2024 (Tsinghua) | Contrastive assembly-NL alignment (83.6% zero-shot) |
| DeepProbLog (Manhaeve et al.) | NeurIPS 2018 (KU Leuven) | Neural predicates in probabilistic logic |
| DeepSeaProbLog | 2024 (KU Leuven) | Discrete-continuous extension |
| NS-CL (Mao et al.) | ICLR 2019 (MIT) | Neuro-symbolic concept learning |
| CLIP (Radford et al.) | ICML 2021 (OpenAI) | Contrastive image-text alignment |
| RAG (Lewis et al.) | NeurIPS 2020 (Meta) | Retrieval-augmented generation |
| Rex | USENIX 2024 | Language-based eBPF safety via Rust |
| CEGIS (Solar-Lezama) | 2006 | Counterexample-guided inductive synthesis |
