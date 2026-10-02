# Laboratorio 1 — Redes virtuales con Linux

## 1. Datos del entorno

- **Estudiante:** Silvio Ariel Sanchez Tenorio
- **Sistema operativo:** Ubuntu 26.04.1 LTS
- **Entorno:** WSL2 sobre Windows
- **Usuario:** `silvio`
- **Kernel:** `6.18.33.2-microsoft-standard-WSL2`
- **Repositorio:** `ETN1011/2026-s2`
- **Directorio de trabajo:** `lab1`
- **Fecha:** 2 de octubre de 2026

---

## 2. Objetivo

Construir y verificar una red virtual aislada utilizando espacios de nombres de red (`network namespaces`) y un par de interfaces virtuales `veth`.

Se configuraron dos espacios de nombres, `hostA` y `hostB`, conectados mediante un enlace virtual `vethA`–`vethB`.

Posteriormente, se configuraron direcciones IPv4, se verificaron las interfaces, rutas y vecinos ARP, y se comprobó la conectividad mediante `ping`.

También se realizó una captura de tráfico utilizando `tcpdump` para observar los paquetes ICMP y ARP intercambiados entre ambos nodos.

Finalmente, se provocó una falla controlada en una de las interfaces para observar su efecto sobre la conectividad, diagnosticar el problema y recuperar el funcionamiento de la red.

---

## 3. Topología

La topología utilizada durante la práctica fue:

```text
                 Linux / WSL2
                      |
             +--------+--------+
             |                 |
           hostA             hostB
        namespace          namespace
             |                 |
          vethA ============= vethB
             |                 |
       10.10.1.1/30       10.10.1.2/30
```

Los namespaces `hostA` y `hostB` proporcionan espacios de red independientes dentro del sistema Linux.

El par de interfaces virtuales `vethA` y `vethB` permite establecer una conexión directa entre ambos namespaces, simulando un enlace de red entre dos dispositivos.

---

## 4. Direccionamiento IPv4

### 4.1. Actividad 2 — Subred `10.10.1.0/30`

Para la red `10.10.1.0/30` se dispone de cuatro direcciones IPv4:

| Dirección | Uso |
|---|---|
| `10.10.1.0` | Dirección de red |
| `10.10.1.1` | Dirección utilizable para host |
| `10.10.1.2` | Dirección utilizable para host |
| `10.10.1.3` | Dirección de broadcast |

La cantidad de direcciones utilizables es **2**, correspondientes a `10.10.1.1` y `10.10.1.2`.

Ambas direcciones pertenecen a la misma subred, por lo que pueden utilizarse para establecer una comunicación directa entre los dos nodos sin necesidad de configurar un gateway.

### 4.2. Asignación de direcciones

| Namespace | Interfaz | Dirección IPv4 |
|---|---|---|
| `hostA` | `vethA` | `10.10.1.1/30` |
| `hostB` | `vethB` | `10.10.1.2/30` |

---

## 5. Construcción de la red virtual

### 5.1. Creación de los namespaces

Se crearon dos espacios de nombres de red:

- `hostA`
- `hostB`

Cada namespace dispone de sus propias interfaces, direcciones IP, tablas de rutas y tablas de vecinos.

Esto permite representar dos nodos de red independientes utilizando una sola instalación de Linux.

### 5.2. Creación del par de interfaces `veth`

Se creó un par de interfaces virtuales conectadas entre sí:

```text
vethA <================> vethB
```

Inicialmente, ambas interfaces pertenecían al namespace principal. Posteriormente, `vethA` fue trasladada a `hostA` y `vethB` a `hostB`.

### 5.3. Configuración de direcciones y activación

Se asignaron las siguientes direcciones IPv4:

```text
hostA / vethA -> 10.10.1.1/30
hostB / vethB -> 10.10.1.2/30
```

También se activaron las interfaces de loopback `lo` y las interfaces virtuales `vethA` y `vethB`.

La verificación mostró:

```text
hostA: vethA ... <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
hostB: vethB ... <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
```

Una interfaz existente es aquella que ha sido creada y se encuentra presente en el namespace.

Una interfaz direccionada tiene una dirección IP asignada.

Una interfaz en estado `UP` ha sido activada administrativamente, mientras que `LOWER_UP` indica que el enlace subyacente se encuentra operativo.

---

## 6. Verificación de la red

### 6.1. Verificación de interfaces

En `hostA` se observó la interfaz `vethA` en estado `UP` y con `LOWER_UP`.

En `hostB` se observó la interfaz `vethB` también en estado `UP` y con `LOWER_UP`.

Además, `vethA` mostró `link-netns hostB` y `vethB` mostró `link-netns hostA`, confirmando que los extremos del enlace pertenecen a namespaces diferentes y están conectados entre sí.

### 6.2. Verificación de direcciones IP

La configuración observada fue:

```text
hostA: vethA -> 10.10.1.1/30
hostB: vethB -> 10.10.1.2/30
```

Ambas interfaces quedaron configuradas dentro de la misma subred `10.10.1.0/30`.

### 6.3. Actividad 3 — Verificación de rutas

En `hostA` se observó la siguiente ruta:

```text
10.10.1.0/30 dev vethA proto kernel scope link src 10.10.1.1
```

En `hostB` se observó:

```text
10.10.1.0/30 dev vethB proto kernel scope link src 10.10.1.2
```

Estas rutas fueron creadas automáticamente por el kernel al asignar las direcciones IP.

Ambos namespaces disponen de una ruta directamente conectada hacia la red `10.10.1.0/30`.

Para comunicarse con el otro nodo, `hostA` utiliza `vethA`, mientras que `hostB` utiliza `vethB`.

No es necesario configurar un gateway porque ambos nodos pertenecen a la misma subred y están conectados directamente mediante el par de interfaces `veth`.

### 6.4. Actividad 4 — Verificación de vecinos ARP

Después de realizar las pruebas de conectividad mediante `ping`, se obtuvieron las siguientes entradas.

En `hostA`:

```text
10.10.1.2 dev vethA lladdr 5a:64:6e:44:a9:e8 STALE
```

En `hostB`:

```text
10.10.1.1 dev vethB lladdr ae:8c:47:46:9e:3b STALE
```

Cada namespace aprendió la dirección MAC correspondiente al otro nodo.

| Namespace | Dirección IP vecina | Dirección MAC |
|---|---|---|
| `hostA` | `10.10.1.2` | `5a:64:6e:44:a9:e8` |
| `hostB` | `10.10.1.1` | `ae:8c:47:46:9e:3b` |

El estado `STALE` indica que la entrada de vecino existe, pero no ha sido validada recientemente.

En una comunicación IPv4 dentro del mismo enlace, el protocolo ARP permite obtener la dirección MAC asociada a una dirección IP para poder entregar las tramas al destino correspondiente.

### 6.5. Verificación de conectividad

Se probó la comunicación desde `hostA` hacia `hostB`.

Resultado:

```text
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 0.096/0.992/3.654/1.536 ms
```

También se realizó la prueba en sentido contrario, desde `hostB` hacia `hostA`.

Resultado:

```text
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 0.115/17.313/68.890/29.777 ms
```

Las pruebas confirmaron que la comunicación funcionó en ambos sentidos, sin pérdida de paquetes.

---

## 7. Actividad 5 — Captura y análisis de tráfico con `tcpdump`

Se utilizó `tcpdump` sobre la interfaz `vethA` de `hostA` mientras se generó tráfico mediante `ping` desde `hostB`.

La captura permitió observar solicitudes ICMP Echo Request y respuestas ICMP Echo Reply.

### 7.1. Tráfico ICMP observado

```text
10.10.1.2 > 10.10.1.1: ICMP echo request
10.10.1.1 > 10.10.1.2: ICMP echo reply
```

Las solicitudes Echo Request corresponden a los paquetes enviados por `ping`, mientras que las respuestas Echo Reply confirman que el destino recibió y respondió a dichas solicitudes.

### 7.2. Tráfico ARP observado

También se observaron mensajes ARP utilizados para resolver las direcciones MAC de los nodos:

```text
ARP, Request who-has 10.10.1.2 tell 10.10.1.1
ARP, Request who-has 10.10.1.1 tell 10.10.1.2
ARP, Reply 10.10.1.1 is-at ae:8c:47:46:9e:3b
ARP, Reply 10.10.1.2 is-at 5a:64:6e:44:a9:e8
```

### 7.3. Resultado de la captura

La captura terminó con los siguientes resultados:

```text
12 packets captured
12 packets received by filter
0 packets dropped by kernel
```

La observación permitió relacionar la resolución de direcciones mediante ARP con el intercambio de paquetes ICMP generado por `ping`.

---

## 8. Actividad 6 — Falla controlada y diagnóstico

### 8.1. Predicción previa

Antes de desactivar `vethB`, se esperaba que la comunicación entre `hostA` y `hostB` dejara de funcionar, debido a que el enlace virtual quedaría interrumpido.

### 8.2. Provocación de la falla

Se ejecutó el siguiente comando:

```bash
sudo ip netns exec hostB ip link set vethB down
```

Después de ejecutar el comando, `vethB` quedó en estado `DOWN` y `vethA` mostró `NO-CARRIER` y `state DOWN`.

Esto indicó que el enlace virtual había perdido su conectividad.

### 8.3. Síntoma observado

Se intentó nuevamente realizar un `ping` desde `hostA` hacia `10.10.1.2`.

El resultado fue:

```text
4 packets transmitted, 0 received, +1 errors, 100% packet loss
From 10.10.1.1 icmp_seq=1 Destination Host Unreachable
```

La comunicación dejó de funcionar debido a la interrupción del enlace.

### 8.4. Diagnóstico

Se comprobó que las direcciones IP seguían configuradas:

```text
hostA -> 10.10.1.1/30
hostB -> 10.10.1.2/30
```

La ruta de `hostA` continuaba existiendo, pero aparecía con el indicador `linkdown`:

```text
10.10.1.0/30 dev vethA proto kernel scope link src 10.10.1.1 linkdown
```

Por tanto, la falla no fue causada por la eliminación de la dirección IP ni de la ruta, sino por la caída de la interfaz `vethB`, que provocó la interrupción del enlace virtual.

### 8.5. Acción correctiva

Para recuperar el enlace se volvió a activar la interfaz `vethB` mediante:

```bash
sudo ip netns exec hostB ip link set vethB up
```

Después de la recuperación, `vethB` volvió a mostrar `UP,LOWER_UP` y `state UP`.

### 8.6. Prueba de recuperación

Se repitió el `ping` desde `hostA` hacia `10.10.1.2`.

El resultado fue:

```text
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 0.098/0.112/0.131/0.013 ms
```

La prueba confirmó que la conectividad se había recuperado después de activar nuevamente la interfaz.

---

## 9. Uso de OpenCode

OpenCode fue utilizado como herramienta de apoyo para comprender los comandos y las salidas obtenidas durante el laboratorio, especialmente en la interpretación de interfaces, rutas, vecinos y diagnóstico de conectividad.

### 9.1. Problema identificado

Se necesitaba interpretar las salidas de los comandos `ip link`, `ip route` e `ip neigh`, además de comprender el funcionamiento de los namespaces y del enlace virtual `veth`.

### 9.2. Consultas realizadas

Se utilizaron consultas orientadas a explicar las salidas de los comandos y comprender el procedimiento de configuración y verificación de la red.

### 9.3. Propuestas obtenidas

Se recibieron explicaciones sobre los siguientes aspectos:

- Diferencia entre una interfaz existente, direccionada y en estado `UP`.
- Significado de las rutas directamente conectadas.
- Relación entre direcciones IP y MAC en la tabla de vecinos.
- Interpretación de los paquetes observados mediante `tcpdump`.
- Diagnóstico de la falla provocada al desactivar `vethB`.

### 9.4. Verificación de las respuestas

Las explicaciones obtenidas se contrastaron ejecutando los comandos en WSL2 y observando los resultados reales de la topología construida.

### 9.5. Explicación técnica

OpenCode se utilizó como herramienta de apoyo para el análisis y comprensión del laboratorio.

La evidencia técnica corresponde a los comandos ejecutados y a los resultados observables obtenidos durante las pruebas.

---

## 10. Conclusiones

1. Se construyó una red virtual aislada utilizando dos network namespaces y un par de interfaces virtuales `veth`.

2. Los namespaces `hostA` y `hostB` fueron configurados en la misma subred `10.10.1.0/30`, utilizando las direcciones `10.10.1.1/30` y `10.10.1.2/30`.

3. Linux creó automáticamente las rutas de red directamente conectada al asignar las direcciones IP, por lo que no fue necesario configurar un gateway para esta topología.

4. Las pruebas de `ping` confirmaron conectividad en ambos sentidos, con un resultado de 0% de pérdida de paquetes.

5. La tabla de vecinos permitió observar la correspondencia entre las direcciones IP y MAC de los extremos del enlace.

6. La herramienta `tcpdump` permitió observar directamente el intercambio de tráfico ICMP y ARP entre ambos namespaces.

7. Al desactivar `vethB`, la conectividad se interrumpió aunque las direcciones IP y la ruta permanecieron configuradas. La ruta pasó a mostrar el indicador `linkdown`.

8. Al volver a activar `vethB`, la conectividad se recuperó, confirmando que la falla había sido provocada por la interrupción del enlace virtual.
