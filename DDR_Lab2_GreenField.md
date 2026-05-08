# Design & Decision Record (DDR)
# GreenField Technologies | IoT Systems Design

**Team Members:**
1. María de los Ángeles Prieto Ortega
2. Mariana Zuluaga Yepes

---

# 1. System Overview

* **System Type:** [x] Component (Lab 1-2) | [ ] System (Lab 3-6) | [ ] Environment (Lab 7-8)

* **Description:**

This laboratory extended the previous RF characterization into a resilient multi-hop Thread mesh network using ESP32-C6 devices and OpenThread over IEEE 802.15.4. Three nodes were configured as Leader, Intermediate Router, and Final Router. The laboratory validated multi-hop IPv6 communication, routing behavior, mesh healing, and convergence after node failure.

The tests included neighbor table inspection, router table analysis, RTT measurement, packet-loss analysis, and recovery validation after disconnecting the intermediate router.

---

# 2. Lab Log & Stakeholder Summaries

## Lab 1: RF Characterization

### To Samuel (Architect):

The ESP32-C6 radio behavior was validated using OpenThread. Channel 23 was selected after performing an energy scan and identifying it as the cleanest channel with -108 dBm. Reliable communication was achieved up to 10 m with 0% packet loss.

### To Edwin (Ops):

The Thread network operated correctly when all nodes shared the same dataset and channel configuration. `NoAck` errors appeared at larger distances due to weak signal or interference.

---

## Lab 2: 6LoWPAN

### To Samuel:

A multi-hop Thread mesh network was successfully formed using three ESP32-C6 nodes.

Topology:

A (Leader) → B (Intermediate Router) → C (Final Router)

The routing path was validated using `neighbor table` and `router table`. Node A could not directly communicate with node C and required node B as forwarding router.

A Tractor Test was performed by disconnecting node B during continuous ping transmission. Communication temporarily failed and automatically recovered after the router rejoined the mesh.

Measured convergence time:

- Approximately 64 seconds

Result:

- PASS (< 120 seconds requirement)

The average RTT increased during multi-hop operation, but latency remained acceptable for actuator-control applications.

---

## Lab 3: Thread & CoAP

### To Daniela (Customer):

Pending future laboratory.

---

## Lab 4: Sensors & Control

### To Edwin:

Pending future laboratory.

---

## Lab 5: Border Router

### To Daniela:

Pending future laboratory.

---

## Lab 6: Security

### To Edward (Security):

Pending future laboratory.

---

## Lab 7: Dashboard

### To Gustavo (Product):

Pending future laboratory.

---

## Lab 8: Final Integration

### To All:

Pending future laboratory.

---

# 3. Architecture Decision Records (ADRs)

## ADR-001: Selection of Thread Channel 23

### Context:

The Thread network operates in the 2.4 GHz IEEE 802.15.4 spectrum, which may contain interference from WiFi and other devices.

### Decision:

Channel 23 was selected.

### Rationale:

The energy scan showed channel 23 with the lowest interference level:

-108 dBm

This provided the best communication conditions.

### Status:

[x] Accepted

---

## ADR-002: Choosing Thread Mesh Instead of LoRaWAN Star Topology

### Context:

The agricultural deployment requires low-latency actuator control, resilience, and communication continuity even if one node fails.

### Decision:

Thread mesh networking was selected instead of LoRaWAN star topology.

### Rationale:

Thread mesh networking provides:

- self-healing,
- multi-hop routing,
- IPv6 native communication,
- local device-to-device communication,
- and lower latency.

LoRaWAN offers long range but depends on a centralized gateway. If the gateway fails, communication is interrupted.

Thread routing allows packets to travel dynamically through intermediate routers.

This behavior was experimentally validated during the Tractor Test.

### Status:

[x] Accepted

---

# 4. ISO/IEC 30141 Mapping

## Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 Nodes | SCD | Sensing and controlling devices |
| Leader Router | SCD | Coordinates mesh communication |
| Intermediate Router | SCD | Performs forwarding and routing |
| IEEE 802.15.4 Radio | SCD | Communication subsystem |
| RF Medium | PED | Physical propagation environment |
| IPv6 Mesh Routing | ASD Enabler | Enables application reachability |

---

## Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Processing | Embedded control | ESP32-C6 | Active | Lab 1 |
| Communication | RF transmission | 802.15.4 Radio | Active | Lab 1 |
| Communication | Mesh routing | OpenThread | Active | Lab 2 |
| Communication | IPv6 forwarding | Router nodes | Active | Lab 2 |
| Transfer | RF propagation | Air medium | Latent | Lab 1 |

---

# 5. First Principles Reflections

## Lab 1

### 1. Why does signal strength decrease with distance?

Electromagnetic energy spreads over a larger area as the wave propagates. Therefore, the received power decreases with increasing distance.

### 2. Why does packet loss increase at larger distances?

The signal-to-noise ratio decreases, making packet decoding more difficult. This causes retransmissions and `NoAck` errors.

---

## Lab 2

### 1. Why does mesh networking improve resilience?

In a mesh topology, communication does not depend on a single centralized node. Multiple routers can forward packets dynamically.

### 2. Why does multi-hop increase latency?

Each hop introduces:

- forwarding delay,
- MAC-layer waiting time,
- retransmissions,
- and processing overhead.

### 3. Why do routers consume more energy?

Routers must continuously listen and forward packets, keeping the radio active most of the time.

---

# 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20m | 10 m reliable | [x] Partial |
| Lab 2: Healing Time | < 120s | 64 s | [x] Pass |
| Lab 2: Multi-hop RTT | Acceptable | ~102 ms avg | [x] Pass |
| Lab 3: CoAP Latency | < 200ms | Pending | [ ] Pass |
| Lab 4: Poll Latency | < 5s | Pending | [ ] Pass |
| Lab 6: DTLS Time | < 3s | Pending | [ ] Pass |

---

# 7. Ethics & Sustainability Checklist

* [x] **Lab 1:** Verified interference doesn't disrupt neighbors.
* [ ] **Lab 4:** Data collection minimized (Privacy).
* [ ] **Lab 5:** System works locally without cloud (Sustainability).
* [ ] **Lab 6:** Encryption enabled (Privacy).
* [ ] **Lab 8:** End-of-Life plan considered.

Additional observations:

- The router was disconnected and reconnected only during controlled testing.
- The network was disabled after testing to avoid unnecessary spectrum occupation and power consumption.

---

# 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational | Lab 1-2 | RF propagation, latency, routing |
| Business | Lab 1-2 | Reliability for agricultural IoT |
| Usage | Lab 1-2 | Deployment spacing and troubleshooting |
| Functional | Lab 2 | Leader and Router mesh functions |
| Trustworthiness | Lab 1-2 | Packet loss and resilience |
| Construction | Lab 1-2 | OpenThread configuration and topology |

---

# 9. Trustworthiness Audit (Lab 6+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability | Partially | Mesh recovery tested | More redundancy needed |
| Confidentiality | No | Not evaluated | DTLS pending |
| Integrity | Partially | ICMP delivery verified | No payload integrity analysis |
| Reliability | Yes | RTT and packet loss measured | More outdoor testing needed |
| Resilience | Yes | Tractor Test completed | Larger topology pending |
| Safety | Yes | Controlled RF testing | None identified |
| Compliance | Partially | IEEE 802.15.4/OpenThread used | Formal certification not evaluated |

---

# 10. Construction Viewpoint - IoT System Pattern (Lab 8)

| Pattern Element | Category | Your System |
|-----------------|----------|-------------|
| IoT System | Distributed IoT | GreenField SoilSense |
| IoT Components | Embedded Nodes | ESP32-C6 |
| Digital Network | Mesh Network | Thread/OpenThread |
| IoT Devices | Routers and Leader | ESP32-C6 nodes |
| Primary Capability (observation) | Sensing | Future soil monitoring |
| Primary Capability (control) | Actuation | Future irrigation control |
| Secondary Capability (processing) | Embedded processing | ESP32-C6 MCU |
| Secondary Capability (transferring) | Communication | IPv6 over Thread |
| Secondary Capability (storage) | Configuration | Dataset persistence |
| Interface (network) | Wireless | IEEE 802.15.4 |
| Interface (human UI) | CLI | OpenThread CLI |
| Interface (application) | IPv6 | ICMPv6 communication |
| Supplemental (security) | Mesh credentials | Thread dataset |
| Supplemental (orchestration) | Routing | Leader and routers |
| Supplemental (management) | Debugging | JTAG + OpenThread logs |

---

# 11. Task B — Multi-hop Latency Analysis

## Topology Validation

The following commands were used:

```bash
ot neighbor table
ot router table
```

The routing tables demonstrated that node A could not directly communicate with node C.

Packets were routed through node B.

---

## RTT Measurement

Command used:

```bash
ot ping fd6d:b7b5:ebe1:c026:0:ff:fe00:1400 64 500 0.5
```

Results:

| Metric | Value |
|---|---|
| Packets Sent | 500 |
| Packets Received | 458 |
| Packet Loss | 8.4% |
| Min RTT | 34 ms |
| Avg RTT | 102.248 ms |
| Max RTT | 597 ms |

Interpretation:

The mesh remained functional during multi-hop communication. RTT increased compared with single-hop communication due to forwarding overhead and retransmissions.

---

# 12. Task C — Tractor Test (Mesh Healing)

## Procedure

A continuous ping was started from node A to node C:

```bash
ot ping fd6d:b7b5:ebe1:c026:0:ff:fe00:1400 64 200 1
```

Node B was physically disconnected.

The following OpenThread errors appeared:

```text
error:NoAck
```

The neighbor table became empty and routing temporarily failed.

After reconnecting node B, OpenThread automatically restored communication.

---

## Convergence Measurement

Observed sequence numbers:

| Event | icmp_seq |
|---|---|
| Last successful packet before failure | 960 |
| First successful packet after recovery | 1024 |

Because the interval was 1 second:

```text
Convergence Time ≈ 64 seconds
```

Result:

| Metric | Requirement | Measured | Status |
|---|---|---|---|
| Healing Time | < 120 s | 64 s | PASS |

---

# 13. Final Analysis

The laboratory successfully demonstrated resilient Thread mesh networking using ESP32-C6 devices and OpenThread.

The network automatically recovered after router failure and restored communication without reflashing or manual intervention.

The Tractor Test experimentally validated self-healing behavior and confirmed that the convergence requirement of less than 120 seconds was achieved.

Compared with single-hop communication, multi-hop routing increased RTT and packet loss, but performance remained acceptable for agricultural sensing and actuator-control applications.

From an ISO/IEC 30141 perspective, the Leader and Router roles belong to the Functional Viewpoint because they coordinate communication, routing, and distributed system operation.

The laboratory also demonstrated why Thread mesh networking is preferable to LoRaWAN star topology for applications requiring resilience and low latency.
