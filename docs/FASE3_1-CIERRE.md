# FASE 3.1 — CIERRE: Configuración avanzada de Continue

> Completada: Junio 2026

---

## Resultado

✅ Ollama corriendo como servicio via NSSM con contexto 16K  
✅ Context providers configurados y validados con proyecto Checkpoint  
✅ Autocomplete ajustado a preferencias de uso  
✅ Prompts custom definidos para flujos de C#, ASP.NET y frontend  
✅ Config final consolidada y validada en `~/.continue/config.yaml`  

---

## Bloque A — Contexto y configuración base

### Teoría cubierta

**Cómo construye Continue el prompt:** cada request a Saturn incluye system prompt, contexto inyectado con `@`, historial de la sesión, archivo activo y mensaje del usuario. El modelo no tiene estado — todo va desde cero en cada request. El autocomplete es un mecanismo separado: solo prefix/suffix, sin historial, más pequeño y más frecuente (FIM).

**El límite real del contexto:** Ollama limita el contexto a 2048 tokens por defecto independientemente de la ventana del modelo. Sin configuración explícita en ambos lados (Ollama + Continue), el truncado es silencioso.

**Costo en tokens de cada provider:**

| Provider | Qué manda | Cuándo usarlo |
|---|---|---|
| `@problems` | Errores activos de Roslyn | Debugging de errores de compilación |
| `@terminal` | Últimas líneas del terminal | Errores de runtime |
| `@diff` | Cambios en staging de git | "Revisa lo que acabo de cambiar" |
| `@code` | Función o clase seleccionada | Implementar algo en un bloque conocido |
| `@file` | Archivo completo | Archivos pequeños con contexto total necesario |
| `@codebase` | Chunks relevantes por embeddings | "Cómo funciona X en todo el proyecto" |

### Configuración aplicada en Saturn (NSSM)

| Parámetro | Valor |
|---|---|
| **Servicio** | NSSM |
| **OLLAMA_CONTEXT_LENGTH** | `16384` |
| **OLLAMA_KEEP_ALIVE** | `-1` |
| **OLLAMA_HOST** | `0.0.0.0:11434` |

**Benchmark de contexto:**

| Contexto | tok/s | VRAM | RAM spillover |
|---|---|---|---|
| 32K | ~9 tok/s | 11.5 GB | ~8 GB |
| 16K | ~23 tok/s | 10.7 GB | ~2 GB |

**Decisión: 16K** — más del doble de velocidad, suficiente para Continue + Checkpoint.

---

## Bloque B — Context providers

### Validación con Checkpoint (C# ASP.NET)

| Ejercicio | Provider | Resultado |
|---|---|---|
| Error de compilación intencional | `@problems` | Identificó correctamente el error y la línea sin indicar el archivo |
| Revisión de cambios recientes | `@diff` | Resumió los cambios y evaluó si ameritaban commit |
| Explicar método específico | `@code` | Respuesta precisa sobre el bloque seleccionado |
| Entender el sistema de autenticación | `@codebase` | Respuesta correcta usando embeddings sobre todo el proyecto |

### Configuración aplicada

```yaml
context:
  - provider: code
  - provider: diff
  - provider: terminal
  - provider: codebase
    params:
      nRetrievedFiles: 10
```

**Notas:**
- `@file` reproduce el contenido en el chat — para overviews usar `@code` con selección, o instruir explícitamente "no reproduzcas el código"
- `@codebase` depende de la calidad de la pregunta — preguntas específicas traen fragmentos relevantes, preguntas vagas traen ruido

---

## Bloque C — Autocomplete

### Configuración final

```yaml
autocomplete:
  debounceDelay: 1000
  maxPromptTokens: 300
  multilineCompletions: "never"
```

**Decisión:** configuración conservadora — 1 segundo de delay para no interrumpir el flujo de escritura, sin multiline para mantener control, 300 tokens suficientes para sugerencias de una línea. Punto de partida para alguien que no usaba autocomplete antes.

**Si en el futuro se quiere abrir multiline:** subir `maxPromptTokens` a 400-500 al mismo tiempo.

---

## Bloque D — Slash commands y prompts custom

### Cómo funcionan los prompts en config.yaml (schema v1)

Los slash commands del `config.json` legacy están deprecados. En `config.yaml` los prompts se definen directamente en la sección `prompts` y se invocan con `/nombre` en el chat. No requieren carpeta separada ni reinicio de VS Code — se actualizan al guardar el archivo.

### Prompts definidos

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
| `/docmd` | Documentación técnica en Markdown — pregunta si algo es ambiguo |
| `/guide` | Orientación de dev senior sin modificar código |
| `/vuereview` | Revisión de componente Vue 3 + TypeScript |
| `/reactreview` | Revisión de componente React + TypeScript |
| `/types` | Generación de tipos TypeScript estrictos |

**Flujo recomendado para `/docmd`:** crear el archivo `.md` vacío, abrirlo en VS Code, y usar `/edit` para que Continue escriba directamente en él.

---

## Bloque E — Config final consolidada

### config.yaml final (`~/.continue/config.yaml`)

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
  # C# / ASP.NET
  - name: review
    ...
  - name: docs
    ...
  - name: test
    ...
  - name: endpoint
    ...
  - name: explain
    ...
  - name: fix
    ...
  - name: migrate
    ...
  - name: secure
    ...
  # Documentación
  - name: docmd
    ...
  - name: guide
    ...
  # Frontend
  - name: vuereview
    ...
  - name: reactreview
    ...
  - name: types
    ...
```

> Archivo completo en el repo de configuración (ver sección de versionado abajo).

---

## Modos de Continue

| Modo | Tools disponibles | Estado con modelos actuales |
|---|---|---|
| **Chat** | Ninguna | ✅ Funcional — uso principal |
| **Plan** | Solo lectura (read files, search, diff) | ⚠️ Funcional con limitaciones — modelos 14B no verificados |
| **Agent** | Todas (crear/editar archivos, terminal) | ⚠️ Inconsistente — requiere modelos 30B+ para uso confiable |

**Para flujos agenticos confiables:** OpenCode en Fase 3.2 (diseñado para modelos locales) o upgrade a RTX 4090 + `qwen3-coder:30b-a3b` (~17GB VRAM, ~176 tok/s, contexto nativo 32K).

---

## Versionado de configuración

**Pendiente al cierre de Fase 3** — crear repo privado con estructura:

```
cybersaturn-config/
├── continue/
│   └── config.yaml
└── docs/
    ├── FASE3_1-A2-CIERRE.md
    ├── FASE3_1-CIERRE.md
    └── (futuros cierres)
```

---

## Notas

- Los slash commands del `config.json` legacy están deprecados en schema v1 — no usar
- `@codebase` requiere que el índice de embeddings esté actualizado — se construye automáticamente la primera vez y se actualiza de forma incremental
- Agent mode con `qwen2.5-coder:14b` usa system message tools (XML fallback) — puede fallar en loops complejos de tool calls
- Para OpenCode se recomienda contexto de al menos 64K — con 16K funciona para tareas cortas pero pierde el hilo en sesiones largas

---

## Fases siguientes

| Fase | Contenido |
|---|---|
| **Fase 3.2** | OpenCode — instalación, configuración y flujos de trabajo como agente de terminal |
| **Fase 3.3** | Continue + OpenCode en laptop del trabajo (replica de Cassini) + versionado de config |
| **Fase 4** | Bot de Telegram con n8n + Ollama |
