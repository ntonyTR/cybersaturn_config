# FASE 2 — CIERRE: Acceso remoto con Tailscale

> Completada: Junio 2026

---

## Resultado

✅ Todos los dispositivos conectados a la VPN mesh de Tailscale  
✅ Ollama accesible desde todos los clientes vía hostname  
✅ Tailnet configurado y personalizado  

---

## Configuración de red

| Parámetro | Valor |
|---|---|
| **Tailnet** | `minotaur-mizar` |
| **Hostname del servidor** | `saturn.minotaur-mizar.ts.net` |
| **IP Tailscale del servidor** | `100.68.149.122` |
| **Endpoint Ollama remoto** | `http://saturn.minotaur-mizar.ts.net:11434` |

---

## Estado de dispositivos

| Dispositivo | OS | Estado |
|---|---|---|
| **Saturn** (PC Gamer) | Windows 11 LTSC IoT | ✅ Conectado — key expiry desactivado |
| **Cassini** | Fedora Linux | ✅ Conectado |
| **Laptop trabajo** | Windows 11 | ✅ Conectado |
| **Teléfono** | Android | ✅ Conectado |

---

## Pruebas realizadas

| Prueba | Resultado |
|---|---|
| Ping Saturn → Cassini (`100.99.64.62`) | ✅ 4/4 paquetes, avg 60ms |
| `curl` Ollama desde Cassini vía IP Tailscale | ✅ JSON con lista de modelos |
| `curl` Ollama desde laptop del trabajo | ✅ JSON con lista de modelos |
| `curl` Ollama vía hostname MagicDNS | ✅ `saturn.minotaur-mizar.ts.net:11434` responde |

---

## Configuraciones aplicadas en Tailscale

- ✅ **MagicDNS** activado — acceso por hostname en lugar de IP
- ✅ **Tailnet renombrado** a `minotaur-mizar`
- ✅ **Key expiry desactivado** en Saturn — el servidor no pedirá reautenticación
- ✅ **2FA** activado en la cuenta de Tailscale
- ✅ **Dispositivos renombrados** con nombres descriptivos

---

## Notas

- La VPN corporativa de la laptop del trabajo opera en **split tunnel** — coexiste con Tailscale sin conflicto.
- Moonlight/Sunshine también puede usarse sobre Tailscale apuntando a `saturn.minotaur-mizar.ts.net`.
- Key expiry se dejó activo en los clientes (Cassini, laptop, teléfono) — solo desactivado en el servidor.

---

## Siguiente fase

**Fase 2.5** — Validación completa de inferencia y Wake on LAN  
o  
**Fase 3** — Continue en VS Code (Cassini + Laptop trabajo)
