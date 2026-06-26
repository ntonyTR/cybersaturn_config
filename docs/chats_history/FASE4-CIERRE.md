# FASE 4 — CIERRE: Bot de Telegram con n8n + Ollama

> Completada: Junio 2026

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
⚠️ ngrok con plan free — URL cambia en cada reinicio (ver Notas)

---

## Stack instalado en Saturn

| Componente | Detalle |
|---|---|
| **Docker Desktop** | v4.78.0 — autostart via shortcut en Startup del usuario |
| **n8n** | v2.27.4 — contenedor Docker con volumen persistente |
| **ngrok** | v3.39.8 — túnel HTTP para exponer n8n a internet |

---

## Configuración de Docker

### Comando de arranque del contenedor n8n

```powershell
docker run -d --name n8n --restart always -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_SECURE_COOKIE=false \
  -e WEBHOOK_URL=https://<ngrok-url> \
  -e N8N_PROXY_HOPS=1 \
  n8nio/n8n
```

### Notas de Docker

- Docker Desktop requiere sesión gráfica activa — funciona gracias al autologin de Saturn
- El shortcut de autostart está en:
  ```
  C:\Users\antony\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Docker Desktop.lnk
  ```
- El contenedor tiene `--restart always` — sobrevive reinicios mientras Docker Desktop esté corriendo
- Docker login desde SSH falla (credential store de Windows no disponible en sesión SSH) — hacer login desde Moonlight

### Problemas encontrados durante instalación de Docker

| Problema | Causa | Solución |
|---|---|---|
| Docker Desktop no arrancaba sin GUI | Requiere sesión gráfica activa | Autologin en Saturn |
| Docker Engine standalone solo corre contenedores Windows | Sin Docker Desktop no hay backend Linux | Usar Docker Desktop |
| Virtualización no habilitada | AMD-V/SVM desactivado en BIOS | ASUS PRIME B550M: Advanced → CPU Configuration → SVM Mode → Enabled |
| Virtual Machine Platform no habilitado | Componente Windows faltante | `dism.exe /online /enable-feature /featurename:VirtualMachinePlatform` |

---

## Bot de Telegram

| Parámetro | Valor |
|---|---|
| **Nombre** | CyberSaturn |
| **Username** | @cybersaturn_bot |
| **Token** | Guardado en credenciales de n8n |

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
      "content": "{{ $json.message.text }}"
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

## ngrok

### Arranque

```powershell
ngrok http 5678
```

### Registrar webhook en Telegram (requerido después de cada reinicio de ngrok)

```powershell
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://<ngrok-url>/webhook/4b33f5a5-62df-4edf-8b83-1dfce53f4546/webhook"
```

### Verificar webhook registrado

```powershell
Invoke-WebRequest -UseBasicParsing -Uri "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

---

## Notas

- **ngrok plan free:** la URL pública cambia en cada reinicio de ngrok. Cada vez que se reinicie Saturn hay que:
  1. Correr `ngrok http 5678`
  2. Obtener la nueva URL de `http://127.0.0.1:4040`
  3. Actualizar `WEBHOOK_URL` en el contenedor n8n (`docker rm -f n8n && docker run ... -e WEBHOOK_URL=<nueva-url>`)
  4. Registrar el nuevo webhook con `setWebhook`
- **Alternativa a largo plazo:** ngrok paid (URL fija) o exponer directamente el puerto 5678 con HTTPS via Let's Encrypt
- **qwen3:14b thinking:** el modelo a veces expone su razonamiento como texto plano cuando el usuario manda comandos como `/think`. El filtro de `<think>` tags del nodo Code no cubre este caso — es comportamiento del modelo
- **`host.docker.internal`** resuelve correctamente a Saturn desde dentro del contenedor n8n — no usar `localhost` ni la IP de Tailscale

---

## Contexto para siguiente chat

**Estado actual:** Fase 4 completa y funcional.

**Pendiente inmediato:**
- Resolver la URL fija de ngrok para no tener que re-registrar el webhook en cada reinicio

**Fases siguientes:**
- No hay fases formales pendientes en el plan original de CyberSaturn
- AIFinance: extender el workflow de Telegram con nodos adicionales (Google Sheets, clasificación de gastos, etc.)
- Fase 3.3: Replicar Continue + OpenCode en laptop del trabajo (pendiente desde Fase 3.1)
