# CYBERSATURN — Fuente de verdad: Red, SSH, Tailscale y acceso remoto
> Última actualización: Junio 2026  
> Estado: Producción — todo operacional

---

## 1. Dispositivos y red

### Dispositivos en el tailnet

| Dispositivo | Hostname Tailscale | IP Tailscale | OS | Estado |
|---|---|---|---|---|
| **Saturn** | `saturn.minotaur-mizar.ts.net` | `100.68.149.122` | Windows 11 LTSC IoT | ✅ Online |
| **Cassini** | `cassini.minotaur-mizar.ts.net` | `100.99.64.62` | Fedora Linux | ✅ Online |
| **Henco** (laptop trabajo) | `henco.minotaur-mizar.ts.net` | `100.124.150.104` | Windows 11 | ⚡ Online cuando en uso |
| **ntonys-s25** (teléfono) | `ntonys-s25.minotaur-mizar.ts.net` | `100.110.200.79` | Android | ⚡ Online cuando en uso |

### Saturn — interfaces de red

| Interfaz | IP |
|---|---|
| **LAN (estática)** | `192.168.0.200` |
| **Tailscale** | `100.68.149.122` |
| **WSL2 (Docker)** | `172.21.112.1` |
| **Docker bridge** | `172.26.16.1` |

### DNS local (LAN) — `/etc/hosts` en clientes Linux

```
192.168.0.200    saturn.lan
```

Configurado en: Cassini. Pendiente verificar en Henco (Windows — archivo en `C:\Windows\System32\drivers\etc\hosts`).

> El router TP-Link AX3000 no soporta DNS local — solución via archivo hosts por cliente.  
> `saturn.lan` solo funciona en LAN. Para acceso remoto siempre usar `saturn.minotaur-mizar.ts.net`.

---

## 2. Tailscale

### Configuración del tailnet

| Parámetro | Valor |
|---|---|
| **Tailnet** | `minotaur-mizar` |
| **Cuenta** | `ntonyTR@` |
| **MagicDNS** | ✅ Activado |
| **2FA cuenta Tailscale** | ✅ Activado |

### Key expiry por dispositivo

| Dispositivo | Key expiry |
|---|---|
| **Saturn** | ✅ Desactivado — el servidor nunca pide reautenticación |
| **Cassini** | Activo (default) |
| **Henco** | Activo (default) |
| **ntonys-s25** | Activo (default) |

### Tipo de conexión

Tailscale establece conexión **direct** (peer-to-peer WireGuard) en LAN — sin relay, sin overhead significativo:

```
cassini → saturn: active; direct 192.168.0.239:41641
```

El tráfico va directo entre dispositivos incluso cuando se usa el hostname de Tailscale. Latencia equivalente a LAN pura + cifrado WireGuard (microsegundos de overhead).

### Comandos útiles

```bash
# Ver estado y tipo de conexión
tailscale status

# Ver IP propia
tailscale ip

# Ping a otro dispositivo
tailscale ping saturn
```

### Por qué Tailscale y no exposición directa

- Ollama **no está expuesto a internet** — solo accesible via LAN o Tailscale
- No hay puertos abiertos en el router hacia internet (excepto lo que maneja Cloudflare Tunnel para n8n)
- Tailscale cifra todo el tráfico peer-to-peer con WireGuard
- Plan personal gratuito — sin costo

---

## 3. SSH

### Servidor SSH en Saturn

| Parámetro | Valor |
|---|---|
| **Implementación** | OpenSSH Server (built-in de Windows) |
| **Puerto** | 22 (default) |
| **Usuario** | `antony` |
| **Startup** | Automático (servicio `sshd`) |
| **Firewall** | Regla `OpenSSH-Server-In-TCP` — perfil `Any` |

### Llaves autorizadas en Saturn

Archivo: `C:\ProgramData\ssh\administrators_authorized_keys`

> ⚠️ Windows SSH ignora `~/.ssh/authorized_keys` para usuarios del grupo Administrators. Las llaves **deben** ir en este archivo global.

| Llave | Tipo | Identificador | Dispositivo |
|---|---|---|---|
| `AAAAC3NzaC1lZDI1NTE5AAAAIG1o/g8JJAfF6TA56neUKeJ7E8gpSpeWPqj+oS0ydSIt` | ed25519 | `antonyetr@outlook.com` | Cassini |
| `AAAAC3NzaC1lZDI1NTE5AAAAILqUUE5Ci5L8BfmjihAz7VASFX5JNP4rA1t3GfXA0iBp` | ed25519 | `antony@henco.com.mx` | Henco (laptop trabajo) |
| `AAAAC3NzaC1lZDI1NTE5AAAAICI3M6TqK9KvPag5fDcFxiIydqCVYC6/c3cHHRIjUK5F` | ed25519 | `termux` | Android (Termux) |

### Configuración SSH en clientes

**`~/.ssh/config` en Cassini y Henco (idéntico):**

```
Host saturn
    HostName saturn.minotaur-mizar.ts.net
    User antony
```

**Comando de conexión desde cualquier cliente:**

```bash
ssh saturn
```

Conexión sin password en todos los clientes configurados: Cassini ✅, Henco ✅, Termux ✅.

### Agregar una llave nueva a Saturn

```powershell
# Desde Saturn (o pegar el contenido manualmente)
Add-Content C:\ProgramData\ssh\administrators_authorized_keys "ssh-ed25519 AAAA... comentario"
```

> Usar `>>` o `Add-Content` — nunca sobreescribir el archivo, o se pierden las llaves existentes.

### Limitaciones de SSH en Saturn

| Limitación | Detalle |
|---|---|
| **Docker login falla via SSH** | El credential store de Windows no está disponible en sesión SSH — usar Moonlight para operaciones que requieran GUI |
| **No hay `cat` en PowerShell** | Usar `Get-Content` en lugar de `cat` |
| **No hay `Remove-Service`** | Usar `sc.exe delete <nombre>` |
| **No hay `Get-EventLog`** | Limitación de Windows 11 LTSC IoT |

---

## 4. Wake on LAN

### Configuración en Saturn

| Parámetro | Estado |
|---|---|
| WoL en BIOS/UEFI | ✅ Habilitado |
| WoL en NIC (Administrador de dispositivos → NIC → Administración de energía → "Permitir que este dispositivo reactive el equipo") | ✅ Habilitado |
| ARP estático en router | ✅ Configurado — magic packet llega aunque Saturn lleve días apagado |
| IP estática LAN | ✅ `192.168.0.200` — el magic packet siempre sabe a dónde ir |

### MAC address de Saturn

| Parámetro | Valor |
|---|---|
| **MAC** | `08-BF-B8-6F-4F-B0` |

### Enviar magic packet desde Cassini

```bash
# wol instalado via: sudo dnf install wol
# Alias configurado (saturn.lan resuelve a 192.168.0.200)
wol -i 192.168.0.255 <MAC>
```

> ⚠️ Usar la IP de broadcast (`192.168.0.255`), no la IP estática de Saturn.

### Enviar magic packet desde Android

App: **WolOn**  
- Configurada con la MAC de Saturn y broadcast IP `192.168.0.255`
- Funciona desde LAN
- Licencia de pago pendiente para habilitar comandos SSH adicionales desde la app

### Flujo de arranque remoto completo

1. Enviar magic packet desde Cassini (`wol`) o Android (WolOn)
2. Esperar ~30–60 segundos para que Windows arranque y los servicios suban
3. Conectar via SSH: `ssh saturn`
4. Verificar que Ollama esté corriendo: `sc.exe query ollama`
5. Si se necesita GUI: conectar via Moonlight

---

## 5. Moonlight / Sunshine (escritorio remoto)

| Parámetro | Valor |
|---|---|
| **Servidor** | Sunshine en Saturn |
| **Cliente** | Moonlight en Cassini y Android |
| **Hostname configurado** | `saturn.minotaur-mizar.ts.net` — una sola entrada para LAN y remoto |
| **Conexión LAN** | Tailscale direct → sin penalización de latencia |
| **Conexión remota** | Vía Tailscale — validado desde Android en datos móviles |

> Moonlight no soporta múltiples IPs para el mismo host de Sunshine — se usa siempre el hostname de Tailscale. En LAN la conexión es direct (WireGuard P2P), sin diferencia práctica de latencia respecto a la IP local.

**Cuándo usar Moonlight vs SSH:**

| Tarea | Herramienta |
|---|---|
| Administración de servicios, logs, archivos | SSH |
| Docker login | Moonlight (requiere GUI) |
| Cambios en BIOS / configuración visual | Moonlight |
| Gaming remoto / escritorio completo | Moonlight |

---

## 6. Acceso externo — URLs públicas

| Servicio | URL | Método |
|---|---|---|
| **n8n** | `https://n8n.ntny.dev` | Cloudflare Tunnel |
| **Telegram webhook** | `https://n8n.ntny.dev/webhook/4b33f5a5-62df-4edf-8b83-1dfce53f4546/webhook` | Cloudflare Tunnel |
| **Ollama** | No expuesto a internet | Solo LAN + Tailscale |

### Dominio

| Parámetro | Valor |
|---|---|
| **Dominio** | `ntny.dev` |
| **Registrar** | Cloudflare Registrar |
| **DNS** | Gestionado por Cloudflare |
| **Subdominio activo** | `n8n.ntny.dev` |
| **Uso futuro** | Portfolio de desarrollador en `ntny.dev` |

### Cloudflare Tunnel

| Parámetro | Valor |
|---|---|
| **Tunnel name** | `n8n` |
| **Tunnel ID** | `dcb83209-98d7-4927-9cac-d655a51a95ea` |
| **Credentials** | `C:\Users\antony\.cloudflared\dcb83209-98d7-4927-9cac-d655a51a95ea.json` |
| **Config** | `C:\Users\antony\.cloudflared\config.yml` |
| **Servicio** | NSSM `cloudflared` — `AUTO_START` |

---

## 7. Router

| Parámetro | Valor |
|---|---|
| **Modelo** | TP-Link AX3000 |
| **Panel de administración** | `192.168.0.1` |
| **DHCP Address Reservation** | Saturn: MAC → `192.168.0.200` |
| **ARP estático** | Configurado para Saturn (requerido por WoL) |
| **DNS local** | No soportado — solucionado via `/etc/hosts` en clientes |
| **Puertos abiertos hacia internet** | Ninguno — todo el acceso externo va por Cloudflare Tunnel o Tailscale |

---

## 8. Seguridad

| Item | Estado |
|---|---|
| Ollama no expuesto a internet | ✅ Solo LAN + Tailscale |
| Tailscale cifra todo el tráfico P2P | ✅ WireGuard |
| 2FA en cuenta Tailscale | ✅ |
| SSH solo con llave pública | ✅ Sin autenticación por password |
| Puertos abiertos en router hacia internet | ✅ Ninguno |
| ngrok eliminado | ✅ Desinstalado y config borrada |

---

## 9. Referencia rápida de conexión

```bash
# SSH a Saturn
ssh saturn

# Verificar conectividad Ollama
curl http://saturn.minotaur-mizar.ts.net:11434/api/tags

# Ver estado Tailscale desde Cassini
tailscale status

# Despertar Saturn
wol -i 192.168.0.255 <MAC>
```

```powershell
# Ver estado de servicios en Saturn (via SSH)
sc.exe query ollama
sc.exe query cloudflared
sc.exe query sshd

# Ver interfaces de red
ipconfig

# Ver llaves SSH autorizadas
Get-Content C:\ProgramData\ssh\administrators_authorized_keys
```
