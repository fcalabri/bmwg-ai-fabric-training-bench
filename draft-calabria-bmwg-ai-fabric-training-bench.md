---
title: "Benchmarking Methodology for AI Training Network Fabrics"
abbrev: "AI Fabric Bench"
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
    email: fcalabri@cisco.com
  - name: Carlos Pignataro
    ins: C. Pignataro
    org: Blue Fern Consulting
    email: carlos@bluefern.consulting
  - name: Qin Wu
    ins: Q. Wu
    org: Huawei
    email: bill.wu@huawei.com
  - name: Giuseppe Fioccola
    ins: G. Fioccola
    org: Huawei
    email: giuseppe.fioccola@huawei.com

normative:
  RFC1242:
  RFC2119:
  RFC2544:
  RFC2889:
  RFC8174:
  RFC8238:
  RFC8239:
  RFC9004:
  UEC-1.0:
    title: "Ultra Ethernet Transport (UET) Specification 1.0"
    author:
      - org: Ultra Ethernet Consortium
    date: 2025-06
    target: "https://ultraethernet.org"

informative:
  RFC6815:
  EVPN-BENCH:
    title: "Benchmarking Methodology for EVPN and PBB-EVPN"
    author:
      - ins: S. Jacob
        name: Saneesh Jacob
      - ins: K. Tiruveedhula
        name: Kishore Tiruveedhula
    date: 2018-06
    seriesinfo:
      Internet-Draft: draft-kishjac-bmwg-evpntest-10
  LLM-BENCH:
    title: "Benchmarking Methodology for Large Language Model Serving"
    author:
      - ins: Gaikwad, et al.
    date: 2026-01
    seriesinfo:
      Internet-Draft: draft-gaikwad-llm-benchmarking-methodology-00
  META-ROCE:
    title: "RDMA over Ethernet for Distributed AI Training at Meta Scale"
    author:
      - ins: A. Gangidi
        name: Anirudh Gangidi
    date: 2024
    seriesinfo:
      DOI: "ACM SIGCOMM 2024"
  DCQCN-PAPER:
    title: "Congestion Control for Large-Scale RDMA Deployments"
    author:
      - ins: Y. Zhu
        name: Yibo Zhu
    date: 2015
    seriesinfo:
      DOI: "ACM SIGCOMM 2015"
  NCCL:
    title: "NVIDIA Collective Communications Library (NCCL)"
    author:
      - org: NVIDIA
    target: "https://developer.nvidia.com/nccl"
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

As large-scale distributed AI/ML training clusters grow to tens of thousands of accelerators (GPUs/XPUs), the backend network fabric becomes the critical bottleneck determining job completion time (JCT), training throughput, and accelerator utilization.

This document establishes vendor-independent, reproducible test procedures for benchmarking fabric-level performance under realistic AI training workloads, covering RDMA/RoCEv2 transport, the Ultra Ethernet Transport (UET) protocol defined by the UEC Specification 1.0 {{UEC-1.0}}, congestion management (PFC, ECN, DCQCN, CBFC), load balancing strategies (ECMP, DLB, packet spraying), collective communication patterns (AllReduce, AlltoAll, AllGather), and scale/soak testing.

The methodology enables apples-to-apples comparison across different switch ASICs, vendor implementations, NIC transport stacks (RoCEv2 vs. UET), and fabric architectures (2-tier Clos, 3-tier Clos, rail-optimized).

--- middle

# Introduction

The rapid growth of distributed AI/ML training workloads has fundamentally changed the performance requirements for data center network fabrics. Unlike traditional data center traffic characterized by diverse flow sizes and protocols, AI training workloads generate highly synchronized, bandwidth-intensive, east-west traffic patterns dominated by collective communication operations (AllReduce, AlltoAll, AllGather). These workloads impose unique demands: lossless transport (via RoCEv2 over RDMA), ultra-low tail latency, near-perfect load balancing across all fabric paths, and the ability to absorb coordinated micro-bursts from thousands of accelerators simultaneously.

Existing BMWG methodologies, while foundational, do not adequately address the characteristics of AI training fabrics. {{RFC2544}} defines benchmarking for general network interconnect devices but does not account for RDMA transport semantics, collective communication patterns, or the unique congestion dynamics of GPU-to-GPU traffic. {{RFC8238}} and {{RFC8239}} establish data center benchmarking terminology and methodology but predate the AI fabric paradigm and do not address RoCEv2-specific behaviors such as Priority Flow Control (PFC) interactions, DCQCN congestion control convergence {{DCQCN-PAPER}}, or the impact of load balancing strategies on Job Completion Time (JCT). Industry experience deploying RoCEv2 at scale {{META-ROCE}} further highlights the need for standardized benchmarking methodology.

The EVPN benchmarking methodology {{EVPN-BENCH}} provides a structural template for service-oriented benchmarking but is scoped to L2VPN services rather than RDMA fabrics.

This document fills the gap by defining a comprehensive benchmarking methodology specifically designed for AI training network fabrics.

## Requirements Language

{::boilerplate bcp14-tagged}

## Scope and Applicability

This document applies to Ethernet-based AI training backend network fabrics employing RoCEv2 and/or UEC Ultra Ethernet Transport (UET) protocols. The scope includes leaf-spine (2-tier Clos) and leaf-spine-superspine (3-tier Clos) topologies.

InfiniBand fabrics are explicitly **out of scope**, though many KPIs defined herein may be adapted for IB benchmarking by future documents. The DUT is the network fabric itself (the collection of switches and interconnecting links), not individual accelerators or host NICs, though host-side configuration MUST be documented as it materially affects results.

The methodology is designed for controlled laboratory environments per the BMWG charter; it is NOT intended for production network measurement.

## Relationship to Existing BMWG Work

| Document | Relationship |
|---|---|
| {{RFC1242}} | Base terminology for network benchmarking; terms reused herein |
| {{RFC2544}} | Base methodology; throughput/latency/loss tests adapted for RDMA |
| {{RFC2889}} | LAN switching methodology; MAC learning concepts adapted for ARP/ND scale |
| {{RFC8238}} | Data center terminology; buffer, congestion, and microburst terms extended |
| {{RFC8239}} | Data center methodology; line-rate and buffer tests adapted for RoCEv2 |
| {{RFC9004}} | Back-to-back frame updates; burst absorption methodology referenced |
| {{LLM-BENCH}} | Complementary document benchmarking the inference serving stack. Treats the network as opaque SUT. This document benchmarks the fabric itself. The two documents MAY be used together but MUST NOT be combined in a single benchmarking report without explicit section demarcation. |
| {{UEC-1.0}} | UET protocol specification; transport services, congestion control, and link-layer enhancements benchmarked in {{test-uec}} |
{: #tab-existing-work title="Relationship to Existing BMWG Work"}

# Terminology and Definitions

The following terms are defined for use in this document. Where a term overlaps with {{RFC1242}} or {{RFC8238}}, the definition herein takes precedence in the context of AI fabric benchmarking.

| Term | Definition |
|---|---|
| **AI Fabric** | The dedicated Ethernet backend network interconnecting accelerators (GPUs/XPUs) for distributed AI training, typically a non-blocking Clos topology running RoCEv2 |
| **JCT** (Job Completion Time) | The wall-clock duration from start to completion of a training job, inclusive of all computation and communication phases |
| **DUT Fabric** | All leaf switches, spine switches, superspine switches (if applicable), and interconnecting links forming the AI training fabric |
| **Roofline JCT** | Theoretical minimum JCT assuming perfect (zero-contention) network behavior |
| **JCT Ratio** | Measured JCT / Roofline JCT; 1.0 = no network overhead; >1.0 = fabric inefficiency |
| **BusBW** (Bus Bandwidth) | Effective per-accelerator throughput during a collective: (data_size x algo_factor) / time |
| **QP** (Queue Pair) | RDMA communication endpoint (Send Queue + Receive Queue); multiple QPs per src-dst pair increase ECMP entropy |
| **Incast Ratio** | Ratio of senders to receivers (e.g., N:1 incast) |
| **MMR** (Max-Mean Ratio) | Flow count on most-loaded link / average flow count; quantifies ECMP imbalance (1.0 = perfect) |
| **PFC Pause Event** | Single PFC PAUSE frame transmitted on a priority class |
| **ECN Marking Ratio** | % of packets marked with Congestion Experienced (CE) over a measurement interval |
| **Collective Operation** | Coordinated cross-accelerator communication: AllReduce, AlltoAll, AllGather |
| **DCQCN** | Data Center Quantized Congestion Notification: ECN + PFC for end-to-end congestion control with RoCEv2 |
| **Packet Spray** | Load balancing distributing individual packets across all ECMP paths; maximizes utilization but may cause reordering |
| **DLB/Flowlet** | Dynamic Load Balancing using flowlet detection; reroutes traffic at flow idle gaps |
| **Zero-Impact Failover** | Sub-microsecond path convergence upon link/switch failure with no measurable JCT impact |
| **UET** (Ultra Ethernet Transport) | Connectionless RDMA transport defined by UEC Spec 1.0; designed as next-generation replacement for RoCEv2 |
| **PDC** (Packet Delivery Context) | Ephemeral, connectionless UET transport endpoint; analogous to but distinct from an RDMA QP |
| **ROD** | Reliable Ordered Delivery: UET service semantically equivalent to RoCEv2 RC mode |
| **RUD** | Reliable Unordered Delivery: UET service enabling native packet spray without receiver reorder buffer overhead |
| **RUDI** | Reliable Unordered Delivery for Idempotent operations; simplified retransmission for RDMA Writes |
| **UUD** | Unreliable Unordered Delivery: best-effort UET service for telemetry/speculative operations |
| **LLR** (Link Layer Retry) | Optional UEC per-hop error recovery (sub-microsecond) at the Ethernet link layer |
| **Packet Trimming** | Optional UEC enhancement; congested switches transmit packet header only instead of dropping the full packet |
| **CBFC** (Credit-Based Flow Control) | Optional UEC per-destination flow control; alternative to PFC that avoids head-of-line blocking |
| **UEC Profile** | Defined UET feature subset: AI Base, AI Full, or HPC |
| **Entropy Value** | Explicit per-packet UET field for ECMP path selection; improves multipath utilization vs. 5-tuple hashing |
{: #tab-terminology title="Terminology and Definitions"}

# Test Topology and Architecture

## Reference Fabric Topologies

Three reference topologies are defined. All tests MUST specify which topology is used. Results obtained under different topologies are NOT directly comparable without normalization.

### Topology A: 2-Tier Clos (Leaf-Spine)

~~~ ascii-art
+--------+ +--------+ +--------+ +--------+
| Spine1 | | Spine2 | | Spine3 | | SpineN |
+--++--+  +--++--+  +--++--+  +--++--+
   ||         ||         ||         ||
   ||   Full Mesh Interconnect       ||
   ||   (ECMP / DLB / Spray)        ||
   ||         ||         ||         ||
+--++--+  +--++--+  +--++--+  +--++--+
| Leaf 1 | | Leaf 2 | | Leaf 3 | | Leaf N |
+--++--+  +--++--+  +--++--+  +--++--+
   ||         ||         ||         ||
[GPU/XPU] [GPU/XPU] [GPU/XPU] [GPU/XPU]
Hosts w/  Hosts w/  Hosts w/  Hosts w/
RoCEv2 NIC            RoCEv2 NIC
~~~
{: #fig-topo-a title="Topology A: 2-Tier Clos (Leaf-Spine)"}

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
|<------ Rail-0 ------->|         |<-Rail-7->|
~~~
{: #fig-topo-c title="Topology C: Rail-Optimized"}

In rail-optimized topologies, each NIC on a multi-NIC host connects to a dedicated leaf switch ("rail"), co-optimizing network locality with the collective communication library (NCCL/RCCL). The DUT boundary and rail mapping MUST be fully documented.

## Device Under Test (DUT) Identification

| Parameter | Description | Example |
|---|---|---|
| Switch Vendor/Model | Vendor name, product family, model number | Vendor Family Model |
| Switch ASIC | Silicon vendor, ASIC family, revision | Silicon Vendor ASIC Family Rev |
| NOS Version | Network operating system name and version | NOS Name Version |
| Port Speed | Per-port line rate | 400GbE, 800GbE |
| Buffer Architecture | Shared/dedicated, total buffer per ASIC/port | 32MB shared + 16MB VOQ per port |
| Optics/Cables | Transceiver type, cable type and length | OSFP 400G-DR4, DAC 3m copper |
| NIC Vendor/Model | RDMA NIC vendor, model, firmware | NIC Vendor Model Speed |
| NIC Firmware | NIC firmware version | Firmware Version |
| Host Config | OS, CCL lib version, driver, BIOS settings | OS Version, CCL Version, OFED Version |
{: #tab-dut-id title="DUT Identification Parameters"}

## Traffic Generator Requirements

### Mandatory Functional Capabilities

The traffic generator MUST support: RoCEv2 transport emulation (QP establishment, RDMA Write/Read, ECN processing, DCQCN rate control); configurable QP scaling (1-256 QPs per source-destination pair); programmable collective communication patterns (AllReduce, AlltoAll, AllGather with configurable message sizes); and nanosecond-precision timestamping.

### Minimum Measurement Accuracy Requirements

| Parameter | Minimum Requirement |
|---|---|
| Timestamp accuracy | <= 100 nanoseconds |
| Frame rate accuracy | +/- 0.1% of specified rate |
| QP scaling range | 1 to 256 QPs per src-dst pair |
| Message size range | 1 KB to 8 GB |
| Flow counter resolution | Per-flow byte and packet counts |
| Loss measurement | 0 ppm resolution |
| Burst generation | 1-1000 frames at line rate |
{: #tab-tgen-accuracy title="Minimum Measurement Accuracy Requirements"}

### Acceptable Implementations

Qualifying platforms: (a) dedicated hardware traffic generators at line-rate RDMA emulation meeting the accuracy requirements of {{minimum-measurement-accuracy-requirements}}, or (b) instrumented GPU clusters with RDMA tooling with fully documented host configuration. When GPU clusters are used, any non-fabric overhead MUST be quantified and reported separately.

# KPI Framework and Metrics Taxonomy

> Target values in this section are NON-NORMATIVE illustrative reference points derived from current industry practice. They do NOT constitute benchmarking acceptance criteria. Per BMWG charter, defining acceptance criteria is explicitly out of scope. Implementers MAY use these values as contextual references; they MUST NOT be used as pass/fail thresholds.

## Primary KPIs

| KPI | Unit | Definition | Reference Target (Non-Normative) |
|---|---|---|---|
| Job Completion Time (JCT) | seconds | Wall-clock time for benchmark iteration (compute + communication) | Minimize |
| JCT Ratio | dimensionless | Measured JCT / Roofline JCT | <= 1.05 (<= 1.15 acceptable) |
| Bus Bandwidth (BusBW) | Gbps/accelerator | Effective per-accelerator throughput during collective | >= 90% of NIC line rate (intra-pod) |
| Aggregate Throughput | Tbps | Total fabric goodput during collective phase | >= 95% of bisection BW |
| Packet Drop Rate | ppm | Frames lost end-to-end not retransmitted | 0 ppm (lossless) |
| Tail Latency (P99/P99.9) | us | 99th/99.9th percentile one-way fabric latency | Minimize |
{: #tab-primary-kpis title="Primary KPIs"}

## Secondary KPIs

| KPI | Unit | Definition |
|---|---|---|
| ECN Marking Ratio | % | Percentage of packets marked CE over measurement interval |
| PFC Pause Count | events/sec | Rate of PFC PAUSE frames per priority per port |
| PFC Pause Duration | us | Cumulative time a port is in PFC-paused state per interval |
| RDMA Retransmission Rate | retx/sec | NIC-level retransmissions due to timeouts or NAKs |
| ECMP Imbalance (MMR) | dimensionless | Max-Mean Ratio of flow counts across parallel uplinks |
| Jain Fairness Index (JFI) | 0.0-1.0 | Fairness of traffic distribution; 1.0 = perfect |
| Queue Depth (P95/Max) | bytes or cells | 95th percentile and maximum egress queue occupancy per port |
| Congestion Control Convergence | us | Time from congestion onset to DCQCN rate stabilization |
| Out-of-Order Packet Rate | pkt/sec | Packets delivered out of sequence (relevant for packet spray) |
| CTS/ACK Delay | us | Delay for control messages (Clear-to-Send, ACKs) |
{: #tab-secondary-kpis title="Secondary KPIs"}

## Fabric Health Indicators

| Indicator | Unit | Definition |
|---|---|---|
| Switch CPU Utilization | % | Average and peak CPU usage on DUT control plane during test |
| Switch Memory Utilization | % | Average and peak memory usage, including FIB/MAC table occupancy |
| FIB/Route Convergence Time | ms | Time to converge routing after topology change |
| Link Flap Count | events | Spurious link state changes during test period |
| CRC/FCS Error Rate | errors/sec | Physical layer errors indicating cable or optics issues |
| Power Consumption | Watts | Per-switch and per-port power draw under test load |
{: #tab-fabric-health title="Fabric Health Indicators"}

# Test Category 1: RDMA Transport Benchmarks {#test-rdma}

These tests establish baseline fabric performance for RDMA traffic independent of collective communication patterns. They extend {{RFC2544}} and {{RFC8239}} methodology for RoCEv2 semantics.

## Baseline Throughput

**Objective:** Determine the maximum sustainable RDMA Write throughput through the DUT fabric at each tested message size.

**Procedure:**

- Configure N host pairs, each establishing Q Queue Pairs per pair
- Initiate RDMA Write operations and measure aggregate goodput
- Test MUST run for at least 60 seconds at each rate
- Binary search per {{RFC2544}} Section 26.1 SHOULD be used
- Message sizes: 64B, 256B, 1KB, 4KB, 64KB, 256KB, 1MB, 4MB
- QP counts: 1, 4, 16, 32 per src-dst pair
- Test both unidirectional and bidirectional traffic

**Reporting:** Report aggregate throughput (Tbps), per-port utilization (%), and throughput efficiency (measured/theoretical). Present as table indexed by message size x QP count, and as graph (message size on X-axis).

## Latency Characterization

**Objective:** Determine one-way and round-trip RDMA latency distribution at the throughput rate from {{baseline-throughput}}.

**Procedure:**

- Inject tagged frames at 60s into a 120s stream (per {{RFC2544}} Section 26.2)
- Nanosecond-precision timestamping
- MUST report: min, mean, P50, P95, P99, P99.9, max
- Repeat at least 20 times; report averages
- Test under both zero-load (single QP) and loaded (full fabric utilization) conditions

**Reporting:** Tabulate latency statistics per message size. Provide histogram and CDF plot. Report latency increase factor (loaded/unloaded).

## Back-to-Back Burst Absorption

**Objective:** Characterize the DUT fabric's ability to absorb back-to-back RDMA bursts without loss, extending {{RFC9004}} methodology for RoCEv2.

**Procedure:**

- Transmit bursts at line rate with minimum inter-frame gap
- Increase burst length until first frame loss is detected
- Test incast ratios: 2:1, 4:1, 8:1, 16:1, 32:1
- Repeat at least 50 times per burst length

**Reporting:** Report burst absorption capacity (frames and bytes) for each message size and incast ratio. Plot burst capacity vs. incast ratio.

# Test Category 1A: UEC Transport Protocol Benchmarks {#test-uec}

The Ultra Ethernet Consortium (UEC) Specification 1.0 {{UEC-1.0}} defines UET, a connectionless RDMA transport designed to replace RoCEv2 for AI/HPC workloads. All UET tests use the libfabric API {{LIBFABRIC}} and run on UEC 1.0-compliant NICs.

The UEC compliance profile (AI Base, AI Full, or HPC) used during testing MUST be documented.

## UET Throughput by Transport Service

**Objective:** Determine maximum sustainable throughput under each UET transport service (ROD, RUD, RUDI, UUD) and compare to RoCEv2 RC/UC on the same DUT fabric.

**Procedure:** Use UEC 1.0-compliant NICs; establish PDCs; use libfabric fi_write. Apply binary search ({{RFC2544}} Section 26.1). Vary PDC counts: 1, 4, 16, 32. A parallel RoCEv2 test series MUST be executed. Both unidirectional and bidirectional configurations MUST be tested.

**Reporting template:**

| Metric | ROD | RUD | RUDI | UUD | RoCEv2 RC | RoCEv2 UC |
|---|---|---|---|---|---|---|
| Throughput @ 1MB (Gbps) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| Throughput @ 4MB (Gbps) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| Efficiency (% line rate) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| PDC/QP Setup Time (us) | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
| Max Sustained PDC/QP Count | (meas) | (meas) | (meas) | (meas) | (meas) | (meas) |
{: #tab-uet-throughput title="UET Throughput by Transport Service"}

## UET Latency Characterization

**Objective:** Measure latency distribution for UET transport services; quantify differential vs. RoCEv2, with particular attention to connectionless PDC establishment overhead.

**Procedure:** Measure latency for: (a) steady-state PDC transfers; (b) first-packet latency (PDC + first data packet, measuring "data before handshake"); (c) zero-load baseline. Test ROD and RUD separately to isolate reordering-related latency.

**Reporting:** Tabulate latency statistics per (transport_service, message_size, load_condition) tuple. Plot latency CDF for UET ROD, UET RUD, and RoCEv2 RC side-by-side.

## Packet Spray Efficacy Under UET RUD

**Objective:** Quantify the load balancing improvement achieved by UET's native per-packet spray with RUD, which eliminates the receiver reorder buffer constraint.

**Procedure:** Test four configurations:

- UET RUD + packet spray
- UET ROD + packet spray
- RoCEv2 RC + packet spray
- RoCEv2 RC + standard ECMP (baseline)

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

> UET RUD SHOULD achieve zero host-visible reordering despite per-packet spray because the transport layer natively tolerates unordered delivery.

## UET Congestion Control Benchmarks

**Objective:** Evaluate UET's dual-sided (sender + receiver) congestion control under N:1 incast conditions vs. RoCEv2 DCQCN.

**Procedure:** Measure: (a) incast throughput at N = {2, 4, 8, 16, 32, 64}; (b) convergence time after doubling active senders (until all flows within 10% of fair share); (c) PFC avoidance with PFC disabled on the DUT; (d) receiver credit utilization.

**Reporting:** Tabulate incast throughput, convergence time, peak queue depth, PFC event count, and packet drop rate for UET vs. DCQCN per incast ratio. **Critical differentiator:** report whether UET achieves zero application-visible loss without PFC.

## Link Layer Enhancement Benchmarks

**Objective:** Measure performance impact of optional link-layer enhancements: LLR, Packet Trimming (PT), and CBFC.

**Procedure:**

- **(a) LLR Retry Latency:** inject controlled bit errors; measure LLR retry latency (expected sub-microsecond per hop) vs. transport-layer retransmission (~10-100us RTT). Run with 80% background load.
- **(b) Packet Trimming Effectiveness:** configure 2:1 oversubscription bottleneck; measure time from congestion onset to first retransmission request, bandwidth saved vs. full-packet drops.
- **(c) CBFC vs. PFC:** identical N:1 (N=32) incast scenarios; measure head-of-line blocking duration (CBFC is per-destination, PFC is per-priority), pause propagation hops, and throughput of non-congested flows.

**Reporting:** Before/after comparison table for each enhancement. Note which features are hardware-supported vs. software-emulated.

## UET Collective Communication Performance

**Objective:** Measure collective communication (AllReduce, AlltoAll, AllGather) performance over UET and compare to RoCEv2.

**Procedure:** Execute the collective benchmark suite from {{test-collective}} over UET RUD transport using a UEC-compliant collective library. Same accelerator count, message sizes, and fabric topology. Run UET RUD + packet spray as primary; UET ROD + ECMP as secondary baseline.

**Reporting template:**

| Collective | Msg Size | N Accels | UET RUD BusBW | UET ROD BusBW | RoCEv2 RC BusBW | Delta UET/RoCEv2 |
|---|---|---|---|---|---|---|
| AllReduce | 1GB | 128 | (meas) | (meas) | (meas) | (meas) |
| AllReduce | 1GB | 512 | (meas) | (meas) | (meas) | (meas) |
| AlltoAll | 1GB | 128 | (meas) | (meas) | (meas) | (meas) |
| AllGather | 1GB | 128 | (meas) | (meas) | (meas) | (meas) |
{: #tab-uet-collective title="UET Collective Communication Performance"}

## UET PDC Scalability and Connection Setup Rate

**Objective:** Measure PDC establishment rate and maximum concurrent PDC count vs. RoCEv2 QP-based connections.

**Procedure:** (a) PDC establishment rate: initiate PDC creation to M = {100, 1000, 10000, 100000} remote endpoints. (b) Data-before-handshake: measure first-byte latency for UET vs. RoCEv2 RDMA Write. (c) Maximum concurrent PDC count: scale until per-PDC throughput drops below 90% of single-PDC rate. The UEC specification targets up to 1 million endpoints.

# Test Category 2: Congestion Management {#test-congestion}

AI training workloads generate repetitive micro-congestion during the back-propagation gradient synchronization phase.

## ECN Marking Accuracy and Threshold

**Objective:** Verify that the DUT marks packets with ECN CE at the configured threshold with correct granularity.

**Procedure:** Configure threshold T on DUT egress queue. Verify: (a) no packets marked below T; (b) 100% marked above maximum threshold; (c) appropriate WRED/RED probability ramp between thresholds. Test thresholds: low (~100KB), medium (~1MB), high (~5MB).

**Reporting:** Plot ECN marking probability vs. instantaneous queue depth. Report measured threshold accuracy (deviation from configured).

## PFC Behavior Under Incast

**Objective:** Characterize DUT's PFC generation behavior under N:1 incast conditions.

**Procedure:** Generate N:1 incast at 100% line rate, N = {2, 4, 8, 16, 32, 64}. Measure PFC PAUSE frame count/sec per hop, PFC PAUSE duration per port, PFC storm onset, and end-to-end throughput. Test SHOULD verify correct headroom sizing and PFC watchdog effectiveness.

## DCQCN Convergence Time

**Objective:** Measure time for DCQCN to converge to fair-share rate after congestion onset.

**Procedure:** Establish M flows through a common bottleneck. At T0, inject additional M flows (creating 2:1 oversubscription). Measure time until all 2M flows achieve rates within 10% of fair share. Repeat for M = {4, 16, 64, 256}. Vary DCQCN parameters and report sensitivity.

## PFC Storm and Deadlock Resilience

**Objective:** Verify the DUT does not enter PFC deadlock or sustained PFC storm under adversarial traffic.

**Procedure:** Generate cyclic traffic patterns known to cause PFC deadlocks. Run for 300 seconds. The DUT MUST demonstrate resilience via PFC watchdog or architectural immunity (e.g., VOQ-based scheduling).

# Test Category 3: Load Balancing Efficacy {#test-lb}

Load balancing across parallel fabric paths is critical for AI training fabrics because the traffic consists of a small number of high-bandwidth, long-lived elephant flows.

## ECMP Entropy and Polarization

**Objective:** Quantify traffic polarization under standard ECMP hashing for AI training flow patterns.

**Procedure:** Configure standard 5-tuple ECMP. Generate traffic with Q = {1, 4, 8, 16, 32} QPs per src-dst pair. Measure per-link utilization, MMR, and JFI. Test with and without BTH-aware hashing. Repeat for fabric sizes of 8, 16, 32, and 64 leaf switches.

## Dynamic Load Balancing (Flowlet)

**Objective:** Evaluate DUT's flowlet-based DLB performance and compare to baseline ECMP.

**Procedure:** Configure vendor-specific DLB (document algorithm type). Generate traffic with Q=4 QPs. Measure MMR, JFI, per-link utilization, out-of-order rate. Vary flowlet gap timer and report sensitivity.

## Packet Spraying

**Objective:** Evaluate DUT's per-packet spraying performance and quantify the utilization vs. reordering tradeoff.

**Procedure:** Configure per-packet load balancing. Measure MMR (expected ~1.0), JFI (expected ~1.0), out-of-order rate, and RDMA retransmission impact. If the DUT provides an in-fabric reorder buffer, document per {{asic-features}}.

## Jain Fairness Index Measurement

**Objective:** Single-number summary of load balancing quality comparable across all strategies.

**Formula:**

~~~ ascii-art
JFI = (Sum LinkTx_i)^2 / (N x Sum LinkTx_i^2)
~~~
{: #fig-jfi title="Jain Fairness Index Formula"}

where LinkTx_i = transmitted traffic on fabric link i, N = total parallel links. Range: 1/N (worst) to 1.0 (perfect).

**Reporting:** Report JFI for each load balancing strategy. Provide bar chart comparing ECMP, DLB, and packet spray.

# Test Category 4: Collective Communication Benchmarks {#test-collective}

These tests evaluate the fabric's performance under realistic collective communication patterns. Unlike synthetic RDMA tests in {{test-rdma}} and {{test-uec}}, these exercise the full stack including the collective library (NCCL {{NCCL}}, RCCL, or equivalent).

## AllReduce Benchmark

**Objective:** Measure fabric performance during AllReduce operations -- the dominant collective for gradient synchronization in data-parallel training.

**Procedure:**

- Message sizes: 1MB, 8MB, 64MB, 256MB, 1GB, 4GB
- Accelerator counts: N = {8, 16, 32, 64, 128, 256, 512, 1024}
- Execute at least 100 iterations per (message_size, N) pair
- Report average, P50, P95, P99 BusBW
- Run under each load balancing strategy (ECMP, DLB, spray)

## AlltoAll Benchmark

**Objective:** Measure fabric performance during AlltoAll operations -- used in Mixture-of-Experts (MoE) models and expert parallelism.

**Procedure:** Execute AlltoAll with same parameters as {{allreduce-benchmark}}. AlltoAll creates the worst-case congestion scenario: every accelerator simultaneously sends to every other. Report JCT per iteration -- the most sensitive indicator of fabric congestion management quality.

## AllGather Benchmark

**Objective:** Measure fabric performance during AllGather operations -- used for parameter distribution in tensor-parallel and pipeline-parallel training.

**Procedure:** Execute AllGather with same parameter sweeps as {{allreduce-benchmark}}. Measure BusBW, JCT, and fabric health indicators.

## Collective Communication Library Bus Bandwidth Summary

**Reporting template:**

| Load Balancing Config | MMR | JFI | OOO Rate | Retx Rate | Effective Goodput (%) |
|---|---|---|---|---|---|
| UET RUD + Packet Spray | (meas) | (meas) | (meas) | (meas) | (meas) |
| UET ROD + Packet Spray | (meas) | (meas) | (meas) | (meas) | (meas) |
| RoCEv2 RC + Packet Spray | (meas) | (meas) | (meas) | (meas) | (meas) |
| RoCEv2 RC + ECMP (baseline) | (meas) | (meas) | (meas) | (meas) | (meas) |
| UET RUD + DLB/Flowlet | (meas) | (meas) | (meas) | (meas) | (meas) |
{: #tab-ccl-summary title="Collective Communication Bus Bandwidth Summary"}

# Test Category 5: Job Completion Time (JCT) Benchmarks {#test-jct}

JCT is the single most important user-facing KPI for AI training fabrics, directly determining accelerator utilization and training cost.

## Synthetic JCT Under Controlled Conditions

**Objective:** Measure JCT for a defined synthetic workload with a known computation-to-communication ratio to isolate fabric-induced overhead.

**Procedure:** Define a synthetic training iteration as:

1. Computation phase of C milliseconds (simulated sleep or GPU compute kernel)
2. Communication phase: AllReduce of S bytes across N accelerators

| Parameter | Values |
|---|---|
| Computation time C | 10ms, 50ms, 100ms, 500ms |
| Message size S | 256MB, 1GB, 4GB |
| Accelerator count N | 64, 128, 256, 512, 1024 |
| Iterations | 1000 |
{: #tab-synthetic-jct-params title="Synthetic JCT Test Parameters"}

~~~ ascii-art
Roofline JCT = Iterations x (C + S x algo_factor / NIC_line_rate)
JCT Ratio    = Measured_JCT / Roofline_JCT
~~~
{: #fig-jct-formula title="JCT Ratio Calculation"}

> JCT Ratio < 1.05 = excellent fabric performance; > 1.15 = significant fabric-induced overhead.

## MLPerf-Aligned JCT

**Objective:** Measure JCT using MLPerf Training benchmark workloads {{MLPERF}} to enable comparison with published industry results.

**Procedure:** Execute MLPerf Training closed-division workloads (e.g., BERT, ResNet, GPT-3 175B) per MLPerf submission rules. Simultaneously capture all fabric KPIs from {{kpi-framework-and-metrics-taxonomy}}. Report time-to-train and/or tokens-per-second.

## Multi-Tenant JCT Interference

**Objective:** Quantify JCT impact when multiple training jobs share the same fabric.

**Procedure:** Configure two or more independent training jobs. Jobs SHOULD overlap in spine-layer link usage. Measure baseline JCT (isolated) and contention JCT (simultaneous).

~~~ ascii-art
JCT Interference Factor = Contention_JCT / Baseline_JCT
~~~
{: #fig-jct-interference title="JCT Interference Factor"}

Test with spine link overlap: 0%, 25%, 50%, 75%.

# Test Category 6: Scale and Convergence {#test-scale}

## Fabric Scale Limits

**Objective:** Determine the maximum fabric scale at which the DUT maintains acceptable KPI performance.

**Procedure:** Progressively increase active accelerator endpoints from N=64 to maximum topology support while running AllReduce ({{allreduce-benchmark}}, S=1GB). At each scale point record JCT Ratio, BusBW, ECN ratio, PFC count, CPU and memory utilization. Also measure BGP/routing convergence time after clearing all adjacencies (analogous to {{EVPN-BENCH}} Section 6).

## Link Failure Convergence

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

# Test Category 7: Soak and Stability {#test-soak}

## 24-Hour Sustained Load {#soak-24h}

**Objective:** Verify DUT fabric stability under sustained AI training load over an extended period, following the methodology pattern from {{EVPN-BENCH}} Section 7.

**Procedure:** Configure DUT at maximum validated scale from {{fabric-scale-limits}}. Generate bidirectional collective communication traffic (alternating AllReduce and AlltoAll). Run continuously for 24 hours. Sample all KPIs from {{kpi-framework-and-metrics-taxonomy}} every 60 seconds.

There SHOULD NOT be any memory leaks, crashes, or CPU spikes. Any anomaly MUST be reported with timestamp and duration.

**Reporting:** Time-series plots of JCT Ratio, BusBW, ECN ratio, PFC count, CPU, and memory over the 24-hour period. Report standard deviation of JCT Ratio (stability metric).

## Resource Leak Detection

**Objective:** Detect memory leaks, handle exhaustion, or gradual performance degradation in DUT software.

**Procedure:** Record per-process memory usage at T=0, T=1h, T=6h, T=12h, T=24h. Compute linear regression slope of memory usage over time. A slope exceeding **1MB/hour** for any process indicates a potential memory leak and MUST be reported. Also monitor forwarding-plane counter wraparounds and hardware table occupancy trends.

# Reporting Format {#reporting}

Test reports MUST include the following sections:

1. **DUT Identification:** Complete parameters from {{device-under-test-dut-identification}} for all fabric components.
2. **Test Topology:** Diagram and description per {{reference-fabric-topologies}}, including physical cabling.
3. **Test Configuration:** All DUT configuration parameters: QoS policies (ECN thresholds, PFC headroom, DCQCN parameters), load balancing mode, buffer allocation, and vendor-specific tuning.
4. **Host Configuration:** Complete host stack description per {{device-under-test-dut-identification}} including NIC firmware, driver, collective library version, and any tuning. For UET tests, additionally report: UEC compliance profile, libfabric provider version, NIC UEC firmware version, and enabled optional link-layer features (LLR, Packet Trimming, PRI, CBFC).
5. **Test Results:** For each test from {{test-rdma}} through {{test-soak}}, provide specified tables, graphs, and statistical summaries. For {{test-uec}} tests, results MUST include side-by-side UET vs. RoCEv2 comparison data on the identical DUT fabric.
6. **Anomalies:** Any deviations from specified procedures, test failures, or unexpected behaviors MUST be documented.
7. **Repeatability Statement:** Report iteration count and coefficient of variation (std deviation / mean) for each test's primary metric. CV below 5% is RECOMMENDED for test validity.

# Security Considerations

This document defines benchmarking methodologies for controlled laboratory environments and does not introduce new security mechanisms or protocols.

Per {{RFC6815}}, the tests defined herein MUST NOT be performed on production networks. The use of dedicated test IP address ranges per {{RFC2544}} Appendix C (198.18.0.0/15) is RECOMMENDED to prevent accidental interaction with production infrastructure.

When RDMA/RoCEv2 traffic is used, the test environment SHOULD be isolated from production RDMA fabrics to prevent QP number space collisions or inadvertent PFC propagation. When UET traffic is used ({{test-uec}}), the test environment MUST ensure that UDP port 4793 traffic does not leak to production networks and that PDC identifier spaces are isolated. UET's optional transport security sub-layer (TSS) SHOULD NOT be enabled during performance benchmarking unless transport security overhead is explicitly being measured.

# IANA Considerations

This document makes no request of IANA.

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
| AlltoAll JCT | {{alltoall-benchmark}} | CCL benchmark | seconds per iteration |
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
| UET Collective BusBW | {{uet-collective-communication-performance}} | AllReduce/AlltoAll over UET | Gbps per accelerator |
| PDC Establishment Rate | {{uet-pdc-scalability-and-connection-setup-rate}} | Sustained PDC creation rate | PDCs/second |
| Max Concurrent PDCs | {{uet-pdc-scalability-and-connection-setup-rate}} | Scale limit per NIC | count |
{: #tab-kpi-mapping title="KPI-to-Test Mapping Summary"}

# ASIC Feature Categories (Informational) {#asic-features}

This appendix identifies ASIC feature categories relevant to AI fabric performance. Implementers SHOULD document which categories are present and enabled on the DUT. Specific vendor names are intentionally omitted.

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

All values MUST be reported based on vendor documentation or measured capability. Additional DUT capabilities affecting benchmark results MUST also be documented.

# RoCEv2 Test Frame Format {#rocev2-frame}

| Offset | Field | Size | Value / Description |
|---|---|---|---|
| 00 | Ethernet Dst MAC | 6B | DUT next-hop MAC |
| 06 | Ethernet Src MAC | 6B | Test equipment MAC |
| 12 | EtherType | 2B | 0x8100 (802.1Q) or 0x0800 (IPv4) |
| 14 | VLAN Tag (optional) | 4B | PCP=3 (RoCEv2 priority), VID |
| 18 | IPv4 Header | 20B | DSCP=26 (ECN-capable), Proto=17 (UDP) |
| 38 | UDP Header | 8B | DstPort=4791 (RoCEv2), SrcPort=var |
| 46 | BTH (Base Transport Header) | 12B | OpCode, DstQP, PSN, P_Key |
| 58 | RETH (if Write) | 16B | VA, R_Key, DMA Length |
| 74 | Payload | var | Test data (incrementing octets) |
| var | ICRC | 4B | Invariant CRC |
| var+4 | FCS | 4B | Ethernet Frame Check Sequence |
{: #tab-rocev2-frame title="RoCEv2 Test Frame Format"}

# UET (Ultra Ethernet Transport) Frame Format {#uet-frame}

UET runs over UDP/IP using IANA-assigned destination port 4793.

| Offset | Field | Size | Value / Description |
|---|---|---|---|
| 00 | Ethernet Dst MAC | 6B | DUT next-hop MAC |
| 06 | Ethernet Src MAC | 6B | Test equipment MAC |
| 12 | EtherType | 2B | 0x8100 (802.1Q) or 0x0800 (IPv4) |
| 14 | VLAN Tag (optional) | 4B | PCP=3 (UET priority class), VID |
| 18 | IPv4 Header | 20B | DSCP=26, ECN=ECT(0), Proto=17 (UDP) |
| 38 | UDP Header | 8B | DstPort=4793 (UET), SrcPort=entropy |
| 46 | UET Common Header | 16B | Version, OpCode, PDC ID, PSN, Entropy Value, Flags |
| 62 | SES Header (Semantic) | var | Operation-specific (Write/Send/etc.) |
| var | PDS Header (Pkt Delivery) | var | Sequence, Credit, Ack fields |
| var | CMS Header (Cong. Mgmt) | var | ECN feedback, rate signals |
| var | Payload | var | Application data |
| var | ICRC | 4B | Invariant CRC |
| var+4 | FCS | 4B | Ethernet Frame Check Sequence |
{: #tab-uet-frame title="UET Frame Format"}

## Key Differences from RoCEv2

| Field | RoCEv2 Value | UET Value | Notes |
|---|---|---|---|
| UDP Dst Port | 4791 | 4793 | IANA-assigned for each protocol |
| Transport Endpoint | QP Number (24b) | PDC ID (variable) | Connectionless in UET |
| Sequence Number | PSN (24b) | PSN (extended) | Larger range for RUD OOO tolerance |
| Congestion Signal | ECN bits only | ECN + CMS sub-header | Sender + receiver signals in UET |
| Entropy Source | UDP src port | Explicit entropy field | Deterministic spray in UET |
| Ordering Guarantee | Always in-order (RC) | Per-service (ROD/RUD) | RUD allows OOO delivery |
| Min Header Overhead | ~74B (Write) | ~78B (est. Write) | Slight increase for sub-layer headers |
{: #tab-rocev2-vs-uet title="RoCEv2 vs. UET Comparison"}

1. **UDP Destination Port:** UET uses port 4793 vs. RoCEv2 port 4791.
2. **Entropy Value:** Explicit entropy field for ECMP path selection. Test equipment MUST vary this field to achieve uniform path distribution.
3. **Transport Service Indicator:** Header encodes transport service (ROD/RUD/RUDI/UUD). Tests MUST set this to match the service being benchmarked.
4. **PDC Identifier:** Connectionless PDC ID replaces RoCEv2's Destination QP. Test equipment MUST track PDC lifecycle for accurate measurement.
5. **Layered Sub-Headers:** UET uses four sub-layers (SES, PDS, CMS, TSS) with variable-length headers. Implementations MUST follow {{UEC-1.0}} Section 4 for wire format details.
6. **Optional Link Layer Headers:** When LLR, Packet Trimming, or PRI features are enabled, additional link-layer framing may be present. Test equipment MUST be configured to recognize and parse these.

# Acknowledgments
{:numbered="false"}

Contributions and review are solicited from the BMWG mailing list (bmwg@ietf.org). The BMWG chairs and Area Director are identified at https://datatracker.ietf.org/group/bmwg/about/.
