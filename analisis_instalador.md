# Análisis del instalador de TikTok LIVE Studio (Windows): playbook y evidencia pública

Fecha de elaboración: 2026-10-16. Autor: Subagente A (solo investigación, sin código de la app).

> **Estado: Análisis binario: NO realizado (instalador no disponible por bloqueo de red).**
> Este documento contiene (1) la herramienta `rea` ya instalada y verificada, (2) un playbook listo para ejecutar cuando el usuario adjunte el instalador, (3) evidencia pública etiquetada con su fuente y confianza, y (4) implicaciones de licencia GPL. Ningún dato de este archivo proviene de un binario de TikTok analizado en esta sesión.

## Convención de etiquetas de evidencia

| Etiqueta | Significado |
|---|---|
| `[BINARIO]` | Dato obtenido de un binario analizado. **Ninguno en este documento.** Reservado para la fase 2. |
| `[PUBLICO]` | Fuente pública (artículo, foro, página oficial de TikTok). Cita URL. No verificado en binario. |
| `[INDIRECTO]` | Deducido del código del cliente de terceros `tiktokstreamkeygenerator` (imita al cliente oficial). Cita archivo y línea. No verificado en binario. |
| `[HERRAMIENTA]` | Verificado en esta sesión sobre la herramienta (`rea`, 7z, etc.) con ficheros de prueba sintéticos, no con TikTok. |

Niveles de confianza: **Alta** (fuente primaria o varias independientes coinciden), **Media** (una fuente creíble, secundaria), **Baja** (inferencia, portal de terceros o dato antiguo).

---

## 1. Resumen ejecutivo

1. `[PUBLICO]` En diciembre de 2021, un miembro del equipo de OBS (r1ch, en Hacker News) afirmó haber desensamblado la beta de TikTok LIVE Studio y confirmado código derivado de OBS Studio (GPLv2) sin oferta de código fuente. Otro desarrollador de OBS (Ben Torell) lo confirmó públicamente. No hay, en las fuentes encontradas, ninguna resolución documentada. Confianza: Alta para la acusación y su fecha; sin datos sobre el desenlace.
2. `[INDIRECTO]` + `[PUBLICO]` Las versiones actuales (1.x) se presentan con un User-Agent de **Electron 36 / Chromium 136 / "TTElectron"** (fork de Electron de ByteDance), y los listados de ficheros de versiones 0.x de portales de terceros muestran `resources\app\...`, `LICENSE.electron.txt`, `electron_elf.dll`, `mediasdk_server.exe` y `LSVCam.dll`. Conclusión provisional: **Electron (UI) + SDK multimedia nativo propio ("MediaSDK") fuera del proceso Electron**. Si ese SDK conserva código derivado de libobs en 2026 es **desconocido** hasta analizar el binario.
3. `[INDIRECTO]` El flujo de ingesta del cliente oficial, según el clon de terceros, es: crear sala por API `webcast/room/create/`, recibir `stream_url.rtmp_push_url` (RTMP en contenedor FLV, con soporte HEVC `hvc1` en el parche de SEI), y **inyectar SEI tipo 100 firmado** (`"JSON"` + payload) en los frames de vídeo. TikTok trata como sospechoso el tráfico RTMP sin ese metadato (afirmación del autor del clon, no verificada).
4. `[HERRAMIENTA]` `rea-agents@6.5.0` instalado y funcionando en Linux para: artefactos ASAR/directorio Electron, recursos PE, fuses de Electron, inventario de artefactos, HAR/mitmproxy/pcap. Los analizadores nativos (Ghidra/Hopper/IDA) **no están instalados**. NSIS/Inno/MSI **no los desempaqueta `rea`**: se usan 7-Zip, innoextract y msitools (instalados y verificados).
5. Lo que solo el usuario puede aportar: el instalador oficial (bajado de `https://www.tiktok.com/studio/download`), la versión instalada, y una captura de tráfico del PC (ver sección 10).

---

## 2. Herramienta `rea`: instalación, versión, requisitos y limitaciones

### 2.1 Instalación realizada `[HERRAMIENTA]`

```bash
mkdir -p /tmp/claude-0/work/rea-install && cd /tmp/claude-0/work/rea-install
npm init -y && npm install rea-agents          # 173 paquetes, 0 vulnerabilidades reportadas
npx rea-agents --version                       # -> 6.5.0
npx rea-agents --help                          # lista ~190 comandos
npm install @electron/asar                     # asar v4.3.1 (para extraer/empacar asar)
```

No se ejecutó `rea setup` ni `rea mcp add`: **no se configuró MCP ni ningún agente**. (`rea doctor` informa `registration:claude_code ... config_drift` y `skill:identity` ausente, es solo informativo.)

### 2.2 Requisitos y limitaciones `[HERRAMIENTA]`

| Aspecto | Estado en este entorno |
|---|---|
| Node | v22.22.0 (requisito del paquete: `^22.19.0 \|\| ^24.11.0 \|\| >=26`). OK |
| SO | Ubuntu 24.04 (Linux x64). OK |
| JDK | OpenJDK 21.0.12 presente (suficiente para Ghidra 12.1) |
| Ghidra | **No instalado** (`GHIDRA_INSTALL_DIR` sin definir). Requiere Ghidra 12.1.x extraído por el operador. La doc indica soporte Linux x64 y verificación cruzada con PE x86-64 |
| Hopper | **No instalado** (opcional; `rea setup` lo instalaría, no se ejecutó) |
| IDA | No configurado (`REA_IDA_MCP_CONFIG`) |
| jadx | No aplica (Android) |
| tshark/capinfos | No instalados (apt ofrece tshark 4.2.2; `rea` pide 4.x vía `REA_TSHARK_COMMAND`) |
| mitmdump | No instalado (apt ofrece 8.1.1, pero `rea` exige **12.2.3** para leer `.mitm` nativo; usar exportación **HAR** como alternativa) |
| Frida, ADB, Cutter | No relevantes / no instalados |

Verificado con ficheros sintéticos (no TikTok):

* `analyze-javascript-application <app.asar> --format json`: OK (inventario, parseo, grafo semántico, Evidence con SHA256).
* `inspect-artifact` y `extract-artifact` sobre `.asar`: OK (la extracción va a `/tmp/rea-extracted-<uuid>`).
* `inspect-pe-resources` e `inspect-electron-fuses` sobre un PE x86: OK en Linux.
* La opción de salida JSON es `--format json` (el flag `--json` aparece en la doc pero `--format json` es el verificado aquí).

**Limitaciones importantes para este caso:**

1. `rea` **no desempaqueta instaladores NSIS/Inno/MSI**. Esa fase es manual (sección 4, paso 2).
2. `analyze-javascript-application` acepta un **ASAR o un directorio extraído**. Los listados públicos sugieren que LIVE Studio usa `resources\app\` (directorio, no `app.asar`), así que se le pasará el directorio. `[PUBLICO]`, Confianza Baja.
3. Para DLL nativas (libobs, mediasdk, etc.) los comandos de decompilación y `search`/`xrefs`/`function` requieren un provider (Ghidra recomendado). Sin Ghidra, el plan B es `strings`, `yara`, `pefile`, `objdump` y los `inspect-*` de metadatos.
4. Las capacidades de captura dinámica (Frida, `capture-electron-scenario`, `observe-*`) ejecutan el objetivo: **fuera de alcance** de un análisis estático (no usar sin VM aislada y sin autorización del usuario).

### 2.3 Comandos de `rea` relevantes para un instalador Windows con Electron

| Comando | Para qué |
|---|---|
| `inspect-artifact <ruta>` | Inventario y grafo seguro de un artefacto (asar, directorio, paquete). Primer paso por cada artefacto |
| `extract-artifact <ruta>` | Extrae contenidos de forma segura con digests y manifiesto |
| `inspect-pe-resources <pe>` | Recursos PE: identidades, rangos, hashes, iconos |
| `inspect-electron-fuses <exe>` | Fuses de Electron (integridad de ASAR, `OnlyLoadAppFromAsar`, etc.) sin lanzar la app |
| `analyze-javascript-application <asar\|dir> --format json` | Grafo de la app JS/Electron: endpoints, IPC, módulos nativos |
| `trace-javascript-semantics`, `trace-application-feature` | Rastrear un literal (p. ej. `stream_url`) por la evidencia |
| `compare-application-versions '<json>'` | Comparar dos versiones de la app JS |
| `inspect-managed-artifact` / `inspect-managed-members` | Solo si aparece algún componente .NET |
| `inspect-go-binary`, `inspect-rust-binary` | Si algún helper es Go o Rust |
| `search <pe> <patrón> [--kind strings\|procedures]` | Búsqueda en strings/procedimientos (requiere provider) |
| `inspect-web-network-capture <har> har` | Analizar la captura HAR del usuario sin refetch |
| `inspect-pcap-capture <pcap>` | Endpoints, DNS, SNI, distribución de protocolos (requiere pcap del usuario) |

---

## 3. Herramientas de desempaquetado y análisis estático: disponibilidad `[HERRAMIENTA]`

Verificado con `apt-get install` y `pip install` en este entorno (Ubuntu 24.04, Python 3.13).

| Herramienta | Vía | Estado | Uso |
|---|---|---|---|
| 7-Zip 23.01 (`7z`) | `apt install 7zip` | **Instalado y verificado**; `7z i` lista el formato `Nsis` | NSIS, 7z anidados, PE, MSI/CAB, extracción general |
| innoextract 1.9 | `apt install innoextract` | **Instalado** | Inno Setup 1.2.10 a 6.0.5 |
| msitools (`msiextract`) | `apt install msitools` | **Instalado** | MSI |
| cabextract | `apt install cabextract` | Instalado | CAB |
| osslsigncode 2.8 | `apt install osslsigncode` | Instalado | Verificar/extraer firma Authenticode |
| exiftool 12.76 | `apt install libimage-exiftool-perl` | Instalado | Metadatos PE rápidos |
| yara 4.5.0 | `apt install yara` | Instalado | Reglas de marcadores OBS/codecs |
| binutils (`strings`, `objdump`) | preinstalado | Disponible | Strings ASCII y UTF-16LE |
| `file` | preinstalado | Disponible | Identificación |
| pefile 2024.8.26, lief 1.0.0, yara-python | `pip` en venv `/tmp/claude-0/work/venv` | **Instalados y probados** (inventario PE de prueba OK) | Versión, exports, imports, firma |
| @electron/asar 4.3.1 | `npm` | Instalado en `rea-install` | Extraer/listar asar |
| unshield | apt (candidato 1.5.1) | No instalado | Solo si fuese InstallShield |
| radare2 | apt (candidato 5.5.0) | No instalado | Alternativa ligera a Ghidra |
| tshark 4.2.2, mitmproxy 8.1.1 | apt (candidatos) | No instalados | Ver limitación de versión de mitmdump |
| Ghidra 12.1.x, Hopper, IDA | descarga manual | **No instalados** | Decompilación de DLL nativas |

Scripts auxiliares ya escritos y probados con un PE de prueba (no TikTok):

* `/tmp/claude-0/work/playbook-scripts/pe_inventory.py`: recorre un directorio, calcula SHA256 de cada `.exe/.dll/.node`, extrae `ProductVersion`, `FileVersion`, `CompanyName`, presencia de firma, exports con aspecto OBS, imports; escribe CSV.
* `/tmp/claude-0/work/playbook-scripts/obs.yar`: reglas YARA con marcadores de OBS y de codificadores/protocolos.

Si el usuario o el agente principal quiere conservarlos con el informe, están reproducidos íntegramente en el Apéndice A.

---

## 4. PLAYBOOK paso a paso (para cuando se adjunte el instalador)

**Reglas de seguridad del playbook**

* Nunca ejecutar el instalador ni nada extraído. Análisis 100 % estático.
* Instalador solo desde la fuente oficial (`https://www.tiktok.com/studio/download`) o adjuntado por el usuario. No usar portales de terceros, mirrors ni torrents.
* Cada artefacto descargado o extraído va en un **directorio nuevo y vacío**; los scripts viven en otro directorio; Python siempre con `-I` (modo aislado); pasar rutas como argumentos.
* Registrar cada comando y su salida en el informe (reproducibilidad).

Variables sugeridas:

```bash
export W=/tmp/claude-0/work/tt-analysis
export INST="$W/in/TikTokLIVEStudio_Setup.exe"      # el fichero adjuntado
export PY=/tmp/claude-0/work/venv/bin/python
export S=/tmp/claude-0/work/playbook-scripts
export REA="npx --prefix /tmp/claude-0/work/rea-install rea-agents"
mkdir -p "$W"/{in,stage1,stage2,reports}
```

### Paso 0. Cadena de custodia, hash y versión del instalador

```bash
# Hashes y tamaño
sha256sum "$INST" | tee "$W/reports/installer.sha256"
sha1sum "$INST"; md5sum "$INST"; stat -c '%s bytes, mtime %y' "$INST"
file "$INST"

# Versión PE del instalador (ProductVersion / FileVersion / CompanyName / OriginalFilename)
exiftool -ProductVersion -FileVersion -ProductName -CompanyName -OriginalFileName -LegalCopyright "$INST"
$PY -I - "$INST" <<'EOF'
import sys, pefile
pe = pefile.PE(sys.argv[1])
for fi in getattr(pe, "FileInfo", []) or []:
    for e in fi:
        if getattr(e, "Key", b"") == b"StringFileInfo":
            for st in e.StringTable:
                for k, v in st.entries.items():
                    print(k.decode(), "=", v.decode(errors="replace"))
print("Machine:", hex(pe.FILE_HEADER.Machine), "Timestamp:", pe.FILE_HEADER.TimeDateStamp)
print("Firma Authenticode presente:", bool(pe.OPTIONAL_HEADER.DATA_DIRECTORY[4].Size))
EOF

# Firma Authenticode: emisor, sujeto, fecha (sin ejecutar)
osslsigncode verify -in "$INST" 2>&1 | tee "$W/reports/installer.signature.txt"

# Recursos PE con rea
$REA inspect-pe-resources "$INST" --format json > "$W/reports/installer.peresources.json"
$REA inspect-artifact "$INST" --format json > "$W/reports/installer.inspect.json"
```

Qué registrar: SHA256, tamaño, `ProductVersion`/`FileVersion` (el informe público de terceros sugiere versiones tipo `1.x.y`; la numeración 0.x/1.x aparece en `[PUBLICO]` portal de terceros), sujeto del certificado (esperable "TikTok Pte. Ltd." según `[PUBLICO]` listados de instalador, Confianza Media), fecha de firma.

Comprobación adicional de versión publicada: el cliente consulta un endpoint de actualización de ByteDance (`https://tron-sg.bytelemon.com/api/sdk/check_update`, ver sección 7). Es **información pública del propio cliente**, pero no se consultó en esta sesión; el usuario puede contrastar la versión instalada con el manifest que devuelve, anotándolo como `[PUBLICO]`.

### Paso 1. Identificar el tipo de instalador

```bash
7z i | grep -i -E 'nsis|inno'        # el formato Nsis aparece listado
7z l "$INST" | head -60              # NSIS: carpeta $PLUGINSDIR, [NSIS].nsi (7-Zip antiguo), a veces app-64.7z
strings -a "$INST" | grep -i -E 'Nullsoft|NSIS|Inno Setup|InstallShield|WiX|Wise|Squirrel|electron-builder' | sort -u | head -20
innoextract --list "$INST" 2>&1 | head   # si es Inno, lista ficheros; si no, error (descartado)
```

Indicadores: `Nullsoft Install System` / firma `EF BE AD DE ... NullsoftInst` (NSIS); `Inno Setup Setup Data` (Inno); cabecera `D0 CF 11 E0` (MSI/OLE).
Pista pública `[PUBLICO]`, Confianza Baja: los listados de terceros muestran un `TikTok LIVE Studio Launcher.exe`, un `TikTok LIVE Studio Uninstaller.exe` y una carpeta con el número de versión (`...\TikTok LIVE Studio\0.94.2\`), patrón compatible con un instalador NSIS/propio con actualizador (`app_shell_updater.exe`) que instala por carpeta de versión. No confirmar sin el binario.

### Paso 2. Desempaquetado estático por tipo

```bash
# --- NSIS (o instalador desconocido que 7z reconoce) ---
7z x "$INST" -o"$W/stage1" -y -snl            # NSIS suele contener un 7z/zip anidado (p. ej. app-64.7z)
find "$W/stage1" -maxdepth 3 -type f | head -50
# Si hay archivos anidados (7z, zip, nupkg), repetir en un directorio nuevo por cada uno:
for f in $(find "$W/stage1" -type f \( -name '*.7z' -o -name '*.zip' -o -name '*.nupkg' \)); do
  d="$W/stage2/$(basename "$f" | tr '.' '_')"; mkdir -p "$d"; 7z x "$f" -o"$d" -y -snl
done
sha256sum $(find "$W/stage1" "$W/stage2" -type f \( -name '*.7z' -o -name '*.zip' \)) > "$W/reports/nested_archives.sha256"

# --- Inno Setup ---
innoextract --list --silent "$INST"
innoextract -d "$W/stage1" --extract "$INST"
innoextract --show-password "$INST" 2>/dev/null   # solo informativo

# --- MSI ---
msiextract -C "$W/stage1" "$INST"
7z x "$INST" -o"$W/stage1" -y                     # alternativa; también extrae CABs

# --- Fallback universal: carving por 7z sobre el overlay ---
7z l -slt "$INST" | head -80
```

Notas:

* Muchos instaladores Electron descargan la carga útil desde Internet (stub/web installer). Si el instalador pesa pocos MB y no contiene el `7z` anidado, **ese hecho es un hallazgo**: registrarlo, y el contenido real solo se obtendrá instalando (que se hace en una VM desechable y solo si el usuario lo autoriza explícitamente). Los listados públicos hablan de ~200 MB instalados (`[PUBLICO]` portal de terceros: 214 MB para 0.94.2).
* Guardar el árbol de ficheros: `find "$W/stage1" "$W/stage2" -type f -printf '%P\t%s\n' | sort > "$W/reports/filetree.tsv"`.

### Paso 3. Detectar Electron frente a nativo (Qt/C++)

```bash
R="$W/stage2"   # ajustar a la raíz que contenga el exe principal
# Marcadores de Electron/Chromium
find "$W" \( -iname 'app.asar' -o -iname 'app.asar.unpacked' -o -iname 'electron*.dll' -o -iname 'icudtl.dat' \
  -o -iname 'v8_context_snapshot*.bin' -o -iname 'chrome_100_percent.pak' -o -iname 'resources.pak' \
  -o -iname 'LICENSE.electron.txt' -o -iname 'LICENSES.chromium.html' -o -iname 'ffmpeg.dll' -o -iname 'libEGL.dll' \) | sort
find "$W" -type d \( -name resources -o -name app -o -name locales \) | head
# Marcadores de Qt (nativo)
find "$W" \( -iname 'Qt?Core*.dll' -o -iname 'Qt6*.dll' -o -iname 'Qt5*.dll' -o -iname 'platforms' -o -iname 'qt.conf' \) | sort
# Cadena de versión de Electron/Chromium dentro del exe principal (ASCII y UTF-16LE)
for exe in $(find "$W" -maxdepth 4 -iname '*.exe' -size +20M); do
  echo "== $exe"; strings -a "$exe" | grep -E -m3 'Electron/[0-9]+|Chrome/[0-9]+|TTElectron'
done
# Fuses (integridad ASAR, OnlyLoadAppFromAsar, etc.)
$REA inspect-electron-fuses "<ruta al exe principal>" --format json
```

Pista previa `[INDIRECTO]`: el UA del cliente es `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) TikTokLIVEStudio/1.27.0 Chrome/136.0.7103.59 Electron/36.4.0-alpha.17 TTElectron/36.4.0-alpha.17 Safari/537.36` (`Libs/domain_routing.py:18-24`, `Libs/device_gen.py:138-144`). Si el binario muestra otra versión de Chromium/Electron, esa es la verdad del binario.

### Paso 4. Análisis de la app JS (si existe `resources/app` o `app.asar`)

```bash
APP="<ruta a resources/app>"            # directorio, o app.asar
$REA inspect-artifact "$APP" --format json > "$W/reports/app.inspect.json"
$REA analyze-javascript-application "$APP" --format json > "$W/reports/app.js.graph.json"
$REA analyze-javascript-application "$APP" --artifact-format directory --format json > "$W/reports/app.js.graph.dir.json"  # si es directorio
# ASAR a mano (alternativa): 
npx --prefix /tmp/claude-0/work/rea-install asar list app.asar | head
npx --prefix /tmp/claude-0/work/rea-install asar extract app.asar "$W/stage2/app_src"
cat "$APP/package.json"
# Rastrear literales con rea (tras el análisis anterior, sobre su Evidence JSON)
$REA trace-javascript-semantics '<json de entrada según --schema>'
```

Atención: algunas apps empaquetan el JS como bytecode V8 (`.jsc`) o ofuscan; el listado público incluye `__META-INF__\win32x64.node` y `Common.Build-...node` (`[PUBLICO]`, Confianza Baja), que sugiere módulos nativos `.node`. Registrar si el JS es legible, minificado, u ofuscado.

### Paso 5. Buscar OBS / libobs y capturas de juego

```bash
# 5a. Nombres de fichero relacionados con OBS
find "$W" -type f \( -iname 'obs*.dll' -o -iname 'libobs*' -o -iname 'obs-ffmpeg*' -o -iname 'obs-x264*' \
  -o -iname 'obs-outputs*' -o -iname 'obs-filters*' -o -iname 'win-capture*' -o -iname 'win-dshow*' -o -iname 'win-wasapi*' \
  -o -iname 'libobs-d3d11*' -o -iname 'libobs-opengl*' -o -iname 'rtmp-services*' -o -iname 'graphics-hook*' \
  -o -iname 'get-graphics-offsets*' -o -iname 'inject-helper*' -o -iname 'w32-pthreads*' -o -iname 'libff*' \) | sort

# 5b. Equivalentes renombrados que se han citado publicamente (buscar por estos nombres y por estructura)
find "$W" -type f \( -iname 'GameDetour*' -o -iname 'Inject64*' -o -iname 'MediaSDK*' -o -iname 'LSVCam*' \
  -o -iname 'mediasdk_server*' -o -iname 'capture_service_client*' -o -iname 'bmf_hydra*' -o -iname 'effect.dll' \) | sort

# 5c. Marcadores internos (los ficheros pueden estar renombrados): YARA + exports
yara -r -s -w "$S/obs.yar" "$W/stage1" "$W/stage2" 2>/dev/null | tee "$W/reports/yara_obs.txt"
$PY -I "$S/pe_inventory.py" "$W/stage2" "$W/reports/pe_inventory.csv"
$PY -I - "$W/reports/pe_inventory.csv" <<'EOF'
import csv, sys
for r in csv.DictReader(open(sys.argv[1], encoding="utf-8")):
    if r.get("obs_like_exports"):
        print(r["path"], r["sha256"][:16], r["obs_like_exports"][:120])
EOF

# 5d. Strings característicos de libobs (ASCII y UTF-16)
for f in $(find "$W" -type f \( -iname '*.dll' -o -iname '*.exe' -o -iname '*.node' \)); do
  n=$( (strings -a "$f"; strings -a -el "$f") | grep -c -E 'obs_source_|obs_output_|obs_encoder_|libobs|OBS Studio|rtmp-services|graphics-hook|obsproject\.com')
  [ "$n" -gt 0 ] && echo "$n\t$f"
done | sort -rn | head -40
```

Criterios de decisión (para el informe):

* Existencia de exports `obs_*` en una DLL de TikTok: indicio fuerte de enlace a libobs (o a una copia modificada).
* Nombres tipo `GameDetour64.dll` / `Inject64.exe` / `MediaSDKGetWinDXOffset64.exe`: citados en 2021 por OBS como análogos de `graphics-hook64.dll` / `inject-helper64.exe` / `get-graphics-offsets64.exe` (`[PUBLICO]`, fuente: comentario de r1ch en HN). Comparar con el `libobs` público: estructuras, cadenas de error, rutas `libobs/obs-*.c`, la shim de `graphics-hook` (nombres de eventos/pipes/mutex).
* Si hay Ghidra: `search ... --kind strings 'libobs'`, `xrefs`, `function` sobre los puntos de entrada (`obs_module_load`, `obs_startup`) para confirmar. Sin Ghidra, quedar en "indicio" y no en "confirmado".

### Paso 6. Codificadores

```bash
for pat in 'x264 - core' 'libx264' 'h264_nvenc' 'hevc_nvenc' 'av1_nvenc' 'nvEncodeAPI' 'nvcuda' \
           'h264_qsv' 'hevc_qsv' 'libmfx' 'mfxhw' 'amfrt64' 'AMFVideoEncoder' 'h264_amf' 'hevc_amf' \
           'libopenh264' 'libvpx' 'libaom' 'svt-av1' 'libavcodec' 'libx265' 'MediaFoundation' 'MFTEnum'; do
  printf '%-18s ' "$pat"
  grep -rIl -a -e "$pat" "$W/stage1" "$W/stage2" 2>/dev/null | wc -l
done
# Detalle por fichero (incluye UTF-16LE)
for f in $(find "$W" -type f \( -iname '*.dll' -o -iname '*.exe' \)); do
  hits=$( (strings -a "$f"; strings -a -el "$f") | grep -o -E 'x264 - core [0-9]+|libx264|[a-z0-9]+_nvenc|nvEncodeAPI(64)?\.dll|[a-z0-9]+_qsv|libmfx[a-z0-9_]*|amfrt64\.dll|AMFVideoEncoder[A-Za-z]*|libavcodec[^ ]*' | sort -u | tr '\n' ' ')
  [ -n "$hits" ] && echo "$f => $hits"
done | tee "$W/reports/encoders.txt"
```

Interpretación:

* `ffmpeg.dll` junto a `libEGL.dll` es normalmente el **ffmpeg de Chromium/Electron** (solo decodificadores, sin x264). No contar como prueba de codificación. Comprobar con `strings ffmpeg.dll | grep -E 'x264|nvenc'` (esperable: nada).
* "x264 - core NNN" con banner = x264 enlazado (GPL, relevante para sección 8). Anotar versión y si se trata de build comercial (licencia x264 comercial).
* NVENC: se carga `nvEncodeAPI64.dll` del driver NVIDIA, no se distribuye; buscar la cadena y el uso de la SDK (licencia NVENC SDK). QSV: `libmfx`/oneVPL. AMF: `amfrt64.dll` del driver AMD.

### Paso 7. Protocolo de ingesta (RTMP/RTMPS/WHIP/SRT)

```bash
PATS='rtmps?://|whip|WHIP|srt://|webrtc|librtmp|RTMP_|rtmp_push_url|rtmp_pull_url|flv|hvc1|enhanced.?rtmp|rtc_net|stream_url|multi_stream_url|push_url|sei_index|ttls_live_scene|live_sei'
for f in $(find "$W" -type f \( -iname '*.dll' -o -iname '*.exe' -o -iname '*.node' \)); do
  hits=$( (strings -a "$f"; strings -a -el "$f") | grep -E -o "$PATS" | sort | uniq -c | sort -rn | head -8 | tr '\n' ';')
  [ -n "$hits" ] && echo "$f => $hits"
done | tee "$W/reports/ingest_protocols.txt"
# En el JS de la app
grep -rIn -E "$PATS" "$APP" --include='*.js' --include='*.json' | head -80
```

Hipótesis a contrastar con el binario `[INDIRECTO]` (sección 7): RTMP (FLV) con `stream_url.rtmp_push_url`; HEVC en FLV mejorado (`hvc1`); campo `rtc_net_enabled`; SEI tipo 100 con carga `"JSON{...}"` y campos `signResult`, `ttls_live_scene="live_studio"`, `push_video_width/height`, `sei_index`. El binario deberá confirmar si el SEI lo genera la DLL del SDK multimedia o el proceso Electron.

### Paso 8. Strings de dominios y endpoints

```bash
# 8a. Extraer URLs/dominios de todo (ASCII y UTF-16LE), con contexto del fichero
for f in $(find "$W" -type f); do
  (strings -a "$f"; strings -a -el "$f") 2>/dev/null | grep -a -o -E "https?://[A-Za-z0-9._~:/?#@!$&'()*+,;=%-]+|wss?://[A-Za-z0-9._~:/?#@!$&'()*+,;=%-]+" | sed "s|^|$f\t|"
done | sort -u > "$W/reports/urls_all.tsv"
cut -f2 "$W/reports/urls_all.tsv" | sed -E 's|^[a-z]+://([^/:?#]+).*|\1|' | sort | uniq -c | sort -rn > "$W/reports/domains_ranked.txt"

# 8b. Semillas (de la sección 7): comprobar presencia/ausencia
grep -a -E 'tiktokv\.(com|eu|us)|tiktok\.com|bytelemon\.com|tiktokcdn|byteoversea|ttwstatic|snssdk|musical\.ly|webcast|passport|ucenter_web|onetap_auth|oidc|device_register|check_update|tnc[0-9]*-|get_domains|live-studio-app' \
  "$W/reports/urls_all.tsv" | head -200
# 8c. Endpoints relativos en JS
grep -rIho -E '/(webcast|passport|service/2|api/sdk)/[A-Za-z0-9_/.-]+' "$APP" | sort | uniq -c | sort -rn | head -100
# 8d. Esquema URL protocolo del cliente (registro)
grep -rIn -E 'live-studio-app' "$W" 2>/dev/null | head
```

### Paso 9. Hash de cada DLL/EXE/NODE (inventario completo)

```bash
find "$W/stage1" "$W/stage2" -type f \( -iname '*.dll' -o -iname '*.exe' -o -iname '*.node' -o -iname '*.sys' \) -exec sha256sum {} + | sort -k2 > "$W/reports/binaries.sha256"
$PY -I "$S/pe_inventory.py" "$W/stage2" "$W/reports/pe_inventory.csv"     # SHA256 + versión + firma + exports + imports
# Contraste con md5.json que el listado público menciona dentro del directorio de versión
find "$W" -name 'md5.json' -exec sh -c 'echo {}; head -c 600 {}' \;
# Firmas por binario
for f in $(find "$W/stage2" -type f \( -iname '*.dll' -o -iname '*.exe' \) | head -400); do
  osslsigncode verify -in "$f" 2>&1 | grep -E 'Subject:|Signature verification' | head -2 | sed "s|^|$f: |"
done > "$W/reports/signatures_per_binary.txt"
```

### Paso 10. Licencias embebidas y avisos (GPL)

```bash
find "$W" -type f \( -iname 'COPYING*' -o -iname 'LICENSE*' -o -iname 'LICENCE*' -o -iname 'NOTICE*' -o -iname 'THIRD*PARTY*' \
  -o -iname 'LICENSES.*' -o -iname '*open*source*notice*' -o -iname 'README*' -o -iname 'AUTHORS*' \) | sort | tee "$W/reports/license_files.txt"
# El listado público indica un fichero llamado literalmente "open source notice" (sin extension) en el directorio de version
find "$W" -iname 'open source notice*' -exec sh -c 'echo "== $1"; wc -c "$1"; head -c 2000 "$1"' _ {} \;
# Buscar menciones relevantes en TODO el arbol
grep -rIl -i -E 'GNU General Public License|GPL-?(2|3)|LGPL|obsproject|OBS Studio|libobs|x264|ffmpeg|written offer|source code' "$W/stage1" "$W/stage2" 2>/dev/null | sort > "$W/reports/license_mentions.txt"
# Buscar oferta escrita de codigo fuente / URL de codigo
grep -rIn -i -E 'written offer|source code (is )?available|corresponding source|github\.com/(bytedance|tiktok)|gitlab|offer.*source' "$W" 2>/dev/null | head -40
# Licencias dentro de binarios (cadenas)
for f in $(find "$W" -type f \( -iname '*.dll' -o -iname '*.exe' \)); do
  (strings -a "$f"; strings -a -el "$f") | grep -m2 -E 'GNU General Public|GPLv|This program is free software|OBS Studio|Hugh Bailey|Jim Bailey' | sed "s|^|$f: |"
done | tee "$W/reports/license_strings_in_binaries.txt"
```

Qué registrar: lista exacta de ficheros de licencia y avisos; si `open source notice` menciona OBS/libobs/x264/FFmpeg y con qué licencia; si existe oferta escrita o URL de código fuente (GPLv2 sección 3); contenido del aviso en el instalador y dentro de la app (pantalla "Acerca de" no es analizable en estático: pedir captura al usuario).

### Paso 11. Capa nativa con Ghidra (opcional, requiere instalar Ghidra)

```bash
# Prerrequisitos: Ghidra 12.1.x extraido (descarga oficial del proyecto NSA en GitHub, la hace el usuario o se autoriza)
export GHIDRA_INSTALL_DIR=/opt/ghidra_12.1.x_PUBLIC
$REA doctor                                          # debe pasar de ghidra=missing a ok
$REA providers --format json
$REA search "<DLL>" 'obs_' --kind procedures --format json
$REA search "<DLL>" 'rtmp' --kind strings --format json
$REA decompile "<DLL>" ... ; $REA function "<DLL>" <dirección>
```

(Los argumentos exactos de `decompile`, `function`, `xrefs` se consultan con `rea <cmd> --help` y `--schema`.)

### Paso 12. Captura de tráfico (la hace el usuario; se analiza aquí)

```bash
# HAR exportado desde mitmproxy/Fiddler/DevTools
$REA inspect-web-network-capture /ruta/absoluta/sesion.har har --sensitive-value "<cookie_o_token>" --format json
# pcap/pcapng (con tshark 4.x instalado y REA_TSHARK_COMMAND definido)
$REA inspect-pcap-capture /ruta/absoluta/captura.pcapng --format json
```

Nota: el cliente es Chromium/Electron con su propia red ("TTNet"/`TTNetwork PC` según el clon, `[INDIRECTO]`); es posible que haya pinning o que el tráfico del SDK multimedia (RTMP) no pase por un proxy HTTP. Por eso se pide pcap además de HAR.

---

## 5. Plantilla del informe final (a llenar con el binario)

```markdown
# Informe de análisis estático: TikTok LIVE Studio <versión> (Windows)

## 0. Metadatos y cadena de custodia
- Fichero: <nombre>   SHA256: <...>   Tamaño: <...>   Origen: <URL oficial / adjuntado por usuario>
- Fecha de descarga: <...>   Firma: <sujeto, emisor, fecha>   ProductVersion/FileVersion: <...>
- Entorno: <SO, versiones de rea, 7z, innoextract, pefile, Ghidra>
- Comandos ejecutados: <enlace/adjunto con registro completo>

## 1. Tipo de instalador y estructura
- Tipo (NSIS/Inno/MSI/web-stub): <...>   Artefactos anidados + hashes: <...>
- Árbol de ficheros (resumen) y tamaño instalado: <...>

## 2. Arquitectura de la aplicación
- Electron / nativo (Qt/C++) / híbrido: <...>   Evidencia: <ficheros, strings, fuses>
- Versiones: Electron <...>, Chromium <...>, Node <...>, "TTElectron" <sí/no>
- app: asar / directorio / JS legible-minificado-ofuscado-bytecode: <...>
- Procesos auxiliares (mediasdk_server, captura, updater, elevation_service...): <rol, hash>

## 3. Pila multimedia
- ¿libobs o derivado? <sí/no/indicio>   Evidencia: <exports obs_*, cadenas, estructuras>
- Módulos tipo OBS presentes (obs-ffmpeg, win-capture, libobs-d3d11, obs-outputs, rtmp-services, graphics-hook, inject-helper...): tabla nombre / presente / hash / equivalente renombrado
- Captura de juego/pantalla: <hooks, API usadas (D3D/DXGI/WGC)>
- Cámara virtual (LSVCam): <...>

## 4. Codificadores
| Codificador | Presente | Dónde | Versión | Observaciones |
| x264 | | | | |
| NVENC | | | | |
| QSV | | | | |
| AMF | | | | |
| HEVC/AV1 | | | | |

## 5. Protocolo de ingesta
- RTMP / RTMPS / WHIP / SRT / otro: <...>   Contenedor: FLV / enhanced FLV: <...>
- Metadatos en banda (SEI tipo 100, campos): <...>   Quién los genera: <proceso/DLL>

## 6. Dominios y endpoints
- Tabla dominio / ruta / propósito (login, webcast, passport, update, telemetría, CDN) / origen (fichero) / confianza
- Esquema URL registrado (live-studio-app://...) y flujo de login (QR / navegador): <...>
- Mecanismo de firma de peticiones (¿en JS, módulo .node o DLL?): <ubicación; sin reproducirlo>

## 7. Inventario de binarios
- CSV adjunto: ruta / SHA256 / ProductVersion / FileVersion / firma / exports OBS-like / imports

## 8. Licencias y cumplimiento
- Ficheros de licencia/aviso: <lista>   Contenido del "open source notice": <resumen>
- Componentes GPL/LGPL detectados (OBS/libobs, x264, FFmpeg, otros) y evidencia: <...>
- ¿Oferta escrita de código fuente o URL? <sí/no, dónde>   Conclusión factual (sin afirmar incumplimiento sin evidencia): <...>

## 9. Diferencias frente a la evidencia pública/indirecta
- Confirmado / contradicho / no verificable: tabla contra secciones 6 y 7 de este documento

## 10. Limitaciones y trabajo pendiente
- Qué no se pudo analizar (ofuscación, cargas descargadas en runtime, DLL sin provider nativo)
```

---

## 6. Hallazgos de fuentes públicas `[PUBLICO]` (no verificados en binario)

### 6.1 Arquitectura: ¿OBS? ¿Electron?

| # | Afirmación | Fuente | Confianza |
|---|---|---|---|
| P1 | En dic. 2021 salió una beta limitada de "TikTok Live Studio" para Windows (unos pocos miles de usuarios en pocos mercados occidentales). | Engadget (2021-12-21), FOSSA (2021-12-22), Verge (2021-12-20) | Alta |
| P2 | Un miembro del equipo OBS (r1ch) abrió los ficheros del instalador sin instalarlo y vio `GameDetour64.dll`, `Inject64.exe`, `MediaSDKGetWinDXOffset64.exe`, análogos a `graphics-hook64.dll`, `inject-helper64.exe`, `get-graphics-offsets64.exe`; tras desensamblar "pudo confirmar que había código de OBS en sus binarios". | Comentario HN de r1ch (2021-12-17) | Media-Alta (declaración de parte técnica competente, no auditada de forma independiente) |
| P3 | Ben Torell (OBS) matizó que "técnicamente no es un fork de OBS, solo usa partes de código de OBS internamente; la GPL aplica igualmente" y que OBS buscaba una solución privada y de buena fe. | Tuit de @dodgepong 2021-12-16 citado por GIGAZINE y Engadget | Media-Alta |
| P4 | Otro comentario de un desarrollador de OBS en HN: no "instala OBS en segundo plano"; envían varios ejecutables con código derivado de OBS sin oferta de código fuente. | HN (comentario de dev. OBS, vía whnex.com) | Media |
| P5 | Se afirmó que además de OBS había otro código GPL y "otros casos de código OBS fuera de ese ejemplo". | HN (comentarios) | Baja-Media |
| P6 | TikTok retiró el enlace de descarga público tras la polémica (redirigía a la página de inicio), aunque los enlaces directos seguían funcionando. | Inverse (2021-12-28), Habr (2021-12-18) | Media |
| P7 | La versión actual de la web de TikTok dice: "LIVE Studio is developed by TikTok" y compara con OBS; **no menciona OBS como base**. | TikTok LIVE Studio Help Center FAQ (actualizado 2026-08-19) | Alta (sobre lo que dice); no prueba nada sobre el código |
| P8 | Listados de ficheros de portales de desinstalación (versiones 0.67.2, 0.83.2, 0.85.0, 0.94.2, 2025): `TikTok LIVE Studio.exe` (~153 MB), `electron_elf.dll`, `LICENSE.electron.txt`, `LICENSES.chromium.html`, `icudtl.dat`, `chrome_*_percent.pak`, `ffmpeg.dll`, `libEGL.dll`, `libGLESv2.dll`, `lynx.dll`, `ppapi_lynx_headless.dll`, `parfait.dll`, `parfait_crash_handler.exe`, `metasecml.dll`, `bmf_hydra.dll`, `capture_service_client.dll`, `effect.dll`, `PresentMon.dll` + `LICENSE.PresentMon.txt`, `exit_detector.dll`, `app_shell_updater.exe`, `elevation_service.exe`, `dump_reporter.exe`, `vc_redist_x64.exe`, un fichero `open source notice`, `md5.json`, `launcher_config.json`, `resources\app\__META-INF__\*.node`, `resources\app\electron\sdk\lib\mediasdk_server.exe`, `MediaSDK_Server.exe` (~7 MB). Estructura `...\TikTok LIVE Studio\<versión>\`. | advanceduninstaller.com (2025-04 a 2025-09) | **Baja-Media** (portal de terceros, lista generada automáticamente de instalaciones de usuarios; **no se descargó nada de ese portal, solo se leyó la página**) |
| P9 | Scott Hanselman (2024-10) localizó `LSVCam.dll` en `C:\Program Files\TikTok LIVE Studio\0.67.2\resources\app\electron\sdk\lib\MediaSDK_V1` (218.984 bytes) como origen de una cámara virtual "LSVCam" que provoca bloqueos de enumeración de cámaras. OBS añadió esa DLL a su lista de bloqueo (`UI/win-dll-blocklist.c`, commit `ab48cd0`, 2024-10-16). | Hanselman blog; GitHub obsproject/obs-studio | Alta (autor identificado, ruta exacta, commit verificable) |
| P10 | LIVE Studio expone un servidor local Socket.IO para el plugin oficial de Elgato Stream Deck (subprotocolo `streamdeck_ttls_v1`, eventos `stream_deck/join_room`, `stream_deck/sync_settings`, `stream_deck/action_emit`, puertos 28189, 39728, 34246, 42205, 38534, 40825, 40622). Compatible con una app basada en Node/Electron. | GitHub `slyboots/ttls-controller` (repo de terceros, 2026) | Baja-Media |
| P11 | Un aviso de código abierto de "Live Studio" fechado 2023-03-08 lista componentes Apache-2.0 (echarts, openssl, mDNSResponder, aeron) y LGPL; aparece en Studocu, **no es una fuente oficial** y puede no ser de este producto. | Studocu (documento subido por tercero) | Muy baja; no usar como evidencia sin el fichero real |

### 6.2 Versiones

* `[PUBLICO]` Numeración tipo `0.x` hasta `0.99.x` y luego `1.x` (portal de terceros lista de 0.5.4 a 1.37.0; la fecha del portal es 2025-09 para 0.94.2). Confianza Baja-Media. La versión más reciente real debe leerse en el instalador o en el manifest de actualización.
* `[INDIRECTO]` El clon de terceros fija 1.27.0 como versión mínima y UA con Electron 36.4.0-alpha.17 / Chromium 136.0.7103.59 (`Libs/device_gen.py:23`, `Libs/domain_routing.py:18-24`). Esto indica que, al menos en la línea 1.27, la app es Electron.
* `[PUBLICO]` Descarga oficial: `https://www.tiktok.com/studio/download` (la página en español indica "Solo admite Windows 10 de 64 bits y versiones superiores"). Existe versión macOS (FAQ oficial).

### 6.3 Relación con Streamlabs y OBS

* `[PUBLICO]` Streamlabs Desktop y el "Streamlabs Plugin for OBS" ofrecen un camino para emitir a TikTok sin clave RTMP manual; TikTok exige contenido mayoritariamente de videojuegos en herramientas de terceros (Streamlabs content hub, 2026-04). Confianza Media (fuente comercial interesada).
* `[PUBLICO]` SE.Live (plugin de OBS) y agencias como vías alternativas; Atomik/TikFinity son fuentes comerciales (Confianza Baja).
* `[PUBLICO]` En 2021 el propio HN recoge que Streamlabs (antes "Streamlabs OBS") tuvo disputa de marca con OBS, no de licencia (comentarios contradictorios; no relevante para LIVE Studio).
* `[INDIRECTO]` El README del clon dice que prefiere otro proyecto del mismo autor basado en Streamlabs (`StreamLabsTikTokStreamKeyGenerator`), es decir: antes se usaba el flujo de Streamlabs para obtener claves; el flujo de LIVE Studio es "más nuevo" (README del clon, líneas 5 y siguientes).

### 6.4 Elegibilidad y requisitos (oficial)

| Dato | Fuente | Confianza |
|---|---|---|
| Se necesita "LIVE access". Acceso inmediato si se cumplen: antigüedad mínima de la cuenta, tiempo reciente en LIVE, número mínimo de seguidores, sin suspensión reciente. Los umbrales **varían por país/región y cambian sin aviso**. | Centro de ayuda oficial "Apply for LIVE access" (`https://www.tiktok.com/live/studio/help/article/Before-you-go-LIVE/Apply-for-LIVE-access`) | Alta |
| Streamers de otras plataformas pueden solicitar acceso con formulario (revisión en días). | Misma página | Alta |
| Prueba (trial): hay que hacer LIVE al menos 25 minutos, dos veces en la primera semana, o se revoca el acceso. | FAQ oficial (`.../FAQ/FAQ`, actualizado 2026-08-19) | Alta |
| Artículo oficial de creadores (antiguo): creadores de juegos en EE. UU. con 1.000 seguidores; no-gaming 10.000; fuera de EE. UU. 1.000. Mencionan "RTMP stream key" para quienes soliciten acceso. | `https://www.tiktok.com/live/creators/en-US/article/tiktok-live-studio-access_en-US` | Media (puede estar desactualizado frente al Help Center) |
| Requisitos de sistema Windows: mínimo i3 9.ª gen / Ryzen 5 3000, GTX 10 / Radeon, 8 GB; intermedio i5 11.ª / RTX 20, 16 GB; avanzado i7 12.ª / RTX 30, 16 GB. Mínimo permite solo cámara, 720p30; recomendado 1080p60. | FAQ y guía de resolución de problemas oficiales | Alta |
| Guías de terceros añaden: 18+ años, cuenta con 30+ días, un LIVE de 25+ min en los últimos 180 días (TikFinity). | Blog de terceros | Baja |
| "Software puede instalarse aunque la cuenta no cumpla requisitos; muestra los requisitos no cumplidos en gris". | FAQ oficial | Alta |

### 6.5 Flujo de login `[PUBLICO]`/`[INDIRECTO]`

* `[PUBLICO]` Login con cuenta TikTok (FAQ oficial: "Launch LIVE Studio and log in to your TikTok account").
* `[INDIRECTO]` Mecanismos observados por el clon (QR en `/passport/web/get_qrcode/` y `check_qrconnect/`, o navegador con `onetap_auth` y retorno por esquema `live-studio-app://login/continue`). Detalle en sección 7.

---

## 7. Evidencia indirecta: lo que imita el clon `tiktokstreamkeygenerator` `[INDIRECTO]`

Repositorio local leído (solo lectura, no ejecutado): `/home/user/loukious/tiktokstreamkeygenerator`. Ficheros: `TiktokStreamKeyGenerator.py` (6006 líneas), `Libs/domain_routing.py`, `Libs/device_gen.py`, `Libs/ffmpeg_sei_proxy.py`, `Libs/signers.py`, `Libs/XFrameSign.py`, `Libs/XArgus.py`, `Libs/XLadon.py`, `Updater.py`, `_version.py` (`__version__ = "1.0.0"`, línea 1). Referencia de rutas abreviada: **TSKG** = `TiktokStreamKeyGenerator.py`, **DR** = `Libs/domain_routing.py`, **DG** = `Libs/device_gen.py`, **SEI** = `Libs/ffmpeg_sei_proxy.py`.

**Advertencia de uso:** el clon obtiene firmas anti-abuso de TikTok mediante un servicio de terceros de pago (RapidAPI) y su propio README admite que el tráfico RTMP sin ese metadato "puede acabar en restricción de visibilidad por integridad/autenticidad". Se cita aquí **únicamente como evidencia del comportamiento del cliente oficial**. No se debe replicar la firma de peticiones ni suplantar al cliente oficial en un producto propio (contradice las reglas del proyecto y probablemente los términos de TikTok). Las firmas (`X-Argus`, `X-Ladon`, `X-Gorgon`/`x-ss-stub`, "frame sign") son un indicio de que el servidor valida que el emisor sea el cliente oficial.

### 7.1 Identidad del cliente (versión, UA, parámetros)

| Dato | Valor imitado | Cita |
|---|---|---|
| Versión mínima de LIVE Studio | `1.27.0` | DG:23 |
| User-Agent | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) TikTokLIVEStudio/1.27.0 Chrome/136.0.7103.59 Electron/36.4.0-alpha.17 TTElectron/36.4.0-alpha.17 Safari/537.36` | DR:18-24; DG:138-144; TSKG:575-578 |
| Cliente HTTP | `curl_cffi` con `impersonate="chrome"` (TLS fingerprint de Chrome) | DR:12-15; DG:26-29 |
| Cabeceras Chromium | `sec-ch-ua: "Not.A/Brand";v="99", "Chromium";v="136"`, `sec-ch-ua-platform: "Windows"`, `sec-fetch-*`, `sdk_aid: 8311`, `X-SS-DP: ""` | TSKG:573-593; DR:88-96 |
| `aid` (app id) | `8311` | TSKG:600; DG:193 |
| Parámetros comunes | `app_name=tiktok_live_studio`, `channel=studio`, `device_platform=windows`, `version_code=<ver>`, `webcast_sdk_version=<ver sin puntos>`, `live_mode=6`, `browser_name=Mozilla`, `browser_platform=Win32`, `screen_width/height`, `timezone_name`, `priority_region`, `device_id`, `install_id` | TSKG:595-620 |
| Cabecera `webcast-ntp-t0` / respuesta `webcast-ntp-t2/t3` | Sincronización de reloj tipo NTP con el servidor para calcular el `timestamp` de las firmas | DR:99-146 |
| Cabecera regional | `x-tt-store-region` | TSKG:683, 705, 734, 889 |
| Comprobación de versión (update) | `GET https://tron-sg.bytelemon.com/api/sdk/check_update?pid=7393277106664249610&uid=0&branch=studio/release/stable&buildId=0`; respuesta `data.manifest.win32.version` | DG:45-48, 107-132 |

Observación: el endpoint de actualización `tron-sg.bytelemon.com/api/sdk/check_update` es el mismo que r1ch cita en HN en 2021 (con `branch=ma...`), lo que cuadra con que sea el canal histórico de distribución (`[PUBLICO]` HN). Confianza Alta de que existe; el binario dirá cuál usa hoy.

### 7.2 Registro de dispositivo

* `POST` a `https://<host log.tiktokv.com resuelto>/service/2/desktop/device_register/` con cuerpo cifrado/empaquetado (`magic_tag: ss_app_log`, `sdk_version 1.0.2`, `package/display_name tiktok_live_studio`, campos `pc_uuid`, `pc_serial`, `mc` (MAC), `os_version`, `device_model`, `resolution`, `tz_*`) y UA `TTNetwork PC` (DG:44, 190-252, 276-298). Devuelve `device_id` e `install_id`. Estos campos revelan qué telemetría de hardware envía el cliente oficial (muy relevante para la sección de privacidad del informe).
* Cifrado del cuerpo de log: `Libs/log_encrypt_codec.py` (`magic_number=29795, version=3, sub_version=3`; DG:276-282).

### 7.3 Descubrimiento de dominios (TNC) y hosts

* TNC: `https://tnc0-normal-alisg.tiktokv.com/get_domains/v4/`, `https://tnc16-platform-useast1a.tiktokv.com/get_domains/v4/`, `https://tnc16-platform-alisg.tiktokv.com/get_domains/v4/`, con `aid=8311&ttwebview_version=1130022001&device_platform=win&tnc_src=6` (DR:27-48). Esto implica que el cliente oficial incluye **TTNet/TNC** (red de ByteDance sobre Chromium).
* Hosts resultantes de ejemplo: `webcast16-normal-c-alisg.tiktokv.com`, `api16-normal-c-alisg.tiktokv.com`, `webcast16-ws-c-alisg.tiktokv.com`, `webcast16-normal-no1a.tiktokv.eu`, `webcast16-normal-useast5.tiktokv.us`, `api16-normal-useast8.tiktokv.us`, seeds `webcast-normal.tiktokv.com`, `webcast.tiktokv.com`, `log.tiktokv.com`, `api.tiktokv.com` (DR:263-377, 399-455). Dominios: `tiktokv.com`, `tiktokv.eu`, `tiktokv.us`.
* Cookies que gobiernan el enrutado: `store-idc`, `store-country-code`, `tt-target-idc` (DR:306-348).

### 7.4 Flujo de login

1. **QR** (web passport): `GET https://<api16-normal-no1a.tiktokv.eu | api16-normal-c-alisg.tiktokv.com>/passport/web/get_qrcode/` y sondeo `/passport/web/check_qrconnect/`, con `aid=8311`, `account_sdk_source=web`, `sdk_version=2.1.9`, `verifyFp=verify_<device_id>`, `referer: https://www.tiktok.com/ucenter_web/live_studio/login` (TSKG:2185-2230, 2260-2300; constante `PASSPORT_WEB_SDK_VERSION` TSKG:137, `PASSPORT_APP_KEY` TSKG:138).
2. **Navegador + OIDC** ("onetap"): `POST {webcast}/passport/oidc/prepare/` con `channel=studio, app_name=tiktok_live_studio, device_type=PC, os_version` devuelve `state`, `nonce`, `ticket` (TSKG:2374-2385); se abre `https://www.tiktok.com/ucenter_web/onetap_auth?state&nonce&ticket&client_id=abad793d-e940-42cb-bc89-e6939b1b3b4d&redirect_uri=live-studio-app://login/continue` (TSKG:134-135, 2040-2047). El navegador devuelve un `id_token` (JWT) al esquema de URL **`live-studio-app://`**, y el cliente lo canjea en `POST {passport}/passport/oidc/login/` (TSKG:2393-2425). El clon registra temporalmente ese protocolo en `HKCU\Software\Classes\live-studio-app` (TSKG:2565-2660). **Sugiere que el cliente oficial registra el esquema `live-studio-app://`** en el sistema (hipótesis a buscar en el instalador NSIS: claves de registro).
3. Verificación de cuenta: `GET /passport/account/info/v2/` (TSKG:2346-2370; también 1766).
4. Cookies de sesión relevantes: `sessionid`, `sessionid_ss`, `sid_tt`, `sid_guard`, `multi_sids` (TSKG:2440).

### 7.5 API Webcast (crear sala, ingesta, salud)

Todas bajo el host Webcast resuelto, con parámetros de 7.1 y cabeceras firmadas.

| Endpoint | Uso | Cita |
|---|---|---|
| `POST webcast/room/create/` | Crea la sala. Cuerpo form: `title, live_studio=1, gen_replay, chat_auth, age_restricted, cover_uri, close_room_when_close_stream, hashtag_id, game_tag_id, game_bitrate_type=high, screenshot_cover_status, multi_stream_scene, gift_auth, chat_l2, star_comment_switch, multi_stream_source, is_group_live_session, open_commercial_content_toggle, commercial_content_promote_*, rtc_net_enabled=false`. Respuesta: `stream_url.rtmp_push_url` y `multi_stream_url.rtmp_push_url` (canvas horizontal/vertical, "dual layout"), `room_id`, `stream_id` | TSKG:838-905, 482-528 |
| `GET webcast/room/create_info/` | Info previa, `ban_status`, `advanced_live_ban_status`, `block_status`, `golive_locale_restricted` | TSKG:924, 1785, 1830-1855 |
| `GET webcast/game/basic/create_info/` (`scene=1`) | `has_live_studio_login`, `allow_multi_stream` | TSKG:1793-1810, 1839 |
| `GET webcast/anchor_tool/threshold/check/` | `{allowed, dual_canvas_used, days_to_reach}`: restricción de "funciones sensibles" para usuarios nuevos (25 min x N sesiones) | TSKG:1816-1825, 5072-5090 |
| `GET webcast/room/continue/` | Reanudar sala existente | TSKG:941 |
| `POST webcast/room/ping/anchor/` | Latido del anfitrión con estados `0 default, 1 prepare, 2 living, 3 pause, 4 finish`; "el Live Studio oficial empieza en prepare y pasa a living tras RoomIsLiving o conexión SDK" | TSKG:149-153 (constantes), 1060-1081, 1102-1110 |
| `POST webcast/room/anchor_pre_finish/` | Fin de directo | TSKG:1189 |
| `GET webcast/game/studio/realtime_stats/`, `.../trends_stats/` | Estadísticas | TSKG:1224, 1263 |
| `GET webcast/room/info/`, `webcast/ranklist/online_audience/` | Sala y audiencia | TSKG:1361, 1402 |
| `GET webcast/eco/violation_list/` | Violaciones de integridad (activas/historial) | TSKG:1532 |
| `POST webcast/room/upload/image/` | Miniatura | TSKG:1909 |
| `GET webcast/room/hashtag/list/` | Hashtags/juegos | TSKG:2450 |
| Códigos | `ROOM_IS_LIVING_CODE=4003150`, `ROOM_HAS_FINISHED_CODES={30003, 30003001}` | TSKG:155-156 |

Elegibilidad en la práctica (según el clon): `can_go_live` exige no estar `golive_locale_restricted`, sin `ban_status.is_ban`, sin `advanced_live_ban_status.is_ban`, `block_status` en 0, y `has_live_studio_login` verdadero (TSKG:1830-1840). Es coherente con "LIVE access" oficial `[PUBLICO]`.

### 7.6 Ingesta: RTMP/FLV con SEI firmado

* Destino: `rtmp_push_url` de la respuesta de `room/create` (RTMP; el clon enmascara URLs que empiezan por `rtmp://`, TSKG:2553-2555). No se observa RTMPS, WHIP ni SRT en el clon.
* El clon dice que "TikTok cierra de inmediato las conexiones directas de OBS" y que sin los metadatos de LIVE Studio el directo puede recibir restricción de integridad (README, sección Architecture). `[INDIRECTO]` Confianza Baja-Media (afirmación del autor).
* Proxy local FFmpeg/SEI: escucha `rtmp://127.0.0.1:19350/live/obs` y reenvía a TikTok añadiendo SEI a cada tag de vídeo FLV (TSKG:158-167, 4501-4560, 4648; SEI:1-100). Soporta H.264 (`codec_id 7`, NAL SEI `0x06`) y HEVC (enhanced FLV `hvc1/hev1`, NAL `0x4e 0x01`). Todo el FLV se reescribe tag a tag (SEI:61-100, 437-529).
* Payload SEI (tipo 100): prefijo ASCII `JSON` + JSON compacto con `live_sei_game_moment{timestamp}`, `live_sei_mute_mic{is_mute_mic}`, `push_video_width/height`, `sei_index`, `signResult` (firma "frame sign" calculada con `aid, uid, did, roomid, frametype="2", timestamp`), `ts`, `ttls_live_scene="live_studio"`; además un payload corto `video_e2e_delay{capture_window, encode}` (SEI:105-300; `signed_payload` en 201, `small_payload` en 271). Parámetros fijos del proxy: `--fps 60`, resolución 1920x1080 por defecto (TSKG:4524-4535).
* **Qué implica para el informe del binario:** buscar en la DLL multimedia/Electron las cadenas `live_sei_game_moment`, `live_sei_mute_mic`, `push_video_width`, `ttls_live_scene`, `video_e2e_delay`, `sei_index`, `signResult`; confirmaría el origen del SEI. Esto también delata que el SDK multimedia incluye un muxer FLV/RTMP propio o derivado (¿librtmp de OBS `obs-outputs`?), pregunta abierta clave.

### 7.7 Otras pistas

* Audio/vídeo y funciones: modo "dual layout" (canvas horizontal y vertical) con `multi_stream_url` (TSKG:500-515, 4677-4850).
* Comprobación de "foco" de ventana y polling para ahorrar llamadas de firma (TSKG:171-177): irrelevante para el cliente oficial.
* Registro del esquema de URL, ver 7.4.

### 7.8 Qué verificar en el binario (checklist derivada)

1. UA, `TTElectron`, `aid 8311`, `tiktok_live_studio`, `studio/release/stable`, `7393277106664249610` (pid de update).
2. Dominios `*.tiktokv.com|eu|us`, `bytelemon.com`, `tnc*`, `ttwstatic`, `tiktokcdn`.
3. Endpoints `webcast/room/*`, `passport/oidc/*`, `passport/web/get_qrcode`.
4. Esquema `live-studio-app`.
5. SEI: cadenas citadas arriba.
6. Mecanismo de firma (X-Argus/X-Ladon/frame sign): ¿en JS, `.node` o DLL `metasecml.dll` (nombre sugiere "MetaSec ML", antifraude)? Solo localizarlo y describirlo, no reproducirlo.

---

## 8. Implicaciones de licencia GPL para un producto propio

**Aviso:** análisis técnico-informativo, no asesoramiento legal. Antes de distribuir, revisar con un abogado de propiedad intelectual. Las citas de licencias son del texto estándar de cada licencia (URLs en Fuentes; las marcadas `[no abierta en esta sesión]` son referencias estándar que no se re-leyeron aquí).

### 8.1 Qué es OBS/libobs

* OBS Studio y **libobs** se distribuyen bajo **GPL-2.0-or-later** (`https://github.com/obsproject/obs-studio/blob/master/COPYING` `[no abierta en esta sesión]`). La documentación de libobs describe una arquitectura modular donde las fuentes, salidas y codificadores son plugins que se cargan dinámicamente (`https://docs.obsproject.com/backend-design`, abierta).
* Los plugins de OBS enlazan libobs, por tanto el resultado es obra derivada y debe ser compatible con GPL. Es el criterio habitual de la comunidad (FSF GPL FAQ sobre plugins `[no abierta en esta sesión]`).

### 8.2 Opción A: producto propio que enlaza libobs (o incrusta código de OBS)

Si se enlaza libobs (estática o dinámicamente) en el mismo programa, o se redistribuye una versión modificada:

1. El conjunto distribuido ("work based on the Program") queda bajo **GPL-2.0-or-later**: se debe ofrecer **el código fuente completo correspondiente** de todo lo enlazado, incluidos los scripts de compilación e instalación, bajo GPL (GPLv2 secciones 2 y 3; resumen en FOSSA, abierta).
2. Cada binario distribuido debe ir acompañado de (a) el código fuente, o (b) una **oferta escrita**, válida al menos 3 años, para cualquier tercero, a coste de distribución; o (c) acceso al código desde el mismo lugar de descarga (cita del texto GPLv2 §3 en el hilo HN, abierta).
3. Mantener avisos de copyright, texto de la licencia, indicar los cambios significativos.
4. No se pueden imponer restricciones adicionales (p. ej. EULA que prohíba redistribuir o hacer ingeniería inversa del programa, DRM que impida modificar y ejecutar).
5. **Combinación con código propietario:** no es posible mantener cerrada la parte enlazada. Alternativa sostenida por la comunidad: separar procesos con interfaz bien delimitada (IPC/sockets, "mera agregación"), pero la frontera es jurídicamente discutible, y los propios desarrolladores de OBS opinan que un producto que no funciona sin OBS es obra derivada (comentarios en HN, `[PUBLICO]`, Confianza Baja como opinión).
6. **Dependencias que condicionan:** x264 es GPL-2.0-or-later (a menos que se compre licencia comercial); FFmpeg es LGPL-2.1+ si se compila sin componentes GPL y sin `--enable-gpl`/`--enable-nonfree`; con libx264 enlazado pasa a GPL; con el SDK propietario de NVENC/QSV/AMF el binario resultante es no redistribuible bajo GPL si se enlazan librerías propietarias en el mismo proceso. OBS soluciona esto cargando NVENC/QSV/AMF desde el driver del sistema (no se redistribuye).
7. **Licencia de OBS no incluye uso de marca**: "OBS" es marca; no usar el nombre ni logotipo (hubo disputa con Streamlabs `[PUBLICO]`).
8. **Compromiso GPL de OBS** (GPL Cooperation Commitment): OBS afirma tratar las infracciones "de buena fe" y dar plazo para subsanar (`[PUBLICO]` HN/Verge). No es una licencia ni una garantía.

Resumen A: viable si el producto propio será **GPL y open source** (o si solo se usa OBS como programa externo, sin enlazarlo).

### 8.3 Opción B: pipeline propio

| Componente | Licencia | Obligaciones prácticas |
|---|---|---|
| FFmpeg (libavcodec, libavformat, libswscale...) | LGPL-2.1+ por defecto (si no se activa `--enable-gpl`) | Enlace dinámico recomendable; permitir reemplazar la biblioteca; incluir texto LGPL y aviso; ofrecer el código de FFmpeg usado y de tus modificaciones a FFmpeg; no usar componentes GPL/nonfree. Ver `https://ffmpeg.org/legal.html` `[no abierta]` |
| x264 | GPL-2.0-or-later (o licencia comercial de pago) | Enlazarlo vuelve GPL todo el binario. Alternativas: no incluirlo y usar codificadores hardware, o OpenH264 (BSD, con cuestiones de patentes/licencia de Cisco), o licencia comercial x264 |
| NVENC (NVIDIA Video Codec SDK) | Licencia propietaria de NVIDIA para SDK; la librería `nvEncodeAPI64.dll` viene con el driver | Se puede usar desde un producto propietario (cargando del driver). Verificar términos del SDK |
| Intel QSV / oneVPL | MIT (oneVPL dispatcher); runtime en driver | Permisivo |
| AMD AMF | MIT (headers AMF); runtime en driver | Permisivo |
| libx265 | GPL-2.0 (o comercial) | Igual que x264 |
| SVT-AV1 | BSD-2/3 con patente (AOM) | Permisivo |
| Protocolos | librtmp (LGPL-2.1), implementación propia, WebRTC (BSD-3), SRT (MPL-2.0) | Según librería elegida |
| Captura de pantalla (WGC/DXGI) | APIs de Windows | Sin implicación GPL |
| Electron/Chromium | MIT / BSD y otras (notas en `LICENSES.chromium.html`) | Distribuir avisos |
| Qt (si UI nativa) | LGPL-3 / comercial | Enlace dinámico y cumplir LGPL, o licencia comercial |

Resumen B: es perfectamente posible un producto **cerrado** si se evita GPL (x264/x265, libobs) y se cumplen LGPL (FFmpeg/Qt) y las licencias de terceros.

### 8.4 Lo que se sabe sobre cómo TikTok cumple o no con la GPL (solo con fuentes)

Hechos documentados:

* `[PUBLICO]` Dic. 2021: OBS (dos desarrolladores) afirmó públicamente que había código derivado de OBS en binarios de LIVE Studio sin oferta de código fuente ni licencia alternativa; dijo que lo resolvería en privado y de buena fe. Fuentes: HN, Engadget, Verge, FOSSA, GIGAZINE, Habr, Inverse, Drew DeVault (opinión). Confianza Alta de que la acusación existió; Media-Alta de su fundamento (declaración de un desarrollador de OBS tras desensamblado; FOSSA señaló que no pudo corroborar los ficheros por su cuenta).
* `[PUBLICO]` TikTok no había respondido a OBS "a fecha de la nota" (Engadget 2021-12-21) y quitó el enlace público de descarga (Inverse/Habr).
* `[PUBLICO]` Búsquedas realizadas en esta sesión **no encontraron** (a) una resolución pública, (b) una publicación de código por parte de TikTok/ByteDance, ni (c) una declaración posterior de OBS. Es decir: **el desenlace es desconocido**. No se afirma ni cumplimiento ni incumplimiento actual.
* `[PUBLICO]` El Help Center oficial actual describe LIVE Studio como "desarrollado por TikTok" y no menciona OBS (FAQ oficial). Esto no es evidencia de cumplimiento ni de incumplimiento.
* `[PUBLICO]` (Baja-Media) Listados de terceros muestran que las versiones 0.x recientes incluyen un fichero `open source notice` y `LICENSE.electron.txt` / `LICENSES.chromium.html`. Que exista un aviso de código abierto **no** equivale a cumplir GPL (que exige oferta de código fuente); su contenido real se desconoce.
* `[PUBLICO]` OBS incluyó (oct. 2024) la DLL de la cámara virtual de TikTok Live Studio en su lista de bloqueo por problemas técnicos, no por licencia.

**Pendiente de verificar en el binario** (sección 4, paso 10): contenido del `open source notice`, presencia de libobs/x264/FFmpeg con banner GPL, y cualquier oferta escrita o URL de código fuente. Hasta entonces, la redacción correcta es: "se reportó públicamente en 2021 y no se conoce el desenlace".

---

## 9. Hipótesis de trabajo (ordenadas por confianza) y cómo se confirmarán

| # | Hipótesis | Evidencia actual | Cómo confirmar/refutar |
|---|---|---|---|
| H1 | La UI de LIVE Studio actual es Electron (Chromium 136, Electron 36 con fork "TTElectron") | `[INDIRECTO]` UA (DR:18-24) + `[PUBLICO]` portal con `LICENSE.electron.txt`, `resources\app` | Paso 3 |
| H2 | La captura/codificación/ingesta corre en un SDK nativo propio (`mediasdk_server.exe`, `MediaSDK_V1\*.dll`, `LSVCam.dll`) fuera de Electron | `[PUBLICO]` Hanselman (ruta `resources\app\electron\sdk\lib\MediaSDK_V1`), portal 3.º | Pasos 3, 5, 6 |
| H3 | Ese SDK conserva código derivado de libobs (2021 sí, según OBS) | `[PUBLICO]` HN 2021; sin datos 2025/2026 | Paso 5 (exports `obs_*`, cadenas, `graphics-hook` análogo) |
| H4 | La ingesta es RTMP/FLV a `rtmp_push_url` con SEI tipo 100 firmado | `[INDIRECTO]` clon (SEI y TSKG) | Pasos 7, 8 |
| H5 | El login usa passport (QR) y OIDC con retorno `live-studio-app://login/continue` | `[INDIRECTO]` TSKG | Pasos 8, instalador NSIS (registro) |
| H6 | El instalador es NSIS/propio con `Launcher.exe` + carpeta por versión + `app_shell_updater.exe` | `[PUBLICO]` portal 3.º (Baja) | Pasos 1, 2 |
| H7 | Hay `open source notice` pero no oferta de código fuente GPL | Desconocido | Paso 10 |

---

## 10. Datos que solo puede aportar el usuario

1. **El instalador oficial** descargado de `https://www.tiktok.com/studio/download` (con su nombre de fichero original), adjuntado a la sesión. El agente no puede obtenerlo (política de red: `tiktok.com`, CDNs de TikTok y `web.archive.org` devuelven 403 en CONNECT) y no se usarán mirrors ni portales de terceros.
2. **Versión instalada** (Configuración > Aplicaciones, o `Acerca de` en LIVE Studio), y si el instalador es "online/stub" o "full".
3. **Carpeta instalada** comprimida (`C:\Program Files\TikTok LIVE Studio\<versión>\`) **sin cuentas ni credenciales**: permite análisis aun si el instalador solo descarga la carga útil. Excluir `AppData\Roaming\TikTok LIVE Studio` (contiene sesión/cookies; no enviar).
4. **Captura de tráfico del PC con mitmproxy** (o Fiddler/pcap), con la cuenta del propio usuario y sin compartir cookies: arrancar LIVE Studio con proxy (`--proxy-server` puede no aplicar; usar proxy del sistema), exportar **HAR** o `.mitm` y, si es posible, un **pcap** (`pktmon`/Wireshark) para ver el tráfico RTMP que no pasa por HTTP. Marcar con `--sensitive-value` los tokens para que `rea` los redacte. Posible pinning de certificados: anotar si falla.
5. **Capturas de pantalla** de: pantalla de acceso/elegibilidad ("Your LIVE access"), `Acerca de`/licencias/avisos de código abierto, ajustes de salida (codificador, bitrate, resolución, protocolo), selector de codificador (x264/NVENC/QSV/AMF).
6. **Regiones y cuenta**: país de la cuenta, si tiene LIVE access, para validar los requisitos de elegibilidad del Help Center.
7. **Log local de la aplicación** (carpeta de logs de LIVE Studio) redactado, con información de módulos cargados y del codificador usado en un directo de prueba.
8. **Autorización explícita** si se quiere instalar en una VM desechable (análisis dinámico) o descargar Ghidra/Hopper (descarga de software que sí requiere decisión del usuario).
9. **Requisitos de producto**: si el producto propio será open source (GPL aceptable) o cerrado (evitar libobs/x264), para decidir la opción A o B de la sección 8.

---

## 11. Estado

* **Análisis binario: NO realizado (instalador no disponible por bloqueo de red).**
* Herramienta `rea` 6.5.0: instalada y verificada con ficheros sintéticos; analizadores nativos no instalados.
* Desempaquetadores (7z, innoextract, msitools, cabextract), osslsigncode, exiftool, yara, pefile, lief: instalados y verificados.
* Playbook: completo y probado en lo que no requiere el binario (scripts `pe_inventory.py` y `obs.yar` ejecutados sobre un PE de prueba; `rea` sobre un `.asar` y un PE sintéticos).
* Investigación pública: realizada (decenas de fuentes revisadas, con Confianza indicada por afirmación).
* Evidencia indirecta del clon: leída (solo lectura, nada ejecutado).
* No se evadió ninguna política de red, captcha ni antiabuso; no se descargó ningún instalador ni mirror; no se usaron credenciales de terceros; no se configuró MCP.

---

## 12. Fuentes

Oficiales de TikTok (`[PUBLICO]`, abiertas vía Exa, no desde la red del entorno):

* Apply for LIVE access: https://www.tiktok.com/live/studio/help/article/Before-you-go-LIVE/Apply-for-LIVE-access
* FAQ (LIVE Studio Help Center, actualizado 2026-08-19): https://www.tiktok.com/live/studio/help/article/FAQ/FAQ
* Install and start LIVE Studio: https://www.tiktok.com/live/studio/help/article/Before-you-go-LIVE/Install-and-start-LIVE-Studio
* Página de descarga: https://www.tiktok.com/studio/download y https://www.tiktok.com/studio/faq
* Guía de resolución de problemas: https://www.tiktok.com/live/creators/en-US/article/live-studio-troubleshooting-guides_en-US
* Artículo de acceso de creadores (antiguo): https://www.tiktok.com/live/creators/en-US/article/tiktok-live-studio-access_en-US

GPL / OBS:

* Engadget (2021-12-21): https://www.engadget.com/tiktok-gpl-violation-obs-163509367.html
* The Verge (2021-12-20): https://www.theverge.com/2021/12/20/22847213/tiktoks-live-studio-obs-open-source-policy
* FOSSA (2021-12-22): https://fossa.com/blog/does-tiktok-live-studio-violate-the-gpl-v2/
* GIGAZINE (2021-12-20): https://gigazine.net/gsc_news/en/20211220-tiktok-live-studio-obs-source-code/
* Hacker News (hilo y comentario de r1ch): https://news.ycombinator.com/item?id=29592103 y https://news.ycombinator.com/item?id=29595612
* Habr (2021-12-18): https://habr.com/ru/news/596441/
* Inverse (2021-12-28): https://www.inverse.com/input/gaming/tiktoks-live-studio-app-accused-of-violating-obs-licensing-terms
* Open Source For You (2021-12-22): https://www.opensourceforu.com/2021/12/obs-licencing-policy-is-allegedly-violated-by-tiktoks-new-live-studio-app/
* Drew DeVault (opinión): https://drewdevault.com/blog/Commercial-forks-of-FOSS-projects/
* Tuit citado de @dodgepong: https://twitter.com/dodgepong/status/1471522226520346632 `[no abierto]`
* OBS commit de bloqueo de DLL: https://github.com/obsproject/obs-studio/commit/ab48cd0b3d4eb2eb8794bddcbdadf2b92dd09265
* Hanselman (2024-10-09): https://www.hanselman.com/blog/webcam-randomly-pausing-in-obs-discord-and-websites-lsvcam-and-tiktok-studio
* Diseño de libobs: https://docs.obsproject.com/backend-design

Estructura del producto (Confianza Baja, terceros):

* Portal de desinstalación con listado de ficheros v0.94.2: https://www.advanceduninstaller.com/TikTok-LIVE-Studio-acba5f5fc056f61cdb046ae34737ba65-application.htm (otras versiones: 0.83.2 `...e5dad0ab69b76026c1e6b901e28935ae...`, 0.85.0 `...c982d109ab61d9785cbb467e70c1ca08...`)
* ttls-controller (Stream Deck/Socket.IO): https://github.com/slyboots/ttls-controller

Alternativas de streaming / requisitos de terceros (Confianza Baja-Media):

* Streamlabs: https://streamlabs.com/content-hub/post/how-to-find-your-tiktok-stream-key
* TikFinity: https://blog.tikfinity.com/how-to-get-your-tiktok-stream-key-in-2025/
* LiveU: https://studiosupport.liveu.tv/hc/en-us/articles/17094667853083-TikTok

Herramienta:

* rea: https://github.com/morluto/rea (clon local `/tmp/claude-0/work/rea`, docs en `/tmp/claude-0/work/rea/docs`)

Licencias (referencias estándar, `[no abiertas en esta sesión]`):

* GPLv2: https://www.gnu.org/licenses/old-licenses/gpl-2.0.html
* GPL FAQ plugins: https://www.gnu.org/licenses/gpl-faq.html#GPLPlugins
* OBS COPYING: https://github.com/obsproject/obs-studio/blob/master/COPYING
* FFmpeg legal: https://ffmpeg.org/legal.html
* x264: https://www.videolan.org/developers/x264.html

Código local leído (`[INDIRECTO]`):

* `/home/user/loukious/tiktokstreamkeygenerator/TiktokStreamKeyGenerator.py`
* `/home/user/loukious/tiktokstreamkeygenerator/Libs/domain_routing.py`
* `/home/user/loukious/tiktokstreamkeygenerator/Libs/device_gen.py`
* `/home/user/loukious/tiktokstreamkeygenerator/Libs/ffmpeg_sei_proxy.py`
* `/home/user/loukious/tiktokstreamkeygenerator/Libs/signers.py` (URL del servicio de firma de terceros en líneas 44-45)
* `/home/user/loukious/tiktokstreamkeygenerator/README.md`

---

## Apéndice A. Scripts auxiliares (probados con un PE de prueba)

### A.1 `pe_inventory.py`

```python
#!/usr/bin/env python3
"""Inventario estatico de PE: hash, version, exports OBS, imports, firma. No ejecuta nada."""
import csv, hashlib, os, re, sys
import pefile

OBS_EXPORT_RE = re.compile(r"^(obs_|gs_|audio_|video_|os_|bmalloc|bfree|base_|config_|dstr_|da_|text_lookup|profile_)")

def sha256(path):
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()

def version_info(pe):
    out = {}
    for fi in getattr(pe, "FileInfo", []) or []:
        for entry in fi:
            if getattr(entry, "Key", b"") == b"StringFileInfo":
                for st in entry.StringTable:
                    for k, v in st.entries.items():
                        out[k.decode(errors="replace")] = v.decode(errors="replace")
    return out

def main(root, out_csv):
    rows = []
    for dp, _, files in os.walk(root):
        for name in files:
            if not name.lower().endswith((".exe", ".dll", ".node", ".sys", ".ocx")):
                continue
            p = os.path.join(dp, name)
            row = {"path": os.path.relpath(p, root), "size": os.path.getsize(p), "sha256": sha256(p)}
            try:
                pe = pefile.PE(p, fast_load=False)
                vi = version_info(pe)
                row.update({
                    "machine": hex(pe.FILE_HEADER.Machine),
                    "is_dll": bool(pe.FILE_HEADER.Characteristics & 0x2000),
                    "ProductName": vi.get("ProductName", ""),
                    "ProductVersion": vi.get("ProductVersion", ""),
                    "FileVersion": vi.get("FileVersion", ""),
                    "CompanyName": vi.get("CompanyName", ""),
                    "LegalCopyright": vi.get("LegalCopyright", ""),
                    "has_signature": bool(pe.OPTIONAL_HEADER.DATA_DIRECTORY[4].Size),
                })
                exports = []
                if hasattr(pe, "DIRECTORY_ENTRY_EXPORT"):
                    exports = [s.name.decode(errors="replace") for s in pe.DIRECTORY_ENTRY_EXPORT.symbols if s.name]
                row["n_exports"] = len(exports)
                row["obs_like_exports"] = ";".join([e for e in exports if OBS_EXPORT_RE.match(e)][:40])
                imports = []
                if hasattr(pe, "DIRECTORY_ENTRY_IMPORT"):
                    imports = sorted({d.dll.decode(errors="replace").lower() for d in pe.DIRECTORY_ENTRY_IMPORT})
                row["imports_dlls"] = ";".join(imports)
                pe.close()
            except Exception as exc:  # archivo no PE o corrupto
                row["error"] = f"{type(exc).__name__}: {exc}"
            rows.append(row)
    keys = sorted({k for r in rows for k in r}, key=lambda k: (k not in ("path","size","sha256"), k))
    with open(out_csv, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=keys)
        w.writeheader()
        w.writerows(rows)
    print(f"{len(rows)} PE/archivos escritos en {out_csv}")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        sys.exit("uso: pe_inventory.py <directorio_extraido> <salida.csv>")
    main(sys.argv[1], sys.argv[2])
```

### A.2 `obs.yar`

```yara
rule obs_libobs_markers {
  meta: description = "Marcadores de codigo derivado de OBS Studio / libobs (indicio, no prueba)"
  strings:
    $a1 = "libobs" ascii wide nocase
    $a2 = "obs_source_info" ascii
    $a3 = "obs_output_info" ascii
    $a4 = "obs_encoder_info" ascii
    $a5 = "obs_module_load" ascii
    $a6 = "obs_get_version_string" ascii
    $a7 = "OBS Studio" ascii wide
    $a8 = "rtmp-services" ascii wide
    $a9 = "graphics-hook" ascii wide
    $a10 = "get-graphics-offsets" ascii wide
    $a11 = "inject-helper" ascii wide
    $a12 = "obsproject.com" ascii wide
    $a13 = "libobs/obs-" ascii
    $a14 = "libobs-d3d11" ascii wide
  condition: 2 of them
}
rule encoders_markers {
  meta: description = "Codificadores y protocolos"
  strings:
    $x264 = "x264 - core" ascii
    $x264b = "libx264" ascii wide
    $nv1 = "nvEncodeAPI" ascii wide
    $nv2 = "h264_nvenc" ascii
    $nv3 = "hevc_nvenc" ascii
    $qsv1 = "libmfx" ascii wide
    $qsv2 = "h264_qsv" ascii
    $amf1 = "amfrt64" ascii wide
    $amf2 = "AMFVideoEncoder" ascii wide
    $rt1 = "rtmps://" ascii wide
    $rt2 = "rtmp://" ascii wide
    $whip = "whip" ascii wide nocase
    $srt = "srt://" ascii wide
    $libavc = "libavcodec" ascii wide
  condition: any of them
}
```

Nota sobre falsos positivos: la regla `libobs` con `nocase` y `wide` puede coincidir con cadenas de Chromium o de bibliotecas no relacionadas; tratar un solo hit como indicio y confirmar con exports `obs_*` o con decompilación.
