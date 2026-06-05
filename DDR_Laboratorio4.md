# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. María de los Ángeles Prieto Ortega
2. Mariana Zuluaga Yepes

---

# 1. System Overview

* **System Type:** [ ] Component (Lab 1-2) | [x] System (Lab 3-6) | [ ] Environment (Lab 7-8)

* **Description:**

This DDR documents the complete results from:

- Lab 1: RF Characterization
- Lab 2: 6LoWPAN Mesh Networking & Resilience
- Lab 3: Efficient Data Transport with CoAP and CBOR
- Lab 4: Reliable Downlink & Actuator Control with CoAP CON

The project validates IEEE 802.15.4 communication using ESP32-C6 devices and OpenThread, then extends that Thread mesh with compact ASD service contracts for `/env/temp` and `/act/valve` using CoAP/UDP, CBOR, Observe, and Confirmable downlink commands.

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

# LAB 3 — Efficient Data Transport (CoAP & CBOR)

## To Daniela (Customer)

One sensor reading became much smaller on the radio side. The measured CoAP/CBOR exchange used a six-byte payload and an estimated `~36 B` compressed Thread/6LoWPAN on-wire cost, while the earlier HTTP/JSON exchange costs about `~500 B`; for one reading, CoAP is therefore about `14x` smaller than HTTP.

Observe also kept the radio from being awakened by constant application polling. During a measured `342.534 s` window, Node B received one initial Observe response and `29` change notifications, while a `1 Hz` polling client would have sent about `343` GET requests; after subscription, that is about `5.08` threshold notifications per minute instead of `60` polling GETs per minute, reducing repeated application exchanges by about `91.55%` and improving battery endurance for SoilSense nodes.

---

# LAB 4 — Downlink & CON

## To Edwin (Ops)

The sleeping valve was controlled successfully through the `/act/valve` CoAP resource. From Node B, a Confirmable `PUT` with `a1617601` opened the valve, `GET act/valve` returned `a1617601`, a second `PUT` with `a1617600` closed it, and a final `GET` returned `a1617600`. Node V's monitor confirmed `valve -> OPEN (GPIO8 = 1)` and `valve -> CLOSED (GPIO8 = 0)`, so the command path and state readback both worked.

Reliability under loss was validated by disabling Node V with `thread stop`, then sending a Confirmable `PUT` from Node B. Node B logged `coap receive response error 28: ResponseTimeout`, proving that the client did not silently assume success when the valve was unreachable. After Node V rejoined as `child`, the same `PUT` succeeded and Node V logged `valve -> OPEN`, so the recovery path was verified. For field use, a `5 s` SED poll period is recommended: it gives a defensible worst-case command delay of about `5 s`, while the estimated poll-only battery life is about `672 days` on 2xAA cells.

User-domain impact: Edwin gets a visible command result and a bounded delay instead of an unknown actuator state; Daniela gets a battery-aware configuration rather than a permanently awake valve. The older rubric names the actuator resource as `/farm/valve`; this implementation follows the updated Lab 4 contract `/act/valve` with the same PUT/GET actuator semantics.

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

# LAB 3 — ADRs

## ADR-003: Use CoAP/UDP + CBOR for the Sensor Uplink

### Context

SoilSense sensor nodes operate over a constrained Thread mesh. HTTP/JSON from Lab 0 carries TCP setup and text payload overhead, and MQTT/JSON from Lab 0.5 still depends on a TCP connection, broker round trips, and keepalive traffic. Lab 3 requires a documented application resource that supports compact reads and push-on-change behavior for temperature data.

### Decision

Use CoAP over UDP on port `5683` with the ASD resource `/env/temp`, a CBOR payload encoded as `{"t": float16}`, and `GET + Observe` for change-driven updates.

### Rationale

The documented `/env/temp` contract returns `application/cbor` Content-Format `60` and an always-six-byte body with the wire form `A1 61 74 F9 hh ll`. In Task C, the measured payload was `6 B`, the CoAP message was `26 B`, and the compressed Thread/6LoWPAN exchange was estimated at `~36 B`, compared with `~500 B` for HTTP/JSON and `~40 B` for a MQTT/JSON publish on an already-open TCP socket.

MQTT was not selected for the radio side because its apparently competitive bytes per reading hide the cost of a maintained TCP socket, TCP/MQTT startup, keepalives, and broker-mediated communication. CoAP can send a reading as one UDP datagram and Observe avoids repeated polls.

### Status

* **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

# LAB 4 — ADRs

## ADR-004: Poll Period for SED Actuators

### Context

Edwin reported that the irrigation command was delayed because the valve node sleeps most of the time. Daniela also cares about battery life, so the valve cannot simply keep its radio always on. Lab 4 tested the valve as a Sleepy End Device (SED): Node V receives downlink commands only after waking and polling its parent mailbox.

### Decision

Use a `5 s` SED poll period for field actuator nodes.

### Rationale

Task C tested `1 s`, `5 s`, and `30 s` poll periods. The observed command responses were fast in the measured trials because the commands happened near a poll instant, but the operational worst-case latency is bounded by the poll period: about `1 s`, `5 s`, and `30 s`, respectively. A `1 s` poll is more responsive but wakes the radio five times more often than the `5 s` setting. A `30 s` poll maximizes battery life but is too slow for a manual irrigation command.

Using a conservative `10 ms` radio-on poll handshake, `I_RX = 75 mA`, `I_sleep = 0.005 mA`, and `2500 mAh` for 2xAA cells in series, the estimated poll-only battery lives are about `138 days` at `1 s`, `672 days` at `5 s`, and `3472 days` at `30 s`. The `5 s` choice keeps Edwin's worst-case downlink delay short enough for field operation while preserving a large battery-life margin.

### User Domain Mapping

Edwin's user-domain requirement is operational confidence: after tapping "open", the system must either confirm the command path or report failure. A `5 s` poll period keeps the expected manual-control delay short enough for field operation and below the lab's `< 30 s` downlink target. Daniela's user-domain requirement is battery sustainability: the `5 s` option avoids the `1 s` poll's higher wake-up cost while remaining much more usable than `30 s`.

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
| IPv6 Mesh Routing | SCD | Enables packet reachability through the device communication subsystem | Lab 2 |
| `/env/temp` CoAP resource | ASD | Published application-service endpoint for temperature readings | Lab 3 |
| CBOR payload contract | ASD | Defines the application data representation consumed by CoAP clients | Lab 3 |
| CDDL schema for `env-reading` | ASD | Makes the compact binary payload transparent and independently decodable | Lab 3 |
| UDP over Thread | SCD | Carries CoAP datagrams through the constrained device communication subsystem | Lab 3 |
| `/act/valve` CoAP resource | ASD | Published application-service endpoint for valve actuation | Lab 4 |
| CDDL schema for `valve-cmd` | ASD | Documents the compact command body `{v: 0|1}` | Lab 4 |
| CoAP CON retransmission | ASD | Application-service reliability mechanism for confirmable actuator commands | Lab 4 |
| SED parent-poll mechanism | SCD | Communication-subsystem behavior that determines when a sleeping child receives downlink traffic | Lab 4 |

---

## Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Processing | Embedded control | ESP32-C6 SoC | Active | Lab 1 y 2 |
| Communication | RF transmission | 802.15.4 Radio | Active | Lab 1 y 2 |
| Communication | Physical interface | Antenna | Active | Lab 1 y 2 |
| Transfer | RF propagation | Air medium | Latent | Lab 1 y 2 |
| Interface | Application interface | CoAP `/env/temp` | Active | Lab 3 |
| Processing | Data encoding | CBOR half-precision temperature body | Active | Lab 3 |
| Documentation | Data contract | CDDL `env-reading` schema | Active | Lab 3 |
| Transferring | Change-driven delivery | CoAP Observe | Active | Lab 3 |
| Communication | Datagram transport | UDP over Thread | Active | Lab 3 |
| Interface | Actuator control interface | CoAP `/act/valve` | Active | Lab 4 |
| Processing | Command encoding | CBOR valve command `{v: 0 o 1}` | Active | Lab 4 |
| Reliability | Confirmable delivery | CoAP CON request and response timeout | Active | Lab 4 |
| Communication | Sleepy downlink polling | SED parent mailbox and poll period | Active | Lab 4 |

---

## Functional Viewpoint

The Lab 2 Thread mesh belongs to the Sensing and Controlling Domain (SCD), but its behavior is described more precisely by functional responsibilities than by node roles alone.

| Functional Component | Lab 2 Element | Function in the SCD | Evidence |
|---|---|---|---|
| Network Management | Leader role on Node A | Coordinates the Thread network, maintains dataset state, and participates in router allocation. | `ot state`, active dataset, leader logs |
| Routing and Forwarding | Router role on Node B | Forwards IPv6 packets through the mesh and rebuilds reachability when the route changes. | neighbor table, router table, Tractor Test |
| Sensing/Control Endpoint | Final Thread node / child-capable device | Represents the field endpoint that must remain reachable for sensing or control traffic. | ping reachability and recovery test |
| Communication Interface | IEEE 802.15.4 + 6LoWPAN + Thread | Carries compressed IPv6 traffic among SCD devices over the low-power radio link. | OpenThread communication tests |

This Functional Viewpoint separates the ISO/IEC 30141 responsibilities that were previously described only as Leader, Router, and Child roles. The Leader is not a separate domain: it implements network-management behavior inside the SCD. The Router implements routing and forwarding in the same domain, while the endpoint represents the sensing or control participant that depends on the communication subsystem.

---

## Application Service Contract - Lab 3

| Contract Field | Lab 3 Definition |
|---|---|
| Resource | `/env/temp` |
| Transport | CoAP / UDP / port `5683` |
| Methods | `GET`, `GET + Observe` |
| Content-Format | `60` (`application/cbor`) |
| Observe policy | Notify when `|new - last_notified| > 0.5 C`; heartbeat freshness support remains available through Max-Age behavior |
| Success response | `2.05 Content` |
| Path mismatch | `4.04 Not Found` |
| Unsupported method | `4.05 Method Not Allowed` |

CBOR payload byte layout:

```text
A1            map(1)
61 74         text(1) "t"
F9 hh ll      float16, big-endian, IEEE 754 half-precision
```

CDDL schema:

```cddl
env-reading = {
  t: float16   ; air temperature, degrees Celsius
}
```

## Application Service Contract - Lab 4

| Contract Field | Lab 4 Definition |
|---|---|
| Resource | `/act/valve` |
| Transport | CoAP / UDP / port `5683` |
| Methods | `PUT` (CON), `GET` |
| Content-Format | `60` (`application/cbor`) |
| Idempotency | Repeating `PUT {v:1}` leaves the valve open; repeating `PUT {v:0}` leaves it closed |
| Success response | `2.04 Changed` on `PUT`, `2.05 Content` on `GET` |
| Bad payload | `4.00 Bad Request` |
| Unsupported method | `4.05 Method Not Allowed` |

Resource naming note: the current Lab 4 guide specifies `/act/valve`; the older grading text refers to `/farm/valve`. The validated behavior is the required actuator resource behavior: Confirmable `PUT` changes the valve state and `GET` reports it.

CBOR payload byte layout:

```text
A1            map(1)
61 76         text(1) "v"
0X            unsigned 0 or 1
```

CDDL schema:

```cddl
valve-cmd = {
  v: 0 / 1   ; 0 = closed, 1 = open
}
```

## Protocol Stack Mapping - Lab 3

| Layer | Protocol / Artifact | Role |
|---|---|---|
| Application service | CoAP `/env/temp` + Observe | Exposes GET and push-on-change sensor reads |
| Application data | CBOR + CDDL | Encodes and documents the temperature body |
| Transport | UDP | Carries each CoAP exchange without a TCP session |
| Network adaptation | IPv6 + 6LoWPAN IPHC | Preserves IP reachability while compressing headers |
| Mesh network | Thread / OpenThread | Supplies mesh addressing, routing, and node roles |
| Link / radio | IEEE 802.15.4 on ESP32-C6 | Transmits the constrained low-power frames |

CoAP carries application sensor data while Thread MLE handles leader election and routing maintenance, keeping the functional data plane separate from Thread management.

## Lab 3 Technical Checks

### Task A - API Live Check

Node B requested:

```text
ot coap get fdbd:7c61:21f8:1952:6ff6:6b9d:55aa:cf5d env/temp
```

One observed payload was:

```text
a1 61 74 f9 4e 1c
```

The prefix `A1 61 74 F9` matches the published CBOR contract for a one-entry map with key `t` and a float16 value, so the live response conforms to `/env/temp`.

### Task B - Observe vs Polling

| Measurement | Value |
|---|---:|
| Observe start | `08:27:35.661371`, `OBS=78` |
| Observe final response | `08:33:18.195202`, `OBS=107` |
| Observe duration | `342.534 s` |
| Observe responses visible, including subscription response | `30` |
| Threshold notifications after subscription response | `29` |
| Heartbeat notifications visible in measured window | `0` |
| Polls at `1 Hz` over same duration | `343` |
| Threshold notifications / polling GETs | `29 / 343 = 8.45%` |
| Repeated application exchange reduction | `91.55%` |

### Task C - Efficiency Audit

| Stack | One-reading exchange | Bytes on the wire | Packets | vs CoAP |
|---|---|---:|---:|---:|
| **HTTP / JSON** (Lab 0) | TCP SYN/SYN-ACK/ACK + GET + 200 OK + FINx2 | **~500 B** | 6 | 14x |
| **MQTT / JSON** (Lab 0.5) | PUBLISH with an already-open TCP socket | **~40 B** | 1* | 1.1x |
| **CoAP / CBOR** (Lab 3) | One UDP datagram | **~36 B** (measured payload: `6 B`) | 1 | 1x |

| CoAP Cost Element | Bytes |
|---|---:|
| Fixed CoAP header | 4 |
| OpenThread CLI token | 4 |
| Uri-Path `env` | 4 |
| Uri-Path `temp` | 5 |
| Content-Format option | 2 |
| Payload marker | 1 |
| Measured CBOR payload | 6 |
| **Total CoAP message** | **26 B** |
| UDP header | 8 |
| Approximate compressed IPv6 after 6LoWPAN IPHC | ~2 |
| **Estimated compressed on-wire total** | **~36 B** |

* MQTT counts one packet per reading after connection setup, but startup connection traffic and periodic keepalives remain part of the radio energy budget.

## Lab 4 Technical Checks

### Task A - `/act/valve` Round Trip

Node V joined the Thread network as a sleepy child and advertised the mesh-local EID:

```text
fde4:9792:435e:fe56:a750:61f0:66af:45c9
```

Node B sent:

```text
ot coap put fde4:9792:435e:fe56:a750:61f0:66af:45c9 act/valve con a1617601
ot coap get fde4:9792:435e:fe56:a750:61f0:66af:45c9 act/valve
ot coap put fde4:9792:435e:fe56:a750:61f0:66af:45c9 act/valve con a1617600
ot coap get fde4:9792:435e:fe56:a750:61f0:66af:45c9 act/valve
```

Node B received `a1617601` after the open command and `a1617600` after the close command. Node V logged `valve -> OPEN (GPIO8 = 1)`, `GET /act/valve -> 1`, `valve -> CLOSED (GPIO8 = 0)`, and `GET /act/valve -> 0`.

### Task B - CON Reliability Under Loss

Node V was disabled with `ot thread stop`, leaving it in `state disabled`. Node B then sent a Confirmable `PUT` and logged:

```text
coap receive response error 28: ResponseTimeout
```

This confirms that the client detected the missing ACK/response rather than assuming actuation success. After Node V rejoined as `child`, the same command received a CoAP response and Node V logged `valve -> OPEN (GPIO8 = 1)`.

For the Wireshark/packet-capture deliverable, the expected successful recovery exchange is a CoAP Confirmable `PUT` from Node B to Node V on UDP port `5683`, followed by a CoAP ACK carrying the success response (`2.04 Changed`) from Node V back to Node B. The serial evidence captured the same reliability event at the endpoint level: timeout while Node V was disabled and successful response after recovery.

### Task C - SED Poll Period vs Latency

| `poll_period` | Observed command-response latency | Operational worst-case by policy |
|---|---:|---:|
| `1 s` | `< 1 s` | `~1 s` |
| `5 s` | `~1 s` worst observed across repeated trials | `~5 s` |
| `30 s` | `< 1 s` in three observed trials | `~30 s` |

The measured trials landed close to Node V's parent-poll instant, so the observed values are below the operational worst case. The defensible worst-case bound is approximately one poll period plus small mesh processing time.

### SCD Actuation and Feedback Loop

Node V validates the Sensing and Controlling Domain because it turns an application command into a physical actuator state on GPIO8. The feedback loop is: Node B sends Confirmable `PUT`, Node V applies the valve state, CoAP returns a response/ACK, and Node B can issue `GET` to read back the stored state. When Node V is unreachable, `ResponseTimeout` breaks the loop explicitly instead of hiding the failure.

---

# 5. First Principles Reflections

# LAB 1

#### 1. Why does the signal weaken as distance increases?

A radio signal propagates as an electromagnetic wave. As it moves away from the transmitter, its energy spreads over a larger area. Therefore, the received power decreases with distance. In free space, this decay can be approximated by a quadratic relationship: if the distance is doubled, the received power decreases significantly.

In practical terms, as the nodes move farther apart, the receiver gets a weaker signal. If the signal becomes too close to the noise floor or environmental interference, retransmissions, higher delays, and packet loss occur.

#### 2. Why does a greater distance lead to higher packet loss?

As distance increases, the signal-to-noise ratio decreases. This means the receiver has more difficulty distinguishing useful information from background noise. When the receiver cannot correctly decode a packet, it does not send an acknowledgment (ACK). In the tests, this was observed with messages such as:
```text
MeshForwarder: Failed to send IPv6 ICMP6 msg ... error:NoAck
```
This message indicates that the node attempted to transmit, but did not receive confirmation from the other end. As a result, the packet loss percentage increases.

#### 3. Why is IEEE 802.15.4 used instead of WiFi for sensors?

IEEE 802.15.4 is designed for low-power, low-data-rate networks. For agricultural or battery-powered sensors, this is more suitable than WiFi. WiFi can provide higher data rates but consumes more energy. In a sensor network, high speed is usually not required; instead, stability, low power consumption, and the ability to operate for long periods are more important.

#### 4. What does selecting the channel with the lowest RSSI mean?

In an energy scan, RSSI indicates how much energy is detected on each channel. This energy can come from other devices, noise, or interference. A more negative value, such as -108 dBm, indicates a cleaner channel. That is why channel 23 was a good choice.

# LAB 2

#### 1. Why does mesh networking improve resilience?

In a mesh topology, packets can travel through intermediate routers instead of depending on a single gateway. If one node fails, the routing tables can eventually adapt and establish another communication path.

This makes the network more fault tolerant than a star topology.

---

#### 2. Why does multi-hop communication increase latency?

Every hop introduces additional:

- transmission delay,
- processing delay,
- ACK waiting time,
- and MAC-layer contention.

Therefore:

```text
More hops → Higher RTT
```

This behavior was experimentally observed during the laboratory.

---

#### 3. Why do routers consume more energy?

Routers must keep their radios active continuously in order to:

- listen for packets,
- maintain neighbor tables,
- forward traffic,
- and participate in routing.

This consumes more energy than sleepy end devices.

---

# LAB 3

#### 1. Why can CoAP use UDP where HTTP cannot?

CoAP can use UDP where HTTP cannot because its request, response, retransmission, and resource semantics are designed for constrained datagrams instead of depending on a persistent TCP stream.

#### 2. Why does Observe save battery compared with `GET every 60 s`?

Observe saves battery because the sensor sends application data only when a subscribed resource changes or needs a freshness notification, instead of waking for unnecessary polling requests.

#### 3. Why is CBOR not a custom binary format?

CBOR is not a custom binary format because it is a standardized data representation with defined types and a publishable CDDL schema that lets other implementations decode the payload.

---

# LAB 4

#### 1. Why is PUT safe to retransmit but POST is not?

`PUT {v:1}` sets the valve to a target state, so sending the same command twice still leaves the valve open; a `POST` usually asks the server to create or trigger a new action, so duplicates can cause repeated side effects.

#### 2. Why does CON exponential backoff fit a lossy radio better than fixed retry?

Exponential backoff retries quickly at first but then spaces attempts out, which avoids flooding a weak IEEE 802.15.4 link while still giving transient loss a chance to recover.

#### 3. Why is the SED poll period a policy, not a CoAP parameter?

The poll period belongs to the Thread child-parent communication subsystem: it decides when the sleepy radio wakes to ask for mail, while CoAP only defines the request, response, and confirmable-message behavior.

---

# 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20 m | 10 m reliable | Partial |
| Lab 2: Healing Time | < 120 s | 64 s | [x] Pass |
| Lab 2: Multi-hop RTT | Acceptable | ~102 ms avg | [x] Pass |
| Lab 2: Mesh Recovery | Automatic | Successful | [x] Pass |
| Lab 3: CoAP Latency | < 200 ms | ~44 ms avg | [x] Pass |
| Lab 4: Downlink Latency | < 30 s at 5 s poll | ~5 s worst-case by poll policy; ~1 s worst observed | [x] Pass |

---

# 7. Ethics & Sustainability Checklist

- [x] *Lab 1:* Verified that interference did not disrupt other groups.
- [x] *Lab 1:* Passive energy scans were used.
- [x] *Lab 2:* Resilience tests did not intentionally interfere with other groups.
- [x] *Lab 2:* Router was disabled after testing to reduce spectrum occupation and save energy.
- [x] *Lab 3:* Observe reduced traffic from `343` hypothetical `1 Hz` polling GETs to `29` threshold notifications after the initial subscription response, a reduction of about `91.55%`.
- [x] *Lab 3:* The `/env/temp` resource contract, CBOR byte layout, Content-Format, and CDDL schema are documented so another client can decode the reading.
- [x] *Lab 4:* The valve command is idempotent and uses Confirmable CoAP so lost downlink delivery is detected by timeout.
- [x] *Lab 4:* A default-closed actuator failsafe is required: if the valve receives `OPEN` but the network dies before a `CLOSE`, firmware should close automatically after a maximum irrigation window using a local watchdog timer.
- [x] *Lab 4:* A `1 s` poll period is only defensible for urgent operations because it reduces poll-only battery life by about `5x` compared with `5 s`; `5 s` is the recommended field trade-off.
- [x] *Lab 4:* Privacy was preserved during downlink tests: no additional uplink sensing data was collected, and the only observed values were valve command state, CoAP response status, Thread role, and timing/log metadata needed for reliability analysis.

## Lab 3 Energy Calculation

The measured post-subscription Observe rate is:

```text
Observe rate = 29 threshold notifications / 342.534 s
             = 0.08466 notifications/s
             = 5.080 notifications/min
```

The local course `references.md` reviewed for this laboratory did not state a radio-RX value, so this estimate uses the ESP32-C6 datasheet value for IEEE 802.15.4 active receive current:

```text
I_RX = 74 mA
```

If CoAP saves `50 ms = 0.05 s` of radio time for each delivered reading:

```text
readings/year = 0.08466 readings/s x 31,536,000 s/year
              ~= 2,670,000 readings/year

saved time/year = 2,670,000 x 0.05 s
                ~= 133,500 s
                ~= 37.08 h

saved charge = 74 mA x 37.08 h
             ~= 2,744 mAh/year
```

At the measured Observe notification rate, saving `50 ms` of radio time per transmission preserves about `37.08 h` of RX radio-on time or about `2.74 Ah` of battery charge over one year. The exact added operating lifetime depends on the selected battery capacity and the rest of the node power budget.

## Lab 4 Energy Calculation

Assumptions for the SED poll cost:

```text
I_RX = 75 mA
I_sleep = 0.005 mA
t_radio_on_per_poll = 10 ms = 0.01 s
battery = 2500 mAh for 2xAA in series
```

Duty cycle and battery life:

```text
duty_cycle = t_radio_on_per_poll / poll_period
I_avg = (I_RX x duty_cycle) + (I_sleep x (1 - duty_cycle))
battery_life_days = 2500 mAh / I_avg / 24
```

| `poll_period` | Worst-case latency used for design | Duty cycle | Estimated average current | Estimated battery life |
|---|---:|---:|---:|---:|
| `1 s` | `~1 s` | `1.000%` | `0.755 mA` | `~138 days` |
| `5 s` | `~5 s` | `0.200%` | `0.155 mA` | `~672 days` |
| `30 s` | `~30 s` | `0.033%` | `0.030 mA` | `~3472 days` |

At the selected `5 s` poll period, the bare SED poll is the dominant radio-energy term compared with Lab 3 Observe traffic. Lab 3 Observe sends application data only on threshold changes, while the SED poll occurs continuously even when no actuator command is waiting.

---

# 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational | Labs 1, 2, 3, 4 | RF propagation and packet loss; Thread mesh routing and convergence; why CoAP/UDP, CBOR, Observe, CON, and 6LoWPAN fit constrained networks. |
| Business | Labs 1, 2, 3, 4 | ESP32-C6 sensor-node feasibility; reliable actuator communication; Daniela's battery-life concern and lower ingress bandwidth. |
| Usage | Labs 1, 2, 3, 4 | Recommended deployment spacing; recovery after router failure; `GET` for reads, Observe for change-driven updates, and `PUT` for idempotent valve commands. |
| Functional | Labs 1, 2, 3, 4 | IPv6 communication validation; SCD network management, routing/forwarding, endpoint reachability, and ASD service contracts separated from Thread MLE management. |
| Trustworthiness | Labs 1, 2, 3, 4 | Reliability degradation with distance; self-healing resilience; transparent compact data contracts; CON timeout when actuation is not confirmed. |
| Construction | Labs 1, 2, 3, 4 | OpenThread + JTAG setup; forced multi-hop topology; ESP32-C6 OpenThread CLI plus libcoap servers on `/env/temp` and `/act/valve`. |

---

# 9. Construction Viewpoint - IoT System Pattern

| Pattern Element | Category | Your System |
|---|---|---|
| IoT Components | Device | ESP32-C6 |
| Digital Network | Network | IEEE 802.15.4 |
| Communication | Application/Communication | IPv6 ping |
| IoT System | System | Distributed IoT |
| Digital Network | Network | Thread/OpenThread Mesh |
| IoT Devices | Device | Leader + Routers |
| Communication | Application/Communication | IPv6 Multi-hop Routing |
| Application Interface | Interface | CoAP `/env/temp` with `GET` and Observe |
| Application Data | Processing | CBOR float16 temperature body documented by CDDL |
| Application Interface | Interface | CoAP `/act/valve` with Confirmable `PUT` and `GET` |
| Application Data | Processing | CBOR valve command `{v: 0|1}` documented by CDDL |

---

# 10. Final Analysis

## Lab 1 - RF Characterization

The RF characterization validated the operation of the ESP32-C6 radio with OpenThread before building larger IoT behavior on top of it. Channel 23 was selected because it presented the lowest measured interference during the scan, at `-108 dBm`.

Under the tested conditions, the ESP32-C6 communication link was reliable up to approximately 10 m. As distance increased, packet loss and RTT increased because the received signal became weaker and retransmissions became more likely. This laboratory established the practical importance of channel selection and radio characterization before deploying a sensor network.

## Lab 2 - 6LoWPAN Mesh Networking and Resilience

Lab 2 extended the radio link into a resilient Thread mesh using ESP32-C6 devices and OpenThread. A forced multi-hop topology was validated through neighbor and router tables so that communication depended on an intermediate router instead of a direct path.

The Tractor Test demonstrated self-healing behavior. When the intermediate router failed, communication was interrupted; after the router rejoined, OpenThread rebuilt the routing path and restored connectivity automatically. The measured convergence time was approximately:

```text
64 seconds
```

This result satisfies the laboratory recovery requirement and supports the decision to use Thread mesh behavior for local agricultural device communication.

## Lab 3 - Efficient Data Transport with CoAP and CBOR

Lab 3 moved the temperature uplink from heavier application exchanges toward a constrained-service interface. The `/env/temp` resource was documented as an ASD contract and validated live through `coap get` responses whose CBOR body followed the six-byte `A1 61 74 F9 hh ll` layout.

CoAP/CBOR reduced one estimated compressed Thread/6LoWPAN reading to `~36 B`, about `14x` smaller than the earlier `~500 B` HTTP/JSON exchange. Observe also reduced repeated application traffic in the measured window: after the initial subscription response, `29` threshold notifications replaced `343` hypothetical `1 Hz` polling GETs, a reduction of about `91.55%`.

The measured CoAP round-trip latency averaged about `44 ms`, below the `< 200 ms` target for one hop. These results support ADR-003: CoAP/UDP + CBOR is a better radio-side sensor uplink than HTTP/JSON and avoids the TCP socket, keepalive, and broker costs that make MQTT less attractive on the constrained radio path.

## Lab 4 - Reliable Downlink and Actuator Control

Lab 4 added the downlink side of the SoilSense control loop. The valve node was configured as a Sleepy End Device and exposed the `/act/valve` resource. A Confirmable `PUT` with `a1617601` opened the valve and `a1617600` closed it; `GET act/valve` returned the matching state, proving that the application contract and actuator state machine worked end to end.

The loss test showed why CON matters for actuation. With Node V disabled, Node B logged `ResponseTimeout` instead of silently treating the command as successful. After Node V rejoined as `child`, the same command succeeded and the valve opened. This supports ADR-004: a `5 s` poll period gives a practical command-latency bound while keeping the SED poll cost near `0.155 mA` average current, or about `672 days` of poll-only battery life on 2xAA cells.
