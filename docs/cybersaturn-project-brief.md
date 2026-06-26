# CYBERSATURN — Brief general del proyecto
> Última actualización: Junio 2026

---

## Qué es CyberSaturn

Servidor privado de IA local, accesible desde cualquier dispositivo en la red personal. El objetivo es correr modelos LLM en el hardware propio — sin APIs de pago, sin datos que salgan a terceros — e integrarlos profundamente en el flujo de trabajo de desarrollo y en la vida diaria vía Telegram.

**Costo total del stack de software: $0 MXN.**

---

## Hardware principal — Saturn

| Componente | Detalle |
|---|---|
| **Rol** | Servidor de inferencia — siempre encendido durante uso |
| **CPU** | AMD Ryzen 5800XT |
| **GPU** | RTX 4070 Super (12GB VRAM) |
| **RAM** | 32GB DDR4 3200MHz |
| **OS** | Windows 11 LTSC IoT (sin Microsoft Store / UWP) |
| **IP LAN** | `192.168.0.200` (estática via DHCP Reservation en router) |
| **IP Tailscale** | `100.68.149.122` |
| **Hostname Tailscale** | `saturn.minotaur-mizar.ts.net` |

### Zona cómoda de modelos con este hardware

Con 12GB VRAM, el sweet spot es **8B a 14B parámetros cuantizados (Q4)**. Modelos 32B+ caen a RAM del sistema y bajan drásticamente de velocidad.

| Modelo | VRAM | tok/s (16K ctx) |
|---|---|---|
| `qwen3:14b` | ~10.7 GB | ~23 tok/s |
| `qwen2.5-coder:14b` | ~10.7 GB | ~23 tok/s |

---

## Dispositivos cliente

| Dispositivo | Nombre | OS | Rol |
|---|---|---|---|
| Laptop personal | Cassini | Fedora Linux | Desarrollo principal |
| Laptop trabajo | Henco | Windows 11 | Desarrollo (VPN corporativa split tunnel) |
| Teléfono | ntonys-s25 | Android | Chat via Telegram, WoL, SSH via Termux |

---

## Stack completo

```
[Cassini / Henco / Teléfono]
            |
        [Tailscale]              ← VPN mesh privada (WireGuard P2P)
            |
      [Saturn — Casa]
            |
        [Ollama]                 ← Servidor de inferencia (NSSM service)
            |
     [qwen3:14b]                 ← Chat general, Telegram, agente
     [qwen2.5-coder:14b]        ← Código, Continue, OpenCode
     [nomic-embed-text]          ← Embeddings (@codebase en Continue)
            |
    ┌───────┴────────┐
    │                │
  [n8n]           [Continue]     ← VS Code en Cassini/Henco
  [Docker]        [OpenCode]     ← Terminal en Cassini
    │
[Cloudflare Tunnel]
    │
[n8n.ntny.dev]
    │
[Telegram Bot]                   ← @cybersaturn_bot
```

---

## Componentes del stack

### Inferencia
| Componente | Versión | Rol |
|---|---|---|
| **Ollama** | latest | Motor de inferencia, API compatible con OpenAI |
| **NSSM** | — | Gestor de servicios Windows (Ollama + cloudflared) |

### Acceso remoto y red
| Componente | Versión | Rol |
|---|---|---|
| **Tailscale** | — | VPN mesh entre dispositivos (plan personal gratuito) |
| **OpenSSH Server** | built-in Windows | Administración remota de Saturn |
| **Moonlight / Sunshine** | — | Escritorio remoto sobre Tailscale |
| **WolOn** (Android) | — | Wake on LAN desde teléfono |

### Automatización y bot
| Componente | Versión | Rol |
|---|---|---|
| **n8n** | v2.27.4 | Orquestador de workflows / bot de Telegram |
| **Docker Desktop** | v4.78.0 | Runtime del contenedor n8n |
| **cloudflared** | v2026.5.2 | Túnel Cloudflare — expone n8n a internet |
| **Telegram Bot** | — | Interfaz de chat móvil (@cybersaturn_bot) |

### Herramientas de desarrollo
| Componente | Versión | Rol |
|---|---|---|
| **Continue** | v2.0.0 | AI en VS Code — chat, edit, autocomplete |
| **OpenCode** | v1.17.10 | Agente de terminal |
| **Node.js** | v24.17.0 | Runtime de OpenCode |

---

## Acceso por escenario

| Escenario | Cómo acceder a Ollama |
|---|---|
| En casa, desde Cassini (LAN) | `http://saturn.lan:11434` o `http://saturn.minotaur-mizar.ts.net:11434` |
| Remoto, desde Cassini | `http://saturn.minotaur-mizar.ts.net:11434` |
| Desde Henco (con o sin VPN corporativa) | `http://saturn.minotaur-mizar.ts.net:11434` |
| Desde teléfono (datos móviles) | `http://saturn.minotaur-mizar.ts.net:11434` |
| Desde contenedores Docker en Saturn | `http://host.docker.internal:11434` |
| n8n (público) | `https://n8n.ntny.dev` |

---

## Estado actual (Junio 2026)

| Fase | Descripción | Estado |
|---|---|---|
| Fase 1 | Ollama en red local | ✅ Completa |
| Fase 2 | Tailscale VPN mesh | ✅ Completa |
| Fase 2.5 | Validación de acceso desde todos los dispositivos | ✅ Completa |
| Fase 2.6 | SSH + Moonlight | ✅ Completa |
| Fase 2.7 | Wake on LAN | ✅ Completa |
| Fase 3 | Continue en VS Code (Cassini) | ✅ Completa |
| Fase 3.1 | Configuración avanzada de Continue — contexto, autocomplete, prompts | ✅ Completa |
| Fase 3.2 | OpenCode en terminal (Cassini) | ✅ Completa |
| Fase 3.3 | Continue + OpenCode en Henco (laptop trabajo) | ✅ Completa |
| Fase 4 | Bot de Telegram con n8n + Cloudflare Tunnel | ✅ Completa |

---

## Pendientes y roadmap

| Item | Prioridad | Descripción |
|---|---|---|
| **Memoria conversacional bot** | Media | Contexto por sesión + `/clearcontext` en Telegram |
| **Portfolio ntny.dev** | Baja | Dominio ya registrado en Cloudflare |
| **AIFinance** | Baja | Extender workflow de Telegram con Google Sheets + clasificación de gastos |
| **RAG / second brain** | Futura | ChromaDB o Qdrant + `nomic-embed-text` |
| **Voice interface** | Futura | Whisper (STT) + Kokoro/Piper (TTS) |
| **Vision** | Futura | `qwen2.5-vl` |
| **MCP integrations** | Futura | Home Assistant, web search |
| **Upgrade GPU** | Futura | RTX 4090 (~20,000 MXN neto) cuando aparezcan bottlenecks reales |

---

## Principios de diseño

- **Sin exposición directa a internet** — Ollama solo accesible via LAN o Tailscale. n8n expuesto únicamente a través de Cloudflare Tunnel con dominio propio.
- **Autostart completo** — tras un reinicio de Saturn, todo el stack sube sin intervención manual.
- **Un hostname para todo** — `saturn.minotaur-mizar.ts.net` funciona en LAN (direct P2P) y en remoto con el mismo string. Sin gestión de IPs dinámicas.
- **Costo cero de software** — todo el stack es open source o plan gratuito.
- **AI como multiplicador, no reemplazo** — las ganancias de productividad se reinvierten en aprendizaje. El objetivo es Senior Web Dev + capacidades AI profundas.

---

## Documentos de referencia

| Documento | Contenido |
|---|---|
| `cybersaturn-n8n-telegram-config.md` | Ollama, n8n, Docker, Cloudflare Tunnel, bot de Telegram — config completa |
| `cybersaturn-dev-tools-config.md` | Continue, OpenCode, modelos, prompts — config completa |
| `cybersaturn-network-config.md` | Tailscale, SSH, WoL, dispositivos, red — config completa |
| `cybersaturn-setup-guide.md` | Guía paso a paso para reinstalar todo el stack desde cero |
| `cybersaturn_config/` | Repo privado GitHub — archivos de config versionados |
