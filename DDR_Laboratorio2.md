# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. María de los Ángeles Prieto Ortega
2. Mariana Zuluaga Yepes

---

## 1. System Overview

* **System Type:** [x] Component (Lab 1-2) | [ ] System (Lab 3-6) | [ ] Environment (Lab 7-8)

* **Description:**

This DDR documents the first two laboratories of the GreenField Technologies SoilSense project. The work focused on validating the ESP32-C6 as a low-power wireless IoT node using OpenThread over IEEE 802.15.4.

In **Lab 1**, the radiofrequency behavior of the ESP32-C6 was characterized. An energy scan was performed to select a clean channel, and IPv6 ping tests were executed at different distances to evaluate packet loss, RTT, and practical communication range.

In **Lab 2**, the system evolved from a point-to-point link to a Thread mesh network using three ESP32-C6 nodes. The objective was to validate 6LoWPAN mesh routing, force a multi-hop topology, measure latency over multiple hops, and evaluate resilience using the “Tractor Test,” where an intermediate router was physically disconnected and later reconnected.

The integrated system used the following topology for Lab 2:

```text
A - Leader  →  B - Intermediate Router  →  C - Final Router
```

The purpose of this stage was to determine whether Thread mesh networking is suitable for a distributed agricultural sensing and actuation system where local reachability, resilience, and low latency are important.

---

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization

* **To Samuel (Architect):**

The RF behavior of the ESP32-C6 using OpenThread was empirically validated. An energy scan was performed across IEEE 802.15.4 channels, and channel 23 was selected because it showed the lowest measured energy level: **-108 dBm**.

After configuring the Thread network on this channel, IPv6 ping tests were performed at different distances. The link was fully stable up to **10 m**, with **0% packet loss**. At **20 m**, communication was still possible, but packet loss increased to **20%**. At longer distances, such as 25 m, 30 m, and 35 m, packet loss increased significantly, reaching values between **31% and 52%**.

The main architectural conclusion is that the ESP32-C6 is suitable for short-range Thread communication, but a single-hop deployment should not assume reliable communication beyond approximately 10 m under these test conditions. For larger coverage, more router nodes or a mesh topology are required.

* **To Edwin (Ops):**

The network operated correctly when both nodes used the same Thread channel and dataset. The selected channel for Lab 1 was channel 23. If a node does not join the network or does not respond to pings, the first checks should be:

1. verify that both nodes use the same Thread channel,
2. verify that both nodes share the same dataset,
3. check that the devices are correctly flashed and running OpenThread,
4. observe logs for errors such as `NoAck`.

As the distance increased, `NoAck` errors appeared. This means that a packet was transmitted but no acknowledgment was received. This can be caused by distance, weak signal, obstacles, antenna orientation, multipath propagation, or interference.

Operational recommendation: in similar indoor test conditions, use a node spacing of approximately **8 m to 10 m** for reliable communication.

---

### Lab 2: 6LoWPAN

* **To Samuel:**

A Thread mesh network was successfully formed using three ESP32-C6 nodes. The network was commissioned with a custom PANID and a shared dataset. The topology was physically forced so that the Leader node A could not directly communicate with the final router C. This allowed the team to validate real multi-hop routing:

```text
A → B → C
```

The `neighbor table` on A showed only B as a direct neighbor. The `router table` showed C with `Link = 0`, meaning C was not reachable directly by radio from A. The next-hop value confirmed that traffic from A to C passed through B.

The main multi-hop latency test produced the following result:

```text
500 packets transmitted
458 packets received
Packet loss = 8.4%
RTT min/avg/max = 34 / 102.248 / 597 ms
```

Compared with the Lab 1 one-hop reliable baseline of approximately **35.480 ms** average RTT at 10 m, the Lab 2 multi-hop average RTT increased to approximately **102.248 ms**. This shows the expected trade-off: mesh routing extends coverage and improves reachability, but increases latency and jitter.

* **To Edwin:**

The “Tractor Test” was performed by starting a continuous ping from A toward C and physically disconnecting the intermediate router B. When B was disconnected, A showed `NoAck` messages toward the next-hop router `0xf400`, confirming that the active route had been interrupted.

After B was reconnected, OpenThread automatically reattached it to the mesh and restored the route. The logs showed B returning as a router:

```text
Role disabled -> detached
Role detached -> router
```

The last successful ping before the failure was approximately:

```text
icmp_seq = 960
```

The first successful ping after recovery was approximately:

```text
icmp_seq = 1024
```

Since the ping interval was 1 second, the estimated convergence time was:

```text
1024 - 960 = 64 s
```

This satisfies the requirement that the network should heal in less than 120 seconds.

---

### Lab 3: Thread & CoAP

* **To Daniela (Customer):**

Pending. This laboratory has not been performed yet.

---

### Lab 4: Sensors & Control

* **To Edwin:**

Pending. This laboratory has not been performed yet.

---

### Lab 5: Border Router

* **To Daniela:**

Pending. This laboratory has not been performed yet.

---

### Lab 6: Security

* **To Edward (Security):**

Pending. This laboratory has not been performed yet.

---

### Lab 7: Dashboard

* **To Gustavo (Product):**

Pending. This laboratory has not been performed yet.

---

### Lab 8: Final Integration

* **To All:**

Pending. This laboratory has not been performed yet.

---

## 3. Architecture Decision Records (ADRs)

### ADR-001: Selection of Channel 23 for the Initial Thread RF Characterization

* **Context:**

The Thread network operates over IEEE 802.15.4 in the 2.4 GHz band. This band is shared with WiFi and other devices, so it was necessary to select a channel with low interference before running communication tests.

An energy scan was performed using:

```bash
ot scan energy 500
```

* **Decision:**

Channel 23 was selected for the Lab 1 Thread network.

* **Rationale:**

Channel 23 showed the lowest detected energy level during the scan:

```text
Channel 23: -108 dBm
```

A more negative energy level indicates a cleaner channel. Therefore, channel 23 provided the best initial RF conditions for the one-hop characterization tests.

* **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

### ADR-002: Selection of Thread Mesh Topology over LoRaWAN Star

* **Context:**

GreenField SoilSense requires communication between distributed sensing and actuation nodes in an agricultural environment. Lab 1 showed that direct ESP32-C6 communication was reliable at short distances but degraded as distance increased. Therefore, Lab 2 required a topology that could extend coverage and improve resilience.

LoRaWAN is useful for long-range and low-power telemetry, but it commonly uses a star or star-of-stars topology where end devices depend on gateways. This is appropriate for low-rate sensing data, but less suitable for local actuator control where low latency and local reachability are needed.

* **Decision:**

Thread mesh networking was selected over a LoRaWAN star topology for this implementation.

* **Rationale:**

Thread provides local IPv6 mesh routing. Intermediate routers can forward packets, allowing nodes outside direct radio range to remain reachable. This was demonstrated in Lab 2 with the topology:

```text
A → B → C
```

where A could reach C only through B.

For actuator control, such as irrigation valves, latency is important. Thread allows local traffic to stay inside the mesh instead of depending on a distant gateway. The trade-off is that router nodes consume more energy because they must remain awake and ready to forward packets.

* **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 SoC | SCD | Embedded sensing/controlling device used as the physical IoT node. |
| IEEE 802.15.4 Radio | SCD | Communication subsystem used by Thread/OpenThread. |
| Antenna | SCD / PED interface | Physical interface between the device and the RF propagation medium. |
| Air / RF medium | PED | Physical environment where electromagnetic propagation occurs. |
| Thread Leader | SCD | Coordinates the local Thread mesh and maintains routing information. |
| Thread Router | SCD | Forwards packets between nodes and extends mesh coverage. |
| Final Router / Far-field node | SCD | Represents a remote device reachable through the mesh. |
| Mesh-local IPv6 addressing | SCD to ASD enabler | Allows later application services to address devices through IPv6. |
| Future CoAP / dashboard traffic | ASD | Application and service traffic that will use the connectivity built by the mesh. |

---

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Processing | Embedded control | ESP32-C6 SoC | Active | Lab 1 |
| Communication | RF transmission | IEEE 802.15.4 radio | Active | Lab 1 |
| Communication | Physical interface | Antenna | Active | Lab 1 |
| Transfer | RF propagation | Air medium | Latent | Lab 1 |
| Communication | Thread network formation | OpenThread CLI dataset | Active | Lab 1 |
| Communication | Mesh routing | Thread Router role | Active | Lab 2 |
| Communication | Network coordination | Thread Leader role | Active | Lab 2 |
| Communication | IPv6 addressing | ML-EID / RLOC / link-local addresses | Active | Lab 2 |
| Trustworthiness | Resilience | Reattachment and route recovery | Active | Lab 2 |
| Application enablement | Service reachability | Mesh-local IPv6 connectivity | Latent | Lab 2 |

---

## 5. First Principles Reflections

**Lab 1:**

1. **Why does the signal weaken as distance increases?**

A radio signal propagates as an electromagnetic wave. As it moves away from the transmitter, its energy spreads over a larger area. Therefore, received power decreases with distance. In practical terms, as nodes move farther apart, the receiver obtains a weaker signal. If the received signal becomes too close to the noise floor, packet loss and retransmissions increase.

2. **Why does packet loss increase at longer distances?**

At longer distances, the signal-to-noise ratio decreases. When the receiver cannot correctly decode a packet, it does not send an acknowledgment. In the logs, this was observed as:

```text
MeshForwarder: Failed to send IPv6 ICMP6 msg ... error:NoAck
```

This indicates that the transmitter sent a packet but did not receive confirmation. As a result, packet loss increases.

3. **Why is IEEE 802.15.4 used instead of WiFi for sensors?**

IEEE 802.15.4 is designed for low-power and low-data-rate communication. Soil and environmental sensors do not normally need high throughput, but they do need low energy consumption and long operating time. WiFi offers higher data rate but consumes more energy, making it less suitable for small battery-powered sensing nodes.

4. **Why does selecting a clean channel matter?**

An energy scan estimates how much RF energy exists on each channel. A more negative value, such as `-108 dBm`, indicates lower interfering energy. Selecting a cleaner channel reduces the probability of collisions, retransmissions, and packet loss.

---

**Lab 2:**

1. **Why does mesh increase coverage?**

Mesh networking allows intermediate routers to forward packets. If A cannot directly reach C, B can act as a bridge. This extends the effective communication area without requiring every node to be within direct radio range of every other node.

2. **Why does multi-hop increase latency?**

Each hop introduces processing, medium access waiting time, packet forwarding time, and possible retransmissions. Therefore, a route such as A → B → C has a higher RTT than a direct one-hop link.

3. **Why do routers consume more energy than sleepy end devices?**

A router must keep its radio available to receive and forward packets for other nodes. This means it spends more time listening and processing. A sleepy end device can turn off its radio for long periods and wake only when needed. Therefore:

```text
More routers  →  more coverage and resilience  →  higher energy consumption
Fewer routers →  lower consumption             →  lower coverage and resilience
```

4. **Why is Thread Mesh preferable for local actuator control?**

Actuator control requires timely local response. In a Thread mesh, commands can remain inside the local IPv6 network. In a star topology, traffic depends more strongly on gateway availability. For GreenField, local control and recovery are more important than minimizing the power consumption of every node.

---

## 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20 m | 10 m reliable / 20 m with 20% loss | [x] Partial |
| Lab 2: Mesh Formation | Custom PANID | PANID `0x2022`, network `GreenField-G2` | [x] Pass |
| Lab 2: Multi-hop Latency | Compare 1 hop vs multi-hop | 35.480 ms vs 102.248 ms avg RTT | [x] Pass |
| Lab 2: Healing Time | < 120 s | ~64 s | [x] Pass |
| Lab 3: CoAP Latency | < 200 ms | Pending | [ ] Pass |
| Lab 4: Poll Latency | < 5 s | Pending | [ ] Pass |
| Lab 6: DTLS Time | < 3 s | Pending | [ ] Pass |

---

### Lab 1 RF Scan Results

Command used:

```bash
ot scan energy 500
```

| Channel | RSSI [dBm] |
|---:|---:|
| 11 | -93 |
| 12 | -78 |
| 13 | -74 |
| 14 | -81 |
| 15 | -101 |
| 16 | -97 |
| 17 | -83 |
| 18 | -99 |
| 19 | -100 |
| 20 | -101 |
| 21 | -98 |
| 22 | -93 |
| 23 | -108 |
| 24 | -96 |
| 25 | -102 |
| 26 | -96 |

**Cleanest channel:** 23  
**Measured energy:** -108 dBm

---

### Lab 1 Range Testing Results

| Distance | Packets Sent | Packets Received | Packet Loss | Min RTT | Avg RTT | Max RTT | Status |
|---:|---:|---:|---:|---:|---:|---:|---|
| 1 m | 100 | 100 | 0% | 15 ms | 34.920 ms | 53 ms | Reliable |
| 5 m | 100 | 100 | 0% | 15 ms | 38.170 ms | 150 ms | Reliable |
| 10 m | 100 | 100 | 0% | 19 ms | 35.480 ms | 61 ms | Reliable |
| 15 m | 100 | 61 | 39% | 21 ms | 281.524 ms | 1841 ms | Unreliable |
| 20 m | 100 | 80 | 20% | 15 ms | 97.112 ms | 746 ms | Unreliable |
| 25 m | 100 | 69 | 31% | 24 ms | 220.347 ms | 991 ms | Unreliable |
| 30 m | 100 | 48 | 52% | 15 ms | 128.979 ms | 996 ms | Unreliable |
| 35 m | 100 | 48 | 52% | 24 ms | 203.958 ms | 1061 ms | Unreliable |

---

### Lab 2 Commissioning Results

| Parameter | Value |
|---|---|
| Network name | `GreenField-G2` |
| PANID | `0x2022` |
| Stack | OpenThread over IEEE 802.15.4 |
| Hardware | ESP32-C6 |

Node identification:

| Node | Role | Extended Address | Observed Role |
|---|---|---|---|
| A | Leader | `6afd403ebcb1d8bb` | Leader |
| B | Intermediate Router | `3ac785ea0ebdc89c` | Router |
| C | Final Router | `ae482162e2527dc0` | Router |

---

### Lab 2 Latency Results

Target address used:

```text
fd6d:b7b5:ebe1:c026:0:ff:fe00:1400
```

Short multi-hop test:

```text
20 packets transmitted
20 packets received
Packet loss = 0.0%
RTT min/avg/max = 57 / 158.600 / 467 ms
```

Long multi-hop test:

```text
500 packets transmitted
458 packets received
Packet loss = 8.4%
RTT min/avg/max = 34 / 102.248 / 597 ms
```

Comparison:

| Test | Topology | Packets Sent | Packets Received | Packet Loss | Avg RTT |
|---|---|---:|---:|---:|---:|
| Lab 1 reliable baseline | 1 hop, 10 m | 100 | 100 | 0% | 35.480 ms |
| Lab 2 short multi-hop | A → B → C | 20 | 20 | 0% | 158.600 ms |
| Lab 2 long multi-hop | A → B → C | 500 | 458 | 8.4% | 102.248 ms |

---

### Lab 2 Tractor Test Results

Continuous ping command:

```bash
ot ping fd6d:b7b5:ebe1:c026:0:ff:fe00:1400 64 200 1
```

Failure evidence:

```text
error:NoAck
to:0xf400
```

Recovery evidence:

```text
Role disabled -> detached
Role detached -> router
RouterTable: Allocate router id 5
```

Convergence calculation:

| Event | Sequence |
|---|---:|
| Last successful ping before failure | 960 |
| First successful ping after recovery | 1024 |

```text
Convergence time = 1024 - 960 = 64 s
```

Result:

```text
64 s < 120 s → Pass
```

---

## 7. Ethics & Sustainability Checklist

* [x] **Lab 1:** Verified interference does not disrupt neighbors. The energy scan was passive and did not jam other users.
* [x] **Lab 2:** Used a custom PANID (`0x2022`) and only disconnected the team’s own router during the Tractor Test.
* [x] **Lab 2:** Router/interface can be disabled after testing to save energy and spectrum.
* [ ] **Lab 4:** Data collection minimized (Privacy). Pending.
* [ ] **Lab 5:** System works locally without cloud (Sustainability). Pending.
* [ ] **Lab 6:** Encryption enabled (Privacy). Pending.
* [ ] **Lab 8:** End-of-Life plan considered. Pending.

Recommended shutdown commands after testing:

```bash
ot thread stop
ot ifconfig down
```

---

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational | Lab 1, Lab 2 | RF propagation, channel selection, packet loss, 6LoWPAN, Thread mesh, routing and convergence. |
| Business | Lab 1, Lab 2 | ESP32-C6 viability for low-cost agricultural sensor and actuator networks. |
| Usage | Lab 1, Lab 2 | Recommended spacing, troubleshooting, node placement, and recovery after router failure. |
| Functional | Lab 1, Lab 2 | IPv6 communication, Leader role, Router role, forwarding, next-hop behavior, and route recovery. |
| Trustworthiness | Lab 1, Lab 2 | Reliability, availability, packet loss, resilience, and recovery time. |
| Construction | Lab 1, Lab 2 | ESP32-C6, OpenThread CLI, IEEE 802.15.4, dataset configuration, router and neighbor tables. |

---

### Functional Viewpoint: Leader and Router Roles

| Functional Role | Thread Role | Node Example | Function in the System |
|---|---|---|---|
| Network coordination | Leader | Node A | Maintains Thread network parameters, router information, and mesh partition coordination. |
| Packet forwarding | Router | Node B | Receives packets from A and forwards them toward C, extending coverage beyond direct radio range. |
| Far-field reachability | Router / final node | Node C | Represents a node outside direct range of A, reachable through mesh routing. |
| Communication endpoint | IPv6 interface | All nodes | Sends and receives ICMPv6 packets and later application traffic. |
| Link monitoring | MLE / neighbor table | All routers | Tracks neighbors, link quality, and attachment state. |
| Route selection | Router table | Leader and routers | Determines next-hop paths and updates routes when topology changes. |

---

## 9. Trustworthiness Audit (Lab 6+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability | Partially | Lab 1 measured packet delivery; Lab 2 measured recovery after router failure. | More tests needed with alternative routes and more nodes. |
| Confidentiality | No | Not evaluated yet. | Lab 6 security pending. |
| Integrity | Partially | ICMPv6 packets were received or lost; protocol behavior observed. | No payload integrity/security validation yet. |
| Reliability | Yes | Packet loss and RTT measured at different distances and over multi-hop. | More repetitions needed for statistical confidence. |
| Resilience | Yes | Tractor Test performed; convergence time measured at ~64 s. | True alternate-path healing requires at least one redundant route. |
| Safety | Yes | Energy scan passive; controlled disconnection of own router only. | Must continue avoiding interference with other groups. |
| Compliance | Partially | Used IEEE 802.15.4/OpenThread and ISO/IEC 30141 mapping. | Formal regulatory compliance not evaluated. |

---

## 10. Construction Viewpoint - IoT System Pattern (Lab 8)

| Pattern Element | Category | Your System |
|-----------------|----------|-------------|
| IoT System | Component-stage SoilSense prototype | ESP32-C6 OpenThread wireless sensor/actuator communication subsystem. |
| IoT Components | Hardware nodes | ESP32-C6 boards used as Leader, Router, and final node. |
| Digital Network | Wireless mesh | Thread / 6LoWPAN over IEEE 802.15.4. |
| IoT Devices | Field nodes | A Leader node, one intermediate router, and one final router. |
| Primary Capability (observation) | Sensing foundation | Prepared for soil/environmental sensor data transmission in later labs. |
| Primary Capability (control) | Actuator foundation | Prepared for local actuator control such as irrigation valves. |
| Secondary Capability (processing) | Embedded processing | ESP32-C6 runs OpenThread and processes routing/control messages. |
| Secondary Capability (transferring) | Communication | IPv6 packets transferred through Thread mesh. |
| Secondary Capability (storage) | Configuration storage | Thread dataset and network credentials stored in device memory. |
| Interface (network) | IEEE 802.15.4 / IPv6 | OpenThread CLI, mesh-local IPv6, RLOC, ML-EID. |
| Interface (human UI) | Developer console | ESP-IDF monitor / OpenThread CLI. |
| Interface (application) | Future CoAP/dashboard | Pending for Lab 3 and Lab 7. |
| Supplemental (security) | Thread dataset security | Shared Thread dataset/network credentials; advanced security pending. |
| Supplemental (orchestration) | Thread Leader role | Leader coordinates network and router information. |
| Supplemental (management) | CLI diagnostics | `neighbor table`, `router table`, `ipaddr`, `state`, `dataset active`. |

---

## Final Integrated Analysis

Labs 1 and 2 show the technical evolution from a basic RF link to a resilient mesh communication subsystem.

Lab 1 demonstrated that the ESP32-C6 can communicate reliably using OpenThread over IEEE 802.15.4 at short distances. The best measured condition was up to 10 m with 0% packet loss. However, performance degraded at longer distances, which showed that a simple point-to-point or star-like deployment would not be sufficient for larger agricultural coverage.

Lab 2 addressed this limitation by using Thread mesh routing. The team successfully formed a network with a custom PANID, forced the topology A → B → C, and verified that the final node C was not directly reachable from A. The router B extended coverage by forwarding packets.

The multi-hop latency increased compared with the one-hop baseline, but remained acceptable for many local IoT control scenarios. The Tractor Test demonstrated resilience: after disconnecting B, communication failed as expected, and after reconnecting B, OpenThread restored the route in approximately 64 seconds, satisfying the target of less than 120 seconds.

The main design conclusion is that Thread mesh is a better fit than a LoRaWAN-style star topology for this specific GreenField deployment when local actuator control, IPv6 reachability, and resilience are required. The cost is higher energy consumption in router nodes, so routers should be placed strategically, ideally where power is available or where forwarding is essential.
