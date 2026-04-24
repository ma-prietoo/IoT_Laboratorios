# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team:** María de los Ángeles Prieto Ortega - Mariana Zuluaga Yepes  

## 1. System Overview

* **System Type:** [x] Component (Lab 1-2) | [ ] System (Lab 3-6) | [ ] Environment (Lab 7-8)

* **Description:**

An initial radiofrequency characterization of the ESP32-C6 was carried out using OpenThread. The objective was to validate whether two ESP32-C6 nodes can communicate reliably using IEEE 802.15.4 radio in the 2.4 GHz band. To achieve this, an energy scan per channel was performed, the channel with the lowest interference was selected, and transmission and reception tests were conducted using `ot ping` at different distances.

The ESP32-C6 was configured and debugged using JTAG, which allowed working with the ESP-IDF development environment and directly observing OpenThread messages during the tests.

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization

#### To Samuel (Architect)

The radio behavior of the ESP32-C6 using OpenThread was empirically validated. The cleanest channel found was channel 23, with an energy level of -108 dBm during the scan. After configuring the network on this channel, IPv6 ping tests were performed at various distances.

The link was completely stable up to 10 m, with 0% packet loss. At 20 m, communication was still achieved, but packet loss increased to 20%. From 25 m and 30 m onwards, packet loss was high, between 31% and 52%, meaning the link can no longer be considered reliable for a sensor network that requires high availability.

Conclusion: the ESP32-C6 works correctly for short-range Thread communication, but based on the obtained data, a practical maximum spacing of around 10 m is recommended to ensure reliability.

#### To Edwin (Ops)

The network operated correctly when both nodes used the same channel. The configured channel was 23. If a node does not join the network or does not respond to ping, the first thing to check is that the `dataset channel` matches on both devices.

It was also observed that as distance increases, `NoAck` errors appear, meaning that a packet was sent but no acknowledgment was received from the other node. This typically occurs due to weak signal, obstacles, antenna orientation, interference, or excessive distance.

Recommendation: keep the nodes at a maximum distance of 10 m for reliable testing in indoor environments or in environments with uncontrolled interference.

## 3. Architecture Decision Records (ADRs)

### ADR-001: Selección del canal 23 para la red Thread

* **Context:**

The Thread network operates over IEEE 802.15.4 in the 2.4 GHz band. This band can also be used by WiFi and other devices, so before creating the network, a channel with low interference must be selected. An energy scan was performed using `ot scan energy 500`.

* **Decision:**

Channel 23 was selected to operate the Thread network.

* **Rationale:**

Channel 23 showed the lowest detected energy level during the scan, at -108 dBm. A more negative value indicates lower interfering energy on that channel. Therefore, channel 23 provided better initial conditions for communication.

* **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 SoC | SCD | Sensing/Controlling device (Section 6.4-6.5 of standard) |
| 802.15.4 Radio | SCD | Communication subsystem |
| Antenna | SCD | Physical interface to PED |
| Air (RF medium) | PED | Physical entity - electromagnetic propagation |

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---|---|---|---|---|
| Communication | Wireless network interface | Radio IEEE 802.15.4 | Active | Lab 1 |
| Communication | IPv6 connectivity | OpenThread IPv6 | Active | Lab 1 |
| Communication | Mesh networking | Thread mesh capability | Latent | Lab 1 |
| Processing | Local embedded processing | CPU ESP32-C6 | Active | Lab 1 |
| Management | Debug and programming | JTAG | Active | Lab 1 |
| Communication | WiFi radio | WiFi del ESP32-C6 | Latent | Lab 1 |
| Communication | Bluetooth LE | BLE del ESP32-C6 | Latent | Lab 1 |
| Security | Secure communication | Seguridad de Thread | Latent | Lab 1 |
| Observation | RF environment scan | Energy scan by channel | Active | Lab 1 |

## 5. First Principles Reflections

### Lab 1

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

## 6. RF Scan Results

### Energy Scan

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

### Scan Results

* **Cleanest channel:** channel 23  
* **Measured energy level:** -108 dBm  
* **Channels with higher relative interference:** channel 13 (-74 dBm), channel 12 (-78 dBm), channel 14 (-81 dBm)

### Applied Configuration

Command used:

```bash
ot dataset channel 23
```

Con esto se configuró el dataset de Thread para operar en el canal 23.

## 7. Range Testing Results

### Base command used
```bash
ot ping fd16:a9da:52bb:2755:0:ff:fe00:1400 64 100 0.2
```

The command sends 100 ICMPv6 packets using OpenThread. It was used to measure packet loss and round-trip time.

### Results Table

| Distance | Packets Sent | Packets Received | Packet Loss | Min RTT | Avg RTT | Max RTT | Status |
|---:|---:|---:|---:|---:|---:|---:|---|
| Initial test | 1 | 1 | 0% | 28 ms | 28 ms | 28 ms | Correct |
| 1 m | 100 | 100 | 0% | 15 ms | 34.920 ms | 53 ms | Reliable |
| 5 m | 100 | 100 | 0% | 15 ms | 38.170 ms | 150 ms | Reliable |
| 10 m | 100 | 100 | 0% | 19 ms | 35.480 ms | 61 ms | Reliable |
| 15 m | 100 | 61 | 39% | 21 ms | 281.524 ms | 1841 ms | Unreliable |
| 20 m | 100 | 80 | 20% | 15 ms | 97.112 ms | 746 ms | Unreliable |
| 25 m | 100 | 69 | 31% | 24 ms | 220.347 ms | 991 ms | Unreliable |
| 30 m | 100 | 48 | 52% | 15 ms | 128.979 ms | 996 ms | Unreliable |
| 35 m | 100 | 48 | 52% | 24 ms | 203.958 ms | 1061 ms | Unreliable |

### Important Observation

Although the packet loss at 20 m was lower than at 15 m, the overall behavior shows link degradation as distance increases. This difference may be due to antenna orientation, reflections, obstacles, multipath effects, or temporary variations in interference. In radiofrequency, degradation is not always perfectly linear.

## 8. Performance Baselines

| Metric | Target | Measured | Status |
|---|---:|---:|---|
| Lab 1: Max Range | > 20 m | 10 m reliable / 20 m with high loss | [ ] Pass / [x] Partial |
| Lab 2: Healing Time | < 120 s | Not measured in this lab | [ ] Pending |
| Lab 3: CoAP Latency | < 200 ms | Not measured in this lab | [ ] Pending |
| Lab 4: Poll Latency | < 5 s | Not measured in this lab | [ ] Pending |
| Lab 6: DTLS Time | < 3 s | Not measured in this lab | [ ] Pending |

### Baseline Interpretation

The laboratory required validating a range greater than 20 m. The nodes achieved communication at 20 m, but not reliably, as there was 20% packet loss. If the criterion is high reliability, the acceptable measured range was 10 m, since up to that distance the packet loss was 0%.

Therefore, the result should be marked as **partial**: communication was achieved beyond 20 m, but not with sufficient quality for a critical sensor network.

## 9. Link Budget Calculation

Since the RSSI of the point-to-point link was not measured at each distance, the link budget calculation is presented approximately using free-space path loss. Operation at 2.4 GHz is assumed.

The free-space path loss is estimated as:

```text
FSPL(dB) = 20 log10(d_km) + 20 log10(f_MHz) + 32.44
```

For 2.4 GHz:

| Distance | Approximate FSPL |
|---:|---:|
| 1 m | 40.05 dB |
| 5 m | 54.03 dB |
| 10 m | 60.05 dB |
| 20 m | 66.07 dB |
| 30 m | 69.59 dB |
| 35 m | 70.94 dB |

### Analysis

From 10 m to 20 m, the theoretical loss increases by approximately 6 dB. In radiofrequency systems, a 6 dB increase can be enough to shift from a stable link to one with retransmissions and packet loss, especially in the presence of obstacles, reflections, or interference.

The ideal calculation only considers free-space conditions. In a real laboratory environment, performance may be worse due to:

* walls,
* human bodies near the antenna,
* module orientation,
* reflections,
* WiFi interference,
* noise from other equipment.

## 10. One-Page Performance Summary

# GreenField SoilSense - Lab 1 Performance Report

**Objective:** Test if ESP32-C6 radio works for farm sensor networks.

## Key Findings

* Maximum reliable range measured: **10 m** with **0% packet loss**.
* Communication was still possible at **20 m**, but with **20% packet loss**.
* Recommended sensor spacing: **8 m to 10 m** in similar conditions.
* Best radio channel: **IEEE 802.15.4 Channel 23**.
* Lowest measured energy: **-108 dBm** on channel 23.
* Risk: channels 12, 13 and 14 showed higher energy levels and should be avoided if possible.

## Impact on Product

* For dense deployments, the ESP32-C6 can be used as a short-range Thread node.
* For larger areas, more nodes or routers would be needed.
* The current measurements do not support assuming 20 m as a reliable spacing under these test conditions.

## Recommendation

**Need more testing.**

The ESP32-C6 works correctly, but the measured reliable distance was 10 m. Before using it in a real agricultural deployment, outdoor line-of-sight tests should be performed. The current indoor or semi-controlled test shows that distance, obstacles and antenna orientation strongly affect reliability.

## 11. Quick Reference for Edwin

# SoilSense Field Deployment - RF Checklist

## Symptom: Device won't join network

1. Check that both devices use the same Thread channel.
2. Verify the dataset configuration with:

```bash
ot dataset channel
```

3. Confirm that the selected channel is 23.
4. Check that the device is correctly flashed and running OpenThread.
5. Check JTAG/serial output for OpenThread errors.

## Symptom: Intermittent packet loss

1. Run an energy scan:

```bash
ot scan energy 500
```

2. Avoid channels with high energy values, such as -74 dBm or -78 dBm.
3. Reduce distance between nodes.
4. Reorient the ESP32-C6 boards.
5. Avoid placing the boards close to metal objects.
6. Avoid testing near WiFi routers or USB 3.0 cables.

## Symptom: `error:NoAck`

This means the transmitted packet was not acknowledged by the receiver.

Possible causes:

* distance too high,
* weak signal,
* interference,
* antenna orientation problem,
* obstacle between nodes,
* temporary channel congestion.

## Maximum Range Guidelines

* Indoor / lab reliable range: **10 m**
* Communication possible but degraded: **15 m to 20 m**
* Not recommended under these test conditions: **25 m to 35 m**

## 12. Ethics & Sustainability Checklist

* [x] **Lab 1:** Verified interference doesn't disrupt neighbors. The energy scan is passive and does not jam other users.
* [ ] **Lab 4:** Data collection minimized (Privacy). Pending.
* [ ] **Lab 5:** System works locally without cloud (Sustainability). Pending.
* [ ] **Lab 6:** Encryption enabled (Privacy). Pending.
* [ ] **Lab 8:** End-of-Life plan considered. Pending.

### Additional note

The radio should be turned off when not being tested. This reduces unnecessary occupation of the shared 2.4 GHz spectrum and saves energy.

## 13. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|---|---|---|
| Foundational | Lab 1 | Radio propagation, channel selection, packet loss, latency. |
| Business | Lab 1 | Whether ESP32-C6 is viable for low-cost sensor nodes. |
| Usage | Lab 1 | Recommended spacing and troubleshooting for deployment. |
| Functional | Lab 1 | IPv6 ping over OpenThread after dataset channel configuration. |
| Trustworthiness | Lab 1 | Reliability decreases strongly after 10 m. |
| Construction | Lab 1 | ESP32-C6 configured with JTAG and OpenThread CLI. |

## 14. Trustworthiness Audit

| Characteristic | Addressed? | How | Gaps |
|---|---|---|---|
| Availability | Partially | Packet loss measured at different distances. | More tests needed outdoors. |
| Confidentiality | No | Not evaluated in Lab 1. | Security/DTLS pending. |
| Integrity | Partially | ICMPv6 packets were either received or lost. | No payload integrity test beyond protocol behavior. |
| Reliability | Yes | Packet delivery rate and RTT measured. | Need RSSI per distance. |
| Resilience | Partially | Errors `NoAck` were observed. | Mesh healing not tested. |
| Safety | Yes | Passive energy scan; no active interference. | None for Lab 1. |
| Compliance | Partially | Used IEEE 802.15.4/OpenThread stack. | Formal regulatory compliance not evaluated. |

## 15. Construction Viewpoint - IoT System Pattern

| Pattern Element | Category | Your System |
|---|---|---|
| IoT System | Component-level validation | Two ESP32-C6 nodes communicating over Thread. |
| IoT Components | Device nodes | ESP32-C6 boards. |
| Digital Network | Wireless network | IEEE 802.15.4 / Thread / IPv6. |
| IoT Devices | Embedded devices | ESP32-C6 with OpenThread firmware. |
| Primary Capability (observation) | RF observation | Energy scan by channel. |
| Primary Capability (control) | Network configuration | Dataset channel configuration. |
| Secondary Capability (processing) | Local processing | OpenThread stack running on ESP32-C6. |
| Secondary Capability (transferring) | Packet communication | ICMPv6 ping packets. |
| Secondary Capability (storage) | Configuration storage | Active dataset configuration. |
| Interface (network) | Wireless | IEEE 802.15.4 radio. |
| Interface (human UI) | CLI | OpenThread command line. |
| Interface (application) | Test command | `ot ping`. |
| Supplemental (security) | Thread security | Present but not deeply evaluated. |
| Supplemental (orchestration) | Network formation | Basic channel selection and communication. |
| Supplemental (management) | Debug | JTAG and terminal logs. |

## 16. Answers to Final Lab Questions

### 1. What is the maximum reliable range between sensor nodes?

The maximum reliable range measured was **10 m**. At 1 m, 5 m and 10 m there was **0% packet loss**. At 15 m, packet loss increased to 39%, so the link was no longer reliable.

### 2. Which channel gives the cleanest spectrum in the 2.4 GHz band?

The cleanest channel was **802.15.4 channel 23**, with an energy level of **-108 dBm**. This was the lowest value in the scan.

### 3. What RSSI threshold predicts 99% packet delivery?

This could not be measured directly because the test collected channel energy and packet loss, but not RSSI of the received packets at each distance.

From the obtained packet-loss results, a practical threshold can only be stated by distance:

* Up to 10 m: 100% packet delivery.
* At 15 m and beyond: packet delivery dropped strongly.

For a future version of the lab, it is necessary to record the neighbor RSSI or link metrics at each distance. Only then can a real RSSI threshold for 99% delivery be calculated.

### 4. Did the ESP32-C6 meet the expected range requirement?

Partially. The target was greater than 20 m, but the reliable measured range was 10 m. At 20 m, communication still occurred, but with 20% packet loss. That is too high for a sensor network that requires stable delivery.

### 5. Why did the packet loss increase after 10 m?

Because the received signal became weaker compared with the noise and interference. The OpenThread logs showed `NoAck`, meaning that some packets were not confirmed by the receiver. This is consistent with a weaker link, more retransmissions and less reliable communication.

### 6. Why is the RTT higher at long distances?

The RTT increases because the system may need retransmissions or additional waiting time at the MAC layer. In IEEE 802.15.4, the radio does not simply transmit whenever it wants; it must share the medium and wait if the channel is busy. If the link is weak, the packet may need to be retried.

### 7. Why can 20 m perform better than 15 m in this experiment?

Radio propagation is affected by more than distance. At 20 m, the antenna orientation or reflections may have temporarily favored the link. At 15 m, there may have been more multipath, a worse antenna angle or an obstacle. This is normal in real RF testing.

### 8. What should be improved in the next test?

* Measure RSSI or link metrics at every distance.
* Repeat each distance test several times.
* Test outdoors with line of sight.
* Record antenna orientation.
* Measure WiFi activity near the test area.
* Compare results using different channels.
* Try different transmit power levels.

## 17. Final Analysis

The laboratory successfully validated the basic RF operation of the ESP32-C6 using OpenThread. The selected channel was technically justified because channel 23 had the lowest measured energy. After configuring this channel, the nodes communicated successfully using IPv6 ping.

The strongest result is that the system was fully reliable up to 10 m, with 0% packet loss and average RTT values around 35 ms to 38 ms. This is acceptable for many sensor applications because soil or environmental sensors usually do not need millisecond-level real-time response.

However, the system degraded sharply after 10 m. At 15 m the packet loss was 39%, and at 20 m it was 20%. Although communication was still possible, the reliability was not good enough for a production sensor network. At 30 m and 35 m the packet loss reached 52%, which means almost half of the messages were not received.

The main engineering conclusion is that the ESP32-C6 radio works, but the tested deployment conditions do not support a 20 m reliable spacing. For a real field deployment, the node spacing should be reduced or the network should include more router nodes. Outdoor line-of-sight testing is required before making a final production decision.
