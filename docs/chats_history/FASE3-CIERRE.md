# FASE 3 — CIERRE: Herramientas de desarrollo (Cassini)

> Completada: Junio 2026

---

## Resultado

✅ Modelos definidos y organizados por rol  
✅ Continue v2.0.0 instalado y funcionando en Cassini  
✅ Chat contra Saturn confirmado — rápido y sin errores  
✅ Autocompletado inline (Tab) confirmado contra Saturn  
✅ Embeddings configurados (`nomic-embed-text`)  
✅ `OLLAMA_KEEP_ALIVE=-1` configurado — modelo permanente en VRAM  
⏸ Configuración avanzada de Continue → Fase 3.1 (chat dedicado)  
⏸ OpenCode (agente de terminal) → Fase 3.2 (chat dedicado)  
⏸ Continue + OpenCode en Laptop trabajo → Fase 3.3  

---

## Modelos en Saturn

| Modelo | Rol principal | Disponible en |
|---|---|---|
| `qwen2.5-coder:14b` | Continue (chat + autocomplete) + OpenCode (default) | Continue, OpenCode, Telegram |
| `qwen3:14b` | Telegram bot (default) — chat general / propósito general | Continue, OpenCode, Telegram |
| `nomic-embed-text` | Embeddings para `@codebase` en Continue | Continue |

### Modelos eliminados
- `qwen2.5:14b` — reemplazado por `qwen3:14b`
- `qwen2.5-coder:7b` — descartado (swap de modelos en VRAM no vale el tradeoff)

---

## Configuración de Ollama en Saturn

| Parámetro | Valor | Motivo |
|---|---|---|
| `OLLAMA_KEEP_ALIVE` | `-1` | Modelo permanente en VRAM — sin lag de carga |
| Consumo en idle | ~10.6 / 12 GB VRAM | Normal — GPU no procesa nada en idle |
| Para liberar VRAM (gaming) | `Stop-Service ollama` | Libera los ~9GB inmediatamente |

---

## Continue en Cassini

| Parámetro | Valor |
|---|---|
| **Versión** | 2.0.0 |
| **Config** | `~/.continue/config.yaml` |
| **Schema** | v1 |
| **Endpoint** | `http://saturn.minotaur-mizar.ts.net:11434` |

### config.yaml actual

```yaml
name: CyberSaturn
version: 1.0.0
schema: v1

models:
  - name: Qwen Coder 14B
    provider: ollama
    model: qwen2.5-coder:14b
    apiBase: http://saturn.minotaur-mizar.ts.net:11434
    roles:
      - chat
      - edit
      - apply
      - autocomplete

  - name: Qwen3 14B
    provider: ollama
    model: qwen3:14b
    apiBase: http://saturn.minotaur-mizar.ts.net:11434
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
```

### Pruebas realizadas

| Prueba | Resultado |
|---|---|
| Chat (`Ctrl+L`) contra Saturn | ✅ Respuesta rápida, sin errores |
| Autocompletado inline (Tab) en `.cs` | ✅ Sugerencias correctas |
| Confirmación que usa Saturn (no local) | ✅ Al detener Ollama en Saturn, chat y autocomplete dejan de funcionar |
| Embeddings (`nomic-embed-text`) | ✅ Modelo descargado y configurado |

---

## Notas

- Continue v2.0.0 es la **versión final** del equipo original — fue adquirido por Cursor en junio 2026. El repo es read-only pero la extensión es estable y funcional. Vigente al menos 6 meses más para uso personal con Ollama.
- El Ollama local de Cassini está **deshabilitado** (`systemctl disable ollama`) — todo el tráfico va a Saturn.
- La config de `qwen2.5-coder:14b` como modelo único para chat + autocomplete elimina el swap de VRAM entre modelos — mejor experiencia sin tradeoff.

---

## Fases siguientes

| Fase | Contenido |
|---|---|
| **Fase 3.1** | Configuración avanzada de Continue — contexto, autocomplete fino, context providers, agent mode, prompts custom |
| **Fase 3.2** | OpenCode — instalación, configuración y flujos de trabajo como agente de terminal |
| **Fase 3.3** | Continue + OpenCode en Laptop trabajo (replica de Cassini) |
| **Fase 4** | Bot de Telegram con n8n + Ollama |
