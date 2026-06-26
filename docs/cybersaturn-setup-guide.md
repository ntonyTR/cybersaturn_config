# CYBERSATURN — Guía de reconstrucción
> Referencia: Junio 2026  
> Esta guía es agnóstica al OS — los comandos exactos están en los documentos de verdad absoluta.

---

## Antes de empezar

Esta guía describe **qué hacer y en qué orden** para reconstruir el stack completo de CyberSaturn en Saturn. No importa si el OS destino es Windows, Linux o macOS — los pasos son los mismos, solo cambia cómo se ejecutan.

**El orden importa.** Cada paso asume que el anterior está completo.

**Documentos de referencia para valores exactos y comandos:**
- `cybersaturn-network-config.md` — Tailscale, SSH, llaves, WoL, dispositivos
- `cybersaturn-n8n-telegram-config.md` — Ollama, n8n, Docker, Cloudflare Tunnel, Telegram
- `cybersaturn-dev-tools-config.md` — Continue, OpenCode, modelos, prompts
- `cybersaturn_config/` — repo privado GitHub con configs versionadas

---

## Parte 1 — Saturn (servidor)

### 1. IP estática en el router

Asignar IP fija a Saturn via DHCP Reservation en el router TP-Link, usando la MAC de la NIC.

- IP objetivo: `192.168.0.200`
- Sin esto, WoL y los configs de los clientes dejan de funcionar

### 2. Arranque autónomo

Configurar el sistema para que Saturn arranque sesión sin intervención manual. En Windows es autologin, en Linux es arranque directo al escritorio o sin display manager según el setup.

- Requerido para que Docker / los servicios suban solos tras un reinicio

### 3. Tailscale

Instalar Tailscale y unirse al tailnet `minotaur-mizar` con la cuenta `ntonyTR@`.

- Verificar que MagicDNS esté activo
- **Desactivar key expiry para Saturn** en el panel de administración de Tailscale — el servidor no debe pedir reautenticación nunca
- Confirmar que el hostname `saturn` queda registrado en el tailnet

### 4. SSH

Instalar y habilitar el servidor SSH con arranque automático.

- Configurar autenticación **solo por llave pública** — sin password
- Agregar las llaves de todos los clientes autorizados (ver `cybersaturn-network-config.md` para las llaves actuales)
- En Windows, el archivo de llaves para administradores va en una ubicación especial — ver el documento de referencia
- Verificar conexión sin password desde Cassini antes de continuar

### 5. Gestor de servicios

En Windows, instalar **NSSM** antes de configurar Ollama y cloudflared — ambos lo requieren.

En Linux y macOS esto no aplica: usar systemd (Linux) o launchd (macOS) directamente.

### 6. Ollama

Instalar Ollama y configurarlo para que:
- Escuche en todas las interfaces (`0.0.0.0:11434`), no solo localhost
- Arranque automáticamente como servicio del sistema
- Use las variables de entorno correctas (ver `cybersaturn-n8n-telegram-config.md` para los valores exactos)

Variables clave: `OLLAMA_HOST`, `OLLAMA_KEEP_ALIVE`, `OLLAMA_MODELS`, `OLLAMA_CONTEXT_LENGTH`.

En Windows hay un problema conocido con el shortcut que deja el instalador — ver el documento de referencia antes de configurar el servicio.

Abrir el puerto `11434` en el firewall del sistema si aplica.

### 7. Modelos

Descargar los tres modelos necesarios:

- `qwen3:14b` — chat general, Telegram, agente
- `qwen2.5-coder:14b` — código, Continue, OpenCode
- `nomic-embed-text` — embeddings para `@codebase` en Continue

Verificar que la inferencia corre en GPU y que el uso de VRAM es el esperado (~10.7GB con un modelo cargado).

### 8. Sunshine (escritorio remoto)

Instalar Sunshine y configurar usuario y contraseña en su panel web.

En Linux requiere configuración adicional para acceso a la GPU y al display. En Windows es más directo.

Verificar conexión desde Moonlight apuntando a `saturn.minotaur-mizar.ts.net` antes de continuar — necesario para el siguiente paso si se usa Windows.

### 9. Wake on LAN

Tres configuraciones independientes que deben estar todas activas:

1. **BIOS/UEFI** — habilitar WoL por PCI-E / red
2. **NIC del sistema** — habilitar la opción de despertar por magic packet en las propiedades del adaptador
3. **Router** — configurar ARP estático para la MAC de Saturn (sin esto, WoL deja de funcionar después de días sin encender)

Verificar enviando un magic packet desde Cassini con Saturn apagado completamente.

### 10. Docker (runtime de contenedores)

Instalar Docker y verificar que puede correr contenedores Linux.

- En Windows: Docker Desktop — requiere sesión gráfica activa (ya resuelta con el autologin del paso 2). Configurar autostart.
- En Linux: Docker Engine directamente, sin Docker Desktop
- En macOS: Docker Desktop

Verificar que `docker ps` responde antes de continuar.

### 11. n8n

Levantar el contenedor n8n con las variables de entorno correctas y política de restart automático.

Ver `cybersaturn-n8n-telegram-config.md` para el comando exacto de arranque y las variables requeridas — especialmente `WEBHOOK_URL`, `N8N_TRUST_PROXY` y `N8N_PROXY_HOPS` que son críticas para que funcione detrás de Cloudflare Tunnel.

Verificar acceso al panel de n8n en el puerto `5678`.

### 12. Cloudflare Tunnel

Instalar `cloudflared` y conectarlo al tunnel existente `n8n` (Tunnel ID: `dcb83209-98d7-4927-9cac-d655a51a95ea`).

- Las credenciales del tunnel están en `cybersaturn_config/` — restaurarlas antes de arrancar
- Configurar cloudflared como servicio de arranque automático
- En Windows, **no usar** `cloudflared service install` — ver el documento de referencia
- Verificar que `n8n.ntny.dev` responde desde el exterior antes de continuar

### 13. Bot de Telegram — workflow en n8n

Restaurar el workflow en n8n e importar la credencial del bot.

**Opción A (recomendada):** importar el workflow desde el JSON exportado si existe backup.

**Opción B:** reconstruir manualmente los 5 nodos — ver `cybersaturn-n8n-telegram-config.md` para la configuración exacta de cada nodo.

Después de activar el workflow, registrar el webhook en Telegram con la nueva URL si el webhook ID cambió. Verificar con `getWebhookInfo`.

---

## Parte 2 — Clientes (Cassini / Henco)

Estos pasos se hacen en las máquinas cliente, no en Saturn. Saturn debe estar completamente operacional antes de empezar esta parte.

### 14. Tailscale en clientes

Instalar Tailscale en cada cliente y unirse al tailnet `minotaur-mizar` con la misma cuenta.

### 15. SSH config en clientes

Agregar el alias `saturn` en `~/.ssh/config` (o equivalente en Windows) apuntando a `saturn.minotaur-mizar.ts.net`.

Ver `cybersaturn-network-config.md` para la config exacta.

### 16. DNS local LAN (opcional)

Agregar `saturn.lan` apuntando a `192.168.0.200` en el archivo hosts de cada cliente.

- Linux/macOS: `/etc/hosts`
- Windows: `C:\Windows\System32\drivers\etc\hosts`

Solo útil en LAN — para acceso remoto siempre se usa el hostname de Tailscale.

### 17. Continue en VS Code

Instalar la extensión Continue en VS Code y restaurar `~/.continue/config.yaml` desde `cybersaturn_config/`.

Ver `cybersaturn-dev-tools-config.md` para la config completa incluyendo modelos, context providers, autocomplete y todos los prompts custom.

### 18. OpenCode

Instalar OpenCode via npm y restaurar `~/.config/opencode/opencode.json` desde `cybersaturn_config/`.

Ver `cybersaturn-dev-tools-config.md` para la config y notas sobre qué modelo usar para qué tipo de tarea.

---

## Verificación final end-to-end

Antes de dar por completa la reconstrucción:

- [ ] `ssh saturn` conecta sin password desde Cassini y Henco
- [ ] `curl http://saturn.minotaur-mizar.ts.net:11434/api/tags` devuelve los tres modelos
- [ ] La inferencia corre en GPU (verificar con nvidia-smi o equivalente)
- [ ] `n8n.ntny.dev` carga el panel desde el navegador externo
- [ ] Mensaje al bot de Telegram desde el teléfono → responde
- [ ] Moonlight conecta desde Cassini y desde el teléfono
- [ ] WoL funciona con Saturn completamente apagado
- [ ] Autocomplete de Continue funciona en VS Code en Cassini
- [ ] OpenCode responde en terminal en Cassini
- [ ] Reiniciar Saturn y verificar que todo el stack sube solo sin intervención

---

## Notas generales

| Tema | Nota |
|---|---|
| **Linux como alternativa a Windows** | Elimina las complejidades de NSSM, autologin y Docker Desktop. Systemd maneja Ollama y cloudflared nativamente. Además da ~5–8% más de tokens/seg. |
| **Webhook ID de Telegram** | Cambia si se elimina y recrea el workflow en n8n — hay que volver a registrar el webhook con `setWebhook` |
| **Credenciales de Cloudflare** | El tunnel existente puede reutilizarse en una instalación nueva — solo restaurar el JSON de credenciales desde el repo |
| **Volumen de n8n** | El volumen Docker `n8n_data` persiste workflows y credenciales aunque se elimine y recree el contenedor |
| **VRAM al cambiar de modelo** | Si se cambia a una GPU con más VRAM en el futuro, reevaluar `OLLAMA_CONTEXT_LENGTH` — con más VRAM se puede subir el contexto sin penalización de velocidad |
