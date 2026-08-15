# Chapin Red — Manual Técnico

Proyecto 1 de Redes de Computadoras 2. Red corporativa multi-edificio para la
organización **Chapin Red**, implementada en Cisco Packet Tracer.

- Carné: **202300335** (X = 35)
- Redes base: `192.188.35.0/24` (VLANs de usuario) y `10.4.35.0/24` (enlaces de enrutamiento)
- Protocolo de enrutamiento: **EIGRP** en todos los edificios y en el MAN (carné termina en dígito impar → EIGRP)
- Archivo de topología: `ChapinRed.pkt` (pendiente de subir una vez armada la topología)

## 1. Topología general

Cuatro edificios interconectados en una MAN mediante switches multicapa Cisco 3650
(fibra óptica / módulos Gigabit), en malla redundante para tolerancia a fallos:

| Edificio | Rol | Switch MAN (3650) |
|---|---|---|
| Principal | Aloja los 2 servidores DHCP (Izquierdo y Derecho) | MSW-NORTE |
| Izquierdo | Arquitectura jerárquica de 3 capas (Core/Distribución/Acceso), VLANs Naranja y Verde, LACP | MSW-IZQ-GW |
| Derecho | VLANs Naranja y Verde, PAgP | MSW-DER-GW |
| Administración | VLAN ADMIN | MSW-ADMIN |

Enlaces MAN (EIGRP, red `10.4.35.0/24`, /30 por enlace): malla redundante entre los
4 switches 3650 (MSW-NORTE, MSW-IZQ-GW, MSW-DER-GW, MSW-ADMIN), simulando fibra óptica
con módulos GigabitEthernet.

## 2. VLANs

Convención de nombres: `VLAN_[Color]_Edificio[IZQ/DER]_202300335`

| VLAN ID | Nombre | Edificio | Propósito |
|---|---|---|---|
| 10 | VLAN_Naranja_EdificioIZQ_202300335 | Izquierdo | Departamento de Proyectos |
| 20 | VLAN_Verde_EdificioIZQ_202300335 | Izquierdo | Departamento de Coordinación |
| 30 | VLAN_Naranja_EdificioDER_202300335 | Derecho | Departamento de Proyectos |
| 40 | VLAN_Verde_EdificioDER_202300335 | Derecho | Departamento de Coordinación |
| 99 | VLAN_ADMIN_202300335 | Administración | Departamento de Administración |

## 3. VTP

- Dominio VTP: `ChapinRed`
- Contraseña VTP: `Chapin2026*` *(cambiar aquí y documentar si se usa otra)*
- VTP Server: switch multicapa del edificio Principal / Core del edificio Izquierdo *(definir al armar la topología)*
- Resto de switches: VTP Client

## 4. Direccionamiento IP

### 4.1 VLSM — `192.188.35.0/24` (VLANs de usuario)

Se reservan bloques con margen de crecimiento por VLAN (30 hosts para Naranja/Verde,
14 hosts para ADMIN):

| VLAN | Subred | Máscara | Rango usable | Gateway | Broadcast |
|---|---|---|---|---|---|
| 10 – Naranja IZQ | 192.188.35.0 | /27 (255.255.255.224) | .2 – .30 | 192.188.35.1 | .31 |
| 20 – Verde IZQ | 192.188.35.32 | /27 (255.255.255.224) | .34 – .62 | 192.188.35.33 | .63 |
| 30 – Naranja DER | 192.188.35.64 | /27 (255.255.255.224) | .66 – .94 | 192.188.35.65 | .95 |
| 40 – Verde DER | 192.188.35.96 | /27 (255.255.255.224) | .98 – .126 | 192.188.35.97 | .127 |
| 99 – ADMIN | 192.188.35.128 | /28 (255.255.255.240) | .130 – .142 | 192.188.35.129 | .143 |

Rango `192.188.35.144 – .255` queda reservado sin usar.

### 4.2 FLSM — `10.4.35.0/24` (enlaces punto a punto, /30)

Se reserva un pool de bloques `/30` (4 IPs, 2 usables c/u) y se asignan conforme se
arma cada enlace real en la topología. Se documentará aquí cada asignación:

| Bloque | Enlace | IP extremo A | IP extremo B |
|---|---|---|---|
| 10.4.35.0/30 | *(pendiente)* | .1 | .2 |
| 10.4.35.4/30 | *(pendiente)* | .5 | .6 |
| 10.4.35.8/30 | *(pendiente)* | .9 | .10 |
| 10.4.35.12/30 | *(pendiente)* | .13 | .14 |
| 10.4.35.16/30 | *(pendiente)* | .17 | .18 |
| 10.4.35.20/30 | *(pendiente)* | .21 | .22 |
| ... | | | |

## 5. Agregación de enlaces

- Edificio Izquierdo: 5 enlaces **LACP** (IEEE 802.3ad, modo active/passive) en capa 2.
- Edificio Derecho: 3 enlaces **PAgP** (modo desirable/auto), capa 2 (y capa 3 si la topología lo requiere).
- Verificación de tolerancia a fallos: `shutdown` de un puerto miembro sin pérdida de paquetes.

## 6. DHCP

| Servidor | Ubicación | VLANs que atiende |
|---|---|---|
| DHCP-IZQ | Edificio Principal (vía MAN) | VLAN 10, VLAN 20 |
| DHCP-DER | Edificio Principal (vía MAN) | VLAN 30, VLAN 40 |

Se documentarán los pools exactos (red, máscara, default-router, dns, exclusiones) en
la Fase 4.

DHCP Relay: `ip helper-address` configurado en cada SVI/subinterfaz apuntando al
servidor DHCP correspondiente.

## 7. ACLs (políticas de seguridad)

| VLAN origen | Puede hablar con | Bloqueado hacia |
|---|---|---|
| Naranja (10/30) | Naranja de ambos edificios | Verde, ADMIN |
| Verde (20/40) | Verde de ambos edificios | Naranja, ADMIN |
| ADMIN (99) | Todas las VLANs (solo tráfico iniciado desde ADMIN) | — (nadie puede iniciar tráfico hacia ADMIN) |

Detalle de las ACLs (números, líneas, interfaz de aplicación) se documentará en la Fase 5.

## 8. Comandos principales utilizados

*(se irá completando por fase: VTP, VLANs, trunking, LACP/PAgP, EIGRP, DHCP, ACLs, con
capturas de verificación en `capturas/`)*

## 9. Roadmap de fases

- [x] Fase 0 — Estructura del repo y diseño de direccionamiento
- [ ] Fase 1 — Topología, VLANs y VTP
- [ ] Fase 2 — Trunking + LACP/PAgP + STP
- [ ] Fase 3 — Enrutamiento inter-VLAN + EIGRP
- [ ] Fase 4 — DHCP + DHCP Relay
- [ ] Fase 5 — ACLs
- [ ] Fase 6 — Pruebas de tolerancia a fallos
- [ ] Fase 7 — Manual técnico final y entrega
