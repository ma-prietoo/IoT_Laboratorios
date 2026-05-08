# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. María de los Ángeles Prieto Ortega
2. Mariana Zuluaga Yepes

---

## 1. System Overview

*   **System Type:** [x] Component (Lab 1-2) | [ ] System (Lab 3-6) | [ ] Environment (Lab 7-8)

*   **Description:**

This laboratory extended the RF characterization performed in Lab 1 into a resilient multi-hop Thread mesh network using OpenThread over IEEE 802.15.4 with ESP32-C6 devices.

The objectives were:

- Validate mesh formation with a custom PANID.
- Configure a Thread network using Leader and Router roles.
- Measure multi-hop communication latency.
- Evaluate self-healing behavior after router failure.
- Analyze the architecture using ISO/IEC 30141 viewpoints.
- Compare Thread mesh networking against LoRaWAN star topology.

Three nodes were used:

- Node A → Leader
- Node B → Intermediate Router
- Node C → Final Router

---

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization

*   **To Samuel (Architect):**

The ESP32-C6 radio using OpenThread was validated experimentally. Channel 23 was selected because it showed the lowest interference during the energy scan (-108 dBm). Reliable communication was achieved up to 10 m with 0% packet loss. Beyond 15 m the network became unreliable due to increasing packet loss and NoAck errors.

*   **To Edwin (Ops):**

The Thread network only worked correctly when all nodes shared the same dataset and channel configuration. Packet loss increased strongly after 10 m. The main operational recommendation is to maintain node spacing below 10 m for stable communication in indoor environments.

---

### Lab 2: 6LoWPAN

*   **To Samuel:**

A resilient Thread mesh network was successfully implemented using three ESP32-C6 nodes. A forced multi-hop topology A → B → C was created so node A could not communicate directly with node C.

The routing path was validated using:

```bash
ot neighbor table
ot router table
```

The Tractor Test demonstrated automatic network recovery after disconnecting the intermediate router. The measured healing time was approximately 64 seconds, satisfying the requirement of convergence in less than 120 seconds.

Thread mesh networking was selected over LoRaWAN because actuator control requires lower latency and local resilience.

---

### Lab 3: Thread & CoAP

*   **To Daniela (Customer):**

Pending future laboratory.

---

### Lab 4: Sensors & Control

*   **To Edwin:**

Pending future laboratory.

---

### Lab 5: Border Router

*   **To Daniela:**

Pending future laboratory.

---

### Lab 6: Security

*   **To Edward (Security):**

Pending future laboratory.

---

### Lab 7: Dashboard

*   **To Gustavo (Product):**

Pending future laboratory.

---

### Lab 8: Final Integration

*   **To All:**

Pending future laboratory.

---

## 3. Architecture Decision Records (ADRs)

### ADR-001: Channel 23 Selection for Thread Network

*   **Context:**

IEEE 802.15.4 operates in the 2.4 GHz band, which can contain WiFi interference. A clean channel was required.

*   **Decision:**

Channel 23 was selected.

*   **Rationale:**

It showed the lowest measured energy during the scan (-108 dBm).

*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

### ADR-002: Choosing Thread Mesh Instead of LoRaWAN Star Topology

*   **Context:**

The agricultural IoT deployment requires:

- low-latency actuator control,
- local communication,
- resilience against node failure,
- and multi-hop routing.

Two candidate topologies were evaluated:

- Thread Mesh
- LoRaWAN Star

*   **Decision:**

Thread Mesh using OpenThread was selected.

*   **Rationale:**

Thread provides:

- self-healing behavior,
- IPv6 native communication,
- distributed routing,
- low latency,
- and local recovery after failures.

LoRaWAN offers larger range but depends heavily on a central gateway and has higher latency for actuator control.

The Tractor Test experimentally demonstrated Thread self-healing capability after disconnecting the intermediate router.

*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 SoC | SCD | Sensing/Controlling device |
| 802.15.4 Radio | SCD | Communication subsystem |
| Leader Router | SCD | Coordinates Thread mesh |
| Intermediate Router | SCD | Performs forwarding and routing |
| IPv6 Mesh Routing | SCD → ASD Enabler | Enables distributed communication |
| RF Medium (Air) | PED | Electromagnetic propagation medium |

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Processing | Embedded control | ESP32-C6 SoC | Active | Lab 1 |
| Communication | RF transmission | 802.15.4 Radio | Active | Lab 1 |
| Communication | Mesh routing | Thread Routers | Active | Lab 2 |
| Communication | Self-healing | OpenThread Mesh | Active | Lab 2 |
| Transfer | Propagation | Air (RF medium) | Latent | Lab 1 |

---

## 5. First Principles Reflections

### Lab 1

1. Radio signals weaken with distance because electromagnetic energy spreads over a larger area.
2. Packet loss increases when the signal-to-noise ratio becomes too low for reliable ACK reception.
3. IEEE 802.15.4 is preferred over WiFi for sensors because it consumes less power.
4. A cleaner channel reduces interference and retransmissions.

---

### Lab 2

1. Mesh networking improves resilience because packets can be forwarded dynamically through intermediate routers.
2. Multi-hop communication increases RTT because every hop introduces forwarding delay and MAC-layer contention.
3. Thread routers consume more power because they must keep their radios active continuously.
4. Self-healing behavior is possible because OpenThread dynamically updates neighbor and routing tables after topology changes.

---

## 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20m | 10 m reliable / 20 m degraded | [x] Partial |
| Lab 2: Healing Time | < 120s | 64 s | [x] Pass |
| Lab 2: Multi-hop RTT | Acceptable | ~102 ms avg | [x] Pass |
| Lab 3: CoAP Latency | < 200ms| Pending | [ ] Pending |
| Lab 4: Poll Latency | < 5s | Pending | [ ] Pending |
| Lab 6: DTLS Time | < 3s | Pending | [ ] Pending |

### Multi-hop Latency Results

| Scenario | Min RTT | Avg RTT | Max RTT | Packet Loss |
|---|---|---|---|---|
| Single-hop | ~15 ms | ~35 ms | ~61 ms | 0% |
| Multi-hop | 34 ms | 102.248 ms | 597 ms | 8.4% |

### Tractor Test Results

| Metric | Value |
|---|---|
| Packets Sent | 200 |
| Packets Received | 49 |
| Packet Loss | 75.5% |
| Convergence Time | ~64 s |

The high packet loss during the Tractor Test is expected because the only route between A and C depended on the intermediate router B.

---

## 7. Ethics & Sustainability Checklist

*   [x] **Lab 1:** Verified interference doesn't disrupt neighbors.
*   [ ] **Lab 4:** Data collection minimized (Privacy).
*   [ ] **Lab 5:** System works locally without cloud (Sustainability).
*   [ ] **Lab 6:** Encryption enabled (Privacy).
*   [ ] **Lab 8:** End-of-Life plan considered.

### Additional Ethical Reflection

- The resilience test was performed without intentionally jamming or disrupting other groups.
- The router was disabled after testing to reduce unnecessary spectrum occupation and energy consumption.

---

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational | Labs 1-2 | RF propagation, routing, latency, packet loss |
| Business | Labs 1-2 | Feasibility of ESP32-C6 for agricultural IoT |
| Usage | Labs 1-2 | Recommended deployment spacing and recovery |
| Functional | Lab 2 | Leader/router roles and mesh communication |
| Trustworthiness | Labs 1-2 | Reliability and resilience after node failure |
| Construction | Labs 1-2 | OpenThread configuration and mesh formation |

### Functional Viewpoint Reflection

The Leader role is responsible for:

- topology coordination,
- router allocation,
- and dataset management.

The Router role is responsible for:

- packet forwarding,
- mesh connectivity,
- and self-healing behavior.

Both roles belong to the Functional Viewpoint because they enable distributed communication inside the IoT system.

---

## 9. Trustworthiness Audit (Lab 6+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability | Yes | Packet delivery and recovery measured | Outdoor testing pending |
| Confidentiality | No | Not evaluated yet | DTLS pending |
| Integrity | Partially | ICMP packet verification | Payload integrity not measured |
| Reliability | Yes | RTT and packet loss measured | More repetitions needed |
| Resilience | Yes | Tractor Test performed | Larger mesh pending |
| Safety | Yes | Passive spectrum analysis | None |
| Compliance | Partially | IEEE 802.15.4/OpenThread stack | Regulatory validation pending |

---

## 10. Construction Viewpoint - IoT System Pattern (Lab 8)

| Pattern Element | Category | Your System |
|-----------------|----------|-------------|
| IoT System | Agricultural IoT | GreenField SoilSense |
| IoT Components | Embedded nodes | ESP32-C6 |
| Digital Network | Mesh network | Thread / 6LoWPAN |
| IoT Devices | Sensor-router nodes | ESP32-C6 routers |
| Primary Capability (observation) | Sensing | Environmental monitoring |
| Primary Capability (control) | Actuation | Future irrigation control |
| Secondary Capability (processing) | Embedded processing | ESP32-C6 MCU |
| Secondary Capability (transferring) | RF communication | IEEE 802.15.4 |
| Secondary Capability (storage) | Configuration | OpenThread dataset |
| Interface (network) | IPv6 | Thread mesh |
| Interface (human UI) | CLI | OpenThread CLI |
| Interface (application) | Future CoAP | Pending |
| Supplemental (security) | DTLS | Pending |
| Supplemental (orchestration) | Leader coordination | OpenThread Leader |
| Supplemental (management) | Routing tables | OpenThread routers |

---

# Final Analysis

The laboratory successfully demonstrated resilient multi-hop Thread mesh communication using ESP32-C6 devices and OpenThread.

The network recovered automatically after failure of the intermediate router, validating the self-healing properties of Thread mesh networking.

The measured convergence time was approximately 64 seconds, satisfying the requirement of recovery in less than 120 seconds.

The comparison between single-hop and multi-hop communication showed that RTT increases with routing complexity, but remains acceptable for agricultural actuator-control systems.

From an architectural perspective, Thread mesh networking is better suited than LoRaWAN star topology for distributed resilient actuator systems requiring local low-latency communication.
