# Conmutación

Diseño e implementación de una red jerárquica tipo **estrella extendida** para una empresa mediana, con segmentación por VLANs, enrutamiento Inter-VLAN, VTP, EtherChannel, seguridad de puertos, acceso SSH y salida a Internet redundante a través de dos routers (ISP principal y respaldo).

---

## 1. Topología de red

![Topología de red](Captura%20de%20pantalla%202026-08-17%20161425.png)

- **S1**: switch núcleo, distribuye VLANs a S2, S3, S4 y S5.
- **Po1**: EtherChannel S1 ↔ S2.
- **Po2**: EtherChannel S1 ↔ S4.
- S1 ↔ S3 y S1 ↔ S5: enlaces troncales estándar (802.1Q).
- **R1**: router principal, subinterfaces (router-on-a-stick) para las 10 VLANs + enlace serial hacia el ISP principal.
- **R2**: router de respaldo, mismas subinterfaces (HSRP standby) + enlace serial hacia un segundo ISP.
- Cada switch de acceso (S2, S3, S4, S5) tiene **2 PCs por VLAN**, distribuidos en switches distintos para cumplir el requisito de redundancia de acceso.

### Distribución por switch

| Switch | VLANs de acceso | PCs conectados |
|---|---|---|
| S2 | 10, 20, 30, 40, 50 | PC1, PC2, PC3, PC4, PC5 |
| S3 | 60, 70, 80, 90, 100 | PC6, PC7, PC8, PC9, PC10 |
| S4 | 10, 20, 30, 40, 50 | PC11, PC12, PC13, PC14, PC15 |
| S5 | 60, 70, 80, 90, 100 | PC16, PC17, PC18, PC19, PC20 |
| S1 | Núcleo / troncal | Sin PCs directos — solo interconexión |

---

## 2. Tabla de VLANs y direccionamiento IP (VLSM)

Cada VLAN tiene un tamaño de subred distinto (VLSM real), calculado a partir de las IP host indicadas en el diseño.

| VLAN | Nombre sugerido | Red | Máscara | Wildcard | Rango útil | Gateway sugerido (R1/HSRP) |
|---|---|---|---|---|---|---|
| 10 | VENTAS | 172.16.0.192 | /27 (255.255.255.224) | 0.0.0.31 | .193–.222 | 172.16.0.222 |
| 20 | MARKETING | 172.16.0.224 | /27 (255.255.255.224) | 0.0.0.31 | .225–.254 | 172.16.0.254 |
| 30 | FINANZAS | 172.16.1.48 | /28 (255.255.255.240) | 0.0.0.15 | .49–.62 | 172.16.1.62 |
| 40 | RRHH | 172.16.0.128 | /26 (255.255.255.192) | 0.0.0.63 | .129–.190 | 172.16.0.190 |
| 50 | LOGISTICA | 172.16.0.0 | /26 (255.255.255.192) | 0.0.0.63 | .1–.62 | 172.16.0.62 |
| 60 | TI | 172.16.1.0 | /27 (255.255.255.224) | 0.0.0.31 | .1–.30 | 172.16.1.30 |
| 70 | LEGAL | 172.16.1.32 | /28 (255.255.255.240) | 0.0.0.15 | .33–.46 | 172.16.1.46 |
| 80 | GERENCIA | 172.16.0.64 | /26 (255.255.255.192) | 0.0.0.63 | .65–.126 | 172.16.0.126 |
| 90 | SOPORTE | 172.16.1.64 | /29 (255.255.255.248) | 0.0.0.7 | .65–.70 | 172.16.1.70 |
| 100 | INVITADOS | 172.16.1.80 | /29 (255.255.255.248) | 0.0.0.7 | .81–.86 | 172.16.1.86 |
| 125 | ADMIN (management) | 172.16.125.0 | /29 (ejemplo, ajustar) | 0.0.0.7 | .1–.6 | 172.16.125.1 |

> Nota: los gateways se ubicaron en la última IP útil de cada subred para no chocar con las IP ya asignadas a los PCs. Ajustar si el diseño final define otra convención.

### Tabla de hosts (PCs)

| PC | IP | Máscara | VLAN | Switch |
|---|---|---|---|---|
| PC1 | 172.16.0.200 | 255.255.255.224 | 10 | S2 |
| PC11 | 172.16.0.201 | 255.255.255.224 | 10 | S4 |
| PC2 | 172.16.0.229 | 255.255.255.224 | 20 | S2 |
| PC12 | 172.16.0.228 | 255.255.255.224 | 20 | S4 |
| PC3 | 172.16.1.50 | 255.255.255.240 | 30 | S2 |
| PC13 | 172.16.1.52 | 255.255.255.240 | 30 | S4 |
| PC4 | 172.16.0.134 | 255.255.255.192 | 40 | S2 |
| PC14 | 172.16.0.132 | 255.255.255.192 | 40 | S4 |
| PC5 | 172.16.0.30 | 255.255.255.192 | 50 | S2 |
| PC15 | 172.16.0.4 | 255.255.255.192 | 50 | S4 |
| PC6 | 172.16.1.5 | 255.255.255.224 | 60 | S3 |
| PC16 | 172.16.1.4 | 255.255.255.224 | 60 | S5 |
| PC7 | 172.16.1.36 | 255.255.255.240 | 70 | S3 |
| PC17 | 172.16.1.38 | 255.255.255.240 | 70 | S5 |
| PC8 | 172.16.0.68 | 255.255.255.192 | 80 | S3 |
| PC18 | 172.16.0.94 | 255.255.255.192 | 80 | S5 |
| PC9 | 172.16.1.67 | 255.255.255.248 | 90 | S3 |
| PC19 | 172.16.1.68 | 255.255.255.248 | 90 | S5 |
| PC10 | 172.16.1.84 | 255.255.255.248 | 100 | S3 |
| PC20 | 172.16.1.82 | 255.255.255.248 | 100 | S5 |

### Enlaces WAN (salida a ISP)

| Enlace | Router | IP | Rol |
|---|---|---|---|
| S0/0/0 | R1 | 200.1.1.2 /30 | Principal |
| S0/0/0 | ISP1 | 200.1.1.1 /30 | — |
| S0/0/1 | R2 | 200.2.2.2 /30 | Respaldo |
| S0/0/0 | ISP2 | 200.2.2.1 /30 | — |

---

## 3. VTP (VLAN Trunking Protocol)

| Parámetro | Valor |
|---|---|
| Dominio | Grupo_2 |
| Versión | 2 |
| Password | grnp0#2 |
| Modo servidor | S1 |
| Modo cliente | S2, S3, S4, S5 |

```
vtp version 2
vtp domain Grupo_2
vtp password grnp0#2
vtp mode server   ! (solo en S1, en el resto: vtp mode client)
```

---

## 4. Configuración base de switches (aplicar en S1–S5, cambiando hostname)

```
enable
configure terminal
hostname S1
enable secret grnp0#2
line console 0
 password grnp0#2
 login
 exit
line vty 0 15
 password grnp0#2
 login
 exit
service password-encryption
ip domain-name grupo2.com
crypto key generate rsa
1024
ip ssh version 2
username admin secret grnp0#2
line vty 0 4
 transport input ssh
 login local
 exit
banner motd #ACCESO RESTRINGIDO SOLO A PERSONAL AUTORIZADO#
do write
end
```

**Puertos no usados**: se deshabilitan por defecto (`shutdown`) y solo se activan (`no shutdown`) los que tienen un dispositivo conectado.

**Seguridad de puertos** (en cada puerto de acceso con PC conectado):
```
interface fastEthernet0/X
 switchport mode access
 switchport access vlan XX
 switchport port-security
 switchport port-security maximum 10
 switchport port-security mac-address sticky
 no shutdown
```

**Troncales** (S1 hacia S2/S3/S4/S5, y S2/S4/S3/S5 hacia sus PCs si aplica):
```
interface range fX/X - X
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60,70,80,90,100,125
 switchport trunk native vlan 125
```

---

## 5. EtherChannel

**Po1 (S1 ↔ S2)**
```
! En S1
interface range fastEthernet0/1 - 2
 channel-group 1 mode active
 switchport mode trunk
 exit
interface port-channel 1
 switchport mode trunk

! En S2 (mismos puertos físicos del otro lado)
interface range fastEthernet0/1 - 2
 channel-group 1 mode active
 switchport mode trunk
```

**Po2 (S1 ↔ S4)** — mismo patrón con `channel-group 2`.

Verificación:
```
show etherchannel summary
show interfaces port-channel 1
```

---

## 6. Inter-VLAN Routing (Router-on-a-Stick) + HSRP en R1 y R2

Ambos routers (R1 activo, R2 standby) tienen subinterfaces para cada VLAN, conectadas por troncal hacia S1, con **HSRP** como gateway redundante.

**Ejemplo en R1 (VLAN 10, repetir el patrón para las 10 VLANs):**
```
interface gigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 172.16.0.221 255.255.255.224
 standby 10 ip 172.16.0.222
 standby 10 priority 110
 standby 10 preempt
 exit
```

**En R2 (mismo VLAN 10, standby):**
```
interface gigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 172.16.0.220 255.255.255.224
 standby 10 ip 172.16.0.222
 standby 10 priority 90
 exit
```

- `172.16.0.222` = IP virtual (gateway real que usan los PCs de la VLAN 10).
- R1 tiene mayor prioridad (110) → es el gateway activo.
- Repetir la misma lógica (subinterface + `standby` con IP virtual única) para las VLANs 20 a 100, usando el gateway sugerido de la tabla de la sección 2.

Verificación:
```
show standby brief
```

---

## 7. Salida a Internet — ISP principal y de respaldo

**R1 — interfaz hacia ISP principal:**
```
interface serial0/0/0
 ip address 200.1.1.2 255.255.255.252
 no shutdown
 exit
ip route 0.0.0.0 0.0.0.0 200.1.1.1
```

**R2 — interfaz hacia ISP de respaldo:**
```
interface serial0/0/1
 ip address 200.2.2.2 255.255.255.252
 no shutdown
 exit
ip route 0.0.0.0 0.0.0.0 200.2.2.1 5
```
(distancia administrativa 5 → ruta flotante, solo se activa si falla la principal)

**NAT overload (en R1, y espejo en R2 para cuando esté activo):**
```
access-list 1 permit 172.16.0.192 0.0.0.31
access-list 1 permit 172.16.0.224 0.0.0.31
access-list 1 permit 172.16.1.48 0.0.0.15
access-list 1 permit 172.16.0.128 0.0.0.63
access-list 1 permit 172.16.0.0 0.0.0.63
access-list 1 permit 172.16.1.0 0.0.0.31
access-list 1 permit 172.16.1.32 0.0.0.15
access-list 1 permit 172.16.0.64 0.0.0.63
access-list 1 permit 172.16.1.64 0.0.0.7
access-list 1 permit 172.16.1.80 0.0.0.7

interface gigabitEthernet0/0.10
 ip nat inside
! (repetir ip nat inside en cada subinterface .20 a .100)

interface serial0/0/0
 ip nat outside
 exit
ip nat inside source list 1 interface serial0/0/0 overload
```

En R2, repetir con `ip nat outside` en `serial0/0/1` y una segunda sentencia `ip nat inside source list 1 interface serial0/0/1 overload` para que el NAT también traduzca cuando el tráfico sale por el enlace de respaldo.

---

## 8. Tabla de usuarios y contraseñas

| Dispositivo | Usuario | Contraseña | Tipo |
|---|---|---|---|
| S1–S5, R1, R2 | — | grnp0#2 | Enable secret |
| S1–S5, R1, R2 | — | grnp0#2 | Consola / VTY (legacy) |
| S1–S5, R1, R2 | admin | grnp0#2 | SSH (username secret) |
| VTP | — | grnp0#2 | Password de dominio |

---

## 9. Checklist de verificación (para el sustento)

- [ ] `show vlan brief` en cada switch
- [ ] `show interfaces trunk`
- [ ] `show etherchannel summary`
- [ ] `show vtp status`
- [ ] `show standby brief` en R1 y R2
- [ ] `show ip route` en R1 y R2
- [ ] `show ip nat translations`
- [ ] `ping` entre PCs de distinta VLAN (Inter-VLAN routing)
- [ ] `ping` desde un PC hacia la nube ISP (salida a Internet)
- [ ] Apagar S0/0/0 en R1 y confirmar failover automático hacia R2 (`show ip route` debe mostrar la ruta con distancia 5 activa)
- [ ] Acceso remoto por SSH desde un PC hacia cada switch/router (`ssh -l admin <ip>`)
- [ ] `show running-config` de cada switch y router

---

## 10. Pendientes / puntos adicionales sugeridos

- Configurar 2 redes inalámbricas con autenticación centralizada (RADIUS/WLC) — puntos adicionales del enunciado, no cubierto en este documento.
- Confirmar y documentar los puertos físicos exactos usados en S1 hacia S3 y S5 (no forman parte de un EtherChannel, solo troncal simple).
- Ajustar el direccionamiento de la VLAN 125 (management) a los valores reales usados en el proyecto.
