# Chapin Red — Diseño técnico completo

Documento de referencia con la topología final, mapa de puertos/cableado,
direccionamiento IP y comandos CLI de cada dispositivo. Carné: **202300335** (X = 35).

> Nota sobre nombres de puertos: los comandos usan `GigabitEthernet1/0/x` para los
> 3650, `FastEthernet0/x` / `GigabitEthernet0/x` para los 3560 y `FastEthernet0/x`
> para los 2960. Packet Tracer puede nombrarlos distinto según el modelo exacto que
> arrastres — verifica con `show ip interface brief` en cada equipo y ajusta el
> número de interfaz si no coincide. La lógica de qué se conecta con qué no cambia.

## 1. Inventario de dispositivos

| Nombre | Modelo | Rol |
|---|---|---|
| MSW-NORTE | 3650-24PS | MAN — edificio Principal, aloja los DHCP |
| MSW-IZQ-GW | 3650-24PS | MAN — gateway edificio Izquierdo |
| MSW-DER-GW | 3650-24PS | MAN — gateway edificio Derecho |
| MSW-ADMIN | 3650-24PS | MAN — edificio Administración (VLAN 99) |
| CORE-IZQ-A | 3560-24PS | Core edificio Izquierdo |
| CORE-IZQ-B | 3560-24PS | Core edificio Izquierdo (redundante) |
| DIST-IZQ | 3560-24PS | Distribución edificio Izquierdo (inter-VLAN + EIGRP) |
| ACC-IZQ-A | 2960-24TT | Acceso — VLAN Naranja Izquierdo |
| ACC-IZQ-B | 2960-24TT | Acceso — VLAN Verde Izquierdo |
| DIST-DER | 3560-24PS | Distribución edificio Derecho (inter-VLAN + EIGRP) |
| ACC-DER-A | 2960-24TT | Acceso — VLAN Naranja Derecho |
| ACC-DER-B | 2960-24TT | Acceso — VLAN Verde Derecho |
| SRV-DHCP-IZQ | Server-PT | DHCP para VLAN 10, 20, 99 |
| SRV-DHCP-DER | Server-PT | DHCP para VLAN 30, 40 |
| PC0, PC1 | PC-PT | VLAN 10 (Naranja IZQ) |
| Laptop0, Laptop1 | Laptop-PT | VLAN 20 (Verde IZQ) |
| PC2, PC3 | PC-PT | VLAN 30 (Naranja DER) |
| Laptop2, Laptop3 | Laptop-PT | VLAN 40 (Verde DER) |
| PC-ADMIN | PC-PT | VLAN 99 (ADMIN) |

## 2. VLANs

| ID | Nombre | Subred | Edificio |
|---|---|---|---|
| 10 | VLAN_Naranja_EdificioIZQ_202300335 | 192.188.35.0/27 | Izquierdo |
| 20 | VLAN_Verde_EdificioIZQ_202300335 | 192.188.35.32/27 | Izquierdo |
| 30 | VLAN_Naranja_EdificioDER_202300335 | 192.188.35.64/27 | Derecho |
| 40 | VLAN_Verde_EdificioDER_202300335 | 192.188.35.96/27 | Derecho |
| 99 | VLAN_ADMIN_202300335 | 192.188.35.128/28 | Administración |
| 100 | VLAN_TransitoIZQ_202300335 (infraestructura, solo IZQ) | 10.4.35.24/30 | Izquierdo |

VLAN 100 es una VLAN de tránsito interna: permite que el puerto de fibra de
`CORE-IZQ-A` hacia `MSW-IZQ-GW` sea un simple puerto de acceso (Core se queda 100%
Capa 2), mientras el enrutamiento real de esa subred lo hace la SVI en `DIST-IZQ`.
Así los 5 enlaces LACP del edificio izquierdo permanecen puros de Capa 2, tal como
pide la rúbrica.

## 3. Mapa de cableado (puerto a puerto)

### 3.1 MAN — malla redundante (fibra óptica, módulos Gigabit)

| Extremo A | Puerto A | Extremo B | Puerto B | Cable |
|---|---|---|---|---|
| MSW-NORTE | Gi1/0/1 | MSW-IZQ-GW | Gi1/0/1 | Fibra óptica |
| MSW-NORTE | Gi1/0/2 | MSW-DER-GW | Gi1/0/1 | Fibra óptica |
| MSW-NORTE | Gi1/0/3 | MSW-ADMIN | Gi1/0/1 | Fibra óptica |
| MSW-IZQ-GW | Gi1/0/2 | MSW-ADMIN | Gi1/0/2 | Fibra óptica |
| MSW-DER-GW | Gi1/0/2 | MSW-ADMIN | Gi1/0/3 | Fibra óptica |
| MSW-IZQ-GW | Gi1/0/3 | MSW-DER-GW | Gi1/0/3 | Fibra óptica |
| MSW-NORTE | Gi1/0/4 | SRV-DHCP-IZQ | FastEthernet0 | Cobre recto |
| MSW-NORTE | Gi1/0/5 | SRV-DHCP-DER | FastEthernet0 | Cobre recto |
| MSW-IZQ-GW | Gi1/0/4 | CORE-IZQ-A | Gi0/1 | Fibra óptica |
| MSW-DER-GW | Gi1/0/4 | DIST-DER | Gi0/1 | Cobre recto (miembro Po3) |
| MSW-DER-GW | Gi1/0/5 | DIST-DER | Gi0/2 | Cobre recto (miembro Po3) |
| MSW-ADMIN | Fa0/1 | PC-ADMIN | FastEthernet0 | Cobre recto |

> En Packet Tracer, para poner fibra en un 3650/3560 primero apaga el equipo
> (botón físico), agrega un módulo `PT-SWITCH-NM-1GBIC` o similar a un slot vacío,
> enciéndelo, y ahí conecta el cable de fibra.

### 3.2 Edificio Izquierdo (interno)

| Extremo A | Puerto A | Extremo B | Puerto B | EtherChannel | Cable |
|---|---|---|---|---|---|
| CORE-IZQ-A | Fa0/1 | CORE-IZQ-B | Fa0/1 | Po1 (LACP) | Cobre |
| CORE-IZQ-A | Fa0/2 | CORE-IZQ-B | Fa0/2 | Po1 (LACP) | Cobre |
| CORE-IZQ-A | Fa0/3 | DIST-IZQ | Fa0/1 | Po2 (LACP) | Cobre |
| CORE-IZQ-A | Fa0/4 | DIST-IZQ | Fa0/2 | Po2 (LACP) | Cobre |
| CORE-IZQ-B | Fa0/3 | DIST-IZQ | Fa0/3 | Po3 (LACP) | Cobre |
| CORE-IZQ-B | Fa0/4 | DIST-IZQ | Fa0/4 | Po3 (LACP) | Cobre |
| DIST-IZQ | Fa0/5 | ACC-IZQ-A | Fa0/1 | Po4 (LACP) | Cobre |
| DIST-IZQ | Fa0/6 | ACC-IZQ-A | Fa0/2 | Po4 (LACP) | Cobre |
| DIST-IZQ | Fa0/7 | ACC-IZQ-B | Fa0/1 | Po5 (LACP) | Cobre |
| DIST-IZQ | Fa0/8 | ACC-IZQ-B | Fa0/2 | Po5 (LACP) | Cobre |
| ACC-IZQ-A | Fa0/3 | PC0 | FastEthernet0 | — | Cobre |
| ACC-IZQ-A | Fa0/4 | PC1 | FastEthernet0 | — | Cobre |
| ACC-IZQ-B | Fa0/3 | Laptop0 | FastEthernet0 | — | Cobre |
| ACC-IZQ-B | Fa0/4 | Laptop1 | FastEthernet0 | — | Cobre |

Total: **5 EtherChannels LACP** (Po1–Po5), todos Capa 2.

### 3.3 Edificio Derecho (interno)

| Extremo A | Puerto A | Extremo B | Puerto B | EtherChannel | Cable |
|---|---|---|---|---|---|
| DIST-DER | Gi0/1 | MSW-DER-GW | Gi1/0/4 | Po3 (PAgP, **Capa 3**) | Cobre |
| DIST-DER | Gi0/2 | MSW-DER-GW | Gi1/0/5 | Po3 (PAgP, **Capa 3**) | Cobre |
| DIST-DER | Fa0/1 | ACC-DER-A | Fa0/1 | Po1 (PAgP, Capa 2) | Cobre |
| DIST-DER | Fa0/2 | ACC-DER-A | Fa0/2 | Po1 (PAgP, Capa 2) | Cobre |
| DIST-DER | Fa0/3 | ACC-DER-B | Fa0/1 | Po2 (PAgP, Capa 2) | Cobre |
| DIST-DER | Fa0/4 | ACC-DER-B | Fa0/2 | Po2 (PAgP, Capa 2) | Cobre |
| ACC-DER-A | Fa0/3 | PC2 | FastEthernet0 | — | Cobre |
| ACC-DER-A | Fa0/4 | PC3 | FastEthernet0 | — | Cobre |
| ACC-DER-B | Fa0/3 | Laptop2 | FastEthernet0 | — | Cobre |
| ACC-DER-B | Fa0/4 | Laptop3 | FastEthernet0 | — | Cobre |

Total: **3 EtherChannels PAgP** (Po1, Po2 Capa 2; Po3 Capa 3 hacia el MAN).

## 4. Direccionamiento IP — enlaces `10.4.35.0/24`

| Bloque /30 | Enlace | IP .A | IP .B |
|---|---|---|---|
| 10.4.35.0/30 | MSW-NORTE ↔ MSW-IZQ-GW | .1 | .2 |
| 10.4.35.4/30 | MSW-NORTE ↔ MSW-DER-GW | .5 | .6 |
| 10.4.35.8/30 | MSW-IZQ-GW ↔ MSW-ADMIN | .9 | .10 |
| 10.4.35.12/30 | MSW-DER-GW ↔ MSW-ADMIN | .13 | .14 |
| 10.4.35.16/30 | MSW-IZQ-GW ↔ MSW-DER-GW | .17 | .18 |
| 10.4.35.20/30 | MSW-NORTE ↔ MSW-ADMIN | .21 | .22 |
| 10.4.35.24/30 | DIST-IZQ (SVI100) ↔ MSW-IZQ-GW (vía Core, VLAN 100) | .25 | .26 |
| 10.4.35.28/30 | DIST-DER (Po3) ↔ MSW-DER-GW | .29 | .30 |
| 10.4.35.32/30 | MSW-NORTE ↔ SRV-DHCP-IZQ | .33 | .34 |
| 10.4.35.36/30 | MSW-NORTE ↔ SRV-DHCP-DER | .37 | .38 |

EIGRP AS **35** en: MSW-NORTE, MSW-IZQ-GW, MSW-DER-GW, MSW-ADMIN, DIST-IZQ, DIST-DER.

## 5. Comandos CLI por dispositivo

### 5.1 MSW-NORTE

```
enable
configure terminal
hostname MSW-NORTE
no ip domain-lookup
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.4.35.1 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.4.35.5 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.4.35.21 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/4
 no switchport
 ip address 10.4.35.33 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/5
 no switchport
 ip address 10.4.35.37 255.255.255.252
 no shutdown
router eigrp 35
 network 10.4.35.0 0.0.0.3
 network 10.4.35.4 0.0.0.3
 network 10.4.35.20 0.0.0.3
 network 10.4.35.32 0.0.0.3
 network 10.4.35.36 0.0.0.3
 no auto-summary
end
write memory
```

### 5.2 MSW-IZQ-GW

```
enable
configure terminal
hostname MSW-IZQ-GW
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.4.35.2 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.4.35.9 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.4.35.17 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/4
 no switchport
 ip address 10.4.35.26 255.255.255.252
 no shutdown
router eigrp 35
 network 10.4.35.0 0.0.0.3
 network 10.4.35.8 0.0.0.3
 network 10.4.35.16 0.0.0.3
 network 10.4.35.24 0.0.0.3
 no auto-summary
end
write memory
```

### 5.3 MSW-DER-GW

```
enable
configure terminal
hostname MSW-DER-GW
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.4.35.6 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.4.35.13 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.4.35.18 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/4
 channel-group 3 mode desirable
interface GigabitEthernet1/0/5
 channel-group 3 mode desirable
interface Port-channel3
 no switchport
 ip address 10.4.35.29 255.255.255.252
 no shutdown
router eigrp 35
 network 10.4.35.4 0.0.0.3
 network 10.4.35.12 0.0.0.3
 network 10.4.35.16 0.0.0.3
 network 10.4.35.28 0.0.0.3
 no auto-summary
end
write memory
```

### 5.4 MSW-ADMIN

```
enable
configure terminal
hostname MSW-ADMIN
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.4.35.10 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.4.35.14 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.4.35.22 255.255.255.252
 no shutdown
vlan 99
 name VLAN_ADMIN_202300335
interface Vlan99
 ip address 192.188.35.129 255.255.255.240
 ip helper-address 10.4.35.34
 no shutdown
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 99
router eigrp 35
 network 10.4.35.8 0.0.0.3
 network 10.4.35.12 0.0.0.3
 network 10.4.35.20 0.0.0.3
 network 192.188.35.128 0.0.0.15
 no auto-summary
end
write memory
```

### 5.5 CORE-IZQ-A

```
enable
configure terminal
hostname CORE-IZQ-A
vtp mode client
vtp domain ChapinRed
vtp password Chapin2026*
interface range FastEthernet0/1-2
 channel-group 1 mode active
interface range FastEthernet0/3-4
 channel-group 2 mode active
interface Port-channel1
 switchport mode trunk
interface Port-channel2
 switchport mode trunk
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 100
spanning-tree mode pvst
end
write memory
```

> `vlan 100` debe existir antes de asignarla: créala igual en el VTP server
> (`DIST-IZQ`) — se propaga sola por VTP a este switch cliente.

### 5.6 CORE-IZQ-B

```
enable
configure terminal
hostname CORE-IZQ-B
vtp mode client
vtp domain ChapinRed
vtp password Chapin2026*
interface range FastEthernet0/1-2
 channel-group 1 mode active
interface range FastEthernet0/3-4
 channel-group 3 mode active
interface Port-channel1
 switchport mode trunk
interface Port-channel3
 switchport mode trunk
spanning-tree mode pvst
end
write memory
```

### 5.7 DIST-IZQ (VTP Server, hace inter-VLAN routing)

```
enable
configure terminal
hostname DIST-IZQ
vtp mode server
vtp domain ChapinRed
vtp password Chapin2026*
vlan 10
 name VLAN_Naranja_EdificioIZQ_202300335
vlan 20
 name VLAN_Verde_EdificioIZQ_202300335
vlan 100
 name VLAN_TransitoIZQ_202300335
interface range FastEthernet0/1-2
 channel-group 2 mode active
interface range FastEthernet0/3-4
 channel-group 3 mode active
interface range FastEthernet0/5-6
 channel-group 4 mode active
interface range FastEthernet0/7-8
 channel-group 5 mode active
interface Port-channel2
 switchport mode trunk
interface Port-channel3
 switchport mode trunk
interface Port-channel4
 switchport mode trunk
interface Port-channel5
 switchport mode trunk
interface Vlan10
 ip address 192.188.35.1 255.255.255.224
 ip helper-address 10.4.35.34
 no shutdown
interface Vlan20
 ip address 192.188.35.33 255.255.255.224
 ip helper-address 10.4.35.34
 no shutdown
interface Vlan100
 ip address 10.4.35.25 255.255.255.252
 no shutdown
ip routing
router eigrp 35
 network 192.188.35.0 0.0.0.31
 network 192.188.35.32 0.0.0.31
 network 10.4.35.24 0.0.0.3
 no auto-summary
spanning-tree mode pvst
spanning-tree vlan 10,20,100 root primary
!--- ACLs de seguridad (ver sección 6)
end
write memory
```

### 5.8 ACC-IZQ-A (acceso, VLAN 10)

```
enable
configure terminal
hostname ACC-IZQ-A
vtp mode client
vtp domain ChapinRed
vtp password Chapin2026*
interface range FastEthernet0/1-2
 channel-group 4 mode active
interface Port-channel4
 switchport mode trunk
interface range FastEthernet0/3-4
 switchport mode access
 switchport access vlan 10
spanning-tree mode pvst
end
write memory
```

### 5.9 ACC-IZQ-B (acceso, VLAN 20)

```
enable
configure terminal
hostname ACC-IZQ-B
vtp mode client
vtp domain ChapinRed
vtp password Chapin2026*
interface range FastEthernet0/1-2
 channel-group 5 mode active
interface Port-channel5
 switchport mode trunk
interface range FastEthernet0/3-4
 switchport mode access
 switchport access vlan 20
spanning-tree mode pvst
end
write memory
```

### 5.10 DIST-DER (VTP Server, hace inter-VLAN routing + uplink L3 PAgP)

```
enable
configure terminal
hostname DIST-DER
vtp mode server
vtp domain ChapinRed
vtp password Chapin2026*
vlan 30
 name VLAN_Naranja_EdificioDER_202300335
vlan 40
 name VLAN_Verde_EdificioDER_202300335
interface range FastEthernet0/1-2
 channel-group 1 mode desirable
interface range FastEthernet0/3-4
 channel-group 2 mode desirable
interface Port-channel1
 switchport mode trunk
interface Port-channel2
 switchport mode trunk
interface range GigabitEthernet0/1-2
 channel-group 3 mode desirable
interface Port-channel3
 no switchport
 ip address 10.4.35.30 255.255.255.252
 no shutdown
interface Vlan30
 ip address 192.188.35.65 255.255.255.224
 ip helper-address 10.4.35.38
 no shutdown
interface Vlan40
 ip address 192.188.35.97 255.255.255.224
 ip helper-address 10.4.35.38
 no shutdown
ip routing
router eigrp 35
 network 192.188.35.64 0.0.0.31
 network 192.188.35.96 0.0.0.31
 network 10.4.35.28 0.0.0.3
 no auto-summary
spanning-tree mode pvst
spanning-tree vlan 30,40 root primary
!--- ACLs de seguridad (ver sección 6)
end
write memory
```

### 5.11 ACC-DER-A (acceso, VLAN 30) y ACC-DER-B (acceso, VLAN 40)

```
enable
configure terminal
hostname ACC-DER-A
vtp mode client
vtp domain ChapinRed
vtp password Chapin2026*
interface range FastEthernet0/1-2
 channel-group 1 mode desirable
interface Port-channel1
 switchport mode trunk
interface range FastEthernet0/3-4
 switchport mode access
 switchport access vlan 30
spanning-tree mode pvst
end
write memory
```

```
enable
configure terminal
hostname ACC-DER-B
vtp mode client
vtp domain ChapinRed
vtp password Chapin2026*
interface range FastEthernet0/1-2
 channel-group 2 mode desirable
interface Port-channel2
 switchport mode trunk
interface range FastEthernet0/3-4
 switchport mode access
 switchport access vlan 40
spanning-tree mode pvst
end
write memory
```

## 6. ACLs (a colocar en DIST-IZQ y DIST-DER, ver Fase 5)

Se documentarán con detalle completo en la Fase 5, cuando el enrutamiento ya esté
verificado. Lógica resumen:

- `ACL_NARANJA_IZQ` / `ACL_NARANJA_DER` (aplicadas `in` en VLAN 10 / VLAN 30):
  permiten Naranja↔Naranja entre edificios, deniegan hacia Verde y ADMIN.
- `ACL_VERDE_IZQ` / `ACL_VERDE_DER` (aplicadas `in` en VLAN 20 / VLAN 40):
  permiten Verde↔Verde entre edificios, deniegan hacia Naranja y ADMIN.
- VLAN ADMIN no necesita ACL propia: al bloquear en el origen (Naranja/Verde) el
  tráfico con destino a `192.188.35.128/28`, ninguna VLAN puede iniciar
  comunicación hacia ADMIN, mientras que ADMIN sí puede iniciar hacia cualquiera
  (no tiene ninguna ACL que se lo impida).

## 7. DHCP — pools (Fase 4)

| Servidor | Pool | Red | Máscara | Gateway | Rango |
|---|---|---|---|---|---|
| SRV-DHCP-IZQ (10.4.35.34) | Naranja_IZQ | 192.188.35.0 | 255.255.255.224 | 192.188.35.1 | .2–.30 |
| SRV-DHCP-IZQ | Verde_IZQ | 192.188.35.32 | 255.255.255.224 | 192.188.35.33 | .34–.62 |
| SRV-DHCP-IZQ | Admin | 192.188.35.128 | 255.255.255.240 | 192.188.35.129 | .130–.142 |
| SRV-DHCP-DER (10.4.35.38) | Naranja_DER | 192.188.35.64 | 255.255.255.224 | 192.188.35.65 | .66–.94 |
| SRV-DHCP-DER | Verde_DER | 192.188.35.96 | 255.255.255.224 | 192.188.35.97 | .98–.126 |
