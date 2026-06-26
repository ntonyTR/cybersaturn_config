# CYBERSATURN — Fuente de verdad: Ollama + n8n + Cloudflare Tunnel
> Última actualización: Junio 2026  
> Estado: Producción — todo operacional

---

## Stack completo

| Componente | Versión | Rol |
|---|---|---|
| **Ollama** | latest | Servidor de inferencia LLM |
| **NSSM** | — | Gestor de servicios Windows para Ollama y cloudflared |
| **Docker Desktop** | v4.78.0 | Runtime para contenedor n8n |
| **n8n** | v2.27.4 | Orquestador / bot de Telegram |
| **cloudflared** | v2026.5.2 | Túnel Cloudflare — expone n8n a internet |
| **Telegram Bot** | — | Interfaz de chat móvil |

---

## Hardware — Saturn

| Parámetro | Valor |
|---|---|
| **Hostname LAN** | `saturn.lan` (vía `/etc/hosts` en clientes) |
| **IP LAN estática** | `192.168.0.200` |
| **Hostname Tailscale** | `saturn.minotaur-mizar.ts.net` |
| **IP Tailscale** | `100.68.149.122` |
| **OS** | Windows 11 LTSC IoT |
| **GPU** | RTX 4070 Super (12GB VRAM) |
| **RAM** | 32GB DDR4 |
| **Autologin** | ✅ Habilitado — requerido para Docker Desktop |

---

## 1. Ollama

### Servicio NSSM

| Parámetro | Valor |
|---|---|
| **Gestor** | NSSM (`C:\Windows\System32\nssm.exe`) |
| **Binario** | `C:\Users\antony\AppData\Local\Programs\Ollama\ollama.exe` |
| **AppParameters** | `serve` |
| **Log stdout** | `C:\Users\antony\.ollama\ollama.log` |
| **Log stderr** | `C:\Users\antony\.ollama\ollama.log` |
| **Start type** | `SERVICE_AUTO_START` |
| **Estado** | RUNNING |

### Variables de entorno (via AppEnvironmentExtra)

| Variable | Valor |
|---|---|
| `OLLAMA_HOST` | `0.0.0.0:11434` |
| `OLLAMA_KEEP_ALIVE` | `-1` |
| `OLLAMA_MODELS` | `C:\Users\antony\.ollama\models` |
| `OLLAMA_CONTEXT_LENGTH` | `16384` |
| `OLLAMA_DEBUG` | `1` |

> ⚠️ Las variables van en `AppEnvironmentExtra` de NSSM — NO en las variables del sistema de Windows. Los servicios NSSM no heredan las variables del sistema.

### Modelos instalados

| Modelo | Rol |
|---|---|
| `qwen3:14b` | Chat general, bot de Telegram, agente |
| `qwen2.5-coder:14b` | Continue (chat + autocomplete), OpenCode |
| `nomic-embed-text` | Embeddings para `@codebase` en Continue |

### Comportamiento de VRAM

- Con `OLLAMA_KEEP_ALIVE=-1` el modelo activo queda anclado en VRAM permanentemente
- Consumo en idle: ~10.6–10.7 GB VRAM
- Para liberar VRAM (gaming): `sc.exe stop ollama`
- `ollama stop` NO funciona con `keep_alive=-1` — descargar modelo via API:
  ```bash
  curl -X POST http://localhost:11434/api/generate -d '{"model":"<nombre>","keep_alive":0}'
  ```

### Benchmark de contexto

| Contexto | tok/s | VRAM | RAM spillover |
|---|---|---|---|
| 32K | ~9 tok/s | 11.5 GB | ~8 GB |
| **16K** | **~23 tok/s** | **10.7 GB** | **~2 GB** |

**Decisión: 16K** — más del doble de velocidad. KV cache genera ~2GB de spillover inevitable con este modelo en 12GB VRAM.

### Endpoint

```
http://saturn.minotaur-mizar.ts.net:11434   ← remoto (Tailscale)
http://saturn.lan:11434                      ← LAN
http://localhost:11434                       ← local en Saturn
http://host.docker.internal:11434            ← desde contenedores Docker
```

### Comandos de administración

```powershell
# Estado
sc.exe query ollama

# Iniciar / detener
sc.exe start ollama
sc.exe stop ollama

# Ver configuración NSSM
nssm get ollama AppEnvironmentExtra

# Ver log (últimas 50 líneas)
Get-Content "C:\Users\antony\.ollama\ollama.log" -Tail 50

# Limpiar log antes de diagnosticar
Clear-Content "C:\Users\antony\.ollama\ollama.log"
```

### Problemas conocidos y soluciones

| Problema | Causa | Solución |
|---|---|---|
| Variables de entorno ignoradas | NSSM no hereda variables del sistema | Pasar via `AppEnvironmentExtra` |
| Puerto 11434 ocupado al arrancar | Instalador deja shortcut en Startup que lanza `ollama app.exe` | Eliminar `C:\Users\antony\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Ollama.lnk` |
| Servicio en estado PAUSED | RivaTuner/RTSS inyecta en procesos GPU y pausa Ollama | En RTSS → `+` → agregar `ollama.exe` → Detection level: None |
| `New-Service` falla con timeout | `ollama.exe serve` no implementa protocolo de servicios Windows | Usar NSSM (método oficial de Ollama para Windows) |
| `Remove-Service` no existe | Windows 11 LTSC IoT — PowerShell limitado | Usar `sc.exe delete <nombre>` |

---

## 2. Docker Desktop + n8n

### Docker Desktop

| Parámetro | Valor |
|---|---|
| **Versión** | v4.78.0 |
| **Autostart** | Shortcut en `C:\Users\antony\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Docker Desktop.lnk` |
| **Backend** | WSL2 — distro `docker-desktop` (VERSION 2) |
| **Requerimiento** | Sesión gráfica activa — funciona via autologin de Saturn |

> ⚠️ Docker login desde SSH falla (credential store de Windows no disponible en sesión SSH). Hacer login desde Moonlight si es necesario.

### Problemas durante instalación de Docker

| Problema | Causa | Solución |
|---|---|---|
| Docker Desktop no arrancaba | Requiere sesión gráfica | Habilitar autologin en Saturn |
| Docker Engine standalone no sirve | Solo corre contenedores Windows sin Docker Desktop | Usar Docker Desktop |
| Virtualización no habilitada | AMD-V/SVM desactivado en BIOS | ASUS PRIME B550M: Advanced → CPU Configuration → SVM Mode → Enabled |
| Virtual Machine Platform faltante | Feature de Windows no habilitado | `dism.exe /online /enable-feature /featurename:VirtualMachinePlatform` |

### Contenedor n8n

**Comando de arranque (fuente de verdad):**

```powershell
docker run -d --name n8n --restart always -p 5678:5678 `
  -v n8n_data:/home/node/.n8n `
  -e N8N_SECURE_COOKIE=false `
  -e WEBHOOK_URL=https://n8n.ntny.dev `
  -e N8N_PROXY_HOPS=1 `
  -e N8N_TRUST_PROXY=true `
  n8nio/n8n
```

**Configuración del contenedor (verificada con `docker inspect`):**

| Parámetro | Valor |
|---|---|
| **Nombre** | `n8n` |
| **Imagen** | `n8nio/n8n` |
| **Versión n8n** | v2.27.4 |
| **Restart policy** | `always` |
| **Puerto** | `5678:5678` |
| **Volumen** | `n8n_data:/home/node/.n8n` |
| `N8N_SECURE_COOKIE` | `false` |
| `WEBHOOK_URL` | `https://n8n.ntny.dev` |
| `N8N_PROXY_HOPS` | `1` |
| `N8N_TRUST_PROXY` | `true` |

> `N8N_TRUST_PROXY=true` es requerido cuando n8n está detrás de Cloudflare Tunnel — sin esto n8n rechaza las requests con 403.  
> `host.docker.internal` resuelve correctamente a Saturn desde dentro del contenedor — no usar `localhost` ni IP de Tailscale para llamar a Ollama.

**Acceso al panel:**
```
http://saturn.lan:5678        ← LAN
http://localhost:5678         ← local en Saturn
```

**Comandos de administración:**

```powershell
# Estado del contenedor
docker ps

# Logs
docker logs n8n
docker logs n8n --tail 50

# Reiniciar
docker restart n8n

# Recrear contenedor (si se necesita cambiar variables)
docker rm -f n8n
docker run -d --name n8n --restart always -p 5678:5678 `
  -v n8n_data:/home/node/.n8n `
  -e N8N_SECURE_COOKIE=false `
  -e WEBHOOK_URL=https://n8n.ntny.dev `
  -e N8N_PROXY_HOPS=1 `
  -e N8N_TRUST_PROXY=true `
  n8nio/n8n
```

> ⚠️ Al recrear el contenedor, el volumen `n8n_data` persiste — los workflows, credenciales y configuración de n8n no se pierden.

---

## 3. Cloudflare Tunnel

### Dominio

| Parámetro | Valor |
|---|---|
| **Dominio** | `ntny.dev` |
| **Registrar** | Cloudflare Registrar |
| **Subdominio activo** | `n8n.ntny.dev` → n8n en Saturn |
| **Uso futuro** | Portfolio de desarrollador en `ntny.dev` |

### Configuración del túnel

| Parámetro | Valor |
|---|---|
| **Tunnel name** | `n8n` |
| **Tunnel ID** | `dcb83209-98d7-4927-9cac-d655a51a95ea` |
| **Credentials file** | `C:\Users\antony\.cloudflared\dcb83209-98d7-4927-9cac-d655a51a95ea.json` |
| **Cert** | `C:\Users\antony\.cloudflared\cert.pem` |
| **Config** | `C:\Users\antony\.cloudflared\config.yml` |
| **DNS** | CNAME `n8n.ntny.dev` → tunnel ID (creado con `cloudflared tunnel route dns`) |

### `C:\Users\antony\.cloudflared\config.yml`

```yaml
tunnel: dcb83209-98d7-4927-9cac-d655a51a95ea
credentials-file: C:\Users\antony\.cloudflared\dcb83209-98d7-4927-9cac-d655a51a95ea.json

ingress:
  - hostname: n8n.ntny.dev
    service: http://localhost:5678
  - service: http_status:404
```

### Servicio NSSM

| Parámetro | Valor |
|---|---|
| **Servicio** | `cloudflared` |
| **Binario** | `C:\Program Files (x86)\cloudflared\cloudflared.exe` |
| **AppParameters** | `tunnel run n8n` |
| **Start type** | `SERVICE_AUTO_START` |
| **SERVICE_START_NAME** | `LocalSystem` |
| **Estado** | RUNNING |

> ⚠️ `cloudflared service install` falla con exit code 1067 — registra el servicio sin argumentos y no sabe qué túnel correr. Siempre usar NSSM con `AppParameters: tunnel run n8n`.

### Comandos de administración

```powershell
# Estado
sc.exe query cloudflared

# Reiniciar
sc.exe stop cloudflared
sc.exe start cloudflared

# Verificar configuración NSSM
nssm get cloudflared AppParameters
nssm get cloudflared Application
```

---

## 4. Bot de Telegram

| Parámetro | Valor |
|---|---|
| **Nombre** | CyberSaturn |
| **Username** | `@cybersaturn_bot` |
| **Token** | `8006609858:AAH3h6zI9bpYt39AZ7buC7kjNfm9xnzztJQ` |
| **Webhook URL** | `https://n8n.ntny.dev/webhook/4b33f5a5-62df-4edf-8b83-1dfce53f4546/webhook` |
| **Webhook ID** | `4b33f5a5-62df-4edf-8b83-1dfce53f4546` — fijo, no cambia a menos que se elimine y recree el workflow en n8n |

### Comandos de gestión del webhook

```powershell
# Verificar webhook activo
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"

# Registrar webhook (solo si se pierde)
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://n8n.ntny.dev/webhook/4b33f5a5-62df-4edf-8b83-1dfce53f4546/webhook"

# Eliminar webhook
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/deleteWebhook"
```

---

## 5. Workflow de n8n

### Estructura

```
Telegram Trigger
      ↓
Send Chat Action (typing)
      ↓
HTTP Request → Ollama
      ↓
Code (limpia <think> + convierte Markdown a HTML)
      ↓
Send Message → Telegram
```

### Nodo 1 — Telegram Trigger

| Campo | Valor |
|---|---|
| Trigger On | Message |
| Credential | Telegram account (token del bot) |

### Nodo 2 — Send Chat Action

| Campo | Valor |
|---|---|
| Resource | Message |
| Operation | Send Chat Action |
| Chat ID | `{{ $json.message.chat.id }}` |
| Action | Typing |

### Nodo 3 — HTTP Request

| Campo | Valor |
|---|---|
| Method | POST |
| URL | `http://host.docker.internal:11434/api/chat` |
| Body Content Type | JSON |
| On Error | Continue |

Body:
```json
{
  "model": "qwen3:14b",
  "messages": [
    {
      "role": "system",
      "content": "Responde siempre en español. Para formato usa solo Markdown compatible con Telegram: *negrita*, _cursiva_, `código inline`, ```bloque de código```. No uses headers (#), ni listas con guiones (-), ni otros elementos Markdown."
    },
    {
      "role": "user",
      "content": "{{ $('Telegram Trigger').item.json.message.text }}"
    }
  ],
  "stream": false
}
```

> ⚠️ Usar `$('Telegram Trigger').item.json.message.text` — NO `$json.message.text`. Las referencias cross-nodo requieren el nombre del nodo origen explícito.

### Nodo 4 — Code (JavaScript)

```javascript
const httpOutput = $('HTTP Request').first();

if (httpOutput.json.error) {
  return { json: { text: "⚠️ Saturn no está disponible en este momento. Intenta más tarde." } };
}

const content = httpOutput.json.message.content;
const cleaned = content.replace(/<think>[\s\S]*?<\/think>/g, '').trim();

const html = cleaned
  .replace(/&/g, '&amp;')
  .replace(/</g, '&lt;')
  .replace(/>/g, '&gt;')
  .replace(/```(\w+)?\n?([\s\S]*?)```/g, '<pre><code>$2</code></pre>')
  .replace(/`([^`]+)`/g, '<code>$1</code>')
  .replace(/\*\*([^*]+)\*\*/g, '<b>$1</b>')
  .replace(/\*([^*]+)\*/g, '<i>$1</i>')
  .replace(/^### .+$/gm, '<b>$&</b>'.replace(/### /g, ''))
  .replace(/^## .+$/gm, '<b>$&</b>'.replace(/## /g, ''))
  .replace(/^# .+$/gm, '<b>$&</b>'.replace(/# /g, ''))
  .replace(/^- (.+)$/gm, '• $1');

return { json: { text: html } };
```

### Nodo 5 — Send Message

| Campo | Valor |
|---|---|
| Credential | Telegram account |
| Resource | Message |
| Operation | Send Message |
| Chat ID | `{{ $('Telegram Trigger').item.json.message.chat.id }}` |
| Text | `{{ $json.text }}` |
| Parse Mode | HTML |

> Parse Mode HTML preferido sobre Markdown Legacy — Markdown Legacy tiene problemas de escaping de caracteres especiales.

---

## 6. Secuencia de arranque tras reinicio de Saturn

Con el setup actual, todo arranca automáticamente sin intervención manual:

1. Windows inicia → autologin como `antony`
2. Docker Desktop arranca (shortcut en Startup del usuario)
3. Contenedor `n8n` arranca automáticamente (`--restart always`)
4. Servicio `cloudflared` arranca automáticamente (NSSM `AUTO_START`)
5. Servicio `ollama` arranca automáticamente (NSSM `AUTO_START`)
6. Túnel establece conexión con Cloudflare
7. Bot disponible en Telegram

**No se requiere intervención manual.**

---

## 7. Diagnóstico rápido

### Checklist si el bot no responde

```powershell
# 1. Verificar servicios
sc.exe query ollama
sc.exe query cloudflared

# 2. Verificar contenedor n8n
docker ps

# 3. Verificar que Ollama responde
curl http://localhost:11434/api/tags

# 4. Verificar webhook de Telegram
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"

# 5. Ver logs de Ollama
Get-Content "C:\Users\antony\.ollama\ollama.log" -Tail 50

# 6. Ver logs de n8n
docker logs n8n --tail 50
```

### Checklist si Ollama está lento o pausado

```powershell
# Verificar si RTSS está pausando el proceso
sc.exe query ollama   # debe estar RUNNING, no PAUSED

# Si está PAUSED: abrir RTSS → ollama.exe → Detection level: None
# Reiniciar el servicio
sc.exe stop ollama
sc.exe start ollama
```

---

## 8. Limitaciones conocidas

| Limitación | Detalle |
|---|---|
| **Bot stateless** | No hay memoria conversacional entre mensajes — cada mensaje es independiente |
| **qwen3 thinking expuesto** | Si el usuario manda `/think`, el modelo puede exponer razonamiento como texto plano — el filtro de `<think>` tags no cubre ese caso |
| **Docker requiere GUI** | Docker Desktop necesita sesión gráfica activa — depende del autologin de Saturn |
| **Docker login desde SSH falla** | El credential store de Windows no está disponible en sesión SSH — usar Moonlight para login |

---

## 9. Pendientes / próximos pasos

| Item | Descripción |
|---|---|
| **Memoria conversacional** | Agregar contexto por sesión al bot con comando `/clearcontext` |
| **Fase 3.3** | Replicar Continue + OpenCode en laptop del trabajo |
| **Portfolio** | `ntny.dev` disponible para uso futuro |
| **AIFinance** | Extender workflow de Telegram con Google Sheets + clasificación de gastos |
| **RAG / second brain** | ChromaDB o Qdrant + `nomic-embed-text` |
| **Upgrade GPU** | RTX 4090 cuando aparezcan bottlenecks reales de VRAM (~20,000 MXN neto) |
