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
- [x] Bloquear dominios y subdominios de itla.edu.do (incluyendo bypass por DNS-over-HTTPS)
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

> **Nota — DNS-over-HTTPS (DoH) no se puede bloquear desde aquí:** se buscó en el buscador de firmas (`dns`) dentro de Application and Filter Overrides y no existe ninguna firma tipo `DNS.over.HTTPS` en esta base. La base de firmas de esta licencia eval está congelada desde 2015, y DoH viaja disfrazado dentro de una sesión HTTPS normal (mismo puerto 443, mismo TLS), por lo que requeriría firmas modernas de inspección de SNI/JA3 que no existen aquí. El bloqueo de DoH se resuelve en el **DNS Filter** (sección 7.2), no en Application Control.

### 7.2 DNS Filter — bloquear itla.edu.do, subdominios, y el bypass por DNS-over-HTTPS

**Security Profiles → DNS Filter → Create New**

- Name: `DNSFilter_ITLA`
- Redirect botnet C&C requests to Block Portal: apagado
- Enforce 'Safe Search': apagado
- FortiGuard Category Based Filter: no tocar, dejar en Monitor

**Bloquear el dominio y sus subdominios:**
- Sección **Static Domain Filter** → activa el toggle `Domain Filter`
- **+ Create New**:
  - Type: `Wildcard`
  - Domain: `*.itla.edu.do`
  - Action: **`Redirect to Block Portal`** ⚠️ (ver nota abajo — no existe una opción literal "Block" en este campo)
  - OK
- **+ Create New** otra vez:
  - Type: `Simple`
  - Domain: `itla.edu.do`
  - Action: `Redirect to Block Portal`
  - OK

> ⚠️ **Sobre el campo Action del Static Domain Filter:** en FortiOS 7.6.7 este campo solo tiene tres opciones: `Redirect to Block Portal`, `Allow`, `Monitor`. No existe un botón "Block" (eso aparece en otras versiones/guías, pero no en esta build). **`Redirect to Block Portal` ES el equivalente funcional de bloquear el dominio**: el FortiGate responde la consulta DNS con la IP del portal de bloqueo en vez de la IP real del sitio, así que el cliente nunca llega a itla.edu.do. Esto ya es un bloqueo efectivo, no hace falta buscar otra opción.

**Bloquear el bypass por DNS-over-HTTPS (DoH):**
Los navegadores modernos (Chrome, Edge, Firefox) traen DoH activado por defecto, que manda las consultas DNS cifradas directo a Cloudflare/Google, saltándose el DNS del FortiGate — esto permitiría resolver `itla.edu.do` sin pasar por el Static Domain Filter de arriba. Como no hay firma de Application Control para DoH (ver nota en 7.1), se cierra esta puerta bloqueando por dominio los proveedores DoH más comunes, en la misma tabla de Static Domain Filter:

- **+ Create New**: Type `Simple`, Domain `cloudflare-dns.com`, Action `Redirect to Block Portal`
- **+ Create New**: Type `Wildcard`, Domain `*.cloudflare-dns.com`, Action `Redirect to Block Portal`
- **+ Create New**: Type `Simple`, Domain `dns.google`, Action `Redirect to Block Portal`
- **+ Create New**: Type `Simple`, Domain `mozilla.cloudflare-dns.com`, Action `Redirect to Block Portal`

> Tabla final esperada (6 entradas): `*.itla.edu.do`, `itla.edu.do`, `cloudflare-dns.com`, `*.cloudflare-dns.com`, `dns.google`, `mozilla.cloudflare-dns.com` — todas en `Redirect to Block Portal` / `Enable`.

`External IP Block Lists` y `DNS Translation`: apagados

**Options** (abajo):
- Redirect Portal IP: `Use FortiGuard Default`
- **Allow DNS requests when a rating error occurs: ACTIVADO** ⚠️ (ver "Errores comunes" #10 — sin esto, cualquier dominio que el FortiGate no logre calificar contra FortiGuard queda bloqueado por defecto, aunque no esté en ninguna lista de bloqueo explícita)
- Log all DNS queries and responses: **ACTIVADO** (recomendado para evidencia en el reporte)
- **Strip Encrypted Client Hello service parameters: ACTIVADO** (viene así por defecto, no tocar). Esta opción elimina el parámetro ECH del registro DNS HTTPS (tipo 65) antes de que llegue al cliente, evitando que el navegador cifre el SNI del handshake TLS — mantiene visibilidad para el resto de los mecanismos de filtrado del FortiGate.
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

### 7.5 Web Filter — bloquear HTTP hacia Internet con página de bloqueo visible

**Por qué existe este perfil:** el requisito "solo permitir HTTP de usuarios hacia servidores, bloquear el resto" se puede cumplir de dos formas: (a) quitando el servicio `HTTP` de la Policy 1 y dejando que el implicit deny corte la sesión (sin página de aviso, ver Errores comunes #11 en la versión anterior de este documento), o (b) permitiendo `HTTP` en la Policy 1 pero aplicando un Web Filter que bloquea todo con `*` — así el usuario ve la página de bloqueo con el logo de Fortinet en vez de un timeout silencioso. Esta sección documenta la opción (b), que es la que se implementó.

**Por qué no rompe el HTTPS:** con SSL Inspection en `no-inspection`, el FortiGate nunca decodifica el tráfico HTTPS — pasa directo sin pasar por el motor de Web Filter. El tráfico HTTP sí es texto plano, así que el proxy (en modo Proxy-based) sí puede leerlo completo y aplicarle el Web Filter. Resultado: HTTP queda bloqueado con página de aviso, HTTPS sigue libre.

> **Importante — por qué el Web Filter NO sirve para bloquear itla.edu.do ni DoH:** con `SSL Inspection: no-inspection`, el motor de Web Filter (Static URL Filter) solo puede leer tráfico HTTP en texto plano. Tanto el acceso a `itla.edu.do` por HTTPS como el tráfico DoH viajan cifrados por el puerto 443, así que el proxy nunca llega a ver la URL — cualquier entrada que se agregue aquí para esos dominios simplemente nunca se dispara. Por eso el bloqueo de `itla.edu.do` y de DoH se resuelve exclusivamente en el **DNS Filter** (sección 7.2), que actúa en la capa de resolución de nombres, antes de que exista cualquier sesión HTTPS.

**Security Profiles → Web Filter → Create New**

- Name: `WebFilter_BlockHTTP`
- Feature set: **Proxy-based** (debe coincidir con el modo de inspección de la Policy 1)
- FortiGuard Category Based Filter: apagado
- Static URL Filter → activa el toggle **URL Filter** (no "Content Filter", ese se deja apagado)
  - **+ Create New**:
    - URL: `*`
    - Type: `Wildcard`
    - Action: `Block`
    - Status: `Enable`
    - OK
- Content Filter: apagado (no se usa)
- Rating Options → "Block all websites" (default, no tocar)
- Proxy Options: default, no tocar
- **OK** (guarda el perfil completo)

---

## 8. Paso 6 — Políticas de firewall (Policy & Objects → Firewall Policy)

Límite de licencia evaluación: **máximo 3 políticas por VDOM**. Los perfiles de seguridad se aplican dentro de estas mismas 3 políticas, no como políticas adicionales — incluyendo el Web Filter de la sección 7.5, que no cuenta como una 4ta política porque es solo un toggle dentro de la Policy 1 ya existente.

> **Nota sobre Incoming interface con múltiples valores:** en esta build, el campo Incoming interface de una policy normal solo admite **una** interfaz salvo que actives `System → Feature Visibility → Multiple Interface Policies`. Si no la activas (o no está disponible en licencia eval), usa **`any`** como Incoming interface y deja que el filtrado real lo hagan los objetos de **Source** (`LAN_USUARIOS`, `LAN_SERVIDORES`) — el efecto es equivalente para este lab.

### 8.1 Policy 1 — LANs-a-Internet (con bloqueos de apps/DNS/IPS/HTTP)

**Create New Policy:**
- Name: `LANs-a-Internet`
- Schedule: `always`
- Action: `ACCEPT`
- Incoming interface: `any` (ver nota arriba si tu build no permite multi-selección)
- Outgoing interface: tu interfaz WAN (alias tipo `ISP_INTERNET` / port1)
- Source: `+` → `LAN_USUARIOS` y `LAN_SERVIDORES`
- Destination: `+` → `all`
- **Service: `HTTP`, `HTTPS`, `DNS`** ⚠️ (ver nota abajo)

> **Por qué Service incluye HTTP ahora:** el requisito real de la práctica es "solo permitir HTTP de usuarios hacia servidores, bloquear el resto". Antes esto se lograba quitando `HTTP` de esta policy y dejando que el implicit deny cortara la sesión — funciona, pero el navegador solo muestra un timeout, sin página de aviso. Para que el bloqueo se vea explícitamente con el logo de Fortinet (mejor evidencia para el reporte de lab), ahora se permite `HTTP` en el Service pero se bloquea todo con el perfil `WebFilter_BlockHTTP` (sección 7.5) aplicado dentro de esta misma policy. El resultado funcional es el mismo (HTTP hacia Internet queda bloqueado), solo cambia el mecanismo y la experiencia visual del bloqueo.
> Recuerda que sin `DNS` en el Service, ni siquiera el `HTTPS` carga por nombre de dominio (solo por IP directa) — si al navegar no carga ningún sitio aunque HTTPS esté permitido, revisa primero que `DNS` siga en la lista de Service.

**Firewall/Network Options:**
- Inspection mode: cambia de `Flow-based` a **`Proxy-based`** (activa las opciones de Security Profiles)
- NAT: `Enable` (déjalo activado)
- IP pool configuration: `Use Outgoing Interface Address`

**Security Profiles** (activa el toggle de cada uno y selecciona el perfil):
- Web filter → `WebFilter_BlockHTTP`
- DNS filter → `DNSFilter_ITLA`
- Application control → `APPSBLOCKEOS`
- IPS → `IPS_AntiScan`
- AntiVirus, Video filter, File filter, Web application firewall: apagados (no van en esta policy)
- SSL inspection: dejar en `no-inspection` (necesario para que el HTTPS no se toque — ver sección 7.5)

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
- **Enable this policy: activado** ⚠️ (ver Errores comunes #12 — al final del formulario, es fácil dejarlo apagado sin querer y la policy entera queda inactiva aunque todo lo demás esté bien configurado)
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
- **Policy & Objects → Firewall Policy**: las 3 reglas visibles, en orden 1→2→3, todas con el toggle "Enable" en verde, con los íconos de los perfiles de seguridad junto a cada una.
- Desde Windows10:
  - `https://algún-sitio.com` → confirma salida a Internet (HTTPS permitido, sin tocar por SSL no-inspection).
  - `http://algún-sitio.com` (ej. `neverssl.com`) → **debe mostrar la página de bloqueo de Fortinet** (bloqueado por `WebFilter_BlockHTTP` en la Policy 1, ver 7.5 y 8.1).
  - `ping 8.8.8.8` → **fallará** (ICMP no está en el Service de la Policy 1; esto es esperado, no es un error).
  - Navegar a un sitio de red social (ej. facebook.com por HTTPS) → debe bloquearse por Application Control.
  - Navegar a `itla.edu.do` → debe bloquearse por DNS Filter (página de Redirect to Block Portal).
  - Ejecutar `ipconfig /flushdns` antes de cada prueba de dominio, para evitar falsos negativos por caché de DNS.
  - Intentar `ping` o cualquier protocolo distinto a HTTP hacia el servidor Web → debe bloquearse (solo HTTP permitido).
  - Navegar a `http://10.13.67.130` → debe permitirse (Policy 2).
- Desde el host Fedora: `curl http://200.13.67.2:8080` → confirma acceso GUI FortiGate (puerto cambiado, ver sección 6).
- **Log & Report → Security Events → DNS Query**: filtra por `itla`, `cloudflare` y `google` para confirmar Action = `Redirect` en tiempo real — esto es la evidencia clave de que tanto el bloqueo directo como el bloqueo del bypass por DoH están funcionando.
- **Log & Report → Security Events**: confirma también los bloqueos de Web Filter, Application Control, IPS y WAF apareciendo en el log en tiempo real — evidencia clave para tu reporte de lab.

---

## 10. Errores comunes ya resueltos en este lab

1. **"Can't change dynamic IP" (-651)** al asignar IP en port1 → el puerto viene en DHCP por defecto. Fix: cambiar `Addressing mode` a **Manual** antes de escribir la IP.
2. **"Too many entries... vdom-max = 3"** al crear una 4ta política → licencia evaluación limita a 3 políticas por VDOM. Fix: consolidar y usar perfiles de seguridad dentro de las mismas 3 políticas en vez de políticas nuevas (esto incluye el bloqueo de HTTP hacia Internet, resuelto con un perfil Web Filter dentro de la Policy 1 en vez de una 4ta policy DENY — ver sección 7.5 y 8.1).
3. **VIP no aparece como opción de Destination** en una política → el objeto VIP no se creó antes. Fix: crear siempre primero el objeto (Address/VIP) y después la política que lo referencia.
4. **HTTPS de gestión no carga** aunque esté habilitado → normal en licencia evaluación FGVMEV. Fix: habilitar también HTTP y entrar por `http://200.13.67.2`.
5. **"Invalid IP Netmask"** al asignar IP a una interfaz → se puso la dirección de red (`10.13.67.0/25`) en vez de la IP de host (`10.13.67.1/25`). Fix: usar siempre la IP específica del equipo en interfaces; la subred completa solo va en objetos de dirección.
6. **WAF, Application Control o Web Filter no aparecen en el perfil de la política** → la política está en modo Flow-based. Fix: cambiar `Inspection Mode` a **Proxy-based** en la política antes de asignar el perfil.
7. **No encuentro dónde agregar `itla.edu.do` en el DNS Filter** → el campo de dominios está oculto hasta activar el toggle `Domain Filter` dentro de la sección **Static Domain Filter** (no confundir con la tabla superior de `FortiGuard Category Based Filter`, que es solo para categorías generales de contenido). Fix: activa el toggle `Domain Filter`, luego usa el `+ Create New` que aparece debajo. Además, el campo **Action** de esta tabla no tiene una opción literal "Block" — solo `Redirect to Block Portal`, `Allow` y `Monitor`. `Redirect to Block Portal` es el equivalente funcional de bloquear el dominio (el FortiGate nunca deja resolver la IP real), así que no hay que buscar otra opción que no existe.
8. **"No results" en el buscador de firmas IPS** (Security Profiles → Intrusion Prevention → Add Signatures) → el buscador requiere texto, dejarlo vacío no muestra nada. Además, la base de firmas de esta licencia eval está congelada desde 2015 (`diagnose autoupdate versions` → Attack Definitions 6.00741), así que nombres modernos como `Nmap.Scan` no existen. Fix: usar `FTP.Bounce.Port.Scan` (firma real de escaneo disponible en esta base) combinado con una entrada tipo `Filter` por `Severity: Medium/High/Critical` para cobertura general.
9. **Incoming interface de una policy solo permite una interfaz** → en esta build no está disponible (o no se activó) `Multiple Interface Policies`. Fix: usar `any` como Incoming interface y dejar que `LAN_USUARIOS` + `LAN_SERVIDORES` en Source filtren el tráfico real.
10. **Búsquedas en buscadores (ej. Bing) fallan a medias — la página principal carga pero al buscar algo se rompe** → en **Log & Report → Security Events**, la tarjeta `DNS Query` muestra decenas de eventos `FortiGuard rating error occurred` con Action `Redirect`. Esto pasa porque el FortiGate no logra consultar la categoría del dominio contra los servidores de FortiGuard (falla de conectividad hacia la nube de FortiGuard, típico en labs de GNS3/EVE-NG), y por defecto el DNS Filter **bloquea/redirige** cualquier dominio que no pueda calificar — justo lo que pasa con los decenas de subdominios nuevos que dispara una búsqueda (CDNs, APIs de autosugerencia, telemetría). No es un bloqueo de contenido real, es un fallo de calificación. Fix: en `Security Profiles → DNS Filter → DNSFilter_ITLA`, sección **Options**, activar **"Allow DNS requests when a rating error occurs"**. Esto no afecta el bloqueo de `itla.edu.do` porque esa es una entrada estática explícita (Static Domain Filter), no depende de la calificación en la nube de FortiGuard.
11. **HTTP hacia Internet no muestra la página de bloqueo con el logo de FortiGuard, simplemente no conecta** → esto pasaba en el diseño anterior, donde el bloqueo se lograba por **ausencia del servicio `HTTP`** en la Policy 1 (implicit deny). El implicit deny corta la sesión a nivel de firewall antes de que se establezca una conversación HTTP completa, por lo que no hay forma de "inyectar" una página de aviso. **Solución aplicada:** se permitió `HTTP` en el Service de la Policy 1 y se agregó el perfil `WebFilter_BlockHTTP` (sección 7.5), con una regla `URL: *` → `Block`. Así el proxy sí completa la conexión HTTP y logra interceptar el contenido con la página de bloqueo de Fortinet, mientras que el HTTPS sigue sin tocarse gracias a `SSL Inspection: no-inspection`.
12. **Policy 2 (`Usuarios-a-Servidores-HTTP`) bien configurada pero `http://10.13.67.130` no carga desde Windows10** → el toggle **"Enable this policy"**, al final del formulario de la política, quedó apagado sin querer. Una policy deshabilitada no aplica aunque el resto de campos (Source, Destination, Service, Action) estén correctos. Fix: Edit Policy → bajar hasta el final → activar el toggle **Enable this policy** → OK.
13. **`itla.edu.do` sigue resolviendo aunque el Static Domain Filter esté bien configurado con `Redirect to Block Portal`** → el navegador del cliente (Chrome/Edge/Firefox) tiene **DNS-over-HTTPS (DoH)** activado por defecto, mandando las consultas DNS cifradas directo a Cloudflare (`1.1.1.1`) o Google (`8.8.8.8`) por HTTPS, saltándose por completo el DNS Filter del FortiGate. Se intentó bloquear DoH por firma en **Application Control**, pero la base de firmas 2015 de esta licencia eval no tiene ninguna firma tipo `DNS.over.HTTPS` (se buscó `dns` en Application and Filter Overrides y solo aparecen protocolos DNS clásicos). Tampoco se puede resolver desde el **Web Filter**, porque con `SSL Inspection: no-inspection` ese motor solo lee HTTP en texto plano, nunca ve el tráfico DoH cifrado. **Fix aplicado (sin tocar configuración en el cliente):** se agregaron al mismo `DNSFilter_ITLA` cuatro entradas más en Static Domain Filter, todas con Action `Redirect to Block Portal`: `cloudflare-dns.com` (simple), `*.cloudflare-dns.com` (wildcard), `dns.google` (simple), `mozilla.cloudflare-dns.com` (simple). Esto bloquea la resolución del hostname del servidor DoH antes de que el navegador pueda establecer sesión con él, forzando el fallback a DNS clásico — que sí pasa por el DNS Filter. Limitación a documentar: si el navegador trae hardcodeada la IP del resolver DoH (sin resolver hostname primero), este método no lo detiene; ahí se necesitaría bloqueo por IP en la Policy 1, fuera del alcance resuelto en este lab.

```
config firewall policy
    edit 1
        unset waf-profile
    next
end
```
