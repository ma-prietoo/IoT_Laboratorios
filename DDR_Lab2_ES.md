# Design & Decision Record (DDR)
**GreenField Technologies | Diseño de Sistemas IoT**

**Equipo:** María de los Ángeles Prieto Ortega - Mariana Zuluaga Yepes  

# Laboratorio 2: Redes Mesh 6LoWPAN y Resiliencia

## 1. Descripción general del sistema

* **Tipo de sistema:** [x] Componente (Lab 1-2) | [ ] Sistema (Lab 3-6) | [ ] Entorno (Lab 7-8)

* **Descripción:**

En este laboratorio se implementó una red mesh Thread utilizando tres nodos ESP32-C6 y OpenThread CLI. El objetivo principal fue validar el funcionamiento de una topología multi-hop, medir la latencia entre nodos y analizar la capacidad de recuperación de la red ante la falla de un router intermedio.

La topología utilizada fue:

```text
A - Leader  →  B - Router Intermedio  →  C - Router Final
```

La red fue configurada utilizando IEEE 802.15.4 en la banda de 2.4 GHz y direccionamiento IPv6 mesh-local.

---

# 2. Registro del laboratorio y resumen para stakeholders

## Para Samuel (Arquitecto)

La red mesh Thread fue creada exitosamente utilizando tres ESP32-C6. La topología fue forzada físicamente para evitar comunicación directa entre A y C. Esto permitió validar el comportamiento multi-hop:

```text
A → B → C
```

La tabla de vecinos (`neighbor table`) mostró que el nodo A solo tenía comunicación directa con B. El nodo C aparecía en la `router table` con:

```text
Link = 0
```

Esto confirmó que C no era vecino directo y que debía alcanzarse a través de B.

Se realizaron pruebas de latencia mediante `ot ping` desde A hasta C. En la prueba prolongada se obtuvo:

```text
500 paquetes transmitidos
458 paquetes recibidos
Pérdida = 8.4%
RTT min/prom/max = 34 / 102.248 / 597 ms
```

Comparando contra el enlace de un solo salto del Lab 1 (aproximadamente 35.480 ms promedio), el RTT promedio aumentó significativamente debido al reenvío mesh.

Conclusión: la red mesh extiende cobertura y resiliencia, pero incrementa latencia y variabilidad temporal.

---

## Para Edwin (Operaciones)

Se realizó la prueba de resiliencia (“Tractor Test”) desconectando físicamente el router B mientras A enviaba pings continuos hacia C.

Cuando B fue desconectado aparecieron mensajes:

```text
error:NoAck
```

Esto confirmó que A perdió el siguiente salto hacia C.

Luego de volver a conectar B, OpenThread restauró automáticamente la ruta y el nodo volvió al estado router:

```text
Role detached -> router
```

La recuperación ocurrió aproximadamente entre:

```text
icmp_seq = 960
icmp_seq = 1024
```

Con un intervalo de 1 segundo:

```text
Tiempo de convergencia ≈ 64 s
```

Esto cumple el requisito de recuperación menor a 120 segundos.

---

# 3. ADR-002: Selección de Thread Mesh en lugar de LoRaWAN Star

## Contexto

GreenField SoilSense requiere conectar sensores y actuadores distribuidos en un entorno agrícola. El sistema necesita:

* baja latencia,
* recuperación automática,
* conectividad local,
* comunicación multi-hop.

LoRaWAN utiliza una topología tipo estrella donde los nodos dependen de gateways centrales. Aunque ofrece gran alcance, introduce mayor dependencia del gateway y no está optimizado para control local de actuadores en tiempo real.

---

## Decisión

Se seleccionó Thread Mesh sobre LoRaWAN Star para esta implementación.

---

## Justificación

Thread permite enrutamiento IPv6 local entre nodos mediante una red mesh distribuida. Esto hace posible:

* extender cobertura mediante routers,
* mantener comunicación local,
* reducir dependencia de un gateway central,
* soportar recuperación automática.

En esta práctica se validó exitosamente la ruta:

```text
A → B → C
```

Incluso cuando A no podía comunicarse directamente con C.

Para actuadores agrícolas, como válvulas de riego, la baja latencia es importante. Thread permite que el tráfico permanezca local dentro de la malla, mientras que LoRaWAN normalmente requiere tráfico ascendente y descendente hacia un gateway.

---

## Consecuencias

### Ventajas

* Mayor cobertura mediante multi-hop.
* Recuperación automática ante fallas.
* IPv6 local para aplicaciones IoT.
* Baja latencia para control de actuadores.

### Desventajas

* Los routers consumen más energía.
* Mayor complejidad que una topología estrella.
* La latencia aumenta con cada salto.

---

## Estado

[x] Aceptado

---

# 4. Formación de la red mesh

## Configuración utilizada

| Parámetro | Valor |
|---|---|
| PANID | `0x2022` |
| Nombre de red | `GreenField-G2` |
| Tecnología | Thread sobre IEEE 802.15.4 |
| Hardware | ESP32-C6 |

---

## Direcciones de los nodos

| Nodo | Rol | Extended Address |
|---|---|---|
| A | Leader | `6afd403ebcb1d8bb` |
| B | Router Intermedio | `3ac785ea0ebdc89c` |
| C | Router Final | `ae482162e2527dc0` |

---

## Resultado

La red mesh se formó correctamente utilizando el PANID personalizado:

```text
0x2022
```

El nodo A se convirtió en Leader y los nodos B y C se integraron exitosamente a la red.

---

# 5. Verificación de topología

## Evidencia de multi-hop

En el nodo A:

```bash
ot neighbor table
```

Mostró únicamente al router B como vecino directo.

Mientras que:

```bash
ot router table
```

Mostró a C con:

```text
Link = 0
Next Hop = 61
```

Esto demuestra que:

* A NO tenía enlace directo con C,
* la comunicación dependía de B,
* la topología mesh fue forzada exitosamente.

---

# 6. Resultados de rendimiento

## Prueba de latencia multi-hop

### Ping corto

```bash
ot ping fd6d:b7b5:ebe1:c026:0:ff:fe00:1400 64 20 1
```

Resultado:

```text
20 paquetes transmitidos
20 paquetes recibidos
Pérdida = 0%
RTT min/prom/max = 57 / 158.600 / 467 ms
```

---

### Ping prolongado

```bash
ot ping fd6d:b7b5:ebe1:c026:0:ff:fe00:1400 64 500 0.5
```

Resultado:

```text
500 paquetes transmitidos
458 paquetes recibidos
Pérdida = 8.4%
RTT min/prom/max = 34 / 102.248 / 597 ms
```

---

## Comparación: 1 salto vs 3 saltos

| Prueba | Topología | RTT Promedio |
|---|---|---|
| Lab 1 | 1 salto | 35.480 ms |
| Lab 2 | A → B → C | 102.248 ms |

---

## Análisis

La comunicación multi-hop aumentó la latencia aproximadamente 3 veces respecto al enlace de un solo salto.

Esto ocurre porque:

* cada router debe recibir y retransmitir paquetes,
* existen tiempos de espera MAC,
* aparecen retransmisiones cuando el enlace no es perfecto.

La malla mejora cobertura y resiliencia, pero incrementa el tiempo de respuesta.

---

# 7. Tractor Test (Convergencia)

## Procedimiento

Desde A se inició:

```bash
ot ping fd6d:b7b5:ebe1:c026:0:ff:fe00:1400 64 200 1
```

Luego se desconectó físicamente B.

---

## Evidencia de falla

Aparecieron mensajes:

```text
error:NoAck
```

Indicando pérdida del siguiente salto.

La tabla de vecinos quedó vacía y la ruta hacia C desapareció.

---

## Reconexión

Después de volver a conectar B:

```text
Role detached -> router
```

La ruta hacia C volvió a aparecer.

---

## Tiempo de convergencia

| Evento | Secuencia |
|---|---|
| Último ping antes de la falla | 960 |
| Primer ping tras recuperación | 1024 |

Con intervalo de 1 segundo:

```text
Tiempo de convergencia ≈ 64 s
```

---

## Resultado

| Métrica | Objetivo | Medido | Estado |
|---|---:|---:|---|
| Tiempo de curación | < 120 s | ~64 s | PASS |

---

# 8. Mapeo ISO/IEC 30141

## Mapeo de dominios

| Componente | Dominio ISO |
|---|---|
| ESP32-C6 Leader | SCD |
| ESP32-C6 Router | SCD |
| Red IEEE 802.15.4 | SCD |
| Dashboard / servicios futuros | ASD |
| Medio RF | PED |

---

## Reflexión funcional

| Rol funcional | Rol Thread | Función |
|---|---|---|
| Coordinación de red | Leader | Mantiene la red mesh |
| Reenvío de paquetes | Router | Extiende cobertura |
| Nodo remoto | Router final | Nodo alcanzado mediante multi-hop |

---

## Interpretación

El Leader coordina la red y mantiene información de routers.

El Router intermedio realiza forwarding de paquetes, permitiendo alcanzar nodos fuera del rango directo.

Esto demuestra el funcionamiento funcional de una arquitectura mesh distribuida.

---

# 9. Reflexión desde primeros principios: consumo energético

Una red mesh mejora cobertura porque los routers retransmiten paquetes.

Sin embargo, esto implica mayor consumo energético porque:

* el radio debe permanecer activo,
* el nodo debe escuchar constantemente,
* deben enviarse ACKs y retransmisiones.

Por tanto:

```text
Más routers → más cobertura → mayor consumo energético
```

Thread sacrifica eficiencia energética absoluta para obtener resiliencia y baja latencia local.

En GreenField esto es aceptable porque el control de actuadores requiere respuesta rápida y recuperación automática.

---

# 10. Línea base de rendimiento

| Métrica | Objetivo | Resultado |
|---|---:|---:|
| Formación mesh con PANID personalizado | Correcta | PASS |
| Latencia multi-hop medida | Sí | PASS |
| Curación < 120 s | ~64 s | PASS |

---

# 11. Ética y sostenibilidad

| Pregunta | Resultado |
|---|---|
| ¿La prueba afectó otros grupos? | No |
| ¿Se usó PANID personalizado? | Sí (`0x2022`) |
| ¿El router fue apagado tras la prueba? | Sí recomendado |

Comandos sugeridos:

```bash
ot thread stop
ot ifconfig down
```

Esto evita uso innecesario de espectro y energía.

---

# 12. Conclusiones finales

El laboratorio validó exitosamente el funcionamiento de una red Thread mesh utilizando ESP32-C6.

Se comprobó:

* formación correcta de la red,
* funcionamiento multi-hop,
* aumento de latencia en rutas mesh,
* recuperación automática ante fallas,
* convergencia menor a 120 segundos.

La topología Thread Mesh resultó más adecuada que LoRaWAN Star para esta implementación debido a:

* menor latencia local,
* capacidad de forwarding,
* resiliencia,
* soporte IPv6 distribuido.

El costo principal es el aumento del consumo energético en routers, pero esto es aceptable para aplicaciones agrícolas con actuadores y necesidad de alta disponibilidad.
