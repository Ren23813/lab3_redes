# Laboratorio 3 – Algoritmos de Enrutamiento

**Planificación IP, VLSM, VLANs y enrutamiento OSPF / EIGRP con redistribución** en Cisco Packet Tracer.

- **Curso:** Redes, sección 10, Universidad del Valle de Guatemala
- **Docente:** Kevin Velásquez
- **Integrantes:**
  - Melisa Mendizabal, 23778
  - Renato Rojas, 23813

---

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| [`Lab3.pkt`](./Lab3.pkt) | Topología de Packet Tracer con los 5 routers, switches, VLANs y protocolos configurados |
| [`PDF1-ConceptosProcedimientosAMano.pdf`](./PDF1-ConceptosProcedimientosAMano.pdf) | Cálculos de VLSM a mano, tabla de direccionamiento, sumarización y conceptos investigados |
| [`PDF2-PacketTracer.pdf`](./PDF2-PacketTracer.pdf) | Evidencia de Packet Tracer: tablas de enrutamiento, balanceo OSPF/EIGRP, redistribución, ping/tracert, vecindades, interfaces pasivas y trunks |

---

## Contexto

Una empresa tiene dos sedes. Una usa OSPF y la otra EIGRP, y ambas se interconectan a través de un router central (R-CENTRAL). Ese router corre los dos protocolos y hace redistribución mutua entre ellos.

## Topología

```
      PC-O1-V10   PC-O1-V30                       PC-E1-V10   PC-E1-V20
             \     /                                     \     /
            SW-OSPF1                                    SW-EIGRP1
               |                                            |
           R-OSPF-1 ──────┐                    ┌────── R-EIGRP-1
               │           \                  /            │
               │            ──  R-CENTRAL  ──              │
               │           /                  \            │
           R-OSPF-2 ──────┘                    └────── R-EIGRP-2
               |                                            |
            SW-OSPF2                                    SW-EIGRP2
               |                                            |
             PC-O2                                        PC-E2
```

| Router | Dominio | VLANs | Función |
|---|---|---|---|
| R-OSPF-1 | OSPF área 0 | 10 (datos), 30 (administración) | Sitio de acceso |
| R-OSPF-2 | OSPF área 0 | LAN sin VLANs | Sitio de acceso |
| R-EIGRP-1 | EIGRP AS 100 | 10 (datos), 20 (voz) | Sitio de acceso |
| R-EIGRP-2 | EIGRP AS 100 | LAN sin VLANs | Sitio de acceso |
| R-CENTRAL | OSPF área 0 + EIGRP AS 100 | No | Frontera y redistribución mutua |

El enrutamiento entre VLANs se hace con router-on-a-stick. Cada VLAN usa una subinterfaz `GigabitEthernet0/0.<VLAN>` con encapsulación 802.1Q, y el enlace hacia el switch está en modo trunk.

---

## Direccionamiento IP (VLSM)

| Uso | Bloque base |
|---|---|
| VLANs, lado OSPF | `192.168.0.0/20` |
| VLANs, lado EIGRP | `172.16.0.0/20` |
| Enlaces router-router | `10.0.0.0/24` |

Los segmentos se ordenaron de mayor a menor cantidad de hosts. A cada uno se le sumó 1 dirección para el gateway y luego se calculó el número de bits de host con `⌈log₂(n)⌉`. Los cálculos completos están en el PDF1.

### Lado OSPF (`192.168.0.0/20`)

| Segmento | Hosts | Red | Máscara | Rango utilizable | Broadcast |
|---|---|---|---|---|---|
| R-OSPF-2, LAN datos | 200 | 192.168.0.0/24 | 255.255.255.0 | 192.168.0.1 – 192.168.0.254 | 192.168.0.255 |
| R-OSPF-1, VLAN 10 datos | 50 | 192.168.1.0/26 | 255.255.255.192 | 192.168.1.1 – 192.168.1.62 | 192.168.1.63 |
| R-OSPF-1, VLAN 30 administración | 10 | 192.168.1.64/28 | 255.255.255.240 | 192.168.1.65 – 192.168.1.78 | 192.168.1.79 |

### Lado EIGRP (`172.16.0.0/20`)

| Segmento | Hosts | Red | Máscara | Rango utilizable | Broadcast |
|---|---|---|---|---|---|
| R-EIGRP-2, LAN datos | 100 | 172.16.0.0/25 | 255.255.255.128 | 172.16.0.1 – 172.16.0.126 | 172.16.0.127 |
| R-EIGRP-1, VLAN 10 datos | 60 | 172.16.0.128/26 | 255.255.255.192 | 172.16.0.129 – 172.16.0.190 | 172.16.0.191 |
| R-EIGRP-1, VLAN 20 voz | 30 | 172.16.0.192/26 | 255.255.255.192 | 172.16.0.193 – 172.16.0.254 | 172.16.0.255 |

> La VLAN de voz necesita 31 direcciones (30 hosts + gateway). Un /27 solo da 30 direcciones utilizables, así que no alcanza y se usó un /26.

### Enlaces punto a punto (`10.0.0.0/24`, máscara /30)

| Enlace | Conecta | Red | Extremos | Broadcast |
|---|---|---|---|---|
| L1 | R-OSPF-1 ↔ R-OSPF-2 | 10.0.0.0/30 | .1 (O1) – .2 (O2) | 10.0.0.3 |
| L2 | R-OSPF-1 ↔ R-CENTRAL | 10.0.0.4/30 | .5 (O1) – .6 (C) | 10.0.0.7 |
| L3 | R-OSPF-2 ↔ R-CENTRAL | 10.0.0.8/30 | .9 (O2) – .10 (C) | 10.0.0.11 |
| L4 | R-EIGRP-1 ↔ R-EIGRP-2 | 10.0.0.16/30 | .17 (E1) – .18 (E2) | 10.0.0.19 |
| L5 | R-EIGRP-1 ↔ R-CENTRAL | 10.0.0.20/30 | .21 (E1) – .22 (C) | 10.0.0.23 |
| L6 | R-EIGRP-2 ↔ R-CENTRAL | 10.0.0.24/30 | .25 (E2) – .26 (C) | 10.0.0.27 |

Los enlaces EIGRP empiezan en `10.0.0.16` y no en `.12` para que queden alineados y se puedan resumir como `10.0.0.16/28`.

---

## Enrutamiento

### OSPF (proceso 1, área 0)
- Lo corren R-OSPF-1 (router-id `1.1.1.1`), R-OSPF-2 (`2.2.2.2`) y R-CENTRAL (`5.5.5.5`).
- Se anuncian todas las VLANs y los enlaces L1, L2 y L3.
- Las interfaces hacia los equipos finales están en `passive-interface`.

### EIGRP (AS 100)
- Lo corren R-EIGRP-1, R-EIGRP-2 y R-CENTRAL.
- Se anuncian todas las VLANs y los enlaces L4, L5 y L6.
- Las interfaces hacia las VLANs están en `passive-interface`.

### Redistribución mutua (R-CENTRAL)

```
router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500
 network 10.0.0.20 0.0.0.3
 network 10.0.0.24 0.0.0.3
router ospf 1
 router-id 5.5.5.5
 log-adjacency-changes
 redistribute static metric-type 1 subnets
 network 10.0.0.4 0.0.0.3 area 0
 network 10.0.0.8 0.0.0.3 area 0
```

- **Métrica semilla hacia EIGRP:** `10000 100 255 1 1500`. Equivale a un ancho de banda de 10 Mbps, un delay de 1000 µs, confiabilidad máxima, carga mínima y MTU de 1500. Sin esta métrica, EIGRP trata la ruta externa como inalcanzable (métrica infinita).
- **Hacia OSPF:** se usa métrica semilla 20 con tipo **E1** (`metric-type 1`). E2 mantiene fija la métrica externa. E1 le suma el costo interno hasta el ASBR, y eso refleja mejor el costo real cuando hay varios caminos hacia R-CENTRAL con costos distintos. Por ejemplo, `O E1 172.16.0.0/24 [110/84]` = 20 de métrica externa + 64 de costo interno.

---

## Requisitos avanzados

### Balanceo de costo igual (ECMP), lado OSPF
Entre R-OSPF-1 y R-OSPF-2 hay dos caminos: el enlace directo L1 y el camino que pasa por R-CENTRAL (L2 + L3). Los costos se ajustaron para que ambos sumen **128**, así OSPF instala las dos rutas y reparte el tráfico entre ellas. La evidencia está en el PDF2 (`show ip ospf interface` y `show ip route 192.168.0.0`).

### Balanceo desigual 75/25 (variance), lado EIGRP
Anchos de banda configurados (`bandwidth`):

| Enlace | Bandwidth |
|---|---|
| R-EIGRP-1 ↔ R-EIGRP-2 (directo) | 5000 kbps |
| R-EIGRP-x ↔ R-CENTRAL | 1250 kbps |
| Seriales de R-CENTRAL | 10000 kbps |

Desde R-EIGRP-1:
- Ruta directa (successor): FD = 1,026,560
- Ruta vía R-CENTRAL (feasible successor): FD = 3,074,560, AD = 770,560

La AD de la ruta por R-CENTRAL (770,560) es menor que la FD del successor (1,026,560), así que cumple la condición de factibilidad. Su métrica es casi el triple de la principal, por lo que con `variance 3` se instalan las dos rutas y el tráfico se reparte aproximadamente 75 % por el enlace directo y 25 % por R-CENTRAL. En R-EIGRP-2 se aplicó lo mismo de forma simétrica.

### Sumarización de rutas

| Dirección | Rutas originales | Resumen |
|---|---|---|
| OSPF → EIGRP | 192.168.0.0/24, 192.168.1.0/26, 192.168.1.64/28 | **192.168.0.0/23** |
| EIGRP → OSPF | 172.16.0.0/25, 172.16.0.128/26, 172.16.0.192/26 | **172.16.0.0/24** |
| EIGRP → OSPF (enlaces) | 10.0.0.16/30, 10.0.0.20/30, 10.0.0.24/30 | **10.0.0.16/28** |

Para obtener cada resumen se compararon en binario las subredes y se contaron los bits en común. Por ejemplo, en el lado OSPF hay 23 bits comunes, lo que da 192.168.0.0/23.

Packet Tracer no soporta `summary-address` dentro de `router ospf`. Por eso, el resumen hacia OSPF se hizo con rutas estáticas hacia Null0, que luego se redistribuyen (`redistribute static metric-type 1 subnets`). La ruta a Null0 también evita bucles para direcciones que caen dentro del resumen pero no pertenecen a ninguna subred real.

---

## Verificación

Se hizo ping y tracert de extremo a extremo desde PC-O1-V10 (VLAN 10 de R-OSPF-1) hacia 172.16.0.10 (PC-E2, LAN de R-EIGRP-2):

```
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

El PDF2 incluye además:
- `show ip route` de los 5 routers
- `show ip ospf interface` y `show ip route 192.168.0.0` (ECMP)
- `show ip eigrp topology` / `all-links` (successor y feasible successor)
- `show ip protocols` (variance, redistribución, interfaces pasivas)
- `show running-config | section router` en R-CENTRAL
- `show ip ospf neighbor` y vecindades EIGRP
- `show ip interface brief` (subinterfaces de router-on-a-stick)
- `show vlan brief` y `show interfaces trunk` en los switches

---

## Cómo abrir el proyecto

1. Instalar Cisco Packet Tracer.
2. Abrir `Lab3.pkt`.
3. Desde cualquier PC, abrir *Desktop → Command Prompt* y probar, por ejemplo:
   ```
   ping 172.16.0.10
   tracert 172.16.0.10
   ```
