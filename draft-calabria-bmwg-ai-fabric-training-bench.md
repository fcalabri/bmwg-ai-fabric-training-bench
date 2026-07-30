---
title: "Benchmarking Methodology for AI Training Network Fabrics"
abbrev: "AI Training Fabric Benchmarking"
docname: draft-calabria-bmwg-ai-fabric-training-bench-latest
category: info

submissiontype: IETF
ipr: trust200902
area: "Operations and Management"
workgroup: "BMWG"
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]
v: 3

venue:
  group: BMWG
  type: Working Group
  mail: bmwg@ietf.org
  github: "fcalabri/bmwg-ai-fabric-training-bench"
  latest: "https://fcalabri.github.io/bmwg-ai-fabric-training-bench/draft-calabria-bmwg-ai-fabric-training-bench.html"

author:
  - name: Fernando Calabria
    ins: F. Calabria
    org: Cisco
    country: United States
    email: fcalabri@cisco.com
  - name: Carlos Pignataro
    ins: C. Pignataro
    org: Blue Fern Consulting
    country: United States
    email: carlos@bluefern.consulting
  - name: Qin Wu
    ins: Q. Wu
    org: Huawei
    country: China
    email: bill.wu@huawei.com
  - name: Giuseppe Fioccola
    ins: G. Fioccola
    org: Huawei
    country: Italy
    email: giuseppe.fioccola@huawei.com
  - name: Sowjanya Reddy
    ins: S. Reddy
    org: Apple
    country: United States
    email: sowjredd@gmail.com

normative:
  RFC1242:
  RFC2544:
  RFC6815:
  RFC8238:
  RFC8239:
  RFC9004:
  UEC-1.0:
    title: "Ultra Ethernet Transport (UET) Specification 1.0"
    author:
      - org: Ultra Ethernet Consortium
    date: 2025-06
    target: "https://ultraethernet.org"
  TERMINOLOGY: I-D.calabria-bmwg-ai-fabric-terminology

informative:
  RFC3849:
  INFERENCE-BENCH: I-D.calabria-bmwg-ai-fabric-inference-bench
  EVPN-BENCH:
    title: "Benchmarking Methodology for EVPN and PBB-EVPN"
    author:
      - ins: S. Jacob
        name: Sudhin Jacob
      - ins: K. Tiruveedhula
        name: Kishore Tiruveedhula
    date: 2023-08
    seriesinfo:
      Internet-Draft: draft-ietf-bmwg-evpntest-11
  LLM-BENCH:
    title: "Benchmarking Methodology for Large Language Model Serving"
    author:
      - ins: Gaikwad, et al.
    date: 2026-01
    seriesinfo:
      Internet-Draft: draft-gaikwad-llm-benchmarking-methodology-00
  META-ROCE:
    title: "RDMA over Ethernet for Distributed Training at Meta Scale"
    author:
      - ins: A. Gangidi
        name: Adithya Gangidi
      - ins: R. Miao
        name: Rui Miao
      - ins: S. Zheng
        name: Shengbao Zheng
      - ins: S. J. Bondu
        name: Sai Jayesh Bondu
      - ins: G. Goes
        name: Guilherme Goes
      - ins: H. Morsy
        name: Hany Morsy
      - ins: R. Puri
        name: Rohit Puri
      - ins: M. Riftadi
        name: Mohammad Riftadi
      - ins: A. J. Shetty
        name: Ashmitha Jeevaraj Shetty
      - ins: J. Yang
        name: Jingyi Yang
      - ins: S. Zhang
        name: Shuqiang Zhang
      - ins: M. J. Fernandez
        name: Mikel Jimenez Fernandez
      - ins: S. Gandham
        name: Shashidhar Gandham
      - ins: H. Zeng
        name: Hongyi Zeng
    date: 2024
    seriesinfo:
      "ACM SIGCOMM '24": "Sydney, NSW, Australia"
      DOI: 10.1145/3651890.3672233
  DCQCN-PAPER:
    title: "Congestion Control for Large-Scale RDMA Deployments"
    author:
      - ins: Y. Zhu
        name: Yibo Zhu
      - ins: H. Eran
        name: Haggai Eran
      - ins: D. Firestone
        name: Daniel Firestone
      - ins: C. Guo
        name: Chuanxiong Guo
      - ins: M. Lipshteyn
        name: Marina Lipshteyn
      - ins: Y. Liron
        name: Yehonatan Liron
      - ins: J. Padhye
        name: Jitendra Padhye
      - ins: S. Raindel
        name: Shachar Raindel
      - ins: M. H. Yahia
        name: Mohamad Haj Yahia
      - ins: M. Zhang
        name: Ming Zhang
    date: 2015
    seriesinfo:
      "ACM SIGCOMM": "pp. 523-536"
      DOI: 10.1145/2785956.2787484
  LIBFABRIC:
    title: "libfabric: Open Fabric Interfaces"
    author:
      - org: OpenFabrics Interfaces Working Group
    target: "https://ofiwg.github.io/libfabric/"
  MLPERF:
    title: "MLPerf Training Benchmark Suite"
    author:
      - org: MLCommons
    target: "https://mlcommons.org"

...

--- abstract

This document defines benchmarking terminology, methodologies, and Key Performance Indicators (KPIs) for evaluating Ethernet-based AI training network fabrics.

As large-scale distributed Artificial Intelligence / Machine Learning (AI/ML) training clusters grow to tens of thousands of accelerators (GPUs or generic accelerator processing units (XPUs)), the backend network fabric determines Job Completion Time (JCT), training throughput, and accelerator utilization.

This document establishes vendor-independent, reproducible test procedures for benchmarking fabric-level performance under realistic AI training workloads. The tests cover Remote Direct Memory Access (RDMA) over Converged Ethernet version 2 (RoCEv2) transport, the Ultra Ethernet Transport (UET) protocol defined by the Ultra Ethernet Consortium (UEC) Specification 1.0, congestion management (Priority Flow Control (PFC), Explicit Congestion Notification (ECN), Data Center Quantized Congestion Notification (DCQCN), Credit-Based Flow Control (CBFC)), load balancing strategies (Equal-Cost Multi-Path (ECMP), Dynamic Load Balancing (DLB), packet spraying), collective communication patterns (AllReduce, AllToAll, AllGather), and scale/soak testing.

The methodology enables direct, reproducible comparison across switch ASICs, NIC transport stacks (RoCEv2 and UET), and fabric architectures (2-tier Clos, 3-tier Clos, and rail-optimized).

--- middle

# Introduction

Distributed AI/ML training workloads impose traffic requirements that standard data center fabrics were not designed to meet. Traditional data center traffic varies in flow size and protocol mix. AI training generates synchronized, bandwidth-intensive east-west traffic dominated by collective communication operations: AllReduce, AllToAll, and AllGather. These workloads require RDMA transport with negligible application-visible loss, bounded tail latency, uniform load distribution across all fabric paths, and the ability to absorb coordinated micro-bursts from thousands of accelerators simultaneously. RoCEv2 deployments typically meet the loss requirement by operating the fabric lossless (PFC/ECN); UET is designed to tolerate wire-level loss and recover via retransmission and packet trimming (see the Zero Packet Loss definition in {{TERMINOLOGY}}).

Existing BMWG methodologies do not address AI training fabrics. {{RFC2544}} defines benchmarking for general network interconnect devices but does not account for RDMA transport semantics, collective communication patterns, or the congestion behavior specific to GPU-to-GPU traffic. {{RFC8238}} and {{RFC8239}} establish data center benchmarking terminology and methodology but predate large-scale RoCEv2 deployment and do not address Priority Flow Control (PFC) interactions, DCQCN congestion control convergence {{DCQCN-PAPER}}, or the impact of load balancing strategies on Job Completion Time (JCT). Industry experience deploying RoCEv2 at scale {{META-ROCE}} shows the need for a standardized benchmarking methodology.

The Ethernet Virtual Private Network (EVPN) benchmarking methodology {{EVPN-BENCH}} provides a structural template for service-oriented benchmarking but is scoped to L2VPN services rather than RDMA fabrics.

This document defines a benchmarking methodology for AI training network fabrics.

## Requirements Language

{::boilerplate bcp14-tagged}

## Scope and Applicability

This document applies to Ethernet-based AI training backend network fabrics employing RoCEv2 and/or UEC Ultra Ethernet Transport (UET) protocols. The scope includes leaf-spine (2-tier Clos) and leaf-spine-superspine (3-tier Clos) topologies.

InfiniBand fabrics are explicitly **out of scope**, though many KPIs defined herein may be adapted for IB benchmarking by future documents. The DUT is the network fabric itself (the collection of switches and interconnecting links), not individual accelerators or host NICs; host-side configuration is documented in the test report as it materially affects results.

The DUT boundary for all measurements in this document is the NIC-to-NIC Ethernet fabric segment.  Intra-node communication (proprietary accelerator interconnects, e.g., NVLink, Infinity Fabric/xGMI, or PCIe) and individual GPU/accelerator performance are explicitly out of scope.
Collective operation measurements (AllReduce, AllGather, AllToAll) are measured at the Ethernet fabric boundary; intra-node accelerator-interconnect contributions are reported separately when characterizing wide Expert Parallelism (wide-EP) or multi-node configurations. The reporting requirements that make this separation auditable, and the normalization basis that keeps it intact, are specified in {{comparability-normalization}}.

The methodology is designed for controlled laboratory environments per the BMWG charter; it is NOT intended for production network measurement.

## Relationship to Existing and Companion Work

| Document | Relationship |
|---|---|
| {{RFC1242}} | Base terminology for network benchmarking; terms reused herein |
| {{RFC2544}} | Base methodology; throughput/latency/loss tests adapted for RDMA |
| {{RFC8238}} | Data center terminology; buffer, congestion, and microburst terms extended |
| {{RFC8239}} | Data center methodology; line-rate and buffer tests adapted for RoCEv2 |
| {{RFC9004}} | Back-to-back frame updates; burst absorption methodology referenced |
| {{LLM-BENCH}} | Complementary document benchmarking the inference serving stack. Treats the network as opaque SUT; this document benchmarks the fabric itself. The two documents MAY be used together but MUST NOT be combined in a single benchmarking report without explicit section demarcation. |
| {{TERMINOLOGY}} | Companion document; normative source for the terminology used throughout this document |
| {{INFERENCE-BENCH}} | Companion fabric-level benchmarking methodology addressing AI inference serving workloads |
| {{UEC-1.0}} | UET protocol specification; transport services, congestion control, and link-layer enhancements benchmarked in {{test-uec}} |
{: #tab-existing-work title="Relationship to Existing and Companion Work"}

# Terminology

Terminology used in this document is defined in {{TERMINOLOGY}}. Readers should consult that document before applying the methodology defined here. Where a term overlaps with {{RFC1242}} or {{RFC8238}}, the terminology document provides AI fabric context extensions; the foundational definitions in those RFCs remain authoritative for general network benchmarking.

All terminology used in this document – including the AI fabric, RoCEv2, UET, RDMA transport, congestion control (PFC, DCQCN, ECN, CBFC), load balancing (ECMP, Packet Spray, DLB/Flowlet), collective communication, and KPI vocabulary (JCT, JCT Ratio, BusBW, MMR, etc.) – is defined normatively in {{TERMINOLOGY}} and is not redefined here. The following table lists the single bench-specific extension introduced by this document:

| Term | Definition |
|---|---|
| **PFC Pause Event** | A single PFC PAUSE frame with a non-zero quanta value transmitted on a priority class. Zero-quanta (X-ON / resume) frames are excluded from the count, since implementations that signal resume explicitly would otherwise report roughly double the event rate of implementations that let the pause timer expire; X-ON frames are instead used to bound the paused interval for the cumulative-duration metric. Used in this document as the unit of count for PFC event-rate metrics (events/sec, cumulative duration) reported by the methodology in {{test-congestion}}. |
{: #tab-terminology title="Bench-Specific Terminology Extensions"}

In addition to the BusBW reporting requirements specified in {{TERMINOLOGY}}, the runtime algorithm selected by the collective library MUST be verified via library tracing and documented as part of the test conditions for any AllReduce, AllGather, or AllToAll benchmark in this document.

The scope of the DUT for the tests defined in this document is the set of leaf switches, spine switches, superspine switches (if applicable), and interconnecting links forming the AI training fabric, consistent with the Fabric DUT Boundary defined in {{TERMINOLOGY}}.

## Acronyms

Acronyms used in this document are expanded in the Acronyms appendix of {{TERMINOLOGY}}. Acronyms unique to the methodology defined herein are expanded on first use in the body of this document.

# Test Topology and Architecture

## Reference Fabric Topologies {#reference-fabric-topologies}

Three reference topologies are defined. Every test report identifies which topology was used. Results obtained under different topologies are compared only under the conditions specified in {{comparability-normalization}}, which defines the normalization basis used by this methodology and the parameters that MUST match before two results are compared.

### Topology A: 2-Tier Clos (Leaf-Spine)

~~~ ascii-art
+--------+ +--------+ +--------+ +--------+
| Spine1 | | Spine2 | | Spine3 | | SpineN |
+---++---+ +---++---+ +---++---+ +---++---+
    ||          ||          ||          ||
    ||    Full Mesh Interconnect        ||
    ||    (ECMP / DLB / Spray)         ||
    ||          ||          ||          ||
+---++---+ +---++---+ +---++---+ +---++---+
| Leaf 1 | | Leaf 2 | | Leaf 3 | | Leaf N |
+---++---+ +---++---+ +---++---+ +---++---+
    ||          ||          ||          ||
[GPU/XPU]  [GPU/XPU]  [GPU/XPU]  [GPU/XPU]
Hosts w/   Hosts w/   Hosts w/   Hosts w/
RoCEv2 NIC             RoCEv2 NIC
~~~
{: #fig-topo-a align="center" title="Topology A: 2-Tier Clos (Leaf-Spine)"}

The DUT boundary encompasses all leaf and spine switches and their interconnecting links. Traffic generators or actual GPU hosts connect at the leaf layer.

### Topology B: 3-Tier Clos (Leaf-Spine-Superspine)

For clusters exceeding thousands of accelerators, a superspine layer is added. Each pod consists of a leaf-spine fabric; pods interconnect via superspine switches. This topology scales to 32,000+ accelerators at 800GbE with current-generation ASICs. The DUT boundary encompasses all three tiers.

### Topology C: Rail-Optimized

~~~ ascii-art
                       SPINE LAYER
+--------+ +--------+ +--------+ +--------+
| Spine1 | | Spine2 | | Spine3 | | SpineN |
+--+--+--+ +--+--+--+ +--+--+--+ +--+--+--+
 |     Full Mesh Interconnect (ECMP/Spray)  |
+--+--+--+ +--+--+--+ +--+--+--+ +--+--+--+
| Rail-0 | | Rail-1 | | Rail-2 | | Rail-7 |  RAIL (LEAF) LAYER
|  Leaf  | |  Leaf  | |  Leaf  | |  Leaf  |  one switch per NIC
+--+--+--+ +--+--+--+ +--+--+--+ +--+--+--+
  |   |       |   |       |   |      |   |
NIC-0 NIC-0 NIC-1 NIC-1 NIC-2 NIC-2 NIC-7 NIC-7
  |   |       |   |       |   |      |   |
+--------+ +--------+ +--------+ +--------+
| Host A | | Host B | | Host C | | Host D |  GPU HOSTS
| GPU[0] | | GPU[0] | | GPU[0] | | GPU[0] |  (each host has
| GPU[1] | | GPU[1] | | GPU[1] | | GPU[1] |   8 NICs, one
|  ...   | |  ...   | |  ...   | |  ...   |   per rail)
| GPU[7] | | GPU[7] | | GPU[7] | | GPU[7] |
+--------+ +--------+ +--------+ +--------+
~~~
{: #fig-topo-c align="center" title="Topology C: Rail-Optimized (schematic; only 4 of N rails and hosts shown)"}

In rail-optimized topologies, each NIC on a multi-NIC host connects to a dedicated leaf switch ("rail"); this co-optimizes network locality with the collective communications library (CCL) in use (e.g., NCCL, RCCL, oneCCL). The diagram is schematic: it depicts a representative subset of rails and hosts, not the full cross-host fan-out; in the complete topology, the Rail-N leaf switch connects to GPU[N] of every host. The DUT boundary and rail mapping are fully documented in the test report.

## Result Comparability and Normalization {#comparability-normalization}

{{reference-fabric-topologies}} requires that results obtained under different topologies be compared only after normalization. This section defines what that normalization is, which parameters MUST be equal before two results are compared, and which quantities MUST NOT be used as normalization factors.

### Normalization Basis

Normalization in this document is achieved by reporting KPIs that are dimensionless or referenced to a rate observable at the Fabric DUT Boundary defined in {{TERMINOLOGY}}. It is not achieved by equalising fabric hardware between the systems being compared.

| KPI | Defined in | Normalizing denominator |
|---|---|---|
| JCT Ratio | {{synthetic-jct-under-controlled-conditions}} | Roofline_seq, whose communication term is (8 × S × algo_factor) / B_acc |
| BusBW | {{test-collective}} | Per-accelerator; made algorithm-invariant by the fixed algo_factor of {{TERMINOLOGY}} |
| BusBW efficiency | {{allreduce-benchmark}}, {{allgather-benchmark}} | Per-accelerator NIC line rate |
| Throughput efficiency | {{baseline-throughput}} | Theoretical rate of the Ethernet links under test |
| Aggregate Throughput | {{kpi-framework-and-metrics-taxonomy}} | Fabric bisection bandwidth, computed from Ethernet link rates |
| MMR, JFI | {{test-lb}} | Dimensionless by construction |
| JCT Interference Factor | {{multi-tenant-jct-interference}} | Baseline JCT measured on the same DUT |
{: #tab-normalization-basis title="Normalization Basis for Reported KPIs"}

Each denominator in {{tab-normalization-basis}} is either a workload parameter fixed by the test operator (S, algo_factor, N) or an Ethernet line rate at the DUT boundary. No denominator contains a term for switch internal capacity, accelerator interconnect capacity, or host bus capacity.

Convergence and failover times ({{link-failure-convergence}}, {{zero-impact-failover-measurement}}), latency percentiles ({{latency-characterization}}), and queue occupancy are reported in absolute units and are not normalized; normalizing them would obscure the behaviour they measure.

### Aggregate Switching Capacity Is Not a Normalization Factor

Aggregate switching capacity (ASIC forwarding capacity, in Tbps) MUST NOT be used as a normalization factor for any KPI defined in this document, and MUST NOT be held constant as a precondition for comparing two fabrics, for three reasons:

1. Aggregate switching capacity is a vendor-declared internal property, not a quantity observable at the Fabric DUT Boundary. Benchmarking in this document is performed on a black-box basis ({{security-considerations}}).

2. Whether a given switching capacity is sufficient for the offered collective pattern is what the tests in {{test-rdma}} through {{test-soak}} measure. Dividing a measured result by the capacity that produced it removes the effect under test: a fabric with half the switching capacity that completes the same job in 1.6 times the time would report a better result per Tbps while performing worse at the job.

3. A capacity-normalized value is a cost or efficiency figure of merit. Per the BMWG charter, and consistent with {{reporting}} and {{indicative-reference-values}}, this document defines what is measured and how it is reported, and does not define figures of merit or acceptance criteria.

Aggregate switching capacity, per-port speed, switch radix, and buffer architecture are reported as DUT characteristics ({{device-under-test-dut-identification}}, {{asic-features}}) so that a reader can attribute an observed difference to them. They are inputs to the interpretation of a result, not divisors applied to it.

The same reasoning applies to intra-node switching and interconnect capacity (accelerator interconnects, PCIe, CXL). Were aggregate switching capacity used as a normalization factor, intra-node switching and interconnect capacity would have to be included for that normalization to be complete, which the DUT boundary of {{scope-and-applicability}} does not permit. Because no denominator in {{tab-normalization-basis}} contains such a term, the normalization defined here does not reach inside the node, and this section and {{scope-and-applicability}} are consistent by construction.

### Comparability Set

Results obtained on two different fabrics are compared directly for the KPIs in {{tab-normalization-basis}} when every parameter in {{tab-comparability-set}} is equal and reported for both results.

| Parameter | Why it must match |
|---|---|
| Participating accelerator count N | Collective cost depends on N through algo_factor and through fabric path length |
| Accelerators per node, and their placement and rail mapping | Determines the fabric-visible fraction of collective traffic ({{fabric-visible-data-volume}}) |
| B_acc, the per-accelerator Ethernet line rate at the DUT boundary | Denominator of Roofline_seq and of BusBW efficiency |
| Leaf oversubscription ratio | Determines whether the fabric can satisfy the offered pattern |
| Collective type, message size S, and algo_factor | Definition of the offered workload |
| Transport (RoCEv2 or UET), and for UET the transport service | Loss and retransmission semantics differ by transport |
| Load balancing strategy | Comparisons are made strategy by strategy |
{: #tab-comparability-set title="Comparability Set"}

When any parameter in {{tab-comparability-set}} differs, the results are reported side by side with the difference stated, and are not combined into a single comparative figure.

Topology class is a reported test condition, not a parameter that normalization removes. Two fabrics of different topology class that match on {{tab-comparability-set}} are compared for the KPIs in {{tab-normalization-basis}}, and the report states, for each result: the topology class; the switch tier count; the typical and worst-case hop count for the measured traffic pattern; the number of equal-cost paths available between a source-destination pair; and the bisection bandwidth. Such a comparison characterizes the topologies under the offered workload; it does not attribute the observed difference to any single fabric component.

Topologies outside the reference set of {{reference-fabric-topologies}}, including mesh, torus, and direct-connect fabrics, are outside the scope stated in {{scope-and-applicability}}. Results obtained on such a fabric are reported as a deviation per {{reporting}}, and the hop-count and equal-cost-path descriptors above are supplied only where they are well defined for that topology.

### Comparing Fabrics of the Same Topology Class

**Different scale.** Comparison is performed at equal N, and scale is characterized rather than normalized away. {{synthetic-jct-under-controlled-conditions}} requires JCT Ratio to be reported for each N in its parameter table and plotted against N; the shape of that curve is the scaling result, and no single scalar replaces it. Two fabrics whose maximum supported scale differs are compared over the range of N that both support; the report gives the JCT Ratio at the largest common N together with the maximum N each fabric sustains per {{fabric-scale-limits}}.

**Different switch capacity, radix, or buffer size.** Comparison is performed without adjustment, provided {{tab-comparability-set}} matches. The difference in the measured KPIs is the result of the comparison. The differing DUT characteristics are reported per {{device-under-test-dut-identification}} and {{asic-features}} to permit attribution, and are not used to scale the measured values.

### Fabric-Visible Data Volume {#fabric-visible-data-volume}

Collective placement determines how much of a collective's traffic crosses the DUT boundary. A hierarchical AllReduce that reduces within a node before reducing across the fabric presents less data to the fabric than a flat AllReduce across the same N accelerators, even though the application-level message size S is identical. Two results obtained under different placement therefore represent different offered fabric workloads at equal S.

For every collective result reported under {{test-collective}}, {{uet-collective-communication-performance}}, or {{test-jct}}, the report states:

- S, the application-level data size per participant;
- S_fabric, the data volume per participant that crosses the Fabric DUT Boundary, together with the method used to obtain it. Measurement from NIC Ethernet port counters is preferred; derivation from the collective algorithm and the placement is acceptable when the derivation is stated;
- the accelerator placement and, in rail-optimized topologies, the rail mapping;
- the Intra-Node Transfer Overhead component defined in {{TERMINOLOGY}}, reported separately. This component is never added to, subtracted from, or folded into a fabric KPI.

Comparisons use the same S and the same placement on both sides. When placement cannot be matched, for example when comparing a rail-optimized fabric using rail-aware placement against a Clos fabric that has no rail structure, the report gives S_fabric for each result so that the difference in offered fabric work is visible, and the results are not presented as an equal-workload comparison.

## Device Under Test (DUT) Identification {#device-under-test-dut-identification}

| Parameter | Description | Example |
|---|---|---|
| Switch Vendor/Model | Vendor name, product family, model number | Vendor Family Model |
| Switch ASIC | Silicon vendor, ASIC family, revision | Silicon Vendor ASIC Family Rev |
| NOS Version | Network operating system name and version | NOS Name Version |
| Port Speed | Per-port line rate | 400GbE, 800GbE |
| Buffer Architecture | Shared/dedicated, total buffer per ASIC/port | 32MB shared + 16MB VOQ per port |
| Optics/Cables | Transceiver type, cable type and length | Octal Small Form-factor Pluggable (OSFP) 400G-DR4, Direct Attach Copper (DAC) 3m cable |
| NIC Vendor/Model | RDMA NIC vendor, model, firmware | NIC Vendor Model Speed |
| NIC Firmware | NIC firmware version | Firmware Version |
| Host Config | OS, CCL lib version, driver, BIOS settings | OS Version, CCL Version, OFED Version |
{: #tab-dut-id title="DUT Identification Parameters"}

## Traffic Generator Requirements {#traffic-generator-requirements}

### Mandatory Functional Capabilities

The traffic generator supports: RoCEv2 transport emulation (QP establishment, RDMA Write/Read, ECN processing, DCQCN rate control); configurable QP scaling (1-256 QPs per source-destination pair); programmable collective communication patterns (AllReduce, AllToAll, AllGather with configurable message sizes); and nanosecond-precision timestamping.

### Minimum Measurement Accuracy Requirements

| Parameter | Minimum Requirement |
|---|---|
| Timestamp accuracy | ≤ 100 nanoseconds |
| Frame rate accuracy | +/- 0.1% of specified rate |
| QP scaling range | 1 to 256 QPs per src-dst pair |
| Message size range | 64 B to 2 GB (single-message ceiling; see note) |
| Flow counter resolution | Per-flow byte and packet counts |
| Loss measurement | Exact per-packet loss counting |
| Burst generation | Burst lengths at line rate sufficient to exceed DUT buffering; configurable beyond 1000 frames |
{: #tab-tgen-accuracy title="Minimum Measurement Accuracy Requirements"}

NOTE: A single RDMA message cannot exceed the 32-bit RETH DMA Length field, an
absolute ceiling of 4 GB, and implementations commonly advertise a practical
max_msg_sz of 1-2 GB. Transfers larger than the single-message ceiling are
composed from multiple RDMA messages, and the message count and per-message
size are reported alongside the aggregate transfer size.

### Acceptable Implementations

The platform used is identified in all test reports.

**(a) Hardware Traffic Generator** – dedicated hardware capable of line-rate RDMA emulation meeting the Measurement Accuracy Requirements specified in this document. Suitable for point-to-point RDMA tests ({{test-rdma}} and {{test-uec}}).  For collective tests ({{test-collective}}), the following limitations are documented: whether synchronization barriers are reproduced, whether flow patterns are schedule-driven or gradient-driven, and whether straggler behavior is modeled.

**(b) Accelerator Cluster** – cluster running an actual collective communication library with RDMA tooling.  Preferred for the collective benchmarks in {{test-collective}}.  Host configuration (accelerator model, collective library name and version, PCIe topology, BIOS power management settings) is documented.  Any non-fabric overhead in timing measurements is quantified and reported separately.

When a hardware generator is used for collective benchmarks, results should be cross-validated against an accelerator cluster at one or more overlapping (message_size, N) configurations.

Discrepancies exceeding 10% in BusBW or JCT Ratio are investigated and reported.

# KPI Framework and Metrics Taxonomy {#kpi-framework-and-metrics-taxonomy}

> NOTE: Per BMWG charter, the definition of acceptance criteria or performance requirements is explicitly outside the scope of this Working Group. The KPI tables in this section define what is measured and how it is reported; they do not set pass/fail criteria. Indicative non-normative reference values reflecting current industry observations are provided in {{indicative-reference-values}}; those values MUST NOT be used as pass/fail criteria in vendor evaluations.

## Primary KPIs

| KPI | Unit | Definition |
|---|---|---|
| Job Completion Time (JCT) | seconds | Wall-clock time for benchmark iteration (compute + communication) |
| JCT Ratio | dimensionless | Measured JCT / Roofline JCT |
| Bus Bandwidth (BusBW) | Gbps/accelerator | Effective per-accelerator throughput during collective. See the BusBW definition in {{TERMINOLOGY}} |
| Aggregate Throughput | Tbps | Total fabric goodput during collective phase |
| Packet Drop Rate | ppm | Frames lost end-to-end not retransmitted |
| Tail Latency (P99/P99.9) | us | 99th/99.9th percentile one-way fabric latency |
{: #tab-primary-kpis title="Primary KPIs"}

## Secondary KPIs

| KPI | Unit | Definition |
|---|---|---|
| ECN Marking Ratio | % | Percentage of packets marked CE over measurement interval |
| PFC Pause Count | events/sec | Rate of PFC PAUSE frames per priority per port |
| PFC Pause Duration | us | Cumulative time a port is in PFC-paused state per interval |
| RDMA Retransmission Rate | retx/sec | NIC-level retransmissions due to timeouts or NAKs |
| ECMP Imbalance (MMR) | dimensionless | Max-Mean Ratio of flow counts across parallel uplinks |
| Jain's Fairness Index (JFI) | 1/N-1.0 | Fairness of traffic distribution; 1.0 = perfect, 1/N = worst (N = number of parallel links) |
| Queue Depth (P95/Max) | bytes or cells | 95th percentile and maximum egress queue occupancy per port |
| Congestion Control Convergence | us | Time from congestion onset to DCQCN rate stabilization |
| Out-of-Order Packet Rate | pkt/sec | Packets delivered out of sequence (relevant for packet spray) |
{: #tab-secondary-kpis title="Secondary KPIs"}

## Fabric Health Indicators

| Indicator | Unit | Definition |
|---|---|---|
| Switch CPU Utilization | % | Average and peak CPU usage on DUT control plane during test |
| Switch Memory Utilization | % | Average and peak memory usage, including FIB/MAC table occupancy |
| Forwarding Information Base (FIB) / Route Convergence Time | ms | Time to converge routing after topology change |
| Link Flap Count | events | Spurious link state changes during test period |
| CRC/FCS Error Rate | errors/sec | Physical layer errors indicating cable or optics issues |
| Power Consumption | Watts | Per-switch and per-port power draw under test load |
{: #tab-fabric-health title="Fabric Health Indicators"}

# Test Category 1: RDMA Transport Benchmarks {#test-rdma}

These tests establish baseline fabric performance for RDMA traffic independent of collective communication patterns. They extend {{RFC2544}} and {{RFC8239}} methodology for RoCEv2 semantics.

## Baseline Throughput {#baseline-throughput}

**Objective:** Determine the maximum sustainable RDMA Write throughput through the DUT fabric at each tested message size.

**Procedure:**

- Configure N host pairs, each establishing Q Queue Pairs per pair
- Initiate RDMA Write operations and measure aggregate goodput
- Each test runs for at least 60 seconds at each rate
- Binary search per {{RFC2544}} Section 26.1 is used
- Message sizes: 64B, 256B, 1KB, 4KB, 64KB, 256KB, 1MB, 4MB
- QP counts: 1, 4, 16, 32 per src-dst pair
- Test both unidirectional and bidirectional traffic

**Reporting:** Report aggregate throughput (Tbps), per-port utilization (%), and throughput efficiency (measured/theoretical). Present as table indexed by message size × QP count, and as graph (message size on X-axis).

## Latency Characterization {#latency-characterization}

**Objective:** Determine one-way and round-trip RDMA latency distribution at the throughput rate from {{baseline-throughput}}.

**Procedure:**

- Inject tagged frames at 60s into a 120s stream (per {{RFC2544}} Section 26.2)
- Nanosecond-precision timestamping
- Reported statistics: min, mean, P50, P95, P99, P99.9, max
- Each run MUST capture at least 10,000 latency samples to support a statistically meaningful P99.9
- Repeat at least 20 times; pool all samples across runs and compute percentiles from the pooled distribution, and report the per-run variability (e.g., min/max across runs) alongside the pooled statistics
- Test under both zero-load (single QP) and loaded (full fabric utilization) conditions

**Reporting:** Tabulate latency statistics per message size. Provide histogram and CDF plot. Report latency increase factor (loaded/unloaded).

## Back-to-Back Burst Absorption {#back-to-back-burst-absorption}

**Objective:** Characterize the DUT fabric's ability to absorb back-to-back RDMA bursts without loss. This test extends {{RFC9004}} methodology for RoCEv2.

**Procedure:**

- Transmit bursts at line rate with minimum inter-frame gap
- Increase burst length until first frame loss is detected
- Test incast ratios: 2:1, 4:1, 8:1, 16:1, 32:1
- Repeat at least 50 times per burst length

**Reporting:** Report burst absorption capacity (frames and bytes) for each message size and incast ratio. Plot burst capacity vs. incast ratio.

# Test Category 2: UEC Transport Protocol Benchmarks {#test-uec}

The Ultra Ethernet Consortium (UEC) Specification 1.0 {{UEC-1.0}} defines UET, an RDMA transport positioned as an alternative to RoCEv2 for AI/HPC workloads. All UET tests use the libfabric API {{LIBFABRIC}} and run on UEC 1.0-compliant NICs.

The UEC compliance profile (AI Base, AI Full, or HPC) used during testing is documented in the test report.

## UET Throughput by Transport Service {#uet-throughput-by-transport-service}

**Objective:** Determine maximum sustainable throughput under each UET transport service (ROD, RUD, RUDI, UUD) and compare to RoCEv2 Reliable Connected (RC) / Unreliable Connected (UC) on the same DUT fabric.

**Procedure:** Use UEC 1.0-compliant NICs; establish PDCs; use libfabric fi_write. Apply binary search ({{RFC2544}} Section 26.1). Vary PDC counts: 1, 4, 16, 32. A parallel RoCEv2 test series is executed for comparison. Both unidirectional and bidirectional configurations are tested.

**Reporting template:**

| Metric | ROD | RUD | RUDI | UUD | RoCEv2 RC | RoCEv2 UC |
|---|---|---|---|---|---|---|
| Throughput @ 1MB (Gbps) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| Throughput @ 4MB (Gbps) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| Efficiency (% line rate) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| Connection/PDC Initiation Latency (us) (see note) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| Max Sustained PDC/QP Count | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
{: #tab-uet-throughput title="UET Throughput by Transport Service"}

NOTE: For RoCEv2 RC/UC, this measures QP connection setup time (handshake round-trip). For UET (ROD/RUD/RUDI/UUD), PDCs have no separate setup handshake; this measures the first-packet latency reflecting in-band PDC initiation instead.

## UET Latency Characterization {#uet-latency-characterization}

**Objective:** Measure latency distribution for UET transport services; quantify differential vs. RoCEv2, with particular attention to connectionless PDC establishment overhead.

**Procedure:** Measure latency for: (a) steady-state PDC transfers; (b) first-packet latency (PDC + first data packet, measuring "data before handshake"); (c) zero-load baseline. Test ROD and RUD separately to isolate reordering-related latency.

**Reporting:** Tabulate latency statistics per (transport_service, message_size, load_condition) tuple. Plot latency CDF for UET ROD, UET RUD, and RoCEv2 RC side-by-side.

## Packet Spray Efficacy Under UET RUD {#packet-spray-efficacy-under-uet-rud}

**Objective:** Quantify the load balancing improvement achieved by UET's native per-packet spray with RUD, which eliminates the receiver reorder buffer constraint.

**Procedure:** Test five configurations:

- UET RUD + packet spray
- UET ROD + packet spray
- RoCEv2 RC + packet spray
- RoCEv2 RC + standard ECMP (baseline)
- UET RUD + DLB/Flowlet

Measure MMR, JFI, out-of-order delivery rate, retransmission rate, and effective goodput. Vary ECMP paths: 4, 8, 16, 32.

**Reporting template:**

| Load Balancing Config | MMR | JFI | OOO Rate | Retx Rate | Effective Goodput (%) |
|---|---|---|---|---|---|
| UET RUD + Packet Spray | (meas) | (meas) | (meas) | (meas) | (meas) |
| UET ROD + Packet Spray | (meas) | (meas) | (meas) | (meas) | (meas) |
| RoCEv2 RC + Packet Spray | (meas) | (meas) | (meas) | (meas) | (meas) |
| RoCEv2 RC + ECMP (baseline) | (meas) | (meas) | (meas) | (meas) | (meas) |
| UET RUD + DLB/Flowlet | (meas) | (meas) | (meas) | (meas) | (meas) |
{: #tab-uet-spray title="Packet Spray Efficacy Under UET RUD"}

> This test evaluates whether UET RUD achieves zero host-visible reordering despite per-packet spray, since the transport layer is designed to tolerate unordered delivery.

## UET Congestion Control Benchmarks {#uet-congestion-control-benchmarks}

**Objective:** Evaluate UET's dual-sided (sender + receiver) congestion control under N:1 incast conditions vs. RoCEv2 DCQCN.

**Procedure:** Measure: (a) incast throughput at N = {2, 4, 8, 16, 32, 64}; (b) convergence time after doubling active senders (until all flows within 10% of fair share); (c) PFC avoidance with PFC disabled on the DUT; (d) receiver credit utilization.

**Reporting:** Tabulate incast throughput, convergence time, peak queue depth, PFC event count, and packet drop rate for UET vs. DCQCN per incast ratio. **Key metric:** report whether UET achieves zero application-visible loss without PFC.

## Link-Layer and Network-Layer Enhancement Benchmarks {#link-layer-enhancement-benchmarks}

**Objective:** Measure performance impact of optional UEC enhancements: LLR and CBFC (link-layer) and Packet Trimming (PT, network-layer).

**Procedure:**

- **(a) LLR Retry Latency:** inject controlled bit errors; measure LLR retry latency (expected sub-microsecond per hop) vs. transport-layer retransmission (~10-100us RTT). Run with 80% background load.
- **(b) Packet Trimming Effectiveness:** configure 2:1 oversubscription bottleneck; measure time from congestion onset to first retransmission request, bandwidth saved vs. full-packet drops.
- **(c) CBFC vs. PFC:** identical N:1 (N=32) incast scenarios; measure head-of-line blocking duration (CBFC is per-destination, PFC is per-priority), pause propagation hops, and throughput of non-congested flows.

**Reporting:** Before/after comparison table for each enhancement. Note which features are hardware-supported vs. software-emulated.

## UET Collective Communication Performance {#uet-collective-communication-performance}

**Objective:** Measure collective communication (AllReduce, AllToAll, AllGather) performance over UET and compare directly to RoCEv2, isolating the transport protocol contribution to collective efficiency.

**Procedure:** Execute the collective benchmark suite from {{test-collective}} over UET RUD transport using a UEC-compliant collective library. The same accelerator count (N), message sizes, and fabric topology are used for both UET and RoCEv2 runs to ensure a valid comparison. Run UET RUD + packet spray as the primary configuration and UET ROD + ECMP as the secondary baseline.

For AllReduce, the UET TSS group-key encryption state (active or inactive) on the DUT NIC is documented as a required result field in the test report.

When UET TSS group-key encryption is active during testing, report the observed BusBW computed from measured bytes transferred per the algo_factor formula defined in {{TERMINOLOGY}} (fixed per collective type); group-key encryption affects per-packet security processing overhead, not the transfer volume itself.

The runtime algorithm in use is reported per message-size bucket. See {{TERMINOLOGY}} for the BusBW definition and algo_factor values.

 **Reporting:**  Report the percentage improvement in BusBW and JCT attributable to UET native packet spray and congestion control.

**Reporting template:**

| Collective | Msg Size | N Accels | UET RUD BusBW | UET ROD BusBW | RoCEv2 RC BusBW | Delta UET/RoCEv2 |
|---|---|---|---|---|---|---|
| AllReduce | 1GB | 128 | (meas) | (meas) | (meas) | (meas) |
| AllReduce | 1GB | 512 | (meas) | (meas) | (meas) | (meas) |
| AllToAll | 1GB | 128 | (meas) | (meas) | (meas) | (meas) |
| AllGather | 1GB | 128 | (meas) | (meas) | (meas) | (meas) |
{: #tab-uet-collective title="UET Collective Communication Performance"}

## UET PDC Scalability and Connection Setup Rate {#uet-pdc-scalability-and-connection-setup-rate}

**Objective:** Measure PDC establishment rate and maximum concurrent PDC count vs. RoCEv2 QP-based connections.

**Procedure:** (a) PDC establishment rate: initiate PDC creation to M = {100, 1000, 10000, 100000} remote endpoints. (b) Data-before-handshake: measure first-byte latency for UET vs. RoCEv2 RDMA Write. (c) Maximum concurrent PDC count: scale until per-PDC throughput drops below 90% of single-PDC rate. The UEC specification {{UEC-1.0}} targets millions of endpoints.

NOTE: PDC and QP scaling limits are a host NIC capability, outside the DUT boundary per {{TERMINOLOGY}} and {{scope-and-applicability}}. They are reported as context because a NIC-side PDC/QP ceiling presents as fabric underperformance in the throughput and latency measurements above.

# Test Category 3: Congestion Management {#test-congestion}

AI training workloads generate repetitive micro-congestion during the back-propagation gradient synchronization phase.

## ECN Marking Accuracy and Threshold {#ecn-marking-accuracy-and-threshold}

**Objective:** Verify that the DUT marks packets with ECN CE at the configured threshold with correct granularity.

**Procedure:** Configure threshold T on DUT egress queue. Verify: (a) no packets marked below T; (b) 100% marked above maximum threshold; (c) appropriate Weighted Random Early Detection (WRED) / Random Early Detection (RED) probability ramp between thresholds. Test thresholds: low (~100KB), medium (~1MB), high (~5MB).

**Reporting:** Plot ECN marking probability vs. instantaneous queue depth. Report measured threshold accuracy (deviation from configured).

## PFC Behavior Under Incast {#pfc-behavior-under-incast}

**Objective:** Characterize DUT's PFC generation behavior under N:1 incast conditions.

**Procedure:** Generate N:1 incast at 100% line rate, N = {2, 4, 8, 16, 32, 64}. Measure PFC PAUSE frame count/sec per hop, PFC PAUSE duration per port, PFC storm onset, and end-to-end throughput. The test characterizes headroom sizing and PFC watchdog effectiveness.

## DCQCN Convergence Time {#dcqcn-convergence-time}

**Objective:** Measure time for DCQCN to converge to fair-share rate after congestion onset.

**Procedure:** Establish M flows through a common bottleneck. At T0, inject additional M flows (creating 2:1 oversubscription). Measure time until all 2M flows achieve rates within 10% of fair share. Repeat for M = {4, 16, 64, 256}. Vary DCQCN parameters and report sensitivity.

## PFC Storm and Deadlock Resilience {#pfc-storm-and-deadlock-resilience}

**Objective:** Verify the DUT does not enter PFC deadlock or sustained PFC storm under adversarial traffic.

**Procedure:** Generate cyclic traffic patterns known to cause PFC deadlocks. Run for 300 seconds. The test characterizes whether the DUT demonstrates resilience via PFC watchdog or architectural immunity (e.g., VOQ-based scheduling); the mechanism observed is reported.

# Test Category 4: Load Balancing Efficacy {#test-lb}

Load balancing across parallel fabric paths is critical for AI training fabrics because the traffic consists of a small number of high-bandwidth, long-lived elephant flows.

## ECMP Entropy and Polarization {#ecmp-entropy-and-polarization}

**Objective:** Quantify traffic polarization under standard ECMP hashing for AI training flow patterns.

**Procedure:** Configure standard 5-tuple ECMP. Generate traffic with Q = {1, 4, 8, 16, 32} QPs per src-dst pair. Measure per-link utilization, MMR, and JFI. Test with and without ECMP hashing that includes the BTH destination QP field as a hash input (in addition to the standard 5-tuple). Repeat for fabric sizes of 8, 16, 32, and 64 leaf switches.

## Dynamic Load Balancing (Flowlet) {#dynamic-load-balancing-flowlet}

**Objective:** Evaluate DUT's flowlet-based DLB performance and compare to baseline ECMP.

**Procedure:** Configure vendor-specific DLB (document algorithm type). Generate traffic with Q=4 QPs. Measure MMR, JFI, per-link utilization, out-of-order rate. Vary flowlet gap timer and report sensitivity.

## Packet Spraying {#packet-spraying}

**Objective:** Evaluate DUT's per-packet spraying performance and quantify the utilization vs. reordering tradeoff.

**Procedure:** Configure per-packet load balancing. Measure MMR (expected ~1.0), JFI (expected ~1.0), out-of-order rate, and RDMA retransmission impact. If the DUT provides an in-fabric reorder buffer, document per {{asic-features}}.

## Jain's Fairness Index Measurement

**Objective:** Single-number summary of load balancing quality comparable across all strategies.

**Formula:**

~~~ ascii-art
JFI = (Sum LinkTx_i)^2 / (N × Sum LinkTx_i^2)
~~~
{: #fig-jfi align="center" title="Jain's Fairness Index Formula"}

where LinkTx_i = transmitted traffic on fabric link i, N = total parallel links. Range: 1/N (worst) to 1.0 (perfect).

**Reporting:** Report JFI for each load balancing strategy. Provide bar chart comparing ECMP, DLB, and packet spray.

# Test Category 5: Collective Communication Benchmarks {#test-collective}

These tests evaluate the fabric's performance under realistic collective communication patterns. Unlike synthetic RDMA tests in {{test-rdma}} and {{test-uec}}, these exercise the full stack including the collective communications library (CCL) in use (e.g., NCCL, RCCL, oneCCL).

Because collective placement determines how much of a collective's traffic crosses the DUT boundary, every result in this section is reported together with the fabric-visible data volume and placement information required by {{fabric-visible-data-volume}}.

## AllReduce Benchmark {#allreduce-benchmark}

**Objective:** Measure fabric performance during AllReduce operations, the dominant collective for gradient synchronization in data-parallel training.

**Procedure:** Using N accelerators connected through the DUT fabric, execute AllReduce (sum) operations using a collective communications library benchmark suite (e.g., nccl-tests, rccl-tests, or equivalent).

Test parameters:

* Message sizes: 1 MB, 8 MB, 64 MB, 256 MB, 1 GB, 4 GB
* Accelerator counts (N): 8, 16, 32, 64, 128, 256, 512, 1024
* Minimum iterations per (message_size, N) pair: 100
* Load balancing strategies: ECMP, DLB, packet spray

For each (message_size, N) pair, record average, P50, P95, and P99 BusBW, ECN marking ratio, PFC pause count, and per-link utilization. BusBW is computed per the BusBW definition in {{TERMINOLOGY}}; algo_factor is fixed per collective type and does not vary with the algorithm the library selects at runtime. The runtime algorithm selected by the library for each message-size bucket is verified via library tracing and documented as part of the test conditions.

**Reporting:** Tabulate BusBW for each (message_size, N, LB_strategy, Algorithm (verified)) combination.  The "Algorithm (verified)" column is required; results without it are incomplete.  Plot BusBW vs. N for each message size. Report BusBW efficiency = BusBW / NIC_line_rate.

## AllToAll Benchmark {#alltoall-benchmark}

**Objective:** Measure fabric performance during AllToAll operations, the dominant collective for Mixture-of-Experts (MoE) expert parallelism dispatch.

**Procedure:** Using the same message sizes, accelerator counts, iteration count, and load balancing strategies as {{allreduce-benchmark}}, execute AllToAll operations via the collective communication library.

AllToAll generates the worst-case fabric stress pattern: every accelerator simultaneously sends a unique payload to every other accelerator in the group, which creates maximum entropy and stresses every fabric link with many-to-many traffic overlap.  This makes AllToAll JCT the most sensitive single indicator of fabric congestion management quality.

BusBW is computed per the BusBW definition in {{TERMINOLOGY}}; algo_factor is fixed per collective type and does not depend on topology or library implementation. The runtime algorithm in use is verified via library tracing and documented as part of the test conditions.

**Measurement:**  Report BusBW (average, P50, P95, P99), JCT per iteration, ECN marking ratio, PFC pause count, and per-link utilization for each (message_size, N, LB_strategy) combination.

**Reporting:** Same table format as {{allreduce-benchmark}}, with the "Algorithm (verified)" column required.  Additionally report JCT for each configuration; JCT degradation relative to the ECMP baseline is highlighted as the primary congestion sensitivity indicator.

## AllGather Benchmark {#allgather-benchmark}

**Objective:** Measure fabric performance during AllGather operations, the dominant collective for weight and activation distribution in tensor-parallel training.

**Procedure:** Using the same message sizes, accelerator counts, iteration count, and load balancing strategies as {{allreduce-benchmark}}, execute AllGather operations via the collective communication library.

AllGather consists of a gather phase only – each accelerator contributes a shard and receives the full concatenated tensor.
There is no reduce phase, which produces lower peak fabric load than AllReduce at equivalent message size and N.  This makes AllGather a useful baseline for isolating the gather-path fabric contribution from the combined send-and-reduce cost.

BusBW is computed per the BusBW definition in {{TERMINOLOGY}}; algo_factor is fixed per collective type and does not depend on the library's algorithm selection. The runtime algorithm in use is verified via library tracing and documented as part of the test conditions.

**Measurement:** Report BusBW (average, P50, P95, P99), JCT per iteration, ECN marking ratio, PFC pause count, and per-link utilization for each (message_size, N, LB_strategy) combination.

**Reporting:** Same table format as {{allreduce-benchmark}}, with the "Algorithm (verified)" column required.  Report BusBW efficiency = BusBW / NIC_line_rate.  Where results are compared to AllReduce under identical parameters, the BusBW ratio (AllGather / AllReduce) quantifies the difference in fabric load between the two traffic patterns; AllReduce's additional network traffic reflects its ReduceScatter phase, not the reduction arithmetic itself, which is performed by the accelerators, not the fabric.

## Collective Communication Library Bus Bandwidth Summary

**Reporting template:**

| Collective | Msg Size | N Accels | ECMP BusBW (Gbps/accel) | DLB BusBW (Gbps/accel) | Spray BusBW (Gbps/accel) |
|---|---|---|---|---|---|
| AllReduce | 1GB | 128 | (meas) | (meas) | (meas) |
| AllReduce | 1GB | 512 | (meas) | (meas) | (meas) |
| AllToAll | 1GB | 128 | (meas) | (meas) | (meas) |
| AllToAll | 1GB | 512 | (meas) | (meas) | (meas) |
| AllGather | 1GB | 128 | (meas) | (meas) | (meas) |
| AllGather | 1GB | 512 | (meas) | (meas) | (meas) |
{: #tab-ccl-summary title="Collective Communication Bus Bandwidth Summary"}

# Test Category 6: Job Completion Time (JCT) Benchmarks {#test-jct}

JCT is the single most important user-facing KPI for AI training fabrics; it directly determines accelerator utilization and training cost.

## Synthetic JCT Under Controlled Conditions {#synthetic-jct-under-controlled-conditions}

**Objective:** Measure JCT for a defined synthetic workload with a known computation-to-communication ratio to isolate fabric-induced overhead.

**Procedure:** Define a synthetic training iteration as a strictly sequential model:

1. Computation phase of C milliseconds (simulated sleep or GPU compute kernel)
2. Communication phase: AllReduce of S bytes across N accelerators

| Parameter                                                    | Values                       |
| ------------------------------------------------------------ | ---------------------------- |
| Computation time C                                           | 10 ms, 50 ms, 100 ms, 500 ms |
| Message size S                                               | 256 MB, 1 GB, 4 GB           |
| Accelerator count N                                          | 64, 128, 256, 512, 1024      |
| Iterations                                                   | 1000                         |
{: #tab-synthetic-jct-params title="Synthetic JCT Test Parameters"}

Execute 1000 iterations and measure total wall-clock JCT.

~~~
Roofline_seq = Iterations × (C + (8 × S × algo_factor) / B_acc)
JCT Ratio    = Measured_JCT / Roofline_seq

  where:
    C            = compute time per iteration, in seconds
                   (convert the millisecond values in the
                   parameter table to seconds)
    S            = message size per iteration (bytes)
    algo_factor  = fixed normalization constant per collective
                   type; see the BusBW definition in the
                   companion terminology document
    B_acc        = aggregate per-accelerator NIC line rate
                   (bits/second); sum across all NICs serving
                   the accelerator (e.g., in rail-optimised
                   topologies, the sum of all rail NIC speeds)
    Iterations   = number of synthetic iterations executed

  The factor of 8 converts S from bytes to bits to match the
  units of B_acc.
~~~
{: #fig-jct-formula align="center" title="JCT Ratio Calculation"}

This model assumes strictly sequential compute and communication phases
and represents a conservative upper bound on communication overhead.
Many frameworks overlap these phases via gradient bucketing or asynchronous collectives, which reduces the effective communication overhead visible in wall-clock JCT.

Implementations using overlapped execution additionally report:

~~~
Overlap_Fraction = 1 - (Measured_JCT - C_total) / Comm_time

  where:
    C_total   = Iterations × C
    Comm_time = Iterations × (8 × S × algo_factor) / B_acc
    S, algo_factor, B_acc as defined for Roofline_seq above.
~~~
{: #fig-overlap-formula align="center" title="Overlap Fraction Calculation"}

An Overlap_Fraction of 0 indicates fully sequential execution; 1.0 indicates communication is perfectly hidden behind compute.

When overlap is present, the residual fabric overhead is reported as:

~~~
Effective_Comm_Overhead = Measured_JCT - C_total
~~~

The Overlap_Fraction and communication-library overlap configuration (e.g., bucket size, number of async streams) are documented as part of the test configuration when this optional measurement is reported.

**Reporting:** Tabulate JCT Ratio for each (C, S, N, LB_strategy) combination.  Plot JCT Ratio vs. N to characterize fabric scalability.

> NOTE: JCT Ratio values of 1.05 and 1.15 are cited elsewhere in this document ({{indicative-reference-values}}) as illustrative reference points, not as pass/fail thresholds. Per the BMWG charter, the definition of acceptance criteria or performance requirements is explicitly outside the scope of this Working Group; deployment-specific thresholds are outside the scope of this document.

## MLPerf-Aligned JCT {#mlperf-aligned-jct}

**Objective:** Measure JCT using MLPerf Training benchmark workloads {{MLPERF}} to enable comparison with published industry results.

**Procedure:** Execute the current MLPerf Training closed-division workloads per MLPerf submission rules ({{MLPERF}}); the specific workload set changes across MLPerf versions, so the test report MUST identify the MLPerf Training version and workload names used. Simultaneously capture all fabric KPIs from {{kpi-framework-and-metrics-taxonomy}}. Report time-to-train and/or tokens-per-second.

## Multi-Tenant JCT Interference {#multi-tenant-jct-interference}

**Objective:** Quantify JCT impact when multiple training jobs share the same fabric.

**Procedure:** Configure two or more independent training jobs. Jobs are configured to overlap in spine-layer link usage. Measure baseline JCT (isolated) and contention JCT (simultaneous).

~~~ ascii-art
JCT Interference Factor = Contention_JCT / Baseline_JCT
~~~
{: #fig-jct-interference align="center" title="JCT Interference Factor"}

Test with spine link overlap: 0%, 25%, 50%, 75%.

# Test Category 7: Scale and Convergence {#test-scale}

## Fabric Scale Limits {#fabric-scale-limits}

**Objective:** Determine the maximum fabric scale at which the DUT maintains acceptable KPI performance.

**Procedure:** Progressively increase active accelerator endpoints from N=64 to maximum topology support while running AllReduce ({{allreduce-benchmark}}, S=1GB). At each scale point record JCT Ratio, BusBW, ECN ratio, PFC count, CPU and memory utilization. Also measure BGP/routing convergence time after clearing all adjacencies (analogous to the convergence testing approach in {{EVPN-BENCH}}).

## Link Failure Convergence {#link-failure-convergence}

**Objective:** Measure traffic disruption and JCT impact when a fabric link fails during active training.

**Procedure:** With the fabric fully loaded (AllReduce, N=128, S=1GB), administratively fail a spine uplink. Measure:

- Duration of packet loss
- Packets lost
- JCT overhead for the failure iteration vs. steady state
- Time for load balancing mechanism to redistribute flows

Repeat for: leaf uplink failure, spine switch failure, superspine link failure (if applicable). Test under each load balancing strategy.

## Zero-Impact Failover Measurement

**Objective:** Verify vendor claims of zero-impact or sub-microsecond failover.

**Procedure:** Execute {{link-failure-convergence}} with nanosecond-precision measurement. A failure is considered "zero-impact" if the measured JCT for the failure iteration is within the P99 JCT of steady-state iterations.

# Test Category 8: Soak and Stability {#test-soak}

## 24-Hour Sustained Load {#soak-24h}

**Objective:** Characterize DUT fabric stability under sustained AI training load over an extended period, following the soak-testing methodology pattern in {{EVPN-BENCH}}.

**Procedure:** Configure DUT at maximum validated scale from {{fabric-scale-limits}}. Generate bidirectional collective communication traffic (alternating AllReduce and AllToAll) at 80% of maximum validated throughput, per the offered load fraction required by the Soak Test definition in {{TERMINOLOGY}}. Run continuously for 24 hours. Sample all KPIs from {{kpi-framework-and-metrics-taxonomy}} every 60 seconds.

The objective of the soak test is to monitor and document fabric behavior under extended load. The methodology does not establish pass/fail criteria for any reported metric. Any memory leaks, crashes, or other anomalies encountered during the test MUST be documented as an application log file or other dedicated file with their timestamps and durations.

**Reporting:** Time-series plots of JCT Ratio, BusBW, ECN ratio, PFC count, CPU, and memory over the 24-hour period. Report standard deviation of JCT Ratio (stability metric).

## Resource Leak Detection

**Objective:** Detect memory leaks, handle exhaustion, or gradual performance degradation in DUT software.

**Procedure:** Record per-process memory usage at T=0, T=1h, T=6h, T=12h, T=24h. Compute linear regression slope of memory usage over time. A slope exceeding **1 MB/hour** for any process indicates a potential memory leak and is reported; this slope is a reporting trigger for investigation, not a pass/fail criterion. Also monitor forwarding-plane counter wraparounds and hardware table occupancy trends.

# Reporting Format {#reporting}

Per the BMWG charter, the definition of acceptance criteria or performance requirements is explicitly outside the scope of this Working Group. This methodology defines what is measured and how it is reported; it does not set minimum acceptable values, certification, or pass/fail criteria. Any deployment-specific performance objectives are outside the scope of this document.

Results from collective communication benchmarks ({{test-collective}}) MUST be reported per the reporting requirements stated in the BusBW definition of {{TERMINOLOGY}}.

Test reports include the following sections:

1. **DUT Identification:** Complete parameters from {{device-under-test-dut-identification}} for all fabric components.
2. **Test Topology:** Diagram and description per {{reference-fabric-topologies}}, including physical cabling.
3. **Test Configuration:** All DUT configuration parameters: QoS policies (ECN thresholds, PFC headroom, DCQCN parameters), load balancing mode, buffer allocation, and vendor-specific tuning.
4. **Host Configuration:** Complete host stack description per {{device-under-test-dut-identification}} including NIC firmware, driver, collective library version, and any tuning. For UET tests, additionally report: UEC compliance profile, libfabric provider version, NIC UEC firmware version, and enabled optional features (LLR, Packet Trimming, Packet Rate Improvement (PRI), CBFC).
5. **Test Results:** For each test from {{test-rdma}} through {{test-soak}}, provide specified tables, graphs, and statistical summaries. For {{test-uec}} tests, results include side-by-side UET vs. RoCEv2 comparison data on the identical DUT fabric.
6. **Anomalies:** Any deviations from specified procedures, test failures, or unexpected behaviors are documented.
7. **Repeatability Statement:** Report iteration count and coefficient of variation (std deviation / mean) for each test's primary metric. A CV of 5% is an illustrative reference point for typical run-to-run variation; per the charter disclaimer above, this document does not set a required or minimum threshold for test validity.
8. **Comparability Statement:** When a report compares two or more fabrics, tabulate the comparability set of {{comparability-normalization}} for each result and state explicitly which parameters differ. Reports comparing fabrics of different topology class additionally provide the structural descriptors listed in that section. Reports that present results from a single fabric do not require this section.

# Security Considerations

This document defines benchmarking methodology for controlled laboratory environments and does not specify any protocol mechanism. It therefore introduces no new protocol-level security considerations beyond those of the underlying technologies it references. The considerations below follow the BMWG convention established in {{RFC8238}} and align with the companion terminology document {{TERMINOLOGY}}.

Benchmarking activities as described in this document are limited to technology characterization of AI training fabrics using controlled stimuli in a laboratory environment, with dedicated address space and the constraints specified herein.

The benchmarking network topology will be an independent test setup and MUST NOT be connected to devices that may forward the test traffic into a production network or misroute traffic to the test management network. This isolation requirement is particularly important for AI fabric benchmarking because the hop-by-hop flow-control mechanisms referenced in this document (PFC, CBFC) propagate backpressure toward traffic sources and can extend the blast radius of a misconfigured test beyond the immediate DUT; DCQCN reduces, but does not eliminate, reliance on these mechanisms.

Benchmarking is performed on a "black-box" basis, relying solely on measurements observable external to the DUT as defined in {{TERMINOLOGY}}.

Special capabilities SHOULD NOT exist in the DUT specifically for benchmarking purposes. Any implications for network security arising from the DUT SHOULD be identical in the lab and in production networks. In particular, RDMA memory-region permissions are properties of the deployed configuration, not of the benchmarking methodology, and SHOULD reflect production posture during testing.

Per {{RFC6815}}, the tests defined herein MUST NOT be performed on production networks. The use of dedicated test IP address ranges per {{RFC2544}} Appendix C (198.18.0.0/15 for IPv4; 2001:db8::/32 per {{RFC3849}} for IPv6) is RECOMMENDED to prevent accidental interaction with production infrastructure.

The following considerations are specific to the methodology defined in this document:

- **PFC leakage:** PFC PAUSE frames generated under incast or storm conditions ({{pfc-behavior-under-incast}}, {{pfc-storm-and-deadlock-resilience}}) that escape the test environment can cause adjacent production switches sharing the same priority class to stop responding. Physical or VLAN-based isolation of the test fabric is required.
- **Line-rate RDMA traffic generators:** the equipment specified in {{traffic-generator-requirements}} is capable of saturating production links at line rate; such generators MUST be confined to the test fabric.
- **PFC disabled in {{uet-congestion-control-benchmarks}}:** the UET PFC-free incast test deliberately disables PFC on the DUT. In this configuration, traffic leaking to adjacent infrastructure cannot be backpressured and will be dropped on the adjacent device's queues. Isolation is mandatory.
- **RDMA QP and PDC namespace isolation:** when RDMA/RoCEv2 traffic is used, the test environment SHOULD be isolated from production RDMA fabrics to prevent QP number space collisions or inadvertent PFC propagation. When UET traffic is used ({{test-uec}}), the test environment MUST ensure that UDP port 4793 traffic does not leak to production networks and that PDC identifier spaces are isolated.
- **UET transport security sub-layer (TSS):** SHOULD NOT be enabled during performance benchmarking unless transport security overhead is explicitly being measured.

# IANA Considerations

This document has no IANA actions.

--- back

# KPI-to-Test Mapping Summary

| KPI | Test Section | Measurement Method | Reporting Unit |
|---|---|---|---|
| Throughput Rate | {{baseline-throughput}} | Binary search, zero-loss | Tbps, % line rate |
| Latency (P99) | {{latency-characterization}} | Tagged frame, loaded / unloaded | us |
| Burst Absorption | {{back-to-back-burst-absorption}} | Max burst without loss | frames, bytes |
| ECN Accuracy | {{ecn-marking-accuracy-and-threshold}} | Queue depth vs. marking | threshold deviation % |
| PFC Behavior | {{pfc-behavior-under-incast}} | Incast sweep N=2..64 | PAUSE events/sec, duration |
| DCQCN Convergence | {{dcqcn-convergence-time}} | Rate stabilization after onset | us |
| PFC Deadlock | {{pfc-storm-and-deadlock-resilience}} | Cyclic adversarial traffic | observed/reported, watchdog events |
| ECMP Imbalance | {{ecmp-entropy-and-polarization}} | MMR, JFI per QP count | dimensionless ratios |
| DLB Efficacy | {{dynamic-load-balancing-flowlet}} | Throughput delta vs. ECMP | %, out-of-order rate |
| Spray Efficacy | {{packet-spraying}} | JFI, retransmission rate | dimensionless, retx/sec |
| AllReduce BusBW | {{allreduce-benchmark}} | CCL benchmark | Gbps per accelerator |
| AllToAll JCT | {{alltoall-benchmark}} | CCL benchmark | seconds per iteration |
| AllGather BusBW | {{allgather-benchmark}} | CCL benchmark | Gbps per accelerator |
| Synthetic JCT Ratio | {{synthetic-jct-under-controlled-conditions}} | Measured / Roofline | dimensionless |
| MLPerf JCT | {{mlperf-aligned-jct}} | Time-to-train | minutes, tokens/sec |
| Multi-Tenant Impact | {{multi-tenant-jct-interference}} | Contention / Baseline JCT | interference factor |
| Scale Limit | {{fabric-scale-limits}} | Max N with JCT Ratio characterized | accelerator count |
| Failover Time | {{link-failure-convergence}} | Loss duration on link fail | us |
| 24h Stability | {{soak-24h}} | JCT Ratio std deviation | dimensionless |
| UET Throughput (RUD) | {{uet-throughput-by-transport-service}} | Binary search per transport service | Gbps, % line rate |
| UET First-Packet Latency | {{uet-latency-characterization}} | PDC establish + first data | us |
| UET Spray Efficacy | {{packet-spray-efficacy-under-uet-rud}} | JFI/MMR under RUD spray | dimensionless, OOO rate |
| UET PFC-Free Loss Rate | {{uet-congestion-control-benchmarks}} | Incast without PFC enabled | %, retx overhead |
| LLR Retry Latency | {{link-layer-enhancement-benchmarks}} | Per-hop error recovery time | nanoseconds |
| Packet Trimming Savings | {{link-layer-enhancement-benchmarks}} | BW saved during congestion | % bandwidth |
| CBFC vs PFC HOL Blocking | {{link-layer-enhancement-benchmarks}} | Head-of-line blocking duration | us |
| UET Collective BusBW | {{uet-collective-communication-performance}} | AllReduce/AllToAll over UET | Gbps per accelerator |
| PDC Establishment Rate | {{uet-pdc-scalability-and-connection-setup-rate}} | Sustained PDC creation rate | PDCs/second |
| Max Concurrent PDCs | {{uet-pdc-scalability-and-connection-setup-rate}} | Scale limit per NIC | count |
{: #tab-kpi-mapping title="KPI-to-Test Mapping Summary"}

# Indicative Reference Values (Non-Normative) {#indicative-reference-values}

This appendix provides indicative reference values for the KPIs defined in {{kpi-framework-and-metrics-taxonomy}}. The values reflect current industry observations for distributed AI training workloads as of 2025-2026. These values are NON-NORMATIVE and do not constitute benchmarking acceptance criteria or performance requirements. Per the BMWG charter, the definition of acceptance criteria or performance requirements is explicitly outside the scope of this Working Group. Implementers may use these values as contextual references when interpreting results; they MUST NOT be used as pass/fail criteria in vendor evaluations. Deployment-specific targets will vary by topology, accelerator architecture, collective library, and operator requirements.

| KPI | Indicative Reference |
|---|---|
| JCT Ratio | ≤ 1.05 (≤ 1.15 acceptable) |
| BusBW | ≥ 90% of NIC line rate (intra-pod) |
| Aggregate Throughput | ≥ 95% of bisection BW |
| Packet Drop Rate | 0 ppm wire-level loss (lossless RoCEv2 profiles); 0 ppm application-visible loss (UET; see the Zero Packet Loss definition in {{TERMINOLOGY}}) |
{: #tab-indicative-values title="Indicative Reference Values for Distributed AI Training Fabrics (Non-Normative)"}

# ASIC Feature Categories (Informational) {#asic-features}

This appendix identifies ASIC feature categories relevant to AI fabric performance. Implementers document which categories are present and enabled on the DUT. Specific vendor names are intentionally omitted.

| Feature Category | Sub-types | Relevance to AI Fabric | What to Report |
|---|---|---|---|
| Aggregate Switching BW | ASIC-level capacity | Cluster scale, bisection BW | Total Tbps; per-port speed (400/800GbE) |
| Buffer Architecture | Shared, VOQ, Cut-through | Microburst absorption, PFC behavior, lossless operation | Buffer type; total bytes; shared vs. dedicated split; per-port/queue allocation |
| Packet Distribution | Per-flow, Per-packet, Flowlet | ECMP load balancing quality and reordering risk | Supported granularities; in-fabric reorder buffer (yes/no) |
| Congestion Control | ECN marking, PFC, DCQCN | DCQCN convergence and lossless behavior | ECN granularity (port/queue/VOQ); PFC priorities; DCQCN parameter range |
| Adaptive Routing | Flowlet, ECMP, Spray, Topology-aware | Load balancing quality under collective patterns | Algorithm type; flowlet gap timer range; topology-aware support |
| Telemetry | Per-port, Per-queue, Per-flow | Required for KPI measurement during benchmarking | Monitoring granularity; streaming interval; INT support |
| Cluster Scale Support | 2-tier, 3-tier | Applicable topology scales | Max cluster size per topology; ASIC count |
{: #tab-asic-features title="ASIC Feature Categories"}

All values are reported based on vendor documentation or measured capability. Additional DUT capabilities affecting benchmark results are also documented.

# RoCEv2 Test Frame Format {#rocev2-frame}

| Offset | Field | Size | Value / Description |
|---|---|---|---|
| 00 | Ethernet Dst MAC | 6B | DUT next-hop MAC |
| 06 | Ethernet Src MAC | 6B | Test equipment MAC |
| 12 | EtherType / TPID | 2B | 0x0800 (IPv4) when untagged; 0x8100 (Tag Protocol Identifier — TPID) when 802.1Q-tagged |
| 14 | 802.1Q Tag (optional) | 4B | When tagged: Tag Control Information (TCI: Priority Code Point (PCP)=3 for RoCEv2 priority, VLAN Identifier (VID)) followed by inner EtherType 0x0800. Omit this row entirely when untagged and shift subsequent offsets back by 4B |
| 18 | IPv4 Header | 20B | DSCP=26 (AF31, Assured Forwarding class 3, drop precedence 1), ECN=ECT(0) (ECN-Capable Transport), Proto=17 (UDP) |
| 38 | UDP Header | 8B | DstPort=4791 (RoCEv2), SrcPort=var |
| 46 | BTH (Base Transport Header) | 12B | OpCode, DstQP, PSN, P_Key |
| 58 | RDMA Extended Transport Header (RETH; if Write) | 16B | Virtual Address (VA), R_Key, Direct Memory Access (DMA) Length |
| 74 | Payload | var | Test data (incrementing octets) |
| var | ICRC | 4B | Invariant CRC |
| var+4 | FCS | 4B | Ethernet Frame Check Sequence |
{: #tab-rocev2-frame title="RoCEv2 Test Frame Format"}

# UET (Ultra Ethernet Transport) Frame Format {#uet-frame}

UET runs over UDP/IP using UDP destination port 4793, IANA-assigned to Ultra Ethernet Transport.

Unlike the RoCEv2 frame format above, this appendix does not specify a byte-accurate UET header layout. UET headers are layered and variable-length, and their wire formats are defined normatively by the UEC; test equipment implementations MUST follow {{UEC-1.0}} Section 4 for all wire-format details. The figure below is explicitly schematic: it shows only the layering and on-wire ordering of the protocol components that test equipment generates and parses.

~~~
+-------------------------------------------------------------+
| Ethernet Header: Dst MAC, Src MAC, EtherType                |
|   (optional 802.1Q tag: PCP=3 for UET priority class, VID)  |
+-------------------------------------------------------------+
| IPv4 Header: DSCP=26 (AF31), ECN=ECT(0), Proto=17 (UDP)     |
+-------------------------------------------------------------+
| UDP Header: DstPort=4793 (UET);                             |
|   SrcPort carries the Entropy Value, varied per packet      |
+-------------------------------------------------------------+
| PDS (Packet Delivery Sublayer) header(s):                   |
|   packet type, PDC identifiers, sequence number,            |
|   ack/credit and congestion-control state                   |
+-------------------------------------------------------------+
| SES (Semantic Sublayer) header(s):                          |
|   operation (Write/Read/Send/Atomic), addressing,           |
|   message identification                                    |
+-------------------------------------------------------------+
| Payload                                                     |
+-------------------------------------------------------------+
| ICRC (4B) | FCS (4B)                                        |
+-------------------------------------------------------------+
~~~
{: #fig-uet-frame align="center" title="UET Frame Layering (Schematic Only; Byte Layouts Per UEC 1.0 Section 4)"}

Layering notes:

- The Entropy Value is carried in the UDP source port field, not in a separate UET header field; see the Entropy Value definition in {{TERMINOLOGY}}. Test equipment varies the UDP source port per packet to exercise packet spraying.
- PDS headers precede SES headers on the wire.
- CMS (Congestion Management Sublayer) state is carried in PDS congestion-control fields and control packets rather than as a separate wire header. TSS (Transport Security Sublayer), when enabled, adds security headers and authentication data per {{UEC-1.0}} Section 4.

## Key Differences from RoCEv2

| Aspect | RoCEv2 | UET | Notes |
|---|---|---|---|
| UDP Dst Port | 4791 (IANA-assigned) | 4793 (IANA-assigned) | Distinct transports, both over UDP/IP |
| Transport Endpoint | QP Number (24b) | PDC ID | PDC state is established in-band with the first packet |
| Entropy Source | UDP src port, typically fixed per QP/connection | UDP src port, varied per packet | Per-packet spraying uses existing ECMP hashing |
| Congestion Signalling | ECN bits; CNP-based feedback (e.g., DCQCN) | ECN plus UET congestion-control state carried in PDS | Sender- and receiver-based CC per {{UEC-1.0}} |
| Ordering Guarantee | Always in-order (RC) | Per-service (ROD/RUD/RUDI/UUD) | RUD/RUDI allow out-of-order delivery |
| Header Structure | Fixed BTH plus per-opcode extension headers | Layered, variable-length (PDS, SES) | Per {{UEC-1.0}} Section 4 |
{: #tab-rocev2-vs-uet title="RoCEv2 vs. UET Comparison"}

1. **UDP Destination Port:** UET uses port 4793 vs. RoCEv2 port 4791.
2. **Entropy Value:** Carried in the UDP source port field and varied per packet for ECMP path selection. Test equipment varies the source port to achieve uniform path distribution.
3. **Transport Service Indicator:** Header encodes transport service (ROD/RUD/RUDI/UUD). Tests set this to match the service being benchmarked.
4. **PDC Identifier:** In-band-established PDC ID replaces RoCEv2's Destination QP. Test equipment tracks PDC lifecycle for accurate measurement.
5. **Layered Sub-Headers:** UET uses four sub-layers (SES, PDS, CMS, TSS) with variable-length headers. Implementations MUST follow {{UEC-1.0}} Section 4 for wire format details.
6. **Optional Feature Headers:** When the LLR or PRI link-layer features, or the network-layer Packet Trimming feature, are enabled, additional or modified framing may be present. Test equipment is configured to recognize and parse these.

# Acknowledgments
{:numbered="false"}

This work has benefited from the discussions that occurred during the joint IPPM and BMWG meeting and on the BMWG mailing list. Thanks to Carsten Rossenhoevel and Mohamed Boucadair for valuable review and comments. Thanks to Andrew Yourtchenko for a thorough review of the document set.
