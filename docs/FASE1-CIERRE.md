# CYBERSATURN — Fase 1: Cierre
> Servidor base: Ollama en red local  
> Completado: 24 de junio de 2026

---

## Resultado: ✅ Fase 1 completa

Todos los pasos ejecutados y verificados. Ollama corre en GPU y es accesible desde Cassini en red local.

---

## Configuración del servidor (saturn)

| Parámetro | Valor |
|---|---|
| **Hostname** | saturn |
| **IP local (fija)** | `192.168.0.200` |
| **Significado** | HTTP 200 OK |
| **Router** | TP-Link AX3000 |
| **Método de reserva** | DHCP Address Reservation (por MAC) |
| **Puerto Ollama** | `11434` |
| **OLLAMA_HOST** | `0.0.0.0:11434` |
| **Firewall** | Puerto TCP 11434 abierto (regla: `Ollama LAN`) |

---

## Hardware del servidor

| Componente | Detalle |
|---|---|
| **CPU** | AMD Ryzen 5800XT |
| **GPU** | RTX 4070 Super (12GB VRAM) |
| **RAM** | 32GB DDR4 3200MHz |
| **OS** | Windows 11 LTSC IoT |

---

## Modelos instalados

| Modelo | Uso asignado | VRAM aprox. |
|---|---|---|
| `qwen2.5:14b` | Chat general | ~9GB |
| `qwen2.5-coder:14b` | Chat de código, refactors, modificaciones | ~9GB |
| `qwen2.5-coder:7b` | Autocompletado FIM en IDE (Continue) | ~5GB |

> Los modelos no corren simultáneamente — Ollama los swapea automáticamente según cuál se use.  
> VRAM en uso durante inferencia: ~10GB de 12GB disponibles.

---

## Pruebas realizadas

| # | Prueba | Resultado |
|---|---|---|
| 1.1 | IP estática asignada en router | ✅ `192.168.0.200` confirmada con `ipconfig` |
| 1.2 | Ollama instalado y corriendo | ✅ Servicio activo en background |
| 1.3 | `OLLAMA_HOST=0.0.0.0:11434` configurada | ✅ Variable visible en System Variables |
| 1.4 | Puerto 11434 abierto en Firewall | ✅ Regla `Ollama LAN` creada |
| 1.5 | Modelo corriendo en GPU | ✅ `nvidia-smi` muestra `llama-server.exe`, 10GB VRAM, 96% GPU util |
| 1.6 | API responde en localhost | ✅ `GET /api/tags` → 200 OK con lista de modelos |
| 1.7 | Inferencia completa en localhost | ✅ `POST /api/chat` → respuesta coherente del modelo |
| 1.8 | Acceso desde Cassini en LAN | ✅ `curl http://192.168.0.200:11434/api/tags` responde correctamente |

---

## Notas relevantes

- **PowerShell no acepta sintaxis curl de Linux.** Usar `Invoke-WebRequest` en su lugar:
  ```powershell
  Invoke-WebRequest -Uri "http://localhost:11434/api/chat" -Method POST -ContentType "application/json" -Body '{...}'
  ```

- **Modelos descartados para este lineup:** `qwen3:14b` (no aporta diferenciación con coder 14b en VRAM disponible), `deepseek-r1:14b` (candidato futuro si se necesita más razonamiento), `qwen3-coder:30b` (requiere 24GB VRAM, fuera de alcance hasta upgrade a RTX 4090).

- **IP original asignada por router:** `192.168.0.129`. Cambiada manualmente a `192.168.0.200` (HTTP 200 OK).

---

## Estado para Fase 2

El servidor está listo y accesible en red local. La Fase 2 (Tailscale) extenderá este acceso a cualquier red mediante VPN mesh.

**Dato clave para Fase 2:** la IP Tailscale del servidor (`100.x.x.x`) reemplazará a `192.168.0.200` para acceso remoto, pero la IP local sigue siendo válida cuando los dispositivos están en la misma red.

**Dispositivos que necesitan Tailscale:**
- saturn (PC Gamer, Windows 11) — el servidor
- Cassini (Laptop, Fedora Linux)
- Laptop del trabajo (Windows 11)
- Teléfono (Android/iOS)

---

*CyberSaturn — Fase 1 cerrada — Junio 2026*
