# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. María de los Ángeles Prieto Ortega
2. Mariana Zuluaga Yepes

---

# 1. System Overview

* **System Type:** [x] Component (Lab 1-2) | [ ] System (Lab 3-6) | [ ] Environment (Lab 7-8)

* **Description:**

This DDR documents the complete results from:

- Lab 1: RF Characterization
- Lab 2: 6LoWPAN Mesh Networking & Resilience

The project validates IEEE 802.15.4 communication using ESP32-C6 devices and OpenThread.

---

# 2. Lab Log & Stakeholder Summaries

# LAB 1 — RF Characterization

## To Samuel (Architect)

The radio behavior of the ESP32-C6 using OpenThread was empirically validated. The cleanest channel found was channel 23, with an energy level of -108 dBm during the scan.

The link was completely stable up to 10 m, with 0% packet loss. At 20 m, communication was still achieved, but packet loss increased to 20%. From 25 m and 30 m onwards, packet loss was high, between 31% and 52%, meaning the link can no longer be considered reliable for a sensor network that requires high availability.

Conclusion: the ESP32-C6 works correctly for short-range Thread communication, but based on the obtained data, a practical maximum spacing of around 10 m is recommended to ensure reliability.

---

## To Edwin (Ops)

The network operated correctly when both nodes used the same channel. The configured channel was 23. If a node does not join the network or does not respond to ping, the first thing to check is that the `dataset channel` matches on both devices.

It was also observed that as distance increases, `NoAck` errors appear, meaning that a packet was sent but no acknowledgment was received from the other node. This typically occurs due to weak signal, obstacles, antenna orientation, interference, or excessive distance.

Recommendation: keep the nodes at a maximum distance of 10 m for reliable testing in indoor environments or in environments with uncontrolled interference.

---

# LAB 2 — 6LoWPAN Mesh Networking & Resilience

## To Samuel (Architect)

A multi-hop Thread mesh network was successfully formed using three ESP32-C6 nodes.

Topology:

A (Leader) → B (Intermediate Router) → C (Final Router)

The routing path was validated using:

```bash
ot neighbor table
ot router table
```

Node A could not directly communicate with node C and required node B as forwarding router.

A Tractor Test was performed by disconnecting node B during continuous ping transmission. Communication temporarily failed and automatically recovered after the router rejoined the mesh.

Measured convergence time:

- Approximately 64 seconds

Result:

- PASS (< 120 seconds requirement)

The average RTT increased during multi-hop operation, but latency remained acceptable for actuator-control applications.

---

## To Edwin (Ops)

The network automatically detected when the intermediate router failed. During the Tractor Test, node B was physically disconnected, interrupting communication between A and C.

OpenThread logs showed:

```text
error:NoAck
```

After reconnecting the router, the network automatically rebuilt the routing path and communication resumed without reflashing or manual reconfiguration.

Measured convergence time:

```text
Approximately 64 seconds
```

Recommendation:

Thread mesh networking should be preferred over star topologies for agricultural deployments because the network can automatically recover from node failures.

---

# 3. Architecture Decision Records (ADRs)

# LAB 1 — ADRs

## ADR-001: Selection of Channel 23 for the Thread Network

### Context

The Thread network operates over IEEE 802.15.4 in the 2.4 GHz band. This band can also be used by WiFi and other devices, so before creating the network, a channel with low interference must be selected.

An energy scan was performed using:

```bash
ot scan energy 500
```

### Decision

Channel 23 was selected to operate the Thread network.

### Rationale

Channel 23 showed the lowest detected energy level during the scan, at:

```text
-108 dBm
```

A more negative value indicates lower interfering energy on that channel.

* **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

# LAB 2 — ADRs

## ADR-002: Choosing Thread Mesh Instead of LoRaWAN Star Topology

### Context

The agricultural deployment requires:

- low-latency actuator control,
- resilience,
- communication continuity,
- and automatic recovery after node failure.

### Decision

Thread mesh networking was selected instead of LoRaWAN star topology.

### Rationale

Thread mesh networking provides:

- self-healing,
- multi-hop routing,
- IPv6 native communication,
- and local device-to-device communication.

LoRaWAN offers very long range but depends on a centralized gateway.

If the gateway fails, communication is interrupted.

Thread routing allows packets to travel dynamically through intermediate routers.

This behavior was experimentally validated during the Tractor Test.

### Status

* **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

# 4. ISO/IEC 30141 Mapping

## Domain Mapping

| Component | ISO Domain | Justification | Lab Introduced |
|-----------|------------|---------------|----------------|
| ESP32-C6 SoC | SCD | Sensing/Controlling device | Lab 1 |
| 802.15.4 Radio | SCD | Communication subsystem | Lab 1 |
| Antenna | SCD | Physical interface | Lab 1 |
| Air (RF medium) | PED | Electromagnetic propagation | Lab 1 |
| Leader Router | SCD | Coordinates mesh communication | Lab 2 |
| Intermediate Router | SCD | Performs forwarding and routing | Lab 2 |
| Child Device | SCD | End sensing/control node | Lab 2 |
| IPv6 Mesh Routing | ASD Enabler | Enables application reachability | Lab 2 |

---

## Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Processing | Embedded control | ESP32-C6 SoC | Active | Lab 1 y 2 |
| Communication | RF transmission | 802.15.4 Radio | Active | Lab 1 y 2 |
| Communication | Physical interface | Antenna | Active | Lab 1 y 2 |
| Transfer | RF propagation | Air medium | Latent | Lab 1 y 2 |

---

# LAB 2 — ISO Mapping

## Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| Leader Router | SCD | Coordinates mesh communication |
| Intermediate Router | SCD | Performs forwarding and routing |
| Child Device | SCD | End sensing/control node |
| IPv6 Mesh Routing | ASD Enabler | Enables application reachability |

---

## Functional Viewpoint Reflection

### Leader Role

The Leader performs:

- router allocation,
- dataset coordination,
- and topology management.

From the ISO/IEC 30141 Functional Viewpoint, the Leader supports distributed IoT coordination.

---

### Router Role

Routers provide:

- packet forwarding,
- mesh connectivity,
- and resilience.

The Router role supports communication continuity inside the Sensing & Controlling Domain.

---

# 5. First Principles Reflections

# LAB 1

## 1. Why does signal strength decrease with distance?

Electromagnetic energy spreads over a larger area as the wave propagates. Therefore, the received power decreases with increasing distance.

---

## 2. Why does packet loss increase at larger distances?

The signal-to-noise ratio decreases, making packet decoding more difficult. This causes retransmissions and `NoAck` errors.

---

# LAB 2

## 1. Why does mesh networking improve resilience?

In a mesh topology, communication does not depend on a single central node. Multiple routers can forward packets dynamically.

---

## 2. Why does multi-hop increase latency?

Each hop introduces:

- forwarding delay,
- MAC-layer waiting time,
- retransmissions,
- and processing overhead.

---

## 3. Why do routers consume more power?

A Thread router must keep its radio active continuously to:

- listen for packets,
- forward traffic,
- maintain neighbor tables,
- and participate in routing.

---

# 6. Performance Baselines

# LAB 1 — RF Performance

| Metric | Target | Measured | Status |
|---|---|---|---|
| Reliable RF Range | > 20 m | 10 m reliable | Partial |
| Best Channel | Lowest interference | Channel 23 (-108 dBm) | PASS |

---

# LAB 2 — Mesh Performance

| Metric | Target | Measured | Status |
|---|---|---|---|
| Healing Time | < 120 s | 64 s | PASS |
| Multi-hop RTT | Acceptable | ~102 ms avg | PASS |
| Mesh Recovery | Automatic | Successful | PASS |

---

# 7. Ethics & Sustainability Checklist

# LAB 1

- [x] Verified that interference did not disrupt other groups.
- [x] Passive energy scans were used.

---

# LAB 2

- [x] Resilience tests did not intentionally interfere with other groups.
- [x] Router was disabled after testing to reduce spectrum occupation and save energy.

---

# 8. Viewpoint Analysis

# LAB 1

| Viewpoint | Concerns |
|---|---|
| Foundational | RF propagation and packet loss |
| Business | Feasibility of ESP32-C6 sensor nodes |
| Usage | Recommended deployment spacing |
| Functional | IPv6 communication validation |
| Trustworthiness | Reliability degradation with distance |
| Construction | OpenThread + JTAG configuration |

---

# LAB 2

| Viewpoint | Concerns |
|---|---|
| Foundational | Mesh routing and convergence |
| Business | Reliability for actuator control |
| Usage | Recovery after router failure |
| Functional | Leader and Router responsibilities |
| Trustworthiness | Resilience and self-healing |
| Construction | Multi-hop topology creation |

---

# 9. Trustworthiness Audit

# LAB 1

| Characteristic | Addressed? | How | Gaps |
|---|---|---|---|
| Reliability | Yes | Packet loss measured | Outdoor testing pending |
| Safety | Yes | Passive energy scan | None identified |

---

# LAB 2

| Characteristic | Addressed? | How | Gaps |
|---|---|---|---|
| Availability | Yes | Mesh recovery tested | Larger mesh pending |
| Resilience | Yes | Tractor Test completed | More alternate routes needed |
| Reliability | Yes | RTT and packet loss measured | Outdoor testing pending |

---

# 10. Construction Viewpoint - IoT System Pattern

# LAB 1

| Pattern Element | Your System |
|---|---|
| IoT Components | ESP32-C6 |
| Digital Network | IEEE 802.15.4 |
| Communication | IPv6 ping |

---

# LAB 2

| Pattern Element | Your System |
|---|---|
| IoT System | Distributed IoT |
| Digital Network | Thread/OpenThread Mesh |
| IoT Devices | Leader + Routers |
| Communication | IPv6 Multi-hop Routing |

---

# 11. LAB 1 — Final Analysis

The laboratory successfully validated the RF operation of the ESP32-C6 using OpenThread.

Channel 23 was technically justified because it presented the lowest measured interference.

The ESP32-C6 demonstrated reliable communication up to approximately 10 m under the tested conditions.

As the distance increased, packet loss and RTT also increased due to weaker signal conditions and retransmissions.

The laboratory demonstrated the practical importance of channel selection and RF characterization before deploying a sensor network.

---

# 12. LAB 2 — Final Analysis

The laboratory successfully demonstrated resilient Thread mesh networking using ESP32-C6 devices and OpenThread.

A forced multi-hop topology was created and validated experimentally using routing and neighbor tables.

The Tractor Test validated the self-healing behavior of the mesh network.

When the intermediate router failed, OpenThread detected the failure, updated the routing tables, and automatically restored communication after reconnection.

The measured convergence time was approximately:

```text
64 seconds
```

which satisfies the requirement of recovery in less than 120 seconds.

Compared with single-hop communication, multi-hop routing increased RTT and packet loss, but performance remained acceptable for actuator-control applications.
