# Integración Claude + Cisco Packet Tracer via MCP

> **Entorno:** Mac M1 · Packet Tracer 9.0 · Claude Desktop (gratuito)  
> **Repositorio MCP:** [Mats2208/MCP-Packet-Tracer](https://github.com/Mats2208/MCP-Packet-Tracer)  
> **Repositorio PTBuilder:** [kimmknight/PTBuilder](https://github.com/kimmknight/PTBuilder)

---

## ¿Qué permite esta integración?

Permite describir una red en lenguaje natural y obtener scripts JavaScript listos para ejecutar en Cisco Packet Tracer. El flujo es:

```
Tú (lenguaje natural) → Claude → Script JS → PTBuilder en PT → Red montada automáticamente
```
## ⚠️ ¿Necesito el MCP instalado?

**No es imprescindible.** El flujo funciona perfectamente sin él:

1. Describes la red a Claude
2. Claude genera el script JS
3. Lo pegas en PTBuilder y das Run
4. La red aparece automáticamente en PT

El MCP solo añade valor con **Claude Code (suscripción Pro)** — en ese caso Claude 
ejecuta el script solo sin intervención manual. Sin Pro, PTBuilder + Claude es suficiente.

**Lo único imprescindible es PTBuilder** (`Builder.pts`) instalado en Packet Tracer.
---

## Requisitos previos

- Mac M1 con macOS
- Cisco Packet Tracer 9.0 instalado
- Cuenta Claude (gratuita)
- Homebrew instalado
- Python 3 instalado via Homebrew (`/opt/homebrew/bin/python3`)

---

## Instalación completa (una sola vez)

### 1. Instalar Node.js (necesario para Claude Code)

```bash
brew install node
node --version && npm --version
```

### 2. Instalar Claude Code

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

> **Nota:** Claude Code requiere suscripción Pro ($20/mes). Para uso gratuito, usamos Claude Desktop con el MCP configurado manualmente.

### 3. Clonar el servidor MCP de Packet Tracer

```bash
cd ~ && git clone https://github.com/Mats2208/MCP-Packet-Tracer
```

### 4. Instalar dependencias Python del MCP

```bash
cd ~/MCP-Packet-Tracer && pip3 install -e . --break-system-packages
```

### 5. Clonar PTBuilder (extensión para Packet Tracer)

```bash
cd ~ && git clone https://github.com/kimmknight/PTBuilder
```

### 6. Configurar Claude Desktop

Crear/editar el archivo de configuración:

```bash
cat > ~/Library/Application\ Support/Claude/claude_desktop_config.json << 'EOF'
{
  "mcpServers": {
    "packet-tracer": {
      "command": "/opt/homebrew/bin/python3",
      "args": ["-m", "packet_tracer_mcp", "--stdio"],
      "cwd": "/Users/TU_USUARIO/MCP-Packet-Tracer"
    }
  }
}
EOF
```

> ⚠️ Sustituye `TU_USUARIO` por tu nombre de usuario real (puedes verlo con `whoami`).

### 7. Instalar PTBuilder en Packet Tracer

1. Abre Packet Tracer
2. Ve a **Extensiones → Scripting → Configure PT Script Modules**
3. Clic en **Add...**
4. Navega a `/Users/TU_USUARIO/PTBuilder/Builder.pts` y selecciónalo
5. Reinicia Packet Tracer
6. Verifica que aparece **Builder Code Editor** en Extensiones

---

## Uso diario

### Paso 1 — Arrancar el servidor MCP

Antes de usar Claude Desktop, arranca el servidor en una terminal y déjalo corriendo:

```bash
cd ~/MCP-Packet-Tracer && python3 -m packet_tracer_mcp --stdio
```

> El servidor MCP se conecta automáticamente a Claude Desktop al arrancar.

### Paso 2 — Pedir la red a Claude

En Claude Desktop describe la red que quieres. Ejemplos:

```
Genera un script PTBuilder para una red con 2 routers 2911, 
2 switches 2960 y 4 PCs con OSPF y DHCP
```

```
Crea una red de laboratorio de ciberseguridad con un router, 
un switch, un servidor y 3 PCs en subredes separadas
```

### Paso 3 — Ejecutar el script en Packet Tracer

El script que genera Claude tiene este formato:

```javascript
// DISPOSITIVOS
addDevice("R1","2911",300,200);
addDevice("SW1","2960-24TT",200,350);
addDevice("PC1","PC-PT",100,500);

// ENLACES
addLink("R1","GigabitEthernet0/1","SW1","GigabitEthernet0/1","straight");
addLink("SW1","FastEthernet0/1","PC1","FastEthernet0","straight");
```

**⚠️ Limitación importante del Builder:** No puede ejecutar demasiadas líneas de una vez. Ejecutar en dos tandas:

1. **Primera ejecución** — solo los `addDevice()`
2. **Segunda ejecución** — solo los `addLink()`

Para ejecutar:
1. Ve a **Extensiones → Builder Code Editor**
2. Borra el contenido previo
3. Pega el bloque correspondiente
4. Clic en **Run**

---

## Ejemplo completo — Red corporativa

### Red a montar
- Sede Central: Router R1 + Switch SW1 + 2 PCs + 1 Servidor
- Sucursal: Router R2 + Switch SW2 + 2 PCs
- Interconexión R1-R2 con OSPF

### Script — Parte 1: Dispositivos

```javascript
addDevice("R1","2911",300,200);
addDevice("R2","2911",600,200);
addDevice("SW1","2960-24TT",200,350);
addDevice("SW2","2960-24TT",700,350);
addDevice("SRV1","Server-PT",100,350);
addDevice("PC1","PC-PT",100,500);
addDevice("PC2","PC-PT",200,500);
addDevice("PC3","PC-PT",600,500);
addDevice("PC4","PC-PT",700,500);
```

### Script — Parte 2: Enlaces

```javascript
addLink("R1","GigabitEthernet0/0","R2","GigabitEthernet0/0","straight");
addLink("R1","GigabitEthernet0/1","SW1","GigabitEthernet0/1","straight");
addLink("R2","GigabitEthernet0/1","SW2","GigabitEthernet0/1","straight");
addLink("SW1","FastEthernet0/1","SRV1","FastEthernet0","straight");
addLink("SW1","FastEthernet0/2","PC1","FastEthernet0","straight");
addLink("SW1","FastEthernet0/3","PC2","FastEthernet0","straight");
addLink("SW2","FastEthernet0/1","PC3","FastEthernet0","straight");
addLink("SW2","FastEthernet0/2","PC4","FastEthernet0","straight");
```

### Configuración CLI — R1

```
enable
configure terminal
hostname R1
no ip domain-lookup

interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

ip dhcp excluded-address 192.168.1.1 192.168.1.1
ip dhcp pool LAN_CENTRAL
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.1
 dns-server 8.8.8.8
 exit

router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.255 area 0
 exit

end
write memory
```

### Configuración CLI — R2

```
enable
configure terminal
hostname R2
no ip domain-lookup

interface GigabitEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet0/1
 ip address 192.168.2.1 255.255.255.0
 no shutdown
 exit

ip dhcp excluded-address 192.168.2.1 192.168.2.1
ip dhcp pool LAN_SUCURSAL
 network 192.168.2.0 255.255.255.0
 default-router 192.168.2.1
 dns-server 8.8.8.8
 exit

router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.2.0 0.0.0.255 area 0
 exit

end
write memory
```

---

## Dispositivos disponibles en el MCP

| Categoría | Modelo | Alias |
|-----------|--------|-------|
| Router | 2911 | `router` |
| Router | 1941 | — |
| Router | 2901 | — |
| Router | ISR4321 | — |
| Switch | 2960-24TT | `switch` |
| Switch | 3560-24PS | — |
| PC | PC-PT | `pc` |
| Servidor | Server-PT | `server` |
| Laptop | Laptop-PT | `laptop` |
| Access Point | AccessPoint-PT | `ap`, `wifi` |
| Cloud/WAN | Cloud-PT | — |

---

## Tipos de cable en PTBuilder

| Tipo | Uso |
|------|-----|
| `straight` | PC/Servidor → Switch, Router → Switch |
| `crossover` | Switch → Switch, Router → Router mismo modelo |
| `serial` | Router WAN (requiere módulo HWIC) |
| `fiber` | Fibra óptica |

---

## Herramientas MCP disponibles

| Herramienta | Descripción |
|-------------|-------------|
| `pt_list_devices` | Lista dispositivos disponibles |
| `pt_plan_topology` | Genera plan completo |
| `pt_validate_plan` | Valida el plan |
| `pt_fix_plan` | Auto-corrige errores |
| `pt_generate_script` | Genera script PTBuilder JS |
| `pt_generate_configs` | Genera configs CLI por dispositivo |
| `pt_full_build` | Pipeline completo de una vez |
| `pt_explain_plan` | Explica las decisiones en lenguaje natural |
| `pt_estimate_plan` | Estimación sin generar (dry-run) |
| `pt_export` | Exporta a archivos |

---

## Solución de problemas

### El MCP no conecta en Claude Desktop
Verificar en el log:
```bash
cat ~/Library/Logs/Claude/mcp-server-packet-tracer.log
```

### Error `No module named 'src'`
Usar `packet_tracer_mcp` en vez de `src.packet_tracer_mcp` en el config.

### El Builder no ejecuta el script completo
Dividir en dos ejecuciones: primero `addDevice()`, luego `addLink()`.

### Los dispositivos aparecen fuera de pantalla
Usar **Ctrl+Shift+F** en Packet Tracer para centrar la vista.

---

## Estructura de archivos instalados

```
~/
├── MCP-Packet-Tracer/          # Servidor MCP
│   ├── src/packet_tracer_mcp/  # Código fuente
│   ├── WIKI.md                 # Referencia API completa
│   └── CLAUDE.md               # Guía para Claude Code
├── PTBuilder/                  # Extensión PTBuilder
│   └── Builder.pts             # Archivo de instalación
└── Library/Application Support/Claude/
    └── claude_desktop_config.json  # Configuración MCP
```

---

*Guía generada el 20/03/2026 — Entorno: Mac M1, PT 9.0, Claude Desktop gratuito*
