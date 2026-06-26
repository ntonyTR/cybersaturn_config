# FASE 2.7 — CIERRE: Wake on LAN

> Completada: Junio 2026

---

## Resultado

✅ WoL habilitado en BIOS/UEFI (ASUS Prime B550M-A AC — Power On By PCI-E)  
✅ WoL habilitado en NIC (Realtek PCIe GbE Family Controller)  
✅ ARP binding configurado en router TP-Link AX3000  
✅ Magic packet funcional desde Cassini (LAN)  
✅ Magic packet funcional desde teléfono vía WolOn (LAN)  
✅ Regla ICMP agregada en Windows Firewall para permitir ping entrante  
✅ Función `wol-saturn` con detección y confirmación configurada en Cassini  

---

## Configuración de hardware

### BIOS/UEFI (ASUS Prime B550M-A AC)

| Opción | Valor |
|---|---|
| **Power On By PCI-E** | Enabled |
| **Power On By RTC** | Disabled (no tocar) |

### NIC (Windows — Realtek PCIe GbE Family Controller)

| Pestaña | Opción | Valor |
|---|---|---|
| Power Management | Allow this device to wake the computer | ✅ |
| Power Management | Only allow a magic packet to wake the computer | ✅ |
| Advanced | Wake on Magic Packet | Enabled |
| Advanced | Wake on Pattern Match | Disabled |

---

## ARP Binding en router

Configurado en **Advanced → Security → IP & MAC Binding**:

| Device Name | MAC Address | IP Address |
|---|---|---|
| saturn | `08:XX:XX:XX:XX:XX` | `192.168.0.200` |

> Garantiza que el magic packet siempre llega a Saturn aunque lleve días apagado.

---

## Clientes configurados

### Cassini (Fedora — zsh)

- **Herramienta:** `wol` (via `sudo dnf install wol`)
- **Alias en `/etc/ethers`:** `08:XX:XX:XX:XX:XX saturn.lan`
- **Comando rápido:** `wol saturn.lan`
- **Función con confirmación en `~/.zshrc`:**

```zsh
wol-saturn() {
  if ping -c 1 -W 2 saturn.lan &>/dev/null; then
    echo "Saturn ya esta encendido"
    return
  fi
  wol saturn.lan
  echo "Magic packet enviado, esperando 60s..."
  sleep 60
  ping -c 3 saturn.lan && echo "Saturn encendido OK" || echo "Sin respuesta, reintenta"
}
```

### Teléfono (Android)

- **Herramienta:** WolOn (Google Play)
- **Configuración:**
  - MAC: `08:XX:XX:XX:XX:XX`
  - IP: `192.168.0.255` (broadcast — necesario para que funcione)
  - Puerto: `9`
- **Widget** agregado en pantalla de inicio para encendido con un toque

---

## Windows Firewall — ICMP

Regla agregada para permitir ping entrante (necesario para confirmación de arranque):

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4" protocol=icmpv4:8,any dir=in action=allow
```

---

## Pruebas realizadas

| Prueba | Resultado |
|---|---|
| WoL desde Cassini con Saturn apagado (`wol saturn.lan`) | ✅ Saturn encendió |
| WoL desde Cassini — detección de PC ya encendida | ✅ "Saturn ya esta encendido" |
| WoL desde Cassini con confirmación (`wol-saturn`) | ✅ Ping exitoso post-arranque |
| WoL desde teléfono con WolOn (broadcast 192.168.0.255) | ✅ Saturn encendió |
| Ping a saturn.lan desde Cassini con Saturn encendido | ✅ Responde correctamente |

---

## Notas

- WolOn requiere IP de broadcast (`192.168.0.255`), no la IP específica del dispositivo
- WoL es Layer 2 — solo funciona en la misma red local. No funciona desde red externa
- Termux: `wol` desinstalado — se usa WolOn como cliente en el teléfono
- Laptop trabajo: sin configurar — Docker Desktop ocupa WSL, sin distro Linux disponible

---

## Pendientes / Mejoras futuras

- **Raspberry Pi Zero como relay de WoL:** considerar para habilitar encendido remoto desde red externa. Dispositivo siempre encendido en LAN que reciba comandos vía Tailscale y reenvíe el magic packet a Saturn.

---

## Siguiente fase

**Fase 3** — Continue en VS Code (Cassini + Laptop trabajo)
