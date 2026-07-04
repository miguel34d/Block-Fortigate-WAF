# Lab P11 — Configuración FortiGate

**Curso:** Seguridad de Redes (TSI-203)
**Estudiante:** Miguel Ramírez Meli — Matrícula 2025-1367
**Institución:** Instituto Tecnológico de las Américas (ITLA)

---

## Tabla de contenido

1. [Requisitos de la topología](#1-requisitos-de-la-topología)
2. [Direccionamiento](#2-direccionamiento)
3. [Interfaces](#3-interfaces)
4. [Ruta estática](#4-ruta-estática)
5. [Addresses](#5-addresses)
6. [VIP del servidor Web](#6-vip-del-servidor-web)
7. [NAT (salida a Internet)](#7-nat-salida-a-internet)
8. [Application Control](#8-application-control-redes-sociales--whatsapp)
9. [DNS Filter](#9-dns-filter-itlaedudo--doh)
10. [Detección de escáneres — IPv4 DoS Policy](#10-detección-de-escáneres--ipv4-dos-policy)
11. [Web Application Firewall](#11-web-application-firewall)
12. [Web Filter](#12-web-filter-bloquear-http-hacia-internet)
13. [Políticas de Firewall](#13-políticas-de-firewall)
14. [Verificación rápida](#14-verificación-rápida)

---

## 1. Requisitos de la topología

| # | Requisito | Estado |
|---|---|:---:|
| 1 | Toda la configuración por GUI | ✅ |
| 2 | Acceso a Internet | ✅ |
| 3 | LAN de usuarios (/25) | ✅ |
| 4 | LAN de servidores (/28) | ✅ |
| 5 | IP en interfaces | ✅ |
| 6 | DHCP en LAN de usuarios | ✅ |
| 7 | Ruta por defecto | ✅ |
| 8 | NAT | ✅ |
| 9 | Solo tráfico HTTP de LAN usuarios → LAN servidores, bloquear el resto | ✅ |
| 10 | Bloquear redes sociales | ✅ |
| 11 | Bloquear llamadas por WhatsApp | ✅ |
| 12 | Bloquear dominios y subdominios de `itla.edu.do` (incluyendo bypass por DoH) | ✅ |
| 13 | Detectar y bloquear escáneres de red | ✅ |
| 14 | Aplicar WAF al servidor Web | ✅ |

---

## 2. Direccionamiento

| Segmento | Red | IP |
|---|---|---|
| Router1 ↔ FortiGate | `200.13.67.0/30` | Port1: `200.13.67.2` |
| LAN Usuarios | `10.13.67.0/25` | Port2: `10.13.67.1` |
| LAN Servidores | `10.13.67.128/28` | Port3: `10.13.67.129` · Web: `10.13.67.130` |

---

## 3. Interfaces

📍 `Network → Interfaces → Edit`

| Interfaz | Rol | Modo | IP / Máscara | Acceso administrativo |
|---|---|---|---|---|
| **Port1** | WAN | Manual | `200.13.67.2 / 255.255.255.252` | HTTPS, HTTP, PING |
| **Port2** | LAN | Manual | `10.13.67.1 / 255.255.255.128` | HTTPS, PING, SSH |
| **Port3** | LAN | Manual | `10.13.67.129 / 255.255.255.240` | HTTPS, PING, SSH |

**DHCP en Port2:**
- Estado: Enable
- Rango: `10.13.67.2` – `10.13.67.126`
- Gateway: `10.13.67.1`

---

## 4. Ruta estática

📍 `Network → Static Routes → Create New`

| Campo | Valor |
|---|---|
| Destination | `0.0.0.0/0.0.0.0` |
| Gateway | `200.13.67.1` |
| Interface | `port1` |

---

## 5. Addresses

📍 `Policy & Objects → Addresses → Create New`

| Nombre | Subnet |
|---|---|
| `LAN_USUARIOS` | `10.13.67.0/25` |
| `LAN_SERVIDORES` | `10.13.67.128/28` |
| `WEB_SERVER` | `10.13.67.130/32` (interface: `port3`) |

---

## 6. VIP del servidor Web

📍 `Policy & Objects → Virtual IPs → Create New`

| Campo | Valor |
|---|---|
| Name | `VIP_WEB` |
| Interface | `port1` |
| External IP | `200.13.67.2` |
| Mapped IP | `10.13.67.130` |
| Port Forwarding | Enable — TCP `80 → 80` |

> ⚠️ **Aviso:** al activar el VIP en el puerto 80, la GUI por HTTP se tapa. Cambia el puerto de administración por CLI:
> ```
> config system global
>     set admin-port 8080
> end
> ```
> Entra por `http://200.13.67.2:8080`.

---

## 7. NAT (salida a Internet)

El NAT permite que el tráfico de `LAN_USUARIOS` y `LAN_SERVIDORES` salga a Internet traducido a la IP pública de `port1` (`200.13.67.2`).

📍 Se configura directamente dentro de la **Policy 1 — LANs-a-Internet** (ver sección 13), en la pestaña de la política:

| Campo | Valor |
|---|---|
| NAT | **Enable** |
| IP Pool Configuration | **Use Outgoing Interface Address** |

> ✅ Toda la configuración del NAT se hace por GUI, dentro de la misma Policy 1 (no requiere CLI). Los comandos que siguen abajo son **solo para verificar** que el NAT esté traduciendo el tráfico correctamente.

**Verificación del NAT en tiempo real (CLI)** — mientras se genera tráfico desde un cliente:
```
diagnose sys session filter clear
diagnose sys session filter src 10.13.67.10
diagnose sys session list
```

Salida esperada — confirma la traducción de origen (`act=snat`) y destino en la respuesta (`act=dnat`):
```
hook=post dir=org act=snat 10.13.67.10:49931->8.8.8.8:53(200.13.67.2:49931)
hook=pre dir=reply act=dnat 8.8.8.8:53->200.13.67.2:49931(10.13.67.10:49931)
```

✅ **Estado: verificado y funcionando.** El `policy_id=1` en las sesiones capturadas confirma que la traducción la aplica la Policy 1.

> 💡 `diagnose sys session list` es solo lectura en memoria — no genera logs permanentes, se puede correr las veces que sea necesario sin impacto.

---

## 8. Application Control (redes sociales + WhatsApp)

📍 `Security Profiles → Application Control → Create New` → **`APPSBLOCKEOS`**

| Sección | Acción |
|---|---|
| Categories → `Social Media` | **Block** |
| Application and Filter Overrides → `+` → buscar `wha` → `WhatsApp_VoIP.Call` | **Block** |

---

## 9. DNS Filter (itla.edu.do + DoH)

📍 `Security Profiles → DNS Filter → Create New` → **`DNSFilter_ITLA`**

Static Domain Filter → activar toggle → `+ Create New` para cada uno:

| Domain | Type | Action |
|---|---|---|
| `*.itla.edu.do` | Wildcard | Redirect to Block Portal |
| `itla.edu.do` | Simple | Redirect to Block Portal |
| `cloudflare-dns.com` | Simple | Redirect to Block Portal |
| `*.cloudflare-dns.com` | Wildcard | Redirect to Block Portal |
| `dns.google` | Simple | Redirect to Block Portal |
| `mozilla.cloudflare-dns.com` | Simple | Redirect to Block Portal |

**Options:**
- ✅ Allow DNS requests when a rating error occurs

---

## 10. Detección de escáneres — IPv4 DoS Policy

📍 `System → Feature Visibility` → activar **IPv4 DoS Policy**

📍 `Policy & Objects → IPv4 DoS Policy → Create New`

| Campo | Valor |
|---|---|
| Source | `all` |
| Destination | `all` |
| Service | `ALL` |
| Enable this policy | ON |

**L4 Anomalies:**

| Anomalía | Logging | Action | Threshold |
|---|---|---|---|
| `tcp_port_scan` | ON | Block | 5 |
| `icmp_sweep` | ON | Block | 5 |

Repetir para las 3 interfaces, o aplicar por CLI:

```
config firewall DoS-policy
    edit 1
        set name "DoS_AntiScan"
        set interface "port2"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        config anomaly
            edit "tcp_port_scan"
                set status enable
                set log enable
                set action block
                set threshold 5
            next
            edit "icmp_sweep"
                set status enable
                set log enable
                set action block
                set threshold 5
            next
        end
    next
    edit 2
        set name "DoS_AntiScan_WAN"
        set interface "port1"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        config anomaly
            edit "tcp_port_scan"
                set status enable
                set log enable
                set action block
                set threshold 5
            next
            edit "icmp_sweep"
                set status enable
                set log enable
                set action block
                set threshold 5
            next
        end
    next
    edit 3
        set name "DoS_AntiScan_SRV"
        set interface "port3"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        config anomaly
            edit "tcp_port_scan"
                set status enable
                set log enable
                set action block
                set threshold 5
            next
            edit "icmp_sweep"
                set status enable
                set log enable
                set action block
                set threshold 5
            next
        end
    next
end
```

> ⚠️ **Caso especial — mismo switch/VLAN (Kali y Windows10):** el DoS Policy no detecta tráfico intra-VLAN. Solución en el switch:
> ```
> enable
> configure terminal
> interface e0/2
>  switchport protected
>  exit
> interface e0/1
>  switchport protected
>  exit
> end
> write memory
> ```
> *(dejar `e0/0`, el uplink, sin tocar)*

---

## 11. Web Application Firewall

📍 `System → Feature Visibility` → activar **Web Application Firewall**

📍 `Security Profiles → Web Application Firewall → Create New` → **`WAF_WebServer`**

Cambiar a **Block**:
- Cross Site Scripting
- SQL Injection
- Generic Attacks
- Trojans

---

## 12. Web Filter (bloquear HTTP hacia Internet)

📍 `Security Profiles → Web Filter → Create New` → **`WebFilter_BlockHTTP`**

| Campo | Valor |
|---|---|
| Feature set | Proxy-based |
| Static URL Filter | Activar toggle |
| Entrada | URL `*` · Type Wildcard · Action **Block** |

---

## 13. Políticas de Firewall

📍 `Policy & Objects → Firewall Policy → Create New`

### Policy 1 — LANs-a-Internet

| Campo | Valor |
|---|---|
| Incoming | `any` |
| Outgoing | `port1` |
| Source | `LAN_USUARIOS`, `LAN_SERVIDORES` |
| Destination | `all` |
| Service | `HTTP`, `HTTPS`, `DNS` |
| Inspection mode | Proxy-based |
| NAT | **Enable** (Use Outgoing Interface Address) |
| Security Profiles | Web Filter `WebFilter_BlockHTTP` · DNS Filter `DNSFilter_ITLA` · App Control `APPSBLOCKEOS` |
| SSL Inspection | `no-inspection` |

### Policy 2 — Usuarios-a-Servidores-HTTP

| Campo | Valor |
|---|---|
| Incoming | `port2` |
| Outgoing | `port3` |
| Source | `LAN_USUARIOS` |
| Destination | `WEB_SERVER` |
| Service | **solo HTTP** |
| Enable this policy | **ON** ⚠️ *(verificar al final, se desactiva fácil)* |

### Policy 3 — Internet-a-Web

| Campo | Valor |
|---|---|
| Incoming | `port1` |
| Outgoing | `port3` |
| Source | `all` |
| Destination | `VIP_WEB` |
| Service | HTTP |
| Inspection mode | Proxy-based |
| Web Application Firewall | `WAF_WebServer` |

---

## 14. Verificación rápida

| Prueba | Dónde | Resultado esperado |
|---|---|---|
| `https://sitio.com` | Windows10 | Carga normal |
| `http://neverssl.com` | Windows10 | Página de bloqueo Fortinet |
| `facebook.com` | Windows10 | Bloqueado (App Control) |
| `itla.edu.do` | Windows10 | Bloqueado (DNS Filter) |
| `http://10.13.67.130` | Windows10 | Carga (Policy 2) |
| `nmap -sS 10.13.67.130` | Kali | Bloqueado |
| `nmap -sS 200.13.67.2` | Kali | Muy lento / bloqueado |
| `nmap -sS 10.13.67.10` | Kali | Bloqueado solo si se aplicó `switchport protected` |
| `curl` con SQLi/XSS a `200.13.67.2` | Fuera del FortiGate | Bloqueado (WAF) |
| Sesión NAT saliente | FortiGate CLI | `act=snat` traduce a `200.13.67.2` |

**Logs:** `Log & Report → Security Events` — filtrar por `tcp_port_scan`, `itla`, `cloudflare` según la prueba.

---

## Evidencia

![Evidencia 1](https://github.com/user-attachments/assets/c4d5bc10-7937-4c15-8e5d-9fd930650c30)

![Evidencia 2](https://github.com/user-attachments/assets/009e7f8c-2930-402f-b8f7-7ba08f4c423e)
