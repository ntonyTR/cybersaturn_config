# CYBERSATURN — Plan de ejecución

> Un chat por fase. Al cerrar cada fase, generar el documento de cierre antes de continuar.

---

## Fase 1 — Servidor base: Ollama en red local

**Dispositivo:** PC Gamer (Windows 11)

| # | Paso | Dónde |
|---|---|---|
| 1.1 | Asignar IP estática al PC gamer en el router TP-Link (`DHCP → Address Reservation`) | Router |
| 1.2 | Instalar Ollama desde `ollama.com` y verificar que corre como servicio en background | Windows |
| 1.3 | Configurar variable de entorno `OLLAMA_HOST=0.0.0.0:11434` y reiniciar el servicio | Windows |
| 1.4 | Abrir puerto TCP `11434` en Windows Defender Firewall | Windows |
| 1.5 | Descargar primer modelo (`qwen2.5:14b`) y confirmar ejecución en GPU con `nvidia-smi` | Windows |
| 1.6 | Prueba de conectividad desde Cassini en red local: `curl http://192.168.0.50:11434/api/tags` | Cassini |

**Cierre:** generar `FASE1-CIERRE.md` con IP asignada, modelos instalados y resultado de pruebas.

---

## Fase 2 — Acceso remoto: Tailscale VPN mesh

**Dispositivos:** Todos

| # | Paso | Dónde |
|---|---|---|
| 2.1 | Instalar Tailscale en PC gamer, Cassini, laptop del trabajo y teléfono. Loguear todos con la misma cuenta | Todos |
| 2.2 | Obtener IP Tailscale del servidor con `tailscale ip` y registrarla | Windows |
| 2.3 | Prueba desde Cassini vía Tailscale: `curl http://100.x.x.x:11434/api/tags` | Cassini |
| 2.4 | Verificar coexistencia con VPN corporativa (split vs full tunnel). Documentar resultado o plan con IT | Laptop trabajo |

**Cierre:** generar `FASE2-CIERRE.md` con IP Tailscale del servidor, estado de cada dispositivo y nota sobre VPN del trabajo.

---

## Fase 2.5 — Validación completa de acceso y Wake on LAN

> Fase de verificación antes de construir encima. No se avanza a Fase 3 hasta que todos los checks pasen.

**Dispositivos:** Todos

### Acceso desde red local

| # | Prueba | Comando | Esperado |
|---|---|---|---|
| 2.5.1 | API responde en LAN | `curl http://192.168.0.50:11434/api/tags` | Lista de modelos en JSON |
| 2.5.2 | Inferencia completa en LAN | `curl -X POST http://192.168.0.50:11434/api/chat -d '{"model":"qwen2.5:14b","messages":[{"role":"user","content":"hola"}],"stream":false}'` | Respuesta coherente del modelo |
| 2.5.3 | GPU en uso durante inferencia | `nvidia-smi` mientras corre el modelo | VRAM ocupada, GPU activa |

### Acceso vía Tailscale

| # | Prueba | Comando | Esperado |
|---|---|---|---|
| 2.5.4 | API responde vía Tailscale (Cassini) | `curl http://100.x.x.x:11434/api/tags` | Lista de modelos en JSON |
| 2.5.5 | Inferencia completa vía Tailscale | Mismo curl de 2.5.2 con IP Tailscale | Respuesta coherente del modelo |
| 2.5.6 | Acceso desde laptop del trabajo | Mismo curl con IP Tailscale, con VPN corporativa activa | Lista de modelos en JSON |
| 2.5.7 | Acceso desde teléfono (datos móviles, sin WiFi) | Cliente HTTP en el teléfono contra IP Tailscale | Respuesta del modelo |

### Wake on LAN

> Configurar WoL aquí, antes de las fases de uso intensivo, para tener encendido remoto desde el principio.

| # | Paso | Dónde |
|---|---|---|
| 2.5.8 | Habilitar WoL en BIOS/UEFI | PC Gamer |
| 2.5.9 | Habilitar WoL en propiedades de la NIC (Administrador de dispositivos → NIC → Administración de energía) | Windows |
| 2.5.10 | Configurar ARP estático en el router para la MAC del servidor | Router |
| 2.5.11 | Instalar `wol` en Cassini: `sudo dnf install wol` | Cassini |
| 2.5.12 | Apagar el PC gamer completamente y enviar magic packet: `wol AA:BB:CC:DD:EE:FF` | Cassini |
| 2.5.13 | Confirmar que el servidor arranca y Ollama queda disponible sin intervención manual | Cassini |

**Cierre:** generar `FASE2.5-CIERRE.md` con tabla de resultados por prueba (✅/❌), MAC del servidor, IP Tailscale confirmada y comando WoL final probado.

---

## Fase 3 — Herramientas de desarrollo: Continue en VS Code

**Dispositivos:** Cassini + Laptop trabajo

| # | Paso | Dónde |
|---|---|---|
| 3.1 | Instalar extensión Continue en VS Code y configurar `~/.continue/config.json` apuntando a la IP Tailscale del servidor | Cassini |
| 3.2 | Configurar n8n para usar Ollama como proveedor OpenAI-compatible en nodos de AI (`host.docker.internal:11434/v1`) | Windows |

**Cierre:** generar `FASE3-CIERRE.md` con configuración de Continue, modelos disponibles y prueba de autocompletado.

---

## Fase 4 — Bot de Telegram: n8n + Ollama

**Dispositivos:** PC Gamer + Teléfono

| # | Paso | Dónde |
|---|---|---|
| 4.1 | Crear bot en Telegram con @BotFather y guardar el token | Teléfono |
| 4.2 | Instalar Docker Desktop en el PC gamer y levantar contenedor n8n con persistencia | Windows |
| 4.3 | Crear workflow en n8n: `Telegram Trigger → HTTP Request (Ollama) → Telegram Send` | Windows |
| 4.4 | Activar workflow y probar end-to-end enviando un mensaje desde el teléfono | Teléfono |

**Cierre:** generar `FASE4-CIERRE.md` con nombre del bot, estructura del workflow y confirmación del flujo funcional.

---

## Mejora A — Wake on LAN: encendido remoto (post-setup, opcional)

| # | Paso | Dónde |
|---|---|---|
| A.1 | Habilitar WoL en BIOS/UEFI y en propiedades de la NIC (Administrador de dispositivos) | Windows |
| A.2 | Configurar ARP estático en el router para que el magic packet llegue aunque la PC lleve días apagada | Router |
| A.3 | Instalar `wol` en Cassini (`sudo dnf install wol`) y probar envío de magic packet | Cassini |

**Cierre:** generar `MEJORA-WOL.md` con MAC del servidor y comando final probado.

---

## Mejora B — Upgrade path (según necesidad)

| Situación | Acción |
|---|---|
| 12GB VRAM insuficientes | Upgrade a RTX 4090 (~20k MXN neto vendiendo 4070 Super) |
| Acceso sin depender de Tailscale | Evaluar Headscale self-hosted en VPS (~$5 USD/mes) |
| PC gamer como servidor dedicado sin pantalla | Migrar a Linux (+5-8% tokens/seg) |

Cada decisión abre su propio chat de implementación en su momento.

---

*CyberSaturn — Junio 2026*
