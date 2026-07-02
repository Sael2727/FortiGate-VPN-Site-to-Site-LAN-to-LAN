# 🔐 FortiGate VPN Site-to-Site — LAN-to-LAN

<div align="center">

![FortiGate](https://img.shields.io/badge/FortiGate-FortiOS-red?style=for-the-badge&logo=fortinet)
![IPSec](https://img.shields.io/badge/IPSec-IKEv1-green?style=for-the-badge)
![VPN](https://img.shields.io/badge/VPN-Site--to--Site-blue?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-gray?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y verificación de una **VPN Site-to-Site LAN-to-LAN entre dos equipos FortiGate** para permitir comunicación segura entre dos redes LAN separadas. La comunicación entre LAN1 y LAN2 se realiza a través de un túnel **IPSec IKEv1**, utilizando un router ISP como red intermedia simulada. La configuración se realiza desde la **GUI HTTP de FortiOS** y se complementa con CLI.

> 💡 **Diferencia clave respecto a Cisco IOS:** En FortiGate no se usa `crypto map` ni `crypto isakmp policy`. En su lugar se configuran objetos **Phase 1** (equivalente a IKE) y **Phase 2** (equivalente a IPSec SA) como interfaces virtuales, y las políticas firewall controlan qué tráfico puede entrar y salir del túnel. **NAT debe estar desactivado** en las políticas VPN.

> ⚠️ **Nota sobre traceroute:** Puede aparecer una dirección `192.168.226.x` como salto intermedio porque `port1` está conectado a la red de administración/Net para acceso HTTP a la GUI. El resultado es válido: el destino final responde y el monitor IPSec muestra tráfico activo.

---

## 🗺️ Topología de Red

Dos FortiGate conectados a través de un router ISP simulado. Cada FortiGate tiene una LAN local con una VPC. La administración de los FortiGate se realiza por `port1` conectado a la red de gestión.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| FGT1 | port3 | 10.7.25.1/30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2/30 | Enlace hacia FGT1 |
| ISP | Ethernet0/1 | 10.7.25.6/30 | Enlace hacia FGT2 |
| FGT2 | port3 | 10.7.25.5/30 | WAN hacia ISP |
| FGT1 | port2 | 10.7.25.65/27 | Gateway LAN1 |
| VPC1-LAN1 | eth0 | 10.7.25.66/27 | Host LAN1 |
| FGT2 | port2 | 10.7.25.97/27 | Gateway LAN2 |
| VPC2-LAN2 | eth0 | 10.7.25.98/27 | Host LAN2 |
| FGT1 | port1 | 192.168.226.210/24 | Gestión HTTP |
| FGT2 | port1 | 192.168.226.211/24 | Gestión HTTP |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Tipo de VPN | IPSec Site-to-Site / LAN-to-LAN |
| Equipos | FortiGate FGT1 y FGT2 |
| Interfaz WAN VPN | port3 en ambos FortiGate |
| Red LAN1 | 10.7.25.64/27 |
| Red LAN2 | 10.7.25.96/27 |
| Remote Gateway FGT1 | 10.7.25.5 |
| Remote Gateway FGT2 | 10.7.25.1 |
| Autenticación | Pre-shared key |
| Clave PSK | VPN12345 |
| IKE Version | IKEv1 |
| Propuesta Phase 1 | des-md5 / des-sha1 |
| Propuesta Phase 2 | des-md5 / des-sha1 |
| Grupo DH | 5 |
| NAT en políticas VPN | Desactivado |

---

## 🔍 Funcionamiento

La VPN FortiGate Site-to-Site funciona con los siguientes componentes:

**Phase 1 (IKE)** — Define los parámetros de negociación IKEv1: interfaz WAN (`port3`), gateway remoto, PSK, propuesta de cifrado y grupo DH. Es el equivalente a `crypto isakmp policy` en Cisco.

**Phase 2 (IPSec SA)** — Define los selectores de tráfico (subnets origen/destino) y la propuesta IPSec. Es el equivalente al `crypto map` + ACL en Cisco.

**Rutas estáticas** — FGT1 tiene una ruta hacia `10.7.25.96/27` saliendo por la interfaz virtual del túnel `VPN-FGT1-FGT2`. FGT2 tiene la ruta inversa hacia `10.7.25.64/27`.

**Políticas firewall** — Dos políticas por FortiGate (LAN→VPN y VPN→LAN) con NAT desactivado permiten el tráfico bidireccional a través del túnel.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
ip cef
interface Ethernet0/0
 description WAN_HACIA_FGT1_PORT3
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description WAN_HACIA_FGT2_PORT3
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### FGT1 — Configuración Principal (FortiOS CLI)
```fortios
config system hostname
    set hostname FGT1
end

config system interface
    edit "port1"
        set alias "MGMT_NET"
        set mode static
        set ip 192.168.226.210 255.255.255.0
        set allowaccess ping http https ssh
        set role wan
    next
    edit "port2"
        set alias "LAN1"
        set mode static
        set ip 10.7.25.65 255.255.255.224
        set allowaccess ping https ssh http
        set role lan
    next
    edit "port3"
        set alias "WAN_HACIA_ISP"
        set mode static
        set ip 10.7.25.1 255.255.255.252
        set allowaccess ping https ssh http
        set role wan
    next
end

config router static
    edit 10
        set dst 0.0.0.0 0.0.0.0
        set gateway 10.7.25.2
        set device "port3"
    next
    edit 20
        set dst 10.7.25.96 255.255.255.224
        set device "VPN-FGT1-FGT2"
    next
    edit 30
        set dst 10.7.25.5 255.255.255.255
        set gateway 10.7.25.2
        set device "port3"
    next
end

config firewall address
    edit "LAN1_10.7.25.64_27"
        set subnet 10.7.25.64 255.255.255.224
    next
    edit "LAN2_10.7.25.96_27"
        set subnet 10.7.25.96 255.255.255.224
    next
end

config vpn ipsec phase1-interface
    edit "VPN-FGT1-FGT2"
        set interface "port3"
        set ike-version 1
        set peertype any
        set proposal des-md5 des-sha1
        set dhgrp 5
        set remote-gw 10.7.25.5
        set psksecret VPN12345
    next
end

config vpn ipsec phase2-interface
    edit "VPN-FGT1-FGT2-P2"
        set phase1name "VPN-FGT1-FGT2"
        set proposal des-md5 des-sha1
        set pfs enable
        set dhgrp 5
        set src-subnet 10.7.25.64 255.255.255.224
        set dst-subnet 10.7.25.96 255.255.255.224
    next
end

config firewall policy
    edit 10
        set name "LAN1_to_LAN2_VPN"
        set srcintf "port2"
        set dstintf "VPN-FGT1-FGT2"
        set srcaddr "LAN1_10.7.25.64_27"
        set dstaddr "LAN2_10.7.25.96_27"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 20
        set name "LAN2_to_LAN1_VPN"
        set srcintf "VPN-FGT1-FGT2"
        set dstintf "port2"
        set srcaddr "LAN2_10.7.25.96_27"
        set dstaddr "LAN1_10.7.25.64_27"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

### FGT2 — Configuración Principal (FortiOS CLI)
```fortios
config system hostname
    set hostname FGT2
end

config system interface
    edit "port1"
        set alias "MGMT_NET"
        set mode static
        set ip 192.168.226.211 255.255.255.0
        set allowaccess ping http https ssh
        set role wan
    next
    edit "port2"
        set alias "LAN2"
        set mode static
        set ip 10.7.25.97 255.255.255.224
        set allowaccess ping https ssh http
        set role lan
    next
    edit "port3"
        set alias "WAN_HACIA_ISP"
        set mode static
        set ip 10.7.25.5 255.255.255.252
        set allowaccess ping https ssh http
        set role wan
    next
end

config router static
    edit 10
        set dst 0.0.0.0 0.0.0.0
        set gateway 10.7.25.6
        set device "port3"
    next
    edit 20
        set dst 10.7.25.64 255.255.255.224
        set device "VPN-FGT2-FGT1"
    next
    edit 30
        set dst 10.7.25.1 255.255.255.255
        set gateway 10.7.25.6
        set device "port3"
    next
end

config firewall address
    edit "LAN1_10.7.25.64_27"
        set subnet 10.7.25.64 255.255.255.224
    next
    edit "LAN2_10.7.25.96_27"
        set subnet 10.7.25.96 255.255.255.224
    next
end

config vpn ipsec phase1-interface
    edit "VPN-FGT2-FGT1"
        set interface "port3"
        set ike-version 1
        set peertype any
        set proposal des-md5 des-sha1
        set dhgrp 5
        set remote-gw 10.7.25.1
        set psksecret VPN12345
    next
end

config vpn ipsec phase2-interface
    edit "VPN-FGT2-FGT1-P2"
        set phase1name "VPN-FGT2-FGT1"
        set proposal des-md5 des-sha1
        set pfs enable
        set dhgrp 5
        set src-subnet 10.7.25.96 255.255.255.224
        set dst-subnet 10.7.25.64 255.255.255.224
    next
end

config firewall policy
    edit 10
        set name "LAN2_to_LAN1_VPN"
        set srcintf "port2"
        set dstintf "VPN-FGT2-FGT1"
        set srcaddr "LAN2_10.7.25.96_27"
        set dstaddr "LAN1_10.7.25.64_27"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 20
        set name "LAN1_to_LAN2_VPN"
        set srcintf "VPN-FGT2-FGT1"
        set dstintf "port2"
        set srcaddr "LAN1_10.7.25.64_27"
        set dstaddr "LAN2_10.7.25.96_27"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

### Configuración de VPCs
```bash
# VPC1-LAN1
ip 10.7.25.66 255.255.255.224 10.7.25.65

# VPC2-LAN2
ip 10.7.25.98 255.255.255.224 10.7.25.97
```

---

## ✅ Verificación

| Acción / Comando | Estado esperado |
|:----------------:|-----------------|
| GUI → Network → Interfaces (FGT1) | port2 y port3 activas |
| GUI → Network → Interfaces (FGT2) | port2 y port3 activas |
| GUI → Network → Static Routes (FGT1) | Ruta a LAN2 por VPN-FGT1-FGT2 |
| GUI → Network → Static Routes (FGT2) | Ruta a LAN1 por VPN-FGT2-FGT1 |
| GUI → VPN → IPSec Tunnels (FGT1) | Túnel `VPN-FGT1-FGT2` UP con tráfico |
| GUI → VPN → IPSec Tunnels (FGT2) | Túnel `VPN-FGT2-FGT1` UP con tráfico |
| GUI → Monitor → IPSec Monitor (FGT1) | Túnel activo con bytes in/out |
| GUI → Monitor → IPSec Monitor (FGT2) | Túnel activo con bytes in/out |
| `ping 10.7.25.98` desde VPC1-LAN1 | ✅ Exitoso |
| `trace 10.7.25.98` desde VPC1-LAN1 | ✅ Llega a destino |
| `ping 10.7.25.66` desde VPC2-LAN2 | ✅ Exitoso |
| `trace 10.7.25.66` desde VPC2-LAN2 | ✅ Llega a destino |

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Interfaces ISP (Ethernet0/0 y 0/1) | ✅ up/up |
| Interfaces FGT1 (port2 LAN1, port3 WAN) | ✅ Activas en GUI |
| Interfaces FGT2 (port2 LAN2, port3 WAN) | ✅ Activas en GUI |
| Rutas estáticas FGT1 (default + LAN2 + host FGT2) | ✅ Configuradas |
| Rutas estáticas FGT2 (default + LAN1 + host FGT1) | ✅ Configuradas |
| Túnel IPSec FGT1 (`VPN-FGT1-FGT2`) | ✅ UP con tráfico |
| Túnel IPSec FGT2 (`VPN-FGT2-FGT1`) | ✅ UP con tráfico |
| Configuración Phase 1 y Phase 2 en FGT1 | ✅ Correcta |
| Configuración Phase 1 y Phase 2 en FGT2 | ✅ Correcta |
| Políticas firewall FGT1 (LAN1↔VPN, NAT off) | ✅ Configuradas |
| Políticas firewall FGT2 (LAN2↔VPN, NAT off) | ✅ Configuradas |
| Monitor IPSec FGT1 y FGT2 | ✅ Bytes in/out activos |
| Ping / Trace VPC1-LAN1 → VPC2-LAN2 | ✅ Exitoso |
| Ping / Trace VPC2-LAN2 → VPC1-LAN1 | ✅ Exitoso |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_FortiGate_Site_to_Site_LAN_to_LAN.txt`](SaelGerman_2025-0725_Script_FortiGate_Site_to_Site_LAN_to_LAN.txt) | Scripts de configuración FortiOS CLI e ISP |
| [`SaelGerman_2025-0725_FortiGate-Site-to-Site_P2.pdf`](SaelGerman_2025-0725_FortiGate-Site-to-Site_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología FortiGate VPN Site-to-Site LAN-to-LAN](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/01_topologia_fortigate_site_to_site_lan_to_lan.png)
- 📸 [Figura 2 — Rutas estáticas en FGT1](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/02_fgt1_static_routes_gui.png)
- 📸 [Figura 3 — Rutas estáticas en FGT2](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/03_fgt2_static_routes_gui.png)
- 📸 [Figura 4 — Topología e interfaces del ISP](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/04_topologia_e_isp_show_ip_interface_brief.png)
- 📸 [Figura 5 — Interfaces de FGT1 desde la GUI](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/05_fgt1_interfaces_gui.png)
- 📸 [Figura 6 — Interfaces de FGT2 desde la GUI](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/06_fgt2_interfaces_gui.png)
- 📸 [Figura 7 — Túnel IPSec en FGT1 en estado UP](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/07_fgt1_ipsec_tunnel_up_gui.png)
- 📸 [Figura 8 — Túnel IPSec en FGT2 en estado UP](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/08_fgt2_ipsec_tunnel_up_gui.png)
- 📸 [Figura 9 — Configuración Phase 1 y Phase 2 en FGT1](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/09_fgt1_ipsec_tunnel_edit_phase1_phase2.png)
- 📸 [Figura 10 — Configuración Phase 1 y Phase 2 en FGT2](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/10_fgt2_ipsec_tunnel_edit_phase1_phase2.png)
- 📸 [Figura 11 — Políticas firewall en FGT1 (NAT desactivado)](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/11_fgt1_firewall_policies_gui.png)
- 📸 [Figura 12 — Políticas firewall en FGT2 (NAT desactivado)](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/12_fgt2_firewall_policies_gui.png)
- 📸 [Figura 13 — Monitor IPSec en FGT1 (tráfico activo)](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/13_fgt1_ipsec_monitor_gui.png)
- 📸 [Figura 14 — Monitor IPSec en FGT2 (tráfico activo)](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/14_fgt2_ipsec_monitor_gui.png)
- 📸 [Figura 15 — Prueba desde VPC1-LAN1 hacia VPC2-LAN2](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/15_vpc1_ping_trace_to_vpc2.png)
- 📸 [Figura 16 — Prueba desde VPC2-LAN2 hacia VPC1-LAN1](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/16_vpc2_ping_trace_to_vpc1.png)
- 📸 [Figura 17 — Evidencia adicional FortiGate](SaelGerman_2025-0725_Capturas_FortiGate_Site_to_Site_LAN_to_LAN_GitHub/17_evidencia_extra_fortigate.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_FortiGate-Site-to-Site_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/HEN8Y3uyZ4s)

---

## 📚 Referencias

1. Fortinet. *FortiGate IPSec VPN Administration Guide*. Documentación oficial FortiOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
