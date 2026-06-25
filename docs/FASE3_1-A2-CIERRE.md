# FASE 3.1 — A.2: Configuración de contexto real en Ollama

> Completada: Junio 2026

---

## Resultado

✅ Ollama corriendo como servicio via NSSM  
✅ Contexto configurado a 16K — óptimo para el hardware  
✅ Shortcut de autoarranque del instalador eliminado  
✅ Conflicto con RTSS/Afterburner resuelto  

---

## Configuración final

| Parámetro | Valor |
|---|---|
| **Servicio** | NSSM (`C:\Windows\System32\nssm.exe`) |
| **Binario** | `C:\Users\antony\AppData\Local\Programs\Ollama\ollama.exe serve` |
| **OLLAMA_HOST** | `0.0.0.0:11434` |
| **OLLAMA_KEEP_ALIVE** | `-1` |
| **OLLAMA_MODELS** | `C:\Users\antony\.ollama\models` |
| **OLLAMA_CONTEXT_LENGTH** | `16384` |
| **Log** | `C:\Users\antony\.ollama\ollama.log` |

### Comandos NSSM aplicados

```powershell
nssm install ollama "C:\Users\antony\AppData\Local\Programs\Ollama\ollama.exe" serve
nssm set ollama AppEnvironmentExtra "OLLAMA_HOST=0.0.0.0:11434" "OLLAMA_KEEP_ALIVE=-1" "OLLAMA_MODELS=C:\Users\antony\.ollama\models" "OLLAMA_CONTEXT_LENGTH=16384"
nssm set ollama AppStdout "C:\Users\antony\.ollama\ollama.log"
nssm set ollama AppStderr "C:\Users\antony\.ollama\ollama.log"
nssm set ollama Start SERVICE_AUTO_START
```

---

## Problemas encontrados y soluciones

### 1. OLLAMA_CONTEXT_LENGTH ignorada — variables del sistema no llegan al servicio

**Causa:** Las variables de entorno del sistema de Windows no se propagan automáticamente a procesos lanzados por NSSM. Ollama las ignoraba y usaba el default de 4096.

**Solución:** Pasar las variables directamente en `AppEnvironmentExtra` de NSSM — no en las variables del sistema.

### 2. Puerto 11434 siempre ocupado al arrancar el servicio

**Causa:** El instalador de Ollama deja un shortcut en el Startup del usuario que lanza `ollama app.exe` automáticamente con Windows, tomando el puerto antes que NSSM.

**Solución:** Eliminar el shortcut:
```powershell
Remove-Item "C:\Users\antony\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Ollama.lnk"
```

### 3. Servicio en estado PAUSED al arrancar

**Causa:** RivaTuner Statistics Server (RTSS) inyecta en todos los procesos GPU y pausaba Ollama.

**Solución:** En RTSS → botón `+` → agregar `ollama.exe` → **Application detection level: None**.

### 4. `ollama stop` no descargaba el modelo de VRAM

**Causa:** `OLLAMA_KEEP_ALIVE=-1` ancla el modelo permanentemente. `ollama stop` no puede desalojarlo.

**Solución:** Forzar descarga vía API:
```bash
curl -X POST http://saturn.minotaur-mizar.ts.net:11434/api/generate \
  -d '{"model":"<nombre>","keep_alive":0}'
```

### 5. `New-Service` nativo de Windows fallaba con timeout

**Causa:** `ollama.exe serve` no implementa el protocolo de servicios de Windows — no puede registrarse como servicio nativo.

**Solución:** NSSM es el método oficial recomendado por la documentación de Ollama para Windows.

---

## Benchmark de contexto

| Contexto | tok/s | VRAM | RAM spillover |
|---|---|---|---|
| 32K | ~9 tok/s | 11.5 GB | ~8 GB |
| 16K | ~23 tok/s | 10.7 GB | ~2 GB |

**Decisión: 16K** — más del doble de velocidad. Para Continue + Checkpoint (C# ASP.NET) es más que suficiente. El KV cache genera ~2GB de spillover inevitable con este modelo en 12GB VRAM — no impacta la velocidad de forma significativa.

---

## Comandos de administración del servicio

```powershell
# Estado
sc.exe query ollama

# Reiniciar
sc.exe stop ollama
sc.exe start ollama

# Ver configuración NSSM
nssm get ollama AppEnvironmentExtra

# Descargar modelo de VRAM manualmente
curl -X POST http://localhost:11434/api/generate -d '{"model":"<nombre>","keep_alive":0}'

# Ver log
Get-Content "C:\Users\antony\.ollama\ollama.log" -Tail 50
```

---

## Configuración de Continue (Cassini)

`contextLength` alineado con Ollama en `~/.continue/config.yaml`:

```yaml
models:
  - name: Qwen Coder 14B
    provider: ollama
    model: qwen2.5-coder:14b
    apiBase: http://saturn.minotaur-mizar.ts.net:11434
    contextLength: 16384
    roles:
      - chat
      - edit
      - apply
      - autocomplete

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
```

---

## Notas

- `Remove-Service` no existe en la versión de PowerShell de Windows 11 LTSC IoT — usar `sc.exe delete <nombre>`.
- NSSM guarda las variables en el registro, no en las variables del sistema — verificar siempre con `nssm get ollama AppEnvironmentExtra`.
- El log de NSSM acumula errores de sesiones anteriores — limpiar con `Clear-Content` antes de diagnosticar.
- `ollama app.exe` (tray icon) ya no es necesario — el servicio NSSM lo reemplaza completamente.

---

## Contexto para Fase 3.1 — Bloque B (siguiente chat)

**Dónde estamos:** Fase 3.1 completada hasta A.2 inclusive.

**Siguiente bloque:** B.1 — Context providers en Continue.

**Proyecto de práctica:** Checkpoint — API REST en C# ASP.NET, actualmente solo con endpoints de login.

**Metodología:** teoría + práctica en cada bloque. No avanzar al siguiente sin hacer los ejemplos.

**Stack confirmado:**
- Saturn: NSSM + Ollama, contexto 16K, `OLLAMA_KEEP_ALIVE=-1`
- Continue v2.0.0 en Cassini, `contextLength: 16384`
- Modelos: `qwen2.5-coder:14b` (chat + autocomplete), `qwen3:14b` (general), `nomic-embed-text` (embeddings)
- Endpoint: `http://saturn.minotaur-mizar.ts.net:11434`

**Bloques pendientes de Fase 3.1:**
- B.1 — Context providers: configuración y uso real (`@codebase`, `@file`, `@code`, `@diff`, `@terminal`, `@problems`, `@url`, `@folder`, `@open`)
- C.1 — Autocomplete: ajuste fino de parámetros
- D.1 — Slash commands / prompts custom
- E.1 — Config final consolidada + checklist de migración a laptop trabajo
