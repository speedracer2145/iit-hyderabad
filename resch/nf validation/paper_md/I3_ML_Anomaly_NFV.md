# Machine Learning-Based Anomaly Detection in NFV: Comprehensive Survey

**Authors:** Not individually listed (multi-author survey)  
**Year:** 2023 | **Venue:** Wireless Personal Communications (Springer)  
**DOI/Link:** https://link.springer.com/article/10.1007/s11277-023-10512-y

---

## 1. Overview

Network Function Virtualization (NFV) moves network functions (firewalls, DPI, NAT, IDS, load balancers) from dedicated hardware appliances to software running on commodity servers as Virtual Network Functions (VNFs). While this delivers operational flexibility and cost savings, it also introduces new correctness and reliability challenges: VNFs can fail, misbehave, experience resource contention, or produce anomalous output for reasons invisible to traditional monitoring tools.

This paper is a **comprehensive systematic literature review** of ML-based anomaly detection approaches for NFV environments. It surveys 80+ papers from 2015–2022 and organizes them into a taxonomy covering: (a) supervised learning approaches (SVM, DNN, Random Forest), (b) unsupervised approaches (clustering, autoencoders, isolation forests), (c) semi-supervised approaches (one-class SVM, variational autoencoders), and (d) hybrid approaches. For each category, it analyzes the telemetry features used, the VNF types targeted, detection accuracy, and operational deployability.

The survey is directly relevant to our research as it establishes the landscape of **behavioral monitoring** approaches for deployed NFs — the runtime complement to our static verification approach.

---

## 2. Technical Details

### 2.1 Core Technique / Approach

This is a survey paper, not a systems paper. Its "technique" is a **systematic literature review methodology**:

1. Define search queries across ACM DL, IEEE Xplore, Springer, arXiv
2. Filter by inclusion/exclusion criteria (must involve NFV, VNFs, anomaly detection)
3. Extract data: ML method, input features, VNF type, dataset, evaluation metrics
4. Taxonomize: organize papers into a classification hierarchy
5. Identify open problems and research gaps

The paper does not implement a new system — it synthesizes existing work.

### 2.2 Core Categories of Approaches Surveyed

**Supervised Learning:**
- Inputs: CPU/memory/network telemetry time series, VNF log events, packet-level statistics
- Methods: SVM, Random Forest, LSTM, CNN-LSTM, Transformer
- Detection: classifies telemetry as normal vs. anomalous given labeled training data
- Representative tools: DeepLog, LogRobust, VNF-Checker

**Unsupervised Learning:**
- Inputs: telemetry streams, control plane events
- Methods: K-means, DBSCAN, Isolation Forest, Autoencoders, VAEs
- Detection: identifies statistical outliers without labeled anomaly examples
- Representative: AnomalyBERT adapted for NFV, USAD

**Semi-supervised:**
- Inputs: mix of labeled normal + unlabeled data
- Methods: One-Class SVM, GAN-based, few-shot approaches
- Detection: trained on normal behavior only; anomaly = deviation from normal
- Representative: DeepSVDD adapted for VNF telemetry

**Hybrid:**
- Combine multiple ML methods or combine ML with rule-based approaches
- Example: LSTM for temporal patterns + threshold rules for rapid detection

### 2.3 Telemetry Features Surveyed

The survey categorizes anomaly detection inputs into:
- **Resource metrics:** CPU utilization, memory footprint, disk I/O, NIC queue depths
- **Performance metrics:** Throughput (Mpps), latency (μs), packet loss rate
- **Control plane events:** VNF instantiation, scaling, migration events from MANO (ETSI NFV)
- **Log events:** VNF application logs, kernel logs, SNMP traps
- **Packet-level stats:** Flow byte/packet counts, protocol distribution, connection state distributions

### 2.4 VNF Types Covered

- **Firewall VNFs** (Snort, Suricata as VNFs)
- **IDS/IPS VNFs**
- **vIMS (IP Multimedia Subsystem):** Clearwater, OpenIMS
- **vEPC (Evolved Packet Core):** 4G/5G core functions as VNFs
- **Load balancer VNFs:** HAProxy, Nginx
- **General compute VNFs:** Apache, MySQL in virtualized NF roles

---

## 3. NF Validation & Verification

### 3.1 What NFs Does It Target?

VNFs running in ETSI NFV / OpenStack / Kubernetes environments:
- vIMS (SIP proxy, HSS, P-CSCF, I-CSCF)
- vEPC (MME, SGW, PGW, HSS)
- Virtual firewalls and IDS (Snort/Suricata as VMs)
- Load balancers (HAProxy VNF)
- General purpose VNFs

**NOT:** eBPF/XDP programs, DPDK NFs, or kernel-level data plane NFs.

### 3.2 How It Validates NF Behavior

The surveyed approaches validate NF behavior through **runtime anomaly detection**:

1. **Train:** Build a normal behavior model from telemetry collected during correct operation
2. **Monitor:** Continuously collect runtime telemetry from deployed VNFs
3. **Detect:** Apply trained ML model to telemetry stream — flag deviations from normal
4. **Alert/React:** Trigger MANO healing actions (restart VNF, scale, migrate) on anomaly detection

The "verification" is not formal — it is statistical: "this VNF's behavior is X standard deviations from its learned normal baseline."

### 3.3 What Properties / Invariants Does It (Implicitly) Check?

- **Resource usage invariants:** CPU/memory/network I/O within expected bounds
- **Throughput correctness:** VNF processes packets at expected rate (deviation → bug or overload)
- **Log pattern correctness:** Log event sequences match expected behavioral patterns
- **Connection count invariants:** Number of active connections within expected range for firewall/NAT VNFs
- **SLA compliance:** Latency/throughput meeting Service Level Agreements

### 3.4 Input Requirements

| Input | What's Needed |
|---|---|
| Training telemetry | Historical time series of normal VNF operation |
| Runtime telemetry | Live CPU/memory/network metrics from deployed VNFs |
| ML infrastructure | Training pipeline + inference endpoint |
| MANO integration | For automated healing responses |

### 3.5 Guarantees Provided

- **No formal guarantees** — probabilistic/statistical
- Detection accuracy: typically 90–99% depending on method and VNF type (per surveyed papers)
- False positive rate: major concern across all surveyed approaches
- **No counterexample generation** — cannot explain why a specific packet was processed incorrectly

---

## 4. NF Chain Verification

Most surveyed approaches target **individual VNFs** in isolation. Chain-level analysis is an identified research gap:

- A few papers monitor the NFV service chain end-to-end via SFC OAM probes
- Some use distributed tracing (Jaeger/Zipkin) to correlate anomalies across chained VNFs
- None provide chain-level semantic verification — they can detect "something went wrong in the chain" but not "NF_2's NAT table state is inconsistent with NF_1's firewall decisions"

The survey explicitly identifies **cross-VNF state correlation** as an open problem.

---

## 5. Relevance to Yaksha-Prashna / Our Research

### 5.1 What Yaksha-Prashna Does

Yaksha-Prashna performs static analysis of eBPF bytecode to extract behavioral models (CFG-NC) and answer assertion queries via Prolog. It works before deployment (offline, static) and targets kernel-level eBPF programs.

### 5.2 Key Differences from This Paper

| Dimension | ML Anomaly Survey | Yaksha-Prashna |
|---|---|---|
| **Approach** | Runtime ML monitoring | Static offline analysis |
| **Target** | VNFs (VM-level, cloud) | eBPF kernel programs |
| **Guarantee** | Probabilistic (ML accuracy) | Deterministic (dataflow) |
| **Timing** | Post-deployment (continuous) | Pre-deployment (offline) |
| **Chain support** | Limited (open problem) | Core feature |
| **Explanation** | Anomaly flag only | Behavioral assertion violation with path |

### 5.3 How This Paper Is Useful For Us

1. **Complementarity framing:** ML anomaly detection catches runtime behavioral deviations; YP catches static structural behavioral bugs. They are complementary layers of NF assurance. Cite to position YP as the pre-deployment static layer vs. ML's post-deployment dynamic layer.

2. **Open problem citation:** The survey explicitly identifies chain-level behavioral analysis as an open problem — this directly motivates YP's chain support as addressing a recognized gap.

3. **Coverage of the NF validation space:** Cite this survey in related work to acknowledge runtime approaches and position YP as occupying a different (static, pre-deployment, formal) point in the design space.

4. **VNF types:** The surveyed VNF types (firewall, IDS, NAT, load balancer) are the same functional classes that YP verifies as eBPF programs — demonstrating that the same NF classes need both runtime and static verification.

### 5.4 Positioning Statement

> "While ML-based runtime anomaly detection approaches, as surveyed comprehensively by [I3], provide continuous behavioral monitoring for deployed VNFs, they offer probabilistic detection with no formal guarantees and cannot identify the root cause of behavioral violations. Yaksha-Prashna complements these runtime approaches by providing pre-deployment static verification of eBPF NF behavior — giving developers formal assurance before any packet is processed in production."

---

## Summary Table

| Attribute | Value |
|---|---|
| **Paper type** | Comprehensive survey / systematic literature review |
| **Methods surveyed** | Supervised, unsupervised, semi-supervised, hybrid ML |
| **Input (surveyed systems)** | VNF telemetry: CPU/mem/network/logs |
| **NF types** | VNFs: vIMS, vEPC, vFW, vIDS, vLB |
| **Chain support** | Limited — identified as open problem |
| **Guarantee** | Probabilistic (statistical anomaly detection) |
| **Venue** | Wireless Personal Communications (Springer) 2023 |
| **Cited by YP** | No |
