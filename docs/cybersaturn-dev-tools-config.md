# CYBERSATURN — Fuente de verdad: Entorno de desarrollo
> Última actualización: Junio 2026  
> Estado: Producción — todo operacional en Cassini

---

## Resumen del stack

| Herramienta | Versión | Rol | Dónde corre |
|---|---|---|---|
| **Ollama** | latest | Servidor de inferencia LLM | Saturn (Windows 11) |
| **Continue** | v2.0.0 | AI en VS Code — chat, edit, autocomplete | Cassini (Fedora) |
| **OpenCode** | v1.17.10 | Agente de terminal | Cassini (Fedora) |
| **Node.js** | v24.17.0 | Runtime de OpenCode | Cassini (Fedora) |
| **npm** | v11.13.0 | — | Cassini (Fedora) |

---

## 1. Red y endpoints

| Contexto | Endpoint Ollama |
|---|---|
| Desde Cassini (LAN) | `http://saturn.lan:11434` |
| Desde Cassini (Tailscale / remoto) | `http://saturn.minotaur-mizar.ts.net:11434` |
| Desde contenedores Docker en Saturn | `http://host.docker.internal:11434` |
| Local en Saturn | `http://localhost:11434` |
| API compatible OpenAI (para OpenCode) | `http://saturn.minotaur-mizar.ts.net:11434/v1` |

> Continue y OpenCode usan siempre el hostname de Tailscale — funciona tanto en LAN como en remoto con el mismo string.

---

## 2. Ollama en Saturn

### Servicio NSSM

| Parámetro | Valor |
|---|---|
| **Gestor** | NSSM (`C:\Windows\System32\nssm.exe`) |
| **Binario** | `C:\Users\antony\AppData\Local\Programs\Ollama\ollama.exe` |
| **AppParameters** | `serve` |
| **Log** | `C:\Users\antony\.ollama\ollama.log` |
| **Start type** | `SERVICE_AUTO_START` |

### Variables de entorno (AppEnvironmentExtra)

| Variable | Valor |
|---|---|
| `OLLAMA_HOST` | `0.0.0.0:11434` |
| `OLLAMA_KEEP_ALIVE` | `-1` |
| `OLLAMA_MODELS` | `C:\Users\antony\.ollama\models` |
| `OLLAMA_CONTEXT_LENGTH` | `16384` |
| `OLLAMA_DEBUG` | `1` |

> ⚠️ Variables en `AppEnvironmentExtra` de NSSM — NO en variables del sistema de Windows. Los servicios NSSM no heredan las variables del sistema.

### Modelos instalados

| Modelo | Rol en dev |
|---|---|
| `qwen2.5-coder:14b` | Continue — chat, edit, apply, autocomplete. Modelo default en OpenCode |
| `qwen3:14b` | Continue — chat, edit, apply. OpenCode — tool calling confiable |
| `nomic-embed-text` | Continue — embeddings para `@codebase` |

### Comportamiento de VRAM

- `OLLAMA_KEEP_ALIVE=-1` ancla el modelo en VRAM permanentemente
- Consumo en idle: ~10.6–10.7 GB / 12 GB VRAM
- Para liberar VRAM (gaming): `sc.exe stop ollama`
- `ollama stop` NO funciona con `keep_alive=-1` — descargar via API:
  ```bash
  curl -X POST http://localhost:11434/api/generate -d '{"model":"<nombre>","keep_alive":0}'
  ```

### Benchmark de contexto

| Contexto | tok/s | VRAM | RAM spillover |
|---|---|---|---|
| 32K | ~9 tok/s | 11.5 GB | ~8 GB |
| **16K** | **~23 tok/s** | **10.7 GB** | **~2 GB** |

**Decisión: 16K** — más del doble de velocidad. KV cache genera ~2GB de spillover inevitable. Suficiente para Continue + proyectos C# ASP.NET.

### Comandos de administración (desde Saturn o SSH)

```powershell
# Estado
sc.exe query ollama

# Iniciar / detener
sc.exe start ollama
sc.exe stop ollama

# Ver variables NSSM
nssm get ollama AppEnvironmentExtra

# Ver log
Get-Content "C:\Users\antony\.ollama\ollama.log" -Tail 50

# Limpiar log
Clear-Content "C:\Users\antony\.ollama\ollama.log"
```

### Problemas conocidos y soluciones

| Problema | Causa | Solución |
|---|---|---|
| Variables de entorno ignoradas | NSSM no hereda variables del sistema | Pasar via `AppEnvironmentExtra` |
| Puerto 11434 ocupado al arrancar | Instalador deja shortcut en Startup que lanza `ollama app.exe` | Eliminar `C:\Users\antony\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Ollama.lnk` |
| Servicio en estado PAUSED | RivaTuner/RTSS inyecta en procesos GPU y pausa Ollama | RTSS → `+` → agregar `ollama.exe` → Detection level: None |
| `ollama stop` no libera VRAM | `OLLAMA_KEEP_ALIVE=-1` ancla el modelo | Usar API con `keep_alive:0` |
| `New-Service` falla con timeout | `ollama.exe serve` no implementa protocolo de servicios Windows | Usar NSSM |
| `Remove-Service` no existe | PowerShell limitado en Windows 11 LTSC IoT | Usar `sc.exe delete <nombre>` |

---

## 3. Continue

### Instalación

| Parámetro | Valor |
|---|---|
| **Versión** | v2.0.0 |
| **Editor** | VS Code (Cassini) |
| **Config** | `~/.continue/config.yaml` |
| **Schema** | v1 |
| **Ollama local en Cassini** | Deshabilitado (`systemctl disable ollama`) — todo el tráfico va a Saturn |

> Continue v2.0.0 es la versión final del equipo original — fue adquirido por Cursor en junio 2026. El repo es read-only pero la extensión es estable y funcional.

### `~/.continue/config.yaml` (completo y verificado)

```yaml
name: CyberSaturn
version: 1.0.0
schema: v1

models:
  - name: Qwen2.5 Coder 14B
    provider: ollama
    model: qwen2.5-coder:14b
    apiBase: http://saturn.minotaur-mizar.ts.net:11434
    contextLength: 16384
    roles:
      - chat
      - edit
      - apply
      - autocomplete
    autocomplete:
      debounceDelay: 1000
      maxPromptTokens: 300
      multilineCompletions: "never"

  - name: Qwen3 14B
    provider: ollama
    model: qwen3:14b
    apiBase: http://saturn.minotaur-mizar.ts.net:11434
    contextLength: 16384
    roles:
      - chat
      - edit
      - apply

  - name: Nomic Embed
    provider: ollama
    model: nomic-embed-text
    apiBase: http://saturn.minotaur-mizar.ts.net:11434
    roles:
      - embed

context:
  - provider: code
  - provider: diff
  - provider: terminal
  - provider: codebase
    params:
      nRetrievedFiles: 10

prompts:
  - name: review
    description: Revisa el código seleccionado en C#
    prompt: |
      Revisa el siguiente código en C# y señala:
      1. Code smells o violaciones de principios SOLID
      2. Posibles excepciones no manejadas
      3. Sugerencias concretas de mejora
      No reescribas el código completo, solo señala los puntos.

  - name: docs
    description: Genera documentación XML para el código seleccionado
    prompt: |
      Genera documentación XML completa para el siguiente código C#.
      Usa el formato estándar de C# con <summary>, <param>, <returns> y <exception> donde aplique.
      Solo devuelve los comentarios XML, no reescribas el código.

  - name: test
    description: Genera unit tests para el código seleccionado
    prompt: |
      Genera unit tests para el siguiente código C# usando xUnit.
      Cubre los casos felices, casos límite y posibles excepciones.
      Usa nombres descriptivos en el formato Metodo_Escenario_ResultadoEsperado.
      Mockea las dependencias con Moq donde sea necesario.

  - name: endpoint
    description: Genera un endpoint REST completo en ASP.NET
    prompt: |
      Basándote en el código seleccionado, genera un endpoint REST completo en ASP.NET Core que incluya:
      1. Controller action con atributos de ruta y método HTTP correctos
      2. Validación del request con Data Annotations o FluentValidation
      3. Manejo de errores con respuestas HTTP apropiadas (400, 404, 500)
      4. Documentación XML básica
      Sigue el estilo del código existente.

  - name: explain
    description: Explica el código seleccionado en detalle
    prompt: |
      Explica el siguiente código de forma clara y concisa:
      1. Qué hace y cuál es su propósito
      2. Cómo funciona paso a paso
      3. Dependencias o efectos secundarios importantes
      No incluyas sugerencias de mejora a menos que haya algo crítico.

  - name: fix
    description: Identifica y corrige bugs en el código seleccionado
    prompt: |
      Analiza el siguiente código en busca de bugs, incluyendo:
      1. Errores lógicos
      2. Null reference potenciales
      3. Race conditions o problemas de concurrencia en código async
      4. Manejo incorrecto de excepciones
      Para cada problema encontrado: señala la línea, explica el bug y propón la corrección.

  - name: migrate
    description: Genera una migración de Entity Framework para el modelo seleccionado
    prompt: |
      Basándote en el modelo de Entity Framework Core seleccionado:
      1. Identifica qué cambios requeriría una migración
      2. Genera el código de migración Up() y Down() correspondiente
      3. Señala si hay datos existentes que podrían verse afectados

  - name: secure
    description: Revisa el código seleccionado en busca de vulnerabilidades de seguridad
    prompt: |
      Revisa el siguiente código en busca de vulnerabilidades de seguridad, enfocándote en:
      1. Inyección SQL o NoSQL
      2. Exposición de datos sensibles en responses o logs
      3. Validación insuficiente de inputs
      4. Problemas de autorización o autenticación
      5. OWASP Top 10 relevantes para APIs REST
      Señala la línea y propón la corrección para cada problema encontrado.

  - name: docmd
    description: Genera documentación en Markdown del código seleccionado
    prompt: |
      Genera documentación técnica en un bloque de texto en formato Markdown para el siguiente código.
      Incluye: propósito, parámetros, valor de retorno, ejemplos de uso y notas relevantes.
      Si algo no está claro o es ambiguo, pregunta antes de asumir.
      No documentes lo que no entiendas — pregunta primero.

  - name: guide
    description: Guía de dev senior sobre el código o tarea, sin modificar nada
    prompt: |
      Actúa como dev senior. No modifiques ni reescribas el código.
      Tu rol es únicamente orientar:
      1. Si te muestro código: explica qué hace, qué patrones usa y por qué están bien o mal aplicados
      2. Si te describo algo que quiero hacer: explica cómo hacerlo siguiendo los patrones del código existente
      Sé directo. Si hay varias formas, indica cuál recomendarías y por qué.
      No generes código a menos que sea un ejemplo mínimo ilustrativo.

  - name: vuereview
    description: Revisa un componente Vue.js/TypeScript
    prompt: |
      Revisa el siguiente componente Vue.js con TypeScript y señala:
      1. Violaciones de las convenciones de Vue 3 Composition API
      2. Props sin tipado correcto o sin validación
      3. Reactivity pitfalls (watchers innecesarios, computed mal usados)
      4. Problemas de performance (re-renders innecesarios, listas sin key)
      5. Manejo incorrecto de eventos o emits
      No reescribas el componente, solo señala los puntos con la línea correspondiente.

  - name: reactreview
    description: Revisa un componente React/TypeScript
    prompt: |
      Revisa el siguiente componente React con TypeScript y señala:
      1. Violaciones de las reglas de hooks
      2. Props sin tipado correcto o interfaces faltantes
      3. useEffect con dependencias incorrectas o faltantes
      4. Re-renders innecesarios (falta de memo, useCallback, useMemo)
      5. Manejo incorrecto de estado o side effects
      No reescribas el componente, solo señala los puntos con la línea correspondiente.

  - name: types
    description: Genera tipos TypeScript para el código seleccionado
    prompt: |
      Basándote en el código seleccionado, genera los tipos TypeScript necesarios:
      1. Interfaces o types para los objetos de datos
      2. Tipos para props de componentes si aplica
      3. Tipos para respuestas de API si aplica
      Usa tipos estrictos — evita any. Si algo es ambiguo, pregunta antes de asumir.
```

### Modelos y roles

| Modelo | chat | edit | apply | autocomplete | embed |
|---|---|---|---|---|---|
| `qwen2.5-coder:14b` | ✅ | ✅ | ✅ | ✅ | — |
| `qwen3:14b` | ✅ | ✅ | ✅ | — | — |
| `nomic-embed-text` | — | — | — | — | ✅ |

### Configuración de autocomplete

| Parámetro | Valor | Motivo |
|---|---|---|
| `debounceDelay` | `1000` | No interrumpe el flujo de escritura |
| `maxPromptTokens` | `300` | Suficiente para sugerencias de una línea |
| `multilineCompletions` | `"never"` | Mantiene control — sin sugerencias de bloque |

> Si se quiere abrir multiline en el futuro: subir `maxPromptTokens` a 400–500 al mismo tiempo.

### Context providers

| Provider | Qué manda | Cuándo usarlo |
|---|---|---|
| `@problems` | Errores activos de Roslyn | Debugging de errores de compilación |
| `@terminal` | Últimas líneas del terminal | Errores de runtime |
| `@diff` | Cambios en staging de git | "Revisa lo que acabo de cambiar" |
| `@code` | Función o clase seleccionada | Implementar algo en un bloque conocido |
| `@file` | Archivo completo | Archivos pequeños con contexto total necesario |
| `@codebase` | Chunks relevantes por embeddings | "Cómo funciona X en todo el proyecto" |

> `@file` reproduce el contenido completo en el chat — para overviews usar `@code` con selección.  
> `@codebase` depende de la calidad de la pregunta — preguntas específicas traen fragmentos relevantes, preguntas vagas traen ruido.  
> `@codebase` requiere que el índice de embeddings esté actualizado — se construye automáticamente la primera vez e incrementalmente después.

### Prompts custom (slash commands)

| Comando | Descripción |
|---|---|
| `/review` | Code review C# — SOLID, excepciones, mejoras |
| `/docs` | Documentación XML para C# |
| `/test` | Unit tests con xUnit + Moq |
| `/endpoint` | Endpoint REST completo en ASP.NET Core |
| `/explain` | Explicación detallada del código seleccionado |
| `/fix` | Identificación y corrección de bugs |
| `/migrate` | Migración de Entity Framework Core |
| `/secure` | Revisión de seguridad — OWASP Top 10 para APIs |
| `/docmd` | Documentación técnica en Markdown |
| `/guide` | Orientación de dev senior sin modificar código |
| `/vuereview` | Revisión de componente Vue 3 + TypeScript |
| `/reactreview` | Revisión de componente React + TypeScript |
| `/types` | Generación de tipos TypeScript estrictos |

> Los slash commands del `config.json` legacy están deprecados en schema v1 — los prompts van directamente en `config.yaml`.  
> Flujo recomendado para `/docmd`: crear el `.md` vacío, abrirlo en VS Code, usar `/edit` para que Continue escriba en él.

### Modos de Continue

| Modo | Tools disponibles | Estado con modelos 14B |
|---|---|---|
| **Chat** | Ninguna | ✅ Funcional — uso principal |
| **Plan** | Solo lectura (read files, search, diff) | ⚠️ Funcional con limitaciones — modelos no verificados |
| **Agent** | Todas (crear/editar archivos, terminal) | ⚠️ Inconsistente — requiere 30B+ para uso confiable |

> Agent mode con `qwen2.5-coder:14b` usa XML fallback para tool calling — puede fallar en loops complejos. Para flujos agénticos confiables usar OpenCode.

### Atajos de teclado principales

| Atajo | Acción |
|---|---|
| `Ctrl+L` | Abrir panel de chat |
| `Ctrl+I` | Inline edit (edit mode) |
| `Tab` | Aceptar sugerencia de autocomplete |
| `Esc` | Rechazar sugerencia de autocomplete |

---

## 4. OpenCode

### Instalación

| Parámetro | Valor |
|---|---|
| **Versión** | v1.17.10 |
| **Instalación** | npm global (`npm install -g opencode-ai`) |
| **Runtime** | Node.js v24.17.0 |
| **Config** | `~/.config/opencode/opencode.json` |
| **Versionado** | `~/dev/cybersaturn_config/opencode/opencode.json` |

### `~/.config/opencode/opencode.json` (verificado)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (Saturn)",
      "options": {
        "baseURL": "http://saturn.minotaur-mizar.ts.net:11434/v1"
      },
      "models": {
        "qwen2.5-coder:14b": {
          "name": "Qwen2.5 Coder 14B"
        },
        "qwen3:14b": {
          "name": "Qwen3 14B"
        }
      }
    }
  },
  "model": "ollama/qwen2.5-coder:14b"
}
```

### Modelos y uso en OpenCode

| Modelo | Uso |
|---|---|
| `qwen2.5-coder:14b` | **Default** — código puro, edición de archivos |
| `qwen3:14b` | Tool calling confiable — preferir para tareas agénticas complejas |

> `qwen2.5-coder:14b` es menos confiable para tool calling en OpenCode. Si una tarea agéntica falla o se comporta errático, cambiar a `qwen3:14b` con `/model ollama/qwen3:14b`.

### Provider

OpenCode usa `@ai-sdk/openai-compatible` como adapter — Ollama se expone como API compatible con OpenAI en `/v1`. No requiere API key (Ollama no valida).

### Contexto

OpenCode recomienda al menos 64K de contexto para sesiones largas. Con 16K configurado en Ollama funciona para tareas cortas pero puede perder el hilo en sesiones extensas. Limitación del hardware actual (12GB VRAM).

### Comandos de uso

```bash
# Iniciar OpenCode en el directorio del proyecto
opencode

# Cambiar modelo durante la sesión
/model ollama/qwen3:14b

# Ver versión
opencode --version
```

---

## 5. Versionado de configuración

### Repo: `cybersaturn_config` (privado en GitHub)

Autenticación via SSH. Estructura actual:

```
cybersaturn_config/
├── continue/
│   └── config.yaml
├── opencode/
│   └── opencode.json
└── docs/
    └── (documentos de cierre de fases)
```

### Symlinks o copia manual

Los archivos de config viven en sus paths originales y se versionan copiándolos al repo manualmente. No hay symlinks configurados — actualizar el repo después de cada cambio en la config.

---

## 6. Tailscale (contexto de red)

| Parámetro | Valor |
|---|---|
| **Tailnet** | `minotaur-mizar` |
| **Hostname Saturn** | `saturn.minotaur-mizar.ts.net` |
| **IP Tailscale Saturn** | `100.68.149.122` |
| **MagicDNS** | ✅ Activado |
| **Key expiry Saturn** | ✅ Desactivado — no pide reautenticación |
| **2FA cuenta** | ✅ Activado |
| **Tipo de conexión LAN** | `direct` (peer-to-peer WireGuard, sin relay) |

### `/etc/hosts` en Cassini (DNS local LAN)

```
192.168.0.200    saturn.lan
```

> `saturn.lan` solo funciona en red local. Para remoto siempre usar `saturn.minotaur-mizar.ts.net`.  
> El router TP-Link AX3000 no soporta DNS local — solución via `/etc/hosts` por cliente.

---

## 7. Diagnóstico rápido del entorno de dev

```bash
# Desde Cassini — verificar que Ollama responde
curl http://saturn.minotaur-mizar.ts.net:11434/api/tags

# Verificar modelos disponibles
curl http://saturn.minotaur-mizar.ts.net:11434/api/tags | jq '.models[].name'

# Test de inferencia rápido
curl -X POST http://saturn.minotaur-mizar.ts.net:11434/api/chat \
  -d '{"model":"qwen2.5-coder:14b","messages":[{"role":"user","content":"hola"}],"stream":false}'
```

```powershell
# Desde Saturn — verificar estado de Ollama
sc.exe query ollama

# Ver VRAM en uso
nvidia-smi
```

---

## 8. Pendientes

| Item | Descripción |
|---|---|
| **Fase 3.3** | Replicar Continue + OpenCode en laptop del trabajo (Windows 11) |
| **Agent mode** | Upgrade a RTX 4090 + `qwen3-coder:30b-a3b` (~17GB VRAM) para tool calling confiable en Continue |
| **Contexto OpenCode** | Con RTX 4090 se puede subir a 32K+ y mejorar sesiones agénticas largas |
