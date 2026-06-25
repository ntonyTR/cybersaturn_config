# FASE 2.5 — CIERRE: Validación completa de acceso

> Completada: Junio 2026

---

## Resultado

✅ Inferencia confirmada en todos los escenarios de red  
✅ GPU utilizada correctamente durante inferencia  
✅ Acceso vía Tailscale validado desde todos los dispositivos y redes  
✅ DNS local configurado en clientes via `/etc/hosts`  
⏸ Wake on LAN — pendiente, se ejecuta en Fase 2.7 (después de SSH)

---

## Configuración de DNS local (LAN)

Agregado en `/etc/hosts` de cada cliente:

```
192.168.0.200    saturn.lan
```

| Cliente | Estado |
|---|---|
| Cassini (Fedora) | ✅ Configurado |
| Laptop trabajo (Windows 11) | ✅ Configurado |

> El router TP-Link Archer AX53 no soporta DNS local — solución implementada via archivo hosts por dispositivo.

---

## Pruebas de acceso realizadas

### Red local (LAN)

| # | Prueba | Resultado |
|---|---|---|
| 2.5.1 | API responde en LAN (`saturn.lan:11434/api/tags`) | ✅ JSON con lista de modelos |
| 2.5.2 | Inferencia completa en LAN (`/api/chat`) | ✅ Respuesta coherente del modelo |
| 2.5.3 | GPU en uso durante inferencia (`nvidia-smi`) | ✅ ~10GB VRAM ocupados |

### Acceso vía Tailscale

| # | Prueba | Escenario | Resultado |
|---|---|---|---|
| 2.5.4 | API vía Tailscale — teléfono en datos móviles | Red externa | ✅ JSON con lista de modelos |
| 2.5.5 | Inferencia completa vía Tailscale — teléfono | Red externa | ✅ Respuesta coherente del modelo |
| 2.5.6 | API vía Tailscale — laptop trabajo + VPN corporativa + hotspot móvil | Red externa + VPN activa | ✅ JSON con lista de modelos + inferencia correcta |

---

## Modelos instalados

| Modelo | Uso |
|---|---|
| `qwen2.5:14b` | Chat general / agente (a reemplazar por `qwen3:14b`) |
| `qwen2.5-coder:14b` | Código y Continue en VS Code |

### Decisión de modelos

- **Chat general / agente personal:** migrar a `qwen3:14b` (~8.3GB VRAM Q4) — mejora significativa sobre 2.5 dentro del mismo footprint de hardware
- **Código:** mantener `qwen2.5-coder:14b` — sigue siendo el mejor para código puro en 14B
- **Con upgrade a RTX 4090 (futuro):** re-evaluar modelos frontier locales disponibles en ese momento (candidato actual: `qwen3.6:27b` Q6)

---

## Notas

- `saturn.lan` funciona solo en red local. Para acceso remoto siempre usar `saturn.minotaur-mizar.ts.net`.
- Tailscale coexiste sin conflicto con la VPN corporativa (split tunnel confirmado) incluso en red externa.
- Wake on LAN queda pendiente — se ejecuta en Fase 2.7, después de configurar SSH en Fase 2.6.

---

## Siguiente fase

**Fase 2.6** — Administración remota de Saturn (SSH + Moonlight)  
**Fase 2.7** — Wake on LAN
