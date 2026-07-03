# Lab P11 — Topología FortiGate (Client-to-Site VPN)

## Configuración 100% GUI — FortiOS 7.6.7

---

## 1. Requisitos de la topología (checklist)

- [x] Toda configuración por GUI
- [x] Acceso a Internet
- [x] LAN de usuarios (/25)
- [x] LAN de servidores (/28)
- [x] IP en interfaces
- [x] DHCP en LAN de usuarios
- [x] Ruta por defecto
- [x] NAT
- [x] Solo tráfico HTTP de LAN usuarios → LAN servidores, bloquear el resto
- [x] Bloquear redes sociales
- [x] Bloquear llamadas por WhatsApp
- [x] Bloquear dominios y subdominios de itla.edu.do
- [x] Detectar y bloquear escáneres de red
- [x] Aplicar WAF al servidor Web

---

## 2. Direccionamiento

| Segmento | Red | IP de host (interfaz) |
|---|---|---|
| WAN (Cloud1) | 10.10.10.0/24 | Router1 e0/1: 10.10.10.2 |
| Router1 ↔ FortiGate | 200.13.67.0/30 | Router1 e0/0: 200.13.67.1 / FortiGate Port1: 200.13.67.2 |
| LAN Usuarios (VLAN10) | 10.13.67.0/**25** | FortiGate Port2: 10.13.67.1 |
| LAN Servidores (VLAN20) | 10.13.67.128/**28** | FortiGate Port3: 10.13.67.129 / Web: 10.13.67.130 |

> **Regla clave para evitar "Invalid IP Netmask":** el campo IP/Netmask de una interfaz siempre lleva la IP de **host**, nunca la dirección de red (`10.13.67.0`, `10.13.67.128`, `200.13.67.0`). La subred completa solo se usa en objetos de dirección (Policy & Objects → Addresses).

---

## 3. Paso 1 — Interfaces (Network → Interfaces)

Clic en el nombre del puerto → **Edit Interface**.

### 3.1 Port1 (WAN)
- Role: WAN
- Addressing mode: Manual
- IP/Netmask: `200.13.67.2/255.255.255.252`
- Administrative Access: `HTTPS`, `HTTP` (respaldo, ver nota abajo), `PING`
- OK

### 3.2 Port2 (LAN Usuarios)
- Role: LAN
- Addressing mode: Manual
- IP/Netmask: `10.13.67.1/255.255.255.128`
- Administrative Access: `HTTPS`, `PING`, `SSH`
- **DHCP Server: Enable** (toggle azul arriba de "Network")
  - Address Range: `10.13.67.2` – `10.13.67.126`
  - Netmask: `255.255.255.128`
  - Default Gateway: `10.13.67.1` (Same as Interface IP)
  - DNS Server: Same as System DNS, o manual `8.8.8.8`
  - Lease Time: default (7 días) está bien para lab
- OK

### 3.3 Port3 (LAN Servidores)
- Role: LAN
- Addressing mode: Manual
- IP/Netmask: `10.13.67.129/255.255.255.240`
- Administrative Access: `HTTPS`, `PING`, `SSH`
- DHCP Server: **no** se activa aquí (el servidor Web usa IP fija)
- OK

> Nota HTTPS: en licencia evaluación FGVMEV, el HTTPS de gestión suele fallar aunque esté marcado. Si no carga, entra por `http://200.13.67.2`.

---

## 4. Paso 2 — Ruta estática (Network → Static Routes)

**Create New**:
- Destination: `0.0.0.0/0.0.0.0`
- Gateway: `200.13.67.1`
- Interface: `port1`
- Status: Enable
- OK

---

## 5. Paso 3 — Objetos de dirección (Policy & Objects → Addresses)

**Create New → Address**:

**LAN_USUARIOS**
- Type: Subnet
- Subnet/IP Range: `10.13.67.0/25`
- Interface: any
- OK

**LAN_SERVIDORES**
- Type: Subnet
- Subnet/IP Range: `10.13.67.128/28`
- Interface: any
- OK

**WEB_SERVER** (host específico, para la política de servicios web y WAF)
- Type: Subnet
- Subnet/IP Range: `10.13.67.130/32`
- Interface: port3
- OK

---

## 6. Paso 4 — VIP para publicar el servidor Web

**Policy & Objects → Virtual IPs → Create New → Virtual IP**
(crear ANTES de la política que lo usa)

- Name: `VIP_WEB`
- Interface: `port1`
- External IP/Range: `200.13.67.2`
- Mapped IP/Range: `10.13.67.130`
- Port Forwarding: Enable
- Protocol: TCP
- External Service Port: `80`
- Map to Port: `80`
- OK

> ⚠️ **Conflicto de puerto con la GUI de administración:** el VIP usa la misma IP (`200.13.67.2`) y puerto (`80`) que el acceso HTTP de management del FortiGate. Una vez creada la Policy 3 con este VIP, entrar a `http://200.13.67.2` ya no muestra la GUI del FortiGate — la Policy 3 hace DNAT y te lleva directo al servidor Web (`10.13.67.130`). Para seguir administrando el FortiGate por HTTP, cambia el puerto de management **antes o justo después** de crear el VIP:
> ```
> config system global
>     set admin-port 8080
> end
> ```
> Luego entra por `http://200.13.67.2:8080`. El puerto HTTPS de management (443) no tiene este conflicto porque el VIP solo mapea el 80.

---

## 7. Paso 5 — Perfiles de seguridad (crear ANTES de las políticas)

### 7.1 Application Control — bloquear redes sociales y llamadas WhatsApp

**Security Profiles → Application Control → Create New**

- Name: `APPSBLOCKEOS`

**a) Bloquear redes sociales (categoría completa):**
- En **Categories**, localiza `Social Media (150, ☁31)`
- Clic en el ícono ✓/Allow junto a la categoría → cambia el dropdown a **Block**
- Todas las demás categorías (`Business`, `Cloud/IT`, `Collaboration`, `Email`, `Game`, `General Interest`, `Mobile`, `Network Service`, `P2P`, `Proxy`, `Remote Access`, `Storage/Backup`, `Update`, `Video/Audio`, `VoIP`, `Web Client`, `Unknown Applications`) se dejan en **Allow** (default) — no se tocan

**b) Bloquear llamadas de WhatsApp (override específico, no la categoría completa):**
WhatsApp está bajo la categoría **Collaboration (293, ☁6)**. Bloquear toda la categoría afectaría otras apps legítimas (Zoom, Teams, correo, etc.), así que aquí se usa un override puntual:

- Baja hasta **Application and Filter Overrides**
- Clic **+ Create New**
- Type: **Application**
- En el buscador escribe `wha` — selecciona:
  - `WhatsApp_VoIP.Call` ← firma de llamadas
- Action: **Block**
- OK — queda como Priority 1, Details `WhatsApp_VoIP.Call`, Type `Application`, Action `Block`
- `Collaboration` en el bloque de categorías de arriba se deja en **Allow** (sin tocar)
- OK (para guardar el perfil completo)
- El resto de categorías: Monitor o Allow según necesites
- OK (para guardar el perfil completo)

### 7.2 DNS Filter — bloquear itla.edu.do y subdominios

**Security Profiles → DNS Filter → Create New**

- Name: `DNSFilter_ITLA`
- Redirect botnet C&C requests to Block Portal: apagado
- Enforce 'Safe Search': apagado
- FortiGuard Category Based Filter: no tocar, dejar en Monitor

**Bloquear el dominio:**
- Sección **Static Domain Filter** → activa el toggle `Domain Filter`
- **+ Create New**:
  - Type: `Wildcard`
  - Domain: `*.itla.edu.do`
  - Action: `Block`
  - OK
- **+ Create New** otra vez:
  - Type: `Simple`
  - Domain: `itla.edu.do`
  - Action: `Block`
  - OK
- `External IP Block Lists` y `DNS Translation`: apagados

**Options** (abajo):
- Redirect Portal IP: `Use FortiGuard Default`
- Allow DNS requests when a rating error occurs: apagado
- Log all DNS queries and responses: opcional, actívalo para evidencia
- Strip Encrypted Client Hello: no tocar (viene activado por defecto)
- **OK** (guarda el perfil completo)

### 7.3 IPS — detectar y bloquear escáneres de red

**Security Profiles → Intrusion Prevention → Create New**

- Name: `IPS_AntiScan`
- Comments: `Deteccion y bloqueo de escaneo de red`
- Block malicious URLs: activado

**Entrada 1 (firma específica):**
- IPS Signatures and Filters → **+ Create New**
- Type: `Signature`
- Buscador: escribe `port.scan` → marca `FTP.Bounce.Port.Scan`
- Action: `Block`
- OK

**Entrada 2 (filtro por severidad):**
- **+ Create New** otra vez
- Type: `Filter`
- Action: `Block`
- Packet logging: `Enable`
- Status: `Enable`
- Filter → `+` → Severity → marca `Medium`, `High`, `Critical`
- OK

**Botnet C&C:**
- Scan Outgoing Connections to Botnet Sites: `Block`
- **OK** (guarda el sensor completo)

> Nota: el buscador de firmas requiere texto para mostrar resultados (deja el campo vacío y sale "No results"). La base de firmas de esta licencia eval es de 2015, por lo que no existen nombres modernos como `Nmap.Scan` — se usa `FTP.Bounce.Port.Scan` + filtro por severidad como cobertura equivalente.

### 7.4 Web Application Firewall — proteger el servidor Web

**Si no aparece "Web Application Firewall" en el menú de Security Profiles, actívalo primero:**
```
System → Feature Visibility → activa Web Application Firewall → Apply
```

**Security Profiles → Web Application Firewall → Create New**

- Name: `WAF_WebServer`

**En la tabla de Signatures, cambia Action a `Block` en estas filas** (vienen en Allow por defecto, click en el ícono ✓ Allow de cada una):
- `Cross Site Scripting`
- `SQL Injection`
- `Generic Attacks`
- `Trojans`

El resto (`Information Disclosure`, las versiones "(Extended)", `Known Exploits`) se dejan como están.

- Constraint Exceptions: no tocar (default)
- HTTP Method Policy: dejar apagado (Enforce HTTP Method Policy)
- **OK**

> El WAF requiere que la política asociada esté en modo de **inspección proxy-based**, no flow-based. Verifícalo en la política del VIP (paso 8, Policy 3) — si la política está en Flow-based, el campo WAF ni aparece para seleccionarlo.

---

## 8. Paso 6 — Políticas de firewall (Policy & Objects → Firewall Policy)

Límite de licencia evaluación: **máximo 3 políticas por VDOM**. Los perfiles de seguridad se aplican dentro de estas mismas 3 políticas, no como políticas adicionales.

> **Nota sobre Incoming interface con múltiples valores:** en esta build, el campo Incoming interface de una policy normal solo admite **una** interfaz salvo que actives `System → Feature Visibility → Multiple Interface Policies`. Si no la activas (o no está disponible en licencia eval), usa **`any`** como Incoming interface y deja que el filtrado real lo hagan los objetos de **Source** (`LAN_USUARIOS`, `LAN_SERVIDORES`) — el efecto es equivalente para este lab.

### 8.1 Policy 1 — LANs-a-Internet (con bloqueos de apps/DNS/IPS)

**Create New Policy:**
- Name: `LANs-a-Internet`
- Schedule: `always`
- Action: `ACCEPT`
- Incoming interface: `any` (ver nota arriba si tu build no permite multi-selección)
- Outgoing interface: tu interfaz WAN (alias tipo `ISP_INTERNET` / port1)
- Source: `+` → `LAN_USUARIOS` y `LAN_SERVIDORES`
- Destination: `+` → `all`
- Service: `+` → `ALL`

**Firewall/Network Options:**
- Inspection mode: cambia de `Flow-based` a **`Proxy-based`** (activa las opciones de Security Profiles)
- NAT: `Enable` (déjalo activado)
- IP pool configuration: `Use Outgoing Interface Address`

**Security Profiles** (activa el toggle de cada uno y selecciona el perfil):
- DNS filter → `DNSFilter_ITLA`
- Application control → `APPSBLOCKEOS`
- IPS → `IPS_AntiScan`
- AntiVirus, Web filter, Video filter, File filter, Web application firewall: apagados (no van en esta policy)
- SSL inspection: dejar en `no-inspection` (el ⚠️ es solo informativo, ignóralo)

**Logging Options:**
- Log allowed traffic: `Security events`

- **OK** (guarda la Policy 1 completa)

### 8.2 Policy 2 — Usuarios-a-Servidores (solo HTTP, resto bloqueado)
- Name: `Usuarios-a-Servidores-HTTP`
- Incoming Interface: interfaz de tu LAN Usuarios
- Outgoing Interface: interfaz de tu LAN Servidores
- Source: `LAN_USUARIOS`
- Destination: `WEB_SERVER` (no uses `LAN_SERVIDORES` completo, solo el host web, para no abrir de más)
- Schedule: always
- **Service: `HTTP` únicamente** (quita `ALL`, agrega solo `HTTP`)
- Action: ACCEPT
- OK

> Con esto, cualquier otro tráfico de LAN_USUARIOS hacia LAN_SERVIDORES que no sea HTTP hacia WEB_SERVER queda bloqueado automáticamente por el **implicit deny** del FortiGate (no hace falta una política explícita de bloqueo — el motor deniega todo lo que no matchea ninguna regla ACCEPT).

### 8.3 Policy 3 — Internet-a-Web (con WAF)
- Name: `Internet-a-Web`
- Incoming Interface: interfaz WAN (port1)
- Outgoing Interface: interfaz LAN Servidores (port3)
- Source: `all`
- Destination: `VIP_WEB`
- Schedule: always
- Service: HTTP
- Action: ACCEPT
- Inspection Mode: **Proxy-based**
- **Web Application Firewall: `WAF_WebServer`**
- OK

---

## 9. Verificación

- **Dashboard → Network**: las 3 interfaces en estado "up".
- **Monitor → DHCP Monitor**: confirma que Windows10 recibió IP por DHCP en el rango `10.13.67.2–126`.
- **Policy & Objects → Firewall Policy**: las 3 reglas visibles, en orden 1→2→3, con los íconos de los perfiles de seguridad junto a cada una.
- Desde Windows10:
  - `ping 8.8.8.8` → confirma salida a Internet.
  - Navegar a un sitio de red social (ej. facebook.com) → debe bloquearse.
  - Navegar a `itla.edu.do` → debe bloquearse.
  - Intentar `ftp://10.13.67.130` o cualquier protocolo distinto a HTTP hacia el servidor Web → debe bloquearse (solo HTTP permitido).
  - Navegar a `http://10.13.67.130` → debe permitirse.
- Desde el host Fedora: `curl http://200.13.67.2` → confirma acceso GUI FortiGate.
- **Log & Report → Forward Traffic / Security Events**: confirma los bloqueos de Application Control, DNS Filter, IPS y WAF apareciendo en el log en tiempo real — esto es evidencia clave para tu reporte de lab.

---

## 10. Errores comunes ya resueltos en este lab

1. **"Can't change dynamic IP" (-651)** al asignar IP en port1 → el puerto viene en DHCP por defecto. Fix: cambiar `Addressing mode` a **Manual** antes de escribir la IP.
2. **"Too many entries... vdom-max = 3"** al crear una 4ta política → licencia evaluación limita a 3 políticas por VDOM. Fix: consolidar y usar perfiles de seguridad dentro de las mismas 3 políticas en vez de políticas nuevas.
3. **VIP no aparece como opción de Destination** en una política → el objeto VIP no se creó antes. Fix: crear siempre primero el objeto (Address/VIP) y después la política que lo referencia.
4. **HTTPS de gestión no carga** aunque esté habilitado → normal en licencia evaluación FGVMEV. Fix: habilitar también HTTP y entrar por `http://200.13.67.2`.
5. **"Invalid IP Netmask"** al asignar IP a una interfaz → se puso la dirección de red (`10.13.67.0/25`) en vez de la IP de host (`10.13.67.1/25`). Fix: usar siempre la IP específica del equipo en interfaces; la subred completa solo va en objetos de dirección.
6. **WAF o Application Control no aparecen en el perfil de la política** → la política está en modo Flow-based. Fix: cambiar `Inspection Mode` a **Proxy-based** en la política antes de asignar el perfil.
7. **No encuentro dónde agregar `itla.edu.do` en el DNS Filter** → el campo de dominios está oculto hasta activar el toggle `Domain Filter` dentro de la sección **Static Domain Filter** (no confundir con la tabla superior de `FortiGuard Category Based Filter`, que es solo para categorías generales de contenido). Fix: activa el toggle `Domain Filter`, luego usa el `+ Create New` que aparece debajo.
8. **"No results" en el buscador de firmas IPS** (Security Profiles → Intrusion Prevention → Add Signatures) → el buscador requiere texto, dejarlo vacío no muestra nada. Además, la base de firmas de esta licencia eval está congelada desde 2015 (`diagnose autoupdate versions` → Attack Definitions 6.00741), así que nombres modernos como `Nmap.Scan` no existen. Fix: usar `FTP.Bounce.Port.Scan` (firma real de escaneo disponible en esta base) combinado con una entrada tipo `Filter` por `Severity: Medium/High/Critical` para cobertura general.
9. **Incoming interface de una policy solo permite una interfaz** → en esta build no está disponible (o no se activó) `Multiple Interface Policies`. Fix: usar `any` como Incoming interface y dejar que `LAN_USUARIOS` + `LAN_SERVIDORES` en Source filtren el tráfico real.
