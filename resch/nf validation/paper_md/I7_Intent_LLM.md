# Intent-Based Management of Next-Generation Networks: An LLM-Centric Approach

**Authors:** Abdelkader Mekrache, Adlen Ksentini, Christos Verikoukis  
**Year:** 2024 | **Venue:** IEEE Network (Magazine)  
**DOI/Link:** https://ieeexplore.ieee.org/document/10398234/

---

## 1. Overview

Managing modern 5G and next-generation (NextG) networks is extraordinarily complex. Operators must configure hundreds of network functions (NFs) across heterogeneous domains — Radio Access Network (RAN), Core, Edge, and Cloud — each with its own configuration syntax, descriptor format, and lifecycle semantics. Traditional Intent-Based Networking (IBN) systems attempted to simplify this by letting operators express high-level goals rather than low-level commands, but existing IBN solutions still required intents to be expressed in structured, semi-formal formats such as JSON or YAML — demanding significant technical expertise and creating a bottleneck between human intent and machine-executable configuration.

This paper proposes a fundamentally different approach: an **LLM-centric intent lifecycle management architecture** where operators express network intents entirely in **natural language**, and Large Language Models (LLMs) handle the full pipeline from natural language understanding through intent decomposition, translation into machine-executable descriptors, negotiation, activation, and continuous assurance. The system was validated in a real deployment at the **EURECOM 5G testbed facility**, demonstrating end-to-end operation across Cloud/Edge and RAN domains.

The core contribution is a practical, systems-level architecture that maps the five canonical stages of the IBN lifecycle (Decomposition → Translation → Negotiation → Activation → Assurance) onto concrete LLM-powered modules, and closes the loop with monitoring feedback and human correction signals. This is one of the first papers to demonstrate full end-to-end LLM-driven IBN management in a real 5G facility rather than simulation, making it a landmark reference for the field of AI-driven network management.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

The paper's central insight is that LLMs — pretrained on vast corpora including technical documentation, networking RFCs, and code — possess sufficient domain knowledge to bridge the gap between natural-language operator intent and machine-executable network configurations. The authors design a multi-stage pipeline where each lifecycle stage of an intent is handled by a dedicated LLM-driven module, with the following principles:

- **Natural language as the sole operator interface**: Operators write plain English descriptions of desired network behavior (e.g., "Deploy a video streaming service on the edge with guaranteed 10 Mbps throughput and low latency").
- **Hierarchical decomposition**: High-level intents are broken into domain-specific sub-intents (RAN configuration, Core NF configuration, Edge/Cloud configuration) by an LLM reasoning about the technological boundaries.
- **Code/Descriptor generation**: The LLM translates each sub-intent into the appropriate machine-readable artifact — Network Service Descriptors (NSDs) for Core/Edge services, and RAN Descriptors (RANDs) for radio access configuration.
- **Few-shot learning and prompt engineering**: The translation module uses few-shot prompting with domain-specific examples stored in a Knowledge Base (KB) to guide accurate generation. The KB stores validated (intent, NSD/RAND) tuples.
- **Closed-loop assurance**: A monitoring module continuously compares deployed NF behavior against the original intent. If a deviation is detected, a re-assurance loop is triggered, potentially involving re-translation or re-negotiation.
- **Human Feedback (HF) loop**: Operators or domain experts can correct LLM-generated configurations, and these corrections are fed back into the KB to improve future translations via in-context learning.

### 2.2 Process Steps

1. **Intent Ingestion**: Operator provides a natural-language intent (e.g., via a web portal or CLI) describing the desired network service or behavior.
2. **Intent Decomposition**: The LLM decomposition module parses the intent and identifies the technological domains involved (RAN, Core, Edge/Cloud). It produces domain-scoped sub-intents, one per domain. The LLM reasons about what each domain must configure to satisfy the overall goal.
3. **Intent Translation**: Each sub-intent is passed to a domain-specific LLM translation module. The module queries the Knowledge Base for similar historical (intent, descriptor) pairs and constructs a few-shot prompt. The LLM generates the corresponding NSD or RAND in the appropriate format (JSON/YAML for ETSI OSM; XML or vendor-specific for RAN).
4. **Intent Negotiation**: Generated configurations are checked for feasibility — resource availability, SLA compatibility, and cross-domain consistency are assessed. If the intent is infeasible, the system can negotiate with the operator (e.g., suggest a modified configuration or ask for relaxed constraints).
5. **Intent Activation**: Validated configurations are submitted to the underlying orchestration plane (ETSI OSM MANO for Core/Edge, RAN controller for radio access). The MANO instantiates the requested NFs using the generated NSDs.
6. **Intent Assurance**: A continuous monitoring module collects telemetry (latency, throughput, packet loss, resource utilization) from deployed NFs. The assurance module compares observed behavior against intent-specified targets. Deviations trigger re-assurance actions.
7. **Feedback Integration**: Successful and corrected configurations are added to the KB as new (intent, NSD) training tuples, improving future few-shot accuracy. Human corrections are incorporated via a Human Feedback (HF) mechanism.

### 2.3 Tools & Formalisms Used

| Tool / Formalism | Role in This Paper |
|---|---|
| **Large Language Models (LLMs)** | Core reasoning engine for intent parsing, decomposition, and descriptor generation. The companion NetSoft 2024 paper specifically uses **Code Llama** for NSD generation. General-purpose LLMs (e.g., GPT-family) may also be used in the IEEE Network paper. |
| **Few-Shot Prompting** | Technique for guiding the LLM's output by providing 2–5 worked examples of (natural-language intent → NSD) translations within the prompt context window. |
| **Prompt Engineering** | Crafting structured prompts with role instructions, domain context, and output format constraints to steer LLM behavior reliably. |
| **Knowledge Base (KB)** | Persistent store of validated (intent, NSD/RAND) tuples used as retrieval corpus for few-shot example selection. Analogous to a case-based reasoning database. |
| **ETSI OSM (Open Source MANO)** | ETSI-standardized NFV orchestrator that consumes NSDs and manages the lifecycle of Network Services on the EURECOM 5G facility. OSM is the activation back-end. |
| **Network Service Descriptor (NSD)** | ETSI-standardized JSON/YAML artifact describing a network service — its constituent VNFs, interconnections, resource requirements, and scaling policies. The primary output artifact of the LLM translation step. |
| **RAN Descriptor (RAND)** | Configuration artifact for the Radio Access Network domain, specifying radio parameters such as bandwidth, modulation, power levels, and scheduling policies. |
| **Human Feedback (HF) Loop** | Mechanism for injecting human corrections into the KB, enabling the system to learn from expert knowledge without retraining the LLM. |
| **Closed-Loop Monitoring** | Telemetry-driven feedback architecture that continuously checks deployed NF behavior against intent-level KPIs. |
| **Natural Language Processing (NLP)** | Underlying capability of LLMs used for understanding ambiguous operator intent statements and resolving semantic gaps. |

### 2.4 Key Data Structures / Models

- **Intent Representation**: Unstructured natural-language string from the operator. No schema or grammar is imposed on the input.
- **Sub-Intent (Decomposed)**: A domain-scoped natural-language statement (e.g., "configure the RAN to allocate 20 MHz bandwidth for video traffic" or "deploy a UPF and SMF on the edge cluster with 4 vCPUs").
- **Knowledge Base (KB) Entry**: A tuple `(intent_text, nsd_yaml)` or `(intent_text, rand_config)` representing a previously validated translation. Used for few-shot retrieval.
- **Network Service Descriptor (NSD)**: ETSI OSM-compliant YAML/JSON descriptor defining NF topology, VNF references, virtual link requirements, and resource constraints.
- **RAN Descriptor (RAND)**: Domain-specific XML or structured config file encoding RAN-side parameters (frequencies, power, scheduling).
- **Telemetry Vector**: Time-series KPI data collected during the assurance phase (throughput, latency, packet loss, CPU/memory utilization of deployed NFs).
- **Intent Compliance Score**: Derived metric comparing observed telemetry against intent-specified targets, used to trigger re-assurance.

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

The paper targets **5G core network functions** and **RAN elements**, specifically in the context of ETSI NFV/SDN deployments. In the EURECOM 5G facility, this includes:

- **User Plane Function (UPF)**: Handles data packet routing, forwarding, and QoS enforcement.
- **Session Management Function (SMF)**: Manages PDU sessions and coordinates between RAN and core.
- **Access and Mobility Management Function (AMF)**: Handles UE registration, mobility, and access control.
- **Generic Virtual Network Functions (VNFs)**: Any service deployed via ETSI OSM, such as custom edge applications, load balancers, or gateways.
- **RAN elements**: gNodeBs or O-RAN components configured via RAN descriptors.

The paper is not restricted to a single NF type — it addresses multi-NF service configurations as defined by NSDs, which can include chains of VNFs connected by virtual links.

### 3.2 How It Validates NF Behavior

Validation in this paper operates at the **intent compliance** level rather than the formal verification level. The process works as follows:

1. **Post-Deployment Monitoring**: After NFs are instantiated by ETSI OSM, a telemetry collection module gathers runtime KPIs from the deployed functions.
2. **Intent-KPI Mapping**: The system maps intent-expressed goals (e.g., "low latency," "10 Mbps guaranteed throughput") to measurable metrics (RTT, throughput, packet drop rate) using the LLM or a predefined ontology.
3. **Compliance Checking**: Observed KPIs are compared against intent-derived thresholds. Deviations outside acceptable bounds are flagged.
4. **Assurance Action**: If compliance fails, the assurance module may trigger re-translation of the sub-intent (generating a new NSD with different resource parameters) or re-negotiation with the operator.
5. **Human Feedback Integration**: Persistent compliance failures may prompt human expert review, with corrections fed back into the system.

Note: The validation is **behavioral and empirical** (runtime telemetry comparison), not formal or static. There is no code analysis, symbolic execution, or theorem proving.

### 3.3 What Properties / Invariants Does It Prove?

This paper does **not prove formal invariants** in the classical verification sense. The properties it checks are intent-compliance properties measured at runtime:

- **Throughput compliance**: Does the deployed NF/service deliver the bandwidth specified in the operator's intent?
- **Latency compliance**: Does end-to-end latency stay within intent-specified bounds?
- **Availability/reliability**: Is the service operational and not dropping connections?
- **Resource allocation correctness**: Were the correct number of vCPUs, memory, and VNF instances allocated as described by the generated NSD?
- **NSD syntactic validity**: Is the generated NSD syntactically correct and parseable by ETSI OSM? (Checked implicitly during activation.)
- **Cross-domain consistency**: Are sub-intents for different domains (RAN, Core, Edge) mutually compatible and non-conflicting?

No formal properties such as memory safety, packet-drop freedom, or RFC compliance are checked.

### 3.4 Input Requirements

| Input | Description |
|---|---|
| **Natural-language intent** | Operator provides a plain English description of the desired network service or behavior. No schema required. |
| **Knowledge Base** | Pre-populated with validated (intent, NSD) examples for few-shot prompting. Must be seeded with domain expert examples initially. |
| **LLM access** | Access to a pre-trained LLM (Code Llama, GPT-family, or similar) for inference. No fine-tuning of the LLM is required — the system relies on in-context learning. |
| **ETSI OSM / MANO credentials** | Backend orchestrator access for activation and telemetry retrieval. |
| **Telemetry infrastructure** | Monitoring agents deployed on NF hosts to collect runtime KPIs. |

No NF source code, binary analysis, or formal specifications are required from the operator.

### 3.5 Guarantees Provided

The paper provides **empirical best-effort guarantees** rather than formal proofs:

- **Best-effort intent compliance**: The system makes a reasonable attempt to satisfy the operator intent, verified via runtime telemetry, but does not guarantee it.
- **Iterative correction**: The closed-loop mechanism provides a mechanism for correcting non-compliant deployments, but convergence is not formally guaranteed.
- **Translation accuracy improvement over time**: The KB and HF loop improve few-shot accuracy incrementally, but accuracy is statistical (dependent on LLM behavior), not proven.
- **No soundness guarantees**: The system can generate incorrect, incomplete, or resource-conflicting NSDs if the LLM hallucinates or if the KB lacks relevant examples. Human validation is the backstop.

---

## 4. NF Chain Verification

This paper **implicitly handles multi-NF chains** through the NSD formalism, but does not explicitly reason about chain-level compositional properties. Here is the nuanced breakdown:

**What is covered:**
- ETSI NSDs can describe multi-VNF topologies — a single NSD may specify a chain of VNFs (e.g., vFirewall → vUPF → vSMF) connected by virtual links. When the LLM generates an NSD from an intent, it may produce a multi-VNF descriptor that represents a service function chain.
- The assurance module validates the **end-to-end behavior** of the deployed service (KPIs observed at service level), which implicitly captures chain effects.

**What is NOT covered:**
- No explicit chain ordering verification — the system does not check whether VNF A's output is always correctly forwarded to VNF B.
- No compositional isolation analysis — NFs in a chain are not verified to be isolated from each other's failures.
- No stateful carry-over verification — the system does not check whether stateful NFs (e.g., a UPF maintaining per-session state) correctly interact across chain boundaries.
- No formal composition proofs — there is no model of chain semantics checked by a solver or proof assistant.

**To extend to chain verification**, one would need to: (a) add a chain topology extraction step, (b) model per-NF input/output contracts, and (c) apply compositional reasoning (e.g., assume-guarantee) to verify inter-NF behavior. This is not part of the current work.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna is a static analysis framework that operates directly on **raw eBPF bytecode** without requiring NF source code. It builds **Control Flow Graphs with Network Context (CFG-NC)** from eBPF programs and performs dataflow analysis to track packet processing behavior across BPF map accesses, helper calls, and tail calls. A **Prolog-based query engine** then evaluates behavioral assertions against the extracted CFG-NC model — enabling chain-level compositional reasoning, stateful map tracking, and property checking over multi-NF eBPF pipelines without execution.

### 5.2 Key Differences from This Paper

| Dimension | Mekrache et al. (I7) | Yaksha-Prashna |
|---|---|---|
| **Input** | Natural-language operator intent | Raw eBPF bytecode (no source needed) |
| **NF representation** | ETSI NSDs / VNF descriptors | eBPF bytecode + CFG-NC model |
| **Analysis technique** | LLM inference + runtime telemetry | Static CFG analysis + Prolog query engine |
| **Validation timing** | Runtime (post-deployment) + lifecycle | Pre-deployment (static) |
| **Guarantees** | Empirical, best-effort, runtime | Static, sound over the bytecode model |
| **Stateful analysis** | Implicit (KPI monitoring) | Explicit BPF map tracking |
| **Chain handling** | Implicit via NSD topology | Explicit multi-NF chain reasoning |
| **Formalism** | None (LLM reasoning is opaque) | Prolog logic + CFG dataflow |
| **Hallucination risk** | High (LLM can generate invalid NSDs) | None (deterministic static analysis) |
| **Scope** | 5G NFV/SDN services | eBPF-based NFs (kernel-level) |
| **Operator expertise needed** | Minimal (natural language) | None (automated, artifact-driven) |

### 5.3 How This Paper Is Useful For Us

1. **Motivation for the NF validation problem**: This paper forcefully articulates why automated NF validation is needed — the complexity of 5G configurations, the scale of modern deployments, and the human expertise bottleneck all motivate the need for automated tools. We can cite this to motivate the broader NF validation research agenda.

2. **Contrast on input requirements**: Mekrache et al. start from natural language and produce configurations that are then deployed and checked empirically. Yaksha-Prashna starts from the already-deployed artifact (eBPF bytecode) and validates it statically before or without deployment. This contrast cleanly defines two ends of the validation spectrum and positions our work as complementary (or necessary for the cases where LLM-generated configurations might be incorrect or unsafe).

3. **Highlighting LLM limitations**: The paper implicitly reveals a critical gap — LLMs can generate NSDs that are syntactically correct but semantically wrong (wrong resource allocations, incorrect chain topologies, missing QoS policies). Yaksha-Prashna addresses this gap for eBPF-based NFs by providing rigorous static analysis of the actual program behavior, independent of how the NF was configured or deployed.

4. **Closed-loop vs. static analysis dichotomy**: This paper represents a pure runtime/empirical validation paradigm. Our work represents a static/formal paradigm. Together, they illustrate the classical dichotomy between dynamic and static verification, allowing us to position Yaksha-Prashna as providing stronger pre-deployment guarantees that complement runtime monitoring.

5. **Reference for Intent-to-NF translation**: If our research considers the validation of NFs deployed as a result of intent-driven management (e.g., NFs deployed by an LLM-based system like this one), we can cite this paper as the deployment source model — "given that NFs may be configured automatically by LLM systems (Mekrache et al., 2024), static verification of the resulting eBPF programs becomes even more critical."

### 5.4 Positioning Statement

> "While Mekrache et al. [I7] demonstrate that LLM-driven intent-based management can automate the lifecycle of 5G NFs from natural-language intent to deployment, their assurance mechanism relies on runtime telemetry comparison — offering no pre-deployment guarantees about the semantic correctness of NF behavior at the bytecode level. Yaksha-Prashna addresses this gap by statically analyzing the eBPF programs that implement such NFs, enabling formal behavioral assertions to be verified before any packet is processed."

---

## 6. Additional Notes

### Related Work by Same Authors
- **"LLM-enabled intent-driven service configuration for next generation networks"** (Mekrache & Ksentini, IEEE NetSoft 2024): A companion paper with more implementation detail. Uses **Code Llama** as the LLM backbone, explicitly employs **few-shot prompting**, and introduces the **Knowledge Base (KB)** architecture for storing validated (intent, NSD) pairs. Validated at EURECOM 5G facility with edge computing deployment.
- **"DMO-GPT: An Intent-driven Framework for Distributed 6G Management and Orchestration"**: Follow-on work extending the architecture toward 6G with multi-agent LLM coordination for distributed domains.

### Open Research Challenges Identified by the Authors
1. **LLM hallucination**: LLMs may generate syntactically valid but semantically incorrect NSDs, particularly for unusual or novel service types.
2. **KB cold-start problem**: Few-shot performance degrades with sparse KB coverage for uncommon intent types.
3. **Multi-domain intent consistency**: Ensuring sub-intents generated for different domains (RAN, Core, Edge) are mutually compatible remains an open problem.
4. **Security of LLM-generated configurations**: Configurations generated by LLMs could inadvertently introduce security misconfigurations. No security verification mechanism is proposed.
5. **Scalability of human feedback**: HF loops require human expert involvement, which does not scale to large deployments.
6. **Ambiguity resolution**: Natural language intents are often ambiguous; the system currently relies on LLM interpretation without formal disambiguation.

### Venue Note
Published in **IEEE Network** — a prestigious IEEE magazine-format publication that prioritizes architectural proposals and forward-looking system designs over theorem proving or deep algorithmic novelty. Results are validated through a real-world deployment demonstration rather than large-scale empirical evaluation.
