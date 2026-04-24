# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Laboratorio:** Lab 1 - RF Characterization  
**Plataforma usada:** ESP32-C6  
**Configuración de depuración:** JTAG  
**Tecnología evaluada:** IEEE 802.15.4 / OpenThread / IPv6  
**Equipo:** ____________________

---

## 1. System Overview

* **System Type:** [x] Component (Lab 1-2) | [ ] System (Lab 3-6) | [ ] Environment (Lab 7-8)

* **Description:**

Se realizó la caracterización inicial de radiofrecuencia del ESP32-C6 usando OpenThread. El objetivo fue validar si dos nodos ESP32-C6 pueden comunicarse de forma estable usando radio IEEE 802.15.4 en la banda de 2.4 GHz. Para ello se realizó un escaneo de energía por canal, se seleccionó el canal con menor interferencia y luego se hicieron pruebas de transmisión y recepción mediante `ot ping` a diferentes distancias.

El ESP32-C6 fue configurado y depurado mediante JTAG, lo cual permitió trabajar con el entorno de desarrollo de ESP-IDF y observar directamente los mensajes de OpenThread durante las pruebas.

---

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization

#### To Samuel (Architect)

Se validó empíricamente el comportamiento de radio del ESP32-C6 usando OpenThread. El canal más limpio encontrado fue el canal 23, con un nivel de energía de -108 dBm durante el escaneo. Después de configurar la red en ese canal, se realizaron pruebas de ping IPv6 a varias distancias.

El enlace fue completamente estable hasta 10 m, con 0% de pérdida de paquetes. A 20 m todavía se logró comunicación, pero la pérdida aumentó a 20%. A partir de 25 m y 30 m la pérdida fue alta, entre 31% y 52%, por lo que el enlace ya no puede considerarse confiable para una red de sensores que requiere alta disponibilidad.

**Respuesta corta:** el ESP32-C6 sí funciona correctamente para comunicación Thread de corto alcance, pero con los datos obtenidos se recomienda usar una separación práctica máxima cercana a 10 m para asegurar confiabilidad.

#### To Edwin (Ops)

La red funcionó correctamente cuando ambos nodos usaron el mismo canal. El canal configurado fue el 23. Si un nodo no se une a la red o no responde a ping, lo primero que debe revisarse es que el `dataset channel` coincida en ambos dispositivos.

También se observó que al aumentar la distancia aparecen errores `NoAck`, lo que significa que un paquete fue enviado, pero no se recibió confirmación desde el otro nodo. Esto normalmente ocurre por baja señal, obstáculos, orientación de antena, interferencia o distancia excesiva.

**Recomendación de operación:** mantener los nodos a una distancia máxima de 10 m para pruebas confiables en interiores o en un entorno con interferencias no controladas.

---

## 3. Architecture Decision Records (ADRs)

### ADR-001: Selección del canal 23 para la red Thread

* **Context:**

La red Thread trabaja sobre IEEE 802.15.4 en la banda de 2.4 GHz. Esta banda también puede ser usada por WiFi y otros dispositivos, por lo que antes de crear la red se debe escoger un canal con baja interferencia. Se ejecutó un escaneo de energía con `ot scan energy 500`.

* **Decision:**

Se seleccionó el canal 23 para operar la red Thread.

* **Rationale:**

El canal 23 presentó el menor nivel de energía detectada durante el escaneo, con -108 dBm. Un valor más negativo indica menor energía interferente en ese canal. Por tanto, el canal 23 ofrecía mejores condiciones iniciales para la comunicación.

* **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

---

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|---|---|---|
| ESP32-C6 | SCD - Sensing and Controlling Domain | Actúa como nodo IoT con capacidad de comunicación, procesamiento local y participación en la red Thread. |
| Radio IEEE 802.15.4 del ESP32-C6 | SCD - Sensing and Controlling Domain | Es el módulo que permite transmitir y recibir datos inalámbricos entre nodos. |
| Canal RF de 2.4 GHz | PED - Physical Entity Domain | Es el medio físico por donde se propaga la señal electromagnética. |
| OpenThread | SCD / Functional Viewpoint | Implementa la pila de red necesaria para comunicación IPv6 sobre IEEE 802.15.4. |
| JTAG | Construction / Management Support | Se usó como interfaz de depuración y programación, no como canal de datos de la aplicación. |
| PC de desarrollo | User / Management Support | Permite configurar, monitorear y depurar los nodos durante el laboratorio. |

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

---

## 5. First Principles Reflections

### Lab 1

#### 1. ¿Por qué la señal se debilita cuando aumenta la distancia?

La señal de radio se propaga como una onda electromagnética. A medida que se aleja del transmisor, la energía se reparte en un área mayor. Por eso, la potencia recibida disminuye con la distancia. En espacio libre, esta caída se aproxima con una relación cuadrática: si se duplica la distancia, la potencia recibida baja de forma importante.

En términos prácticos, cuando los nodos se separan más, el receptor recibe una señal más débil. Si la señal queda muy cerca del ruido o de la interferencia del ambiente, aparecen retransmisiones, retardos altos y pérdida de paquetes.

#### 2. ¿Por qué una distancia mayor genera más pérdida de paquetes?

Al aumentar la distancia, baja la relación señal a ruido. Esto significa que al receptor le cuesta más distinguir la información útil del ruido de fondo. Cuando el receptor no puede decodificar correctamente un paquete, no envía confirmación o ACK. En las pruebas esto se observó con mensajes como:

```text
MeshForwarder: Failed to send IPv6 ICMP6 msg ... error:NoAck
```

Ese mensaje indica que el nodo intentó transmitir, pero no recibió confirmación del otro extremo. Por eso aumenta el porcentaje de pérdida de paquetes.

#### 3. ¿Por qué se usa IEEE 802.15.4 en lugar de WiFi para sensores?

IEEE 802.15.4 está diseñado para redes de bajo consumo y baja tasa de datos. Para sensores agrícolas o sensores alimentados por batería, esto es más adecuado que WiFi. WiFi puede entregar más velocidad, pero consume más energía. En una red de sensores, normalmente no se necesita alta velocidad, sino estabilidad, bajo consumo y posibilidad de operar durante largos periodos.

#### 4. ¿Qué significa elegir el canal con menor RSSI?

En el escaneo de energía, el RSSI indica cuánta energía se detecta en cada canal. Esa energía puede venir de otros dispositivos, ruido o interferencias. Un valor más negativo, como -108 dBm, indica un canal más limpio. Por eso el canal 23 fue una buena elección.

---

## 6. RF Scan Results

### Energy Scan

Comando usado:

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

### Resultado del escaneo

* **Canal más limpio:** canal 23
* **Nivel de energía medido:** -108 dBm
* **Canales con más interferencia relativa:** canal 13 (-74 dBm), canal 12 (-78 dBm), canal 14 (-81 dBm)

### Configuración aplicada

Comando usado:

```bash
ot dataset channel 23
```

Con esto se configuró el dataset de Thread para operar en el canal 23.

---

## 7. Range Testing Results

### Comando base usado

```bash
ot ping fd16:a9da:52bb:2755:0:ff:fe00:1400 64 100 0.2
```

El comando envía 100 paquetes ICMPv6 usando OpenThread. Se usó para medir pérdida de paquetes y tiempo de ida y vuelta.

### Tabla de resultados

| Distancia | Paquetes transmitidos | Paquetes recibidos | Packet Loss | RTT mínimo | RTT promedio | RTT máximo | Estado |
|---:|---:|---:|---:|---:|---:|---:|---|
| Prueba inicial | 1 | 1 | 0% | 28 ms | 28 ms | 28 ms | Correcto |
| 1 m | 100 | 100 | 0% | 15 ms | 34.920 ms | 53 ms | Confiable |
| 5 m | 100 | 100 | 0% | 15 ms | 38.170 ms | 150 ms | Confiable |
| 10 m | 100 | 100 | 0% | 19 ms | 35.480 ms | 61 ms | Confiable |
| 15 m | 100 | 61 | 39% | 21 ms | 281.524 ms | 1841 ms | No confiable |
| 20 m | 100 | 80 | 20% | 15 ms | 97.112 ms | 746 ms | No confiable |
| 25 m | 100 | 69 | 31% | 24 ms | 220.347 ms | 991 ms | No confiable |
| 30 m | 100 | 48 | 52% | 15 ms | 128.979 ms | 996 ms | No confiable |
| 35 m | 100 | 48 | 52% | 24 ms | 203.958 ms | 1061 ms | No confiable |

### Observación importante

Aunque a 20 m la pérdida fue menor que a 15 m, el comportamiento general muestra degradación del enlace cuando aumenta la distancia. Esta diferencia puede deberse a orientación de antenas, reflexiones, obstáculos, multipath o variaciones momentáneas de interferencia. En radiofrecuencia no siempre el deterioro es perfectamente lineal.

---

## 8. Performance Baselines

| Metric | Target | Measured | Status |
|---|---:|---:|---|
| Lab 1: Max Range | > 20 m | 10 m confiables / 20 m con pérdida alta | [ ] Pass / [x] Partial |
| Lab 2: Healing Time | < 120 s | No medido en este laboratorio | [ ] Pendiente |
| Lab 3: CoAP Latency | < 200 ms | No medido en este laboratorio | [ ] Pendiente |
| Lab 4: Poll Latency | < 5 s | No medido en este laboratorio | [ ] Pendiente |
| Lab 6: DTLS Time | < 3 s | No medido en este laboratorio | [ ] Pendiente |

### Interpretación del baseline

El laboratorio pedía validar un rango mayor a 20 m. Los nodos lograron comunicación a 20 m, pero no de forma confiable porque hubo 20% de pérdida de paquetes. Si el criterio es confiabilidad alta, el rango aceptable medido fue 10 m, ya que hasta esa distancia la pérdida fue 0%.

Por tanto, el resultado debe marcarse como **parcial**: sí hubo comunicación más allá de 20 m, pero no con calidad suficiente para una red de sensores crítica.

---

## 9. Link Budget Calculation

Como no se midió el RSSI del enlace punto a punto en cada distancia, el cálculo de presupuesto de enlace se presenta de forma aproximada usando pérdida en espacio libre. Se asume operación en 2.4 GHz.

La pérdida en espacio libre se estima como:

```text
FSPL(dB) = 20 log10(d_km) + 20 log10(f_MHz) + 32.44
```

Para 2.4 GHz:

| Distancia | FSPL aproximado |
|---:|---:|
| 1 m | 40.05 dB |
| 5 m | 54.03 dB |
| 10 m | 60.05 dB |
| 20 m | 66.07 dB |
| 30 m | 69.59 dB |
| 35 m | 70.94 dB |

### Análisis

Desde 10 m hasta 20 m la pérdida teórica aumenta cerca de 6 dB. En radiofrecuencia, 6 dB puede ser suficiente para pasar de un enlace estable a un enlace con retransmisiones y pérdidas, especialmente si hay obstáculos, reflexiones o interferencia.

El cálculo ideal solo considera espacio libre. En un laboratorio real, el comportamiento puede ser peor por:

* paredes,
* cuerpos humanos cerca de la antena,
* orientación de los módulos,
* reflexiones,
* interferencia WiFi,
* ruido de otros equipos.

---

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

---

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

---

## 12. Ethics & Sustainability Checklist

* [x] **Lab 1:** Verified interference doesn't disrupt neighbors. The energy scan is passive and does not jam other users.
* [ ] **Lab 4:** Data collection minimized (Privacy). Pending.
* [ ] **Lab 5:** System works locally without cloud (Sustainability). Pending.
* [ ] **Lab 6:** Encryption enabled (Privacy). Pending.
* [ ] **Lab 8:** End-of-Life plan considered. Pending.

### Additional note

The radio should be turned off when not being tested. This reduces unnecessary occupation of the shared 2.4 GHz spectrum and saves energy.

---

## 13. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|---|---|---|
| Foundational | Lab 1 | Radio propagation, channel selection, packet loss, latency. |
| Business | Lab 1 | Whether ESP32-C6 is viable for low-cost sensor nodes. |
| Usage | Lab 1 | Recommended spacing and troubleshooting for deployment. |
| Functional | Lab 1 | IPv6 ping over OpenThread after dataset channel configuration. |
| Trustworthiness | Lab 1 | Reliability decreases strongly after 10 m. |
| Construction | Lab 1 | ESP32-C6 configured with JTAG and OpenThread CLI. |

---

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

---

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

---

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

---

## 17. Final Analysis

The laboratory successfully validated the basic RF operation of the ESP32-C6 using OpenThread. The selected channel was technically justified because channel 23 had the lowest measured energy. After configuring this channel, the nodes communicated successfully using IPv6 ping.

The strongest result is that the system was fully reliable up to 10 m, with 0% packet loss and average RTT values around 35 ms to 38 ms. This is acceptable for many sensor applications because soil or environmental sensors usually do not need millisecond-level real-time response.

However, the system degraded sharply after 10 m. At 15 m the packet loss was 39%, and at 20 m it was 20%. Although communication was still possible, the reliability was not good enough for a production sensor network. At 30 m and 35 m the packet loss reached 52%, which means almost half of the messages were not received.

The main engineering conclusion is that the ESP32-C6 radio works, but the tested deployment conditions do not support a 20 m reliable spacing. For a real field deployment, the node spacing should be reduced or the network should include more router nodes. Outdoor line-of-sight testing is required before making a final production decision.

---

## 18. Final Recommendation

Use the ESP32-C6 as a valid platform for short-range Thread communication, but do not assume 20 m reliable spacing based on this test.

Recommended design value:

```text
Recommended spacing = 8 m to 10 m
```

This gives a safety margin below the maximum reliable distance measured.

