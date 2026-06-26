# FASE 4 — CIERRE: Bot de Telegram con n8n + Ollama

> Completada: Junio 2026  
> Actualizada: Junio 2026 — ngrok reemplazado por Cloudflare Tunnel

---

## Resultado

✅ Docker Desktop instalado y con autostart en Saturn  
✅ n8n corriendo en Docker con persistencia  
✅ Bot de Telegram creado y conectado  
✅ Workflow completo: Telegram → n8n → Ollama → Telegram  
✅ Typing indicator ("escribiendo...")  
✅ Filtro de thinking de qwen3  
✅ Conversión de Markdown a HTML para Telegram  
✅ Error handling cuando Saturn no responde  
✅ Cloudflare Tunnel con URL fija permanente (`n8n.ntny.dev`)  
✅ cloudflared corriendo como servicio NSSM autostart  
✅ Dominio `ntny.dev` registrado en Cloudflare Registrar  

---

## Stack instalado en Saturn

| Componente | Detalle |
|---|---|
| **Docker Desktop** | v4.78.0 — autostart via shortcut en Startup del usuario |
| **n8n** | v2.27.4 — contenedor Docker con volumen persistente |
| **cloudflared** | v2026.5.2 — túnel Cloudflare, servicio NSSM autostart |

---

## Dominio

| Parámetro | Valor |
|---|---|
| **Dominio** | `ntny.dev` |
| **Registrar** | Cloudflare Registrar |
| **Subdominio n8n** | `n8n.ntny.dev` |
| **Uso futuro** | Portfolio de desarrollador en `ntny.dev` |

---

## Cloudflare Tunnel

### Configuración del túnel

| Parámetro | Valor |
|---|---|
| **Tunnel name** | `n8n` |
| **Tunnel ID** | `dcb83209-98d7-4927-9cac-d655a51a95ea` |
| **Credentials** | `C:\Users\antony\.cloudflared\dcb83209-98d7-4927-9cac-d655a51a95ea.json` |
| **Cert** | `C:\Users\antony\.cloudflared\cert.pem` |
| **Config** | `C:\Users\antony\.cloudflared\config.yml` |
| **DNS** | CNAME `n8n.ntny.dev` → tunnel ID (creado con `tunnel route dns`) |

### config.yml

```yaml
tunnel: dcb83209-98d7-4927-9cac-d655a51a95ea
credentials-file: C:\Users\antony\.cloudflared\dcb83209-98d7-4927-9cac-d655a51a95ea.json

ingress:
  - hostname: n8n.ntny.dev
    service: http://localhost:5678
  - service: http_status:404
```

### Servicio Windows (NSSM)

| Parámetro | Valor |
|---|---|
| **Servicio** | `cloudflared` |
| **Binario** | `C:\Program Files (x86)\cloudflared\cloudflared.exe` |
| **Gestor** | NSSM (`C:\Windows\System32\nssm.exe`) |
| **AppParameters** | `tunnel run n8n` |
| **START_TYPE** | `AUTO_START` |
| **SERVICE_START_NAME** | `LocalSystem` |
| **AppEnvironmentExtra** | (ninguno — credenciales via config.yml) |

### Comandos de administración

```powershell
# Estado
sc.exe query cloudflared

# Reiniciar
sc.exe stop cloudflared
sc.exe start cloudflared

# Verificar configuración NSSM
nssm get cloudflared AppParameters
```

### Por qué NSSM y no `cloudflared service install`

`cloudflared service install` registra el servicio sin argumentos — esto hace que falle con exit code 1067 porque necesita `tunnel run <nombre>` para saber qué túnel ejecutar. NSSM permite pasar `tunnel run n8n` como `AppParameters` explícitamente.

---

## Configuración de Docker

### Comando de arranque del contenedor n8n

```powershell
docker run -d --name n8n --restart always -p 5678:5678 `
  -v n8n_data:/home/node/.n8n `
  -e N8N_SECURE_COOKIE=false `
  -e WEBHOOK_URL=https://n8n.ntny.dev `
  -e N8N_PROXY_HOPS=1 `
  -e N8N_TRUST_PROXY=true `
  n8nio/n8n
```

### Variables de entorno del contenedor

| Variable | Valor |
|---|---|
| `N8N_SECURE_COOKIE` | `false` |
| `WEBHOOK_URL` | `https://n8n.ntny.dev` |
| `N8N_PROXY_HOPS` | `1` |
| `N8N_TRUST_PROXY` | `true` |

### Notas de Docker

- Docker Desktop requiere sesión gráfica activa — funciona gracias al autologin de Saturn
- El shortcut de autostart está en:
  ```
  C:\Users\antony\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Docker Desktop.lnk
  ```
- El contenedor tiene `--restart always` — sobrevive reinicios mientras Docker Desktop esté corriendo
- Docker login desde SSH falla (credential store de Windows no disponible en sesión SSH) — hacer login desde Moonlight
- `host.docker.internal` resuelve correctamente a Saturn desde dentro del contenedor — no usar `localhost` ni IP de Tailscale

---

## Bot de Telegram

| Parámetro | Valor |
|---|---|
| **Nombre** | CyberSaturn |
| **Username** | @cybersaturn_bot |
| **Token** | Guardado en credenciales de n8n |
| **Webhook URL** | `https://n8n.ntny.dev/webhook/4b33f5a5-62df-4edf-8b83-1dfce53f4546/webhook` |

### Verificar webhook activo

```powershell
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

### Registrar webhook (solo si se pierde)

```powershell
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://n8n.ntny.dev/webhook/4b33f5a5-62df-4edf-8b83-1dfce53f4546/webhook"
```

### Eliminar webhook

```powershell
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/deleteWebhook"
```

---

## Workflow de n8n

### Estructura

```
Telegram Trigger
      ↓
Send Chat Action (typing)
      ↓
HTTP Request → Ollama
      ↓
Code (limpia thinking + convierte MD a HTML)
      ↓
Send Message → Telegram
```

### Nodo 1 — Telegram Trigger

- **Trigger On:** Message
- **Credential:** Telegram account (token del bot)

### Nodo 2 — Send Chat Action

- **Resource:** Message
- **Operation:** Send Chat Action
- **Chat ID:** `{{ $json.message.chat.id }}`
- **Action:** Typing

### Nodo 3 — HTTP Request

- **Method:** POST
- **URL:** `http://host.docker.internal:11434/api/chat`
- **Body Content Type:** JSON
- **On Error:** Continue
- **Body:**
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

- **Credential:** Telegram account
- **Resource:** Message
- **Operation:** Send Message
- **Chat ID:** `{{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Text:** `{{ $json.text }}`
- **Parse Mode:** HTML

---

## Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| `cloudflared` instalado pero no en PATH | WinGet lo instaló en `Program Files (x86)` sin agregar al PATH | Agregar manualmente con `SetEnvironmentVariable` |
| `cloudflared service install` falla (exit 1067) | El servicio se registra sin argumentos — no sabe qué túnel correr | Instalar via NSSM con `AppParameters: tunnel run n8n` |
| ngrok URL cambiaba en cada reinicio | Plan free de ngrok no tiene URL fija | Reemplazado por Cloudflare Tunnel con dominio propio |
| n8n devolvía 403 detrás del túnel | Cloudflare Tunnel agrega headers de proxy que n8n rechaza por defecto | `N8N_TRUST_PROXY=true` y `N8N_PROXY_HOPS=1` |

---

## Comportamiento en reinicios de Saturn

Con el setup actual, tras un reinicio de Saturn:

1. Windows inicia → autologin
2. Docker Desktop arranca (shortcut en Startup)
3. Contenedor n8n arranca automáticamente (`--restart always`)
4. Servicio `cloudflared` arranca automáticamente (NSSM AUTO_START)
5. Túnel establece conexión con Cloudflare
6. Bot disponible en Telegram

**No se requiere intervención manual.**

---

## Notas

- `N8N_TRUST_PROXY=true` es requerido cuando n8n está detrás de Cloudflare Tunnel — sin esto n8n rechaza las requests con 403
- El webhook ID (`4b33f5a5-62df-4edf-8b83-1dfce53f4546`) es fijo — no cambia a menos que se elimine y recree el workflow en n8n
- qwen3:14b expone su razonamiento como texto plano cuando el usuario manda `/think` — el filtro de `<think>` tags no cubre este caso, es comportamiento del modelo
- El dominio `ntny.dev` está disponible para uso futuro como portfolio de desarrollador

---

## Fases siguientes

| Fase | Contenido |
|---|---|
| **Opcional** | Memoria conversacional en el bot de Telegram (contexto por sesión + `/clearcontext`) |
| **Fase 3.3** | Replicar Continue + OpenCode en laptop del trabajo |
| **Futuro** | Portfolio en `ntny.dev` |
| **Futuro** | AIFinance — extender workflow con Google Sheets + clasificación de gastos |
