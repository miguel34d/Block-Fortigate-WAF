# Lab P11 — FortiGate (Guía rápida)

## Requisitos de la topología

- ✅ Toda configuración por GUI
- ✅ Acceso a Internet
- ✅ LAN de usuarios (/25)
- ✅ LAN de servidores (/28)
- ✅ IP en interfaces
- ✅ DHCP en LAN de usuarios
- ✅ Ruta por defecto
- ✅ NAT
- ✅ Solo tráfico HTTP de LAN usuarios → LAN servidores, bloquear el resto
- ✅ Bloquear redes sociales
- ✅ Bloquear llamadas por WhatsApp
- ✅ Bloquear dominios y subdominios de itla.edu.do (incluyendo bypass por DNS-over-HTTPS)
- ✅ Detectar y bloquear escáneres de red
- ✅ Aplicar WAF al servidor Web

---

## Direccionamiento
| Segmento | Red | IP |
|---|---|---|
| Router1↔FortiGate | 200.13.67.0/30 | Port1: 200.13.67.2 |
| LAN Usuarios | 10.13.67.0/25 | Port2: 10.13.67.1 |
| LAN Servidores | 10.13.67.128/28 | Port3: 10.13.67.129 / Web: 10.13.67.130 |

### Topología de red

![Topología FortiGate](./topologia.png)

*(coloca aquí tu imagen del diagrama, o reemplaza `./topologia.png` por la ruta/nombre real del archivo)*

---

## 1. Interfaces
`Network → Interfaces → Edit`

**Port1:** Role WAN, Manual, IP `200.13.67.2/255.255.255.252`, Access: HTTPS/HTTP/PING

**Port2:** Role LAN, Manual, IP `10.13.67.1/255.255.255.128`, Access: HTTPS/PING/SSH
- DHCP: Enable → Rango `10.13.67.2-126`, GW `10.13.67.1`

**Port3:** Role LAN, Manual, IP `10.13.67.129/255.255.255.240`, Access: HTTPS/PING/SSH

---

## 2. Ruta estática
`Network → Static Routes → Create New`
- Destination `0.0.0.0/0.0.0.0` · Gateway `200.13.67.1` · Interface `port1`

---

## 3. Addresses
`Policy & Objects → Addresses → Create New`

| Nombre | Subnet |
|---|---|
| LAN_USUARIOS | 10.13.67.0/25 |
| LAN_SERVIDORES | 10.13.67.128/28 |
| WEB_SERVER | 10.13.67.130/32 (interface: port3) |

---

## 4. VIP del servidor Web
`Policy & Objects → Virtual IPs → Create New`
- Name `VIP_WEB` · Interface `port1` · External IP `200.13.67.2` · Mapped IP `10.13.67.130` · Port Forwarding Enable · TCP 80→80

⚠️ Después de esto, la GUI por HTTP se tapa. Cambia el puerto admin:
```
config system global
    set admin-port 8080
end
```
Entra por `http://200.13.67.2:8080`

---

## 5. Application Control (redes sociales + WhatsApp)
`Security Profiles → Application Control → Create New` → `APPSBLOCKEOS`
- Categories → `Social Media` → **Block**
- Application and Filter Overrides → `+` → buscar `wha` → `WhatsApp_VoIP.Call` → **Block**

---

## 6. DNS Filter (itla.edu.do + DoH)
`Security Profiles → DNS Filter → Create New` → `DNSFilter_ITLA`
- Static Domain Filter → activar toggle → `+ Create New` para cada uno:

| Domain | Type | Action |
|---|---|---|
| `*.itla.edu.do` | Wildcard | Redirect to Block Portal |
| `itla.edu.do` | Simple | Redirect to Block Portal |
| `cloudflare-dns.com` | Simple | Redirect to Block Portal |
| `*.cloudflare-dns.com` | Wildcard | Redirect to Block Portal |
| `dns.google` | Simple | Redirect to Block Portal |
| `mozilla.cloudflare-dns.com` | Simple | Redirect to Block Portal |

- Options → activar **"Allow DNS requests when a rating error occurs"** + **"Log all DNS queries"**

---

## 7. Detectar escáneres — IPv4 DoS Policy
`System → Feature Visibility` → activa **IPv4 DoS Policy**

`Policy & Objects → IPv4 DoS Policy → Create New`
- Source `all` · Destination `all` · Service `ALL`
- En **L4 Anomalies**:
  - `tcp_port_scan` → Logging ON, Action **Block**, Threshold `5`
  - `icmp_sweep` → Logging ON, Action **Block**, Threshold `5`
- Enable this policy: ON

Repite para las 3 interfaces (o pega por CLI cambiando `interface`):

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

**Caso especial — mismo switch/VLAN (Kali y Windows10):** el DoS Policy no lo ve. Fix en el switch:
```
enable
configure terminal
interface e0/2
 switchport protected
 exit
interface e0/1
 switchport protected
 exit
end
write memory
```
(deja `e0/0`, el uplink, sin tocar)

---

## 8. Web Application Firewall
`System → Feature Visibility` → activa **Web Application Firewall**

`Security Profiles → Web Application Firewall → Create New` → `WAF_WebServer`
- Cambiar a **Block**: `Cross Site Scripting`, `SQL Injection`, `Generic Attacks`, `Trojans`

---

## 9. Web Filter (bloquear HTTP hacia Internet)
`Security Profiles → Web Filter → Create New` → `WebFilter_BlockHTTP`
- Feature set: Proxy-based
- Static URL Filter → activar toggle → `+ Create New`: URL `*` · Type Wildcard · Action **Block**

---

## 10. Políticas de Firewall (máx. 3)
`Policy & Objects → Firewall Policy → Create New`

### Policy 1 — LANs-a-Internet
- Incoming `any` · Outgoing `port1` · Source `LAN_USUARIOS`+`LAN_SERVIDORES` · Dest `all`
- Service: `HTTP`, `HTTPS`, `DNS`
- Inspection mode: **Proxy-based** · NAT: Enable
- Security Profiles: Web filter `WebFilter_BlockHTTP` · DNS filter `DNSFilter_ITLA` · App control `APPSBLOCKEOS`
- SSL inspection: `no-inspection`

### Policy 2 — Usuarios-a-Servidores-HTTP
- Incoming `port2` · Outgoing `port3` · Source `LAN_USUARIOS` · Dest `WEB_SERVER`
- Service: **solo HTTP**
- Enable this policy: **ON** (checar al final, se apaga fácil)

### Policy 3 — Internet-a-Web
- Incoming `port1` · Outgoing `port3` · Source `all` · Dest `VIP_WEB`
- Service: HTTP · Inspection: **Proxy-based**
- Web Application Firewall: `WAF_WebServer`

---

## 11. Verificación rápida

| Prueba | Dónde | Resultado esperado |
|---|---|---|
| `https://sitio.com` | Windows10 | Carga normal |
| `http://neverssl.com` | Windows10 | Página de bloqueo Fortinet |
| `facebook.com` | Windows10 | Bloqueado (App Control) |
| `itla.edu.do` | Windows10 | Bloqueado (DNS Filter) |
| `http://10.13.67.130` | Windows10 | Carga (Policy 2) |
| `nmap -sS 10.13.67.130` | Kali | Bloqueado |
| `nmap -sS 200.13.67.2` | Kali | Muy lento / bloqueado |
| `nmap -sS 10.13.67.10` | Kali | Bloqueado solo si aplicaste switchport protected |
| `curl` con SQLi/XSS a `200.13.67.2` | Fuera del FortiGate | Bloqueado (WAF) |

**Logs:** `Log & Report → Security Events` — filtra por `tcp_port_scan`, `itla`, `cloudflare` según la prueba.


- <img width="773" height="894" alt="image" src="https://github.com/user-attachments/assets/c4d5bc10-7937-4c15-8e5d-9fd930650c30" />
- <img width="774" height="857" alt="image" src="https://github.com/user-attachments/assets/009e7f8c-2930-402f-b8f7-7ba08f4c423e" />

