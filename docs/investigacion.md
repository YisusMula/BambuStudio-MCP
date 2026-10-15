# Investigación: controlar Bambu Studio desde Claude con MCP

Resumen de lo investigado y verificado (octubre de 2026). Las referencias `archivo:línea` son del código de Bambu Studio en la etiqueta `v02.08.04.61` o en master `47a13ee`.

## 1. Qué trae Bambu Studio oficial

- **No tiene API, plugins ni scripting.** No hay Python, Lua ni sistema de extensiones. Bambu no ha respondido a las peticiones de API (issues #1443 de 2023 y #9614 de febrero de 2026, sobre LLM/MCP).
- **Puertas de entrada que sí existen en la app en marcha:**
  - **Reenvío de archivos de instancia única.** En Windows usa WM_COPYDATA, en Linux D-Bus y en macOS NSDistributedNotification (`InstanceCheck.cpp`). Solo acepta rutas de archivos existentes y las abre con `Plater::load_files`. Un `.3mf` sobre una placa con objetos muestra siempre un diálogo.
  - **`bambustudio://open?file=<url>`.** Solo descarga modelos por http(s), pensado para MakerWorld, y pide confirmación si el dominio no es de confianza. No sirve para archivos locales.
  - **`HttpServer.cpp` (127.0.0.1:13618).** Solo gestiona el inicio de sesión (`/refresh_token`, ticket). No sirve para controlar la app.
  - **Atajos de teclado** (`MainFrame.cpp:714-736`, `GLCanvas3D.cpp:4731-4749`): Ctrl+R lamina (se salta las comprobaciones del AMS), Ctrl+G exporta, Ctrl+Shift+G abre el diálogo de imprimir, A organiza, R orienta (con Shift, solo la placa actual).
- **Línea de comandos:** permite laminar sin interfaz (`--slice`, `--export-3mf`, `--load-settings`, `--arrange`, `--orient`), pero no actúa sobre la ventana abierta.
- **Automatizar la interfaz** (computer use o accesibilidad) es frágil: hay unos 628 diálogos modales, el lienzo 3D y los paneles ImGui no se ven por accesibilidad, y la búsqueda de ajustes no funciona con teclado.

## 2. Proyectos públicos (revisados el 14-oct-2026)

| Proyecto | ¿Controla la ventana de Bambu Studio? | Notas |
|---|---|---|
| **PR bambulab/BambuStudio#12485** (otarkhan) | Sí, 90 herramientas | Servidor MCP dentro de la app. Solo escucha en 127.0.0.1:27183/mcp, exige token, viene apagado por defecto y las acciones de impresora van aparte. Sin revisar, sin binarios, solo compilado en macOS arm64. **Es la base de este repo.** |
| PR #12481 (nemalon) | Sí, 9 herramientas | **Inseguro:** sin autenticación, acepta cualquier origen (CORS `*`) y `set_slicing_params` puede fijar `post_process`, que se ejecuta en el shell. No usar. |
| Ding-Ding-Projects/BambuStudio ("MD3") | Sí, solo su propio fork | Solo Windows, interfaz rehecha, sin firmar, se auto-actualiza, MCP limitado (5 ajustes numéricos), nunca probado de punta a punta. Desaconsejado. |
| otarkhan/bambu-studio-mcp | — | Repo vacío; la versión "anterior" del PR nunca se publicó. |
| bambu-printer-mcp, schwarztim/bambu-mcp, beambam, spikeon/bambu-studio-mcp, bambu-studio-profile-mcp… | No | Laminan por línea de comandos, controlan la impresora (MQTT/FTPS) o editan perfiles. |
| MaxEllis/OrcaSlicer + `orcaslicer-mcp` | Sí, pero en OrcaSlicer | Listo para usar en Windows, Mac y Linux. No puede imprimir y sus herramientas de impresora rechazan las Bambu. |
| QIDI Studio MCP Edition (sslinkyy) | Sí, en QIDI Studio | Solo impresoras QIDI. |
| ⚠️ offthehook-implication870/bambu-printer-mcp | — | Probable malware: empuja a descargar un ZIP escondido en el repo. |

## 3. Compilar una versión propia: qué se mantiene y qué no

- **Opciones obligatorias:** `-DBBL_RELEASE_TO_PUBLIC=1 -DBBL_INTERNAL_TESTING=0`, en modo Release. Sin ellas la app usa los servidores de pruebas (`api-pre.bambulab.net`) y una carpeta de datos "Internal". `build_win.bat` y los presets de MSVC **no** pasan esas opciones; el CI oficial sí.
- **Plugin de red:** se descarga según la serie de versión (`GUI_App.cpp:1646`, `02.08.04.xx` → `02.08.04.00`). Hay que compilar desde una etiqueta publicada y nunca inventar una versión.
- **Actualizaciones:**
  - **App:** no se actualiza sola. La app solo abre el instalador oficial en el navegador, y ese instalador no lleva MCP.
  - **Perfiles:** llegan por la línea 02.08.
  - **Plugin:** recibe parches dentro de la serie 02.08.04.
  - **Firmware:** se actualiza por la impresora.
- **Imprimir:**
  - **Linux:** muy probablemente funciona (la versión de Flathub está compilada por terceros).
  - **Windows:** riesgo, porque el plugin comprueba la firma del `.exe` (posible pérdida de la cámara remota o de la impresión por la nube; no probado).
  - **Siempre funciona:** el modo LAN con Developer Mode.
- **Versión base:** el parche del PR #12485 solo se aplica en la línea 2.8.4 (beta). La última estable, `v02.08.02.61`, no es compatible.
- **Política de Bambu:** modificar el código AGPL está permitido; "suplantar al cliente oficial" ante su nube, no.

## 4. Decisiones de este repo

- **Plataforma y visibilidad:** Windows, repo público (GitHub Actions gratis).
- **Contenido:** el repo solo guarda parches y workflows; el CI descarga Bambu Studio oficial en la versión de `UPSTREAM_TAG`.
- **Parches:**
  - `0001` es el PR #12485 sin cambios.
  - `0002` añade tres correcciones:
    - la conversión de ruta en `McpAdvanced.inc`, para que compile en Windows;
    - recuperar la cancelación forzada de Bambu para los laminados que no son del MCP;
    - bloquear el cierre de la ventana solo durante operaciones del MCP.
- **Pendiente:**
  - la primera compilación en Actions;
  - probar en la impresora real: iniciar sesión, LAN, nube, cámara y, por último, el MCP.
