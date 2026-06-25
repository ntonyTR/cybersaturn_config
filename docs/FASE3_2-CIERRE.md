# FASE 3.2 — CIERRE: OpenCode (Agente de terminal)

> Completada: Junio 2026

---

## Resultado

✅ OpenCode instalado en Cassini  
✅ Configurado apuntando a Saturn (Ollama)  
✅ Tool calling validado con qwen3:14b  
✅ Edición de archivos confirmada en proyecto Checkpoint  
⚠️ qwen2.5-coder:14b inconsistente para tool calling — qwen3:14b como default  
⚠️ Contexto 16K es el techo — limitación conocida para sesiones largas  

---

## Instalación

```bash
npm install -g opencode-ai
# versión instalada: 1.17.10
```

**Requisitos:** Node.js (probado con v24.17.0 / npm 11.13.0)

---

## Configuración

Archivo: `~/.config/opencode/opencode.json`

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
  "model": "ollama/qwen3:14b"
}
```

**Notas de configuración:**
- El archivo se llama `opencode.json`, no `config.json`
- El formato requiere `npm: "@ai-sdk/openai-compatible"` y `options.baseURL` — la sintaxis simplificada anterior no funciona en v1.17+
- El endpoint de Ollama debe incluir `/v1` al final
- Los modelos deben declararse explícitamente bajo `models`

---

## Modelos

| Modelo | Tool calling | Velocidad | Uso en OpenCode |
|---|---|---|---|
| `qwen3:14b` | ✅ Confiable | ~1-3 min/tarea | **Default — usar siempre** |
| `qwen2.5-coder:14b` | ⚠️ Inconsistente | Más rápido | No recomendado para OpenCode |

**Decisión:** qwen3:14b como modelo default en OpenCode. qwen2.5-coder:14b se mantiene como default en Continue (chat + autocomplete), donde tool calling no aplica.

---

## Uso común

### Ritual de apertura

```bash
cd /home/ntny/dev/projects/checkpoint/backend
opencode
# Verificar que el footer muestre: Qwen3 14B · Ollama (Saturn)
# Si no: Ctrl+P → Switch model → Qwen3 14B
```

### Una sesión por tarea

```
Ctrl+P → New session
```

No acumular tareas en la misma sesión — el contexto de 16K se agota.

### Anatomía de un buen prompt

```
crea un endpoint POST /api/users que reciba { nombre, email }
y lo guarde usando el patrón de repositorio existente.
sigue la estructura de AuthController como referencia.
```

Elementos clave: acción específica + ruta exacta + input/output claro + referencia a código existente.

---

## Cuándo usar OpenCode vs Continue

| Situación | Herramienta |
|---|---|
| Crear endpoint, servicio, DTO completo | OpenCode |
| Refactorizar una clase | OpenCode |
| Agregar validaciones a código existente | OpenCode |
| Autocompletado mientras escribes | Continue |
| Entender código con `@codebase` / `@code` | Continue |
| Tarea que toca 5+ archivos | Continue + tú mismo |
| Refactor grande que requiere contexto de todo el proyecto | Continue + tú mismo |

---

## Limitaciones conocidas

- **Contexto 16K:** OpenCode recomienda 64K para uso cómodo. Con 16K funciona para tareas acotadas pero se corta en sesiones largas o tareas con muchos archivos. Resolución futura: upgrade a RTX 4090 permite subir contexto sin penalización significativa.
- **qwen3 es lento:** El thinking extendido suma 30s-2min por tarea. Es el tradeoff por tool calling confiable con modelos 14B.
- **No delegar refactors grandes:** Tareas que requieren razonar sobre todo el proyecto a la vez exceden las capacidades confiables de 14B + 16K.

---

## Navegación TUI

| Tecla | Acción |
|---|---|
| `Ctrl+P` | Menú de comandos |
| `Ctrl+X N` | Nueva sesión |
| `Ctrl+X L` | Switch session |
| `Ctrl+X M` | Switch model |
| `Ctrl+X E` | Abrir prompt en editor ($EDITOR) |
| `Ctrl+R` | Renombrar sesión |
| `q` | Salir |

---

## Pruebas realizadas

| Prueba | Resultado |
|---|---|
| Conexión a Saturn (Ollama Saturn en footer) | ✅ |
| Lectura autónoma de proyecto Next.js | ✅ |
| Lectura autónoma de proyecto Checkpoint (C# ASP.NET) | ✅ |
| Tool calling con qwen2.5-coder:14b | ⚠️ Inconsistente |
| Tool calling con qwen3:14b | ✅ Confiable |
| Edición de archivo real (AuthController.cs) | ✅ |

---

## Fases siguientes

| Fase | Contenido |
|---|---|
| **Fase 3.3** | Continue + OpenCode en laptop del trabajo (réplica de Cassini) + versionado de config |
| **Fase 4** | Bot de Telegram con n8n + Ollama |
