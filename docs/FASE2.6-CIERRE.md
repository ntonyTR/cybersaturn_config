# FASE 2.6 — CIERRE: Administración remota de Saturn (SSH + Moonlight)

> Completada: Junio 2026

---

## Resultado

✅ SSH instalado y funcionando en LAN y vía Tailscale  
✅ Moonlight funcionando en LAN y vía Tailscale (directo, sin relay)  
✅ Conexión Tailscale confirmada como `direct` — sin overhead significativo  

---

## SSH

| Parámetro | Valor |
|---|---|
| **Servidor** | OpenSSH Server (Windows built-in) |
| **Puerto** | 22 (default) |
| **Usuario** | `antony` |
| **Startup** | Automático (servicio `sshd`) |
| **Firewall** | Regla `OpenSSH-Server-In-TCP` — perfil `Any` |

### Comandos de conexión

```bash
# Alias corto (todos los clientes)
ssh saturn

# Completo vía Tailscale
ssh antony@saturn.minotaur-mizar.ts.net
```

### Configuración ~/.ssh/config (Cassini, laptop trabajo, Termux)

```
Host saturn
    HostName saturn.minotaur-mizar.ts.net
    User antony
```

### Autenticación por llave pública (sin password)

- Llaves registradas en Saturn: Cassini, laptop trabajo, Termux
- **Archivo de llaves para usuarios administradores en Windows:**
  ```
  C:\ProgramData\ssh\administrators_authorized_keys
  ```
  > ⚠️ Windows SSH ignora `~/.ssh/authorized_keys` para usuarios del grupo Administrators. Las llaves deben ir en este archivo global.

- Llaves de cada cliente agregadas con `>>` para no sobreescribir entradas previas

---

## Moonlight

- Configurado previamente en LAN
- Validado vía Tailscale desde Cassini y teléfono (datos móviles)
- **No se separaron entradas LAN/Tailscale** — Moonlight no soporta múltiples IPs para el mismo host de Sunshine
- **Decisión:** usar siempre la entrada de Tailscale (`saturn.minotaur-mizar.ts.net`) para todo

### Por qué no hay penalización de latencia

Tailscale establece conexión `direct` (peer-to-peer WireGuard) en LAN:

```
tailscale status → saturn: active; direct 192.168.0.200:41641
```

El tráfico va directo entre dispositivos sin pasar por servidores de Tailscale. El overhead es de microsegundos — imperceptible para streaming.

---

## Pruebas realizadas

| Prueba | Escenario | Resultado |
|---|---|---|
| SSH desde Cassini | Red local | ✅ |
| SSH desde Cassini | Vía Tailscale | ✅ |
| SSH desde Termux (Android) | Vía Tailscale | ✅ |
| SSH desde laptop trabajo (W11) | Vía Tailscale | ✅ |
| `ssh saturn` sin password — Cassini | ✅ |
| `ssh saturn` sin password — laptop trabajo | ✅ |
| `ssh saturn` sin password — Termux | ✅ |
| Moonlight desde teléfono | Datos móviles | ✅ |
| Moonlight desde Cassini | Vía Tailscale | ✅ |
| Tailscale connection type | LAN | ✅ `direct` (no relay) |

---

## Siguiente fase

**Fase 2.7** — Wake on LAN
