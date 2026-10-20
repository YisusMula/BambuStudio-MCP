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
  - `0003` solo toca el CMake de los tests (`WIN32_LEAN_AND_MEAN`, `BOOST_ALL_NO_LIB`, `crypt32`).
  - `0004` deja como la oficial cuatro cambios del PR que también afectaban sin MCP (ver sección 5).
- **Pendiente:**
  - probar en la impresora real: iniciar sesión, LAN, nube, cámara y, por último, el MCP.

## 5. Revisión: ¿se comporta igual que la oficial salvo el MCP? (oct-2026)

Revisión hunk a hunk de `v02.08.04.61` contra `0001`+`0002`+`0003`, con dos verificadores independientes por hallazgo.

**Con el MCP apagado (por defecto) es igual que la oficial**, salvo lo que corrige `0004` y el workflow:
- Sin hilos, sockets, temporizadores ni archivos nuevos. El manifiesto de 90 herramientas se lee al primer `tools/list`. `mcp.token` solo se crea al activar el MCP o pulsar los botones del token.
- El laminado y el G-code no cambian. `0002` recupera exactamente la cancelación forzada oficial (1 s en `stop()`, 5 s en `stop_internal()`).
- El cierre de la ventana y la versión del plugin de red (`get_plugin_url`) son idénticos.
- Diferencias que quedan, todas inocuas:
  - la sección "Local MCP server" en Preferencias → General, sin traducir;
  - cuatro claves `mcp_*` en el `BambuStudio.conf` compartido; la oficial las conserva sin problema.

**Corregido en `0004`** (venía de `0001` y afectaba sin MCP):
- `GUI_App::can_recreate_GUI()` impedía, sin avisar, cambiar el idioma o el LOD mientras se laminaba o había una tarea. Ahora solo lo impide el trabajo del MCP.
- `ExclusiveJobGroup::start()` ignoraba organizar, orientar o rellenar la placa si la tarea anterior no se paraba en 10 s.
- `Job::start()`: si no se podía crear el hilo, pasaba por `on_exception()` en vez de `finalize()`. Con `UpgradeNetworkJob` acababa en recursión infinita.
- `BackgroundSlicingProcess`: los eventos de los laminados normales (progreso, laminado terminado, fin o cancelación) pasaban por `McpWorkerLifetime::publish`. Eso añadía un mutex por mensaje y, tras el cambio de idioma de arriba, se tragaba el evento de cancelación del laminado antiguo. Ahora solo pasan por ahí los del MCP; los normales se publican directamente, como en la oficial.

**Corregido en el workflow:** el `WIN10SDK_PATH` 10.0.22000.0, copiado del CI oficial, no existe en `windows-2025-vs2026`. Sin él no se define `HAS_WIN10SDK` y desaparecen "Reparar modelo" (Netfabb), la reparación tras el corte y la de la importación de texturas. El CI público de Bambu tiene el mismo fallo; el instalador oficial no. Ahora el workflow detecta el SDK y falla si se va a compilar sin Netfabb.

**Sin corregir, solo afectan con el MCP en uso:**
- Mientras hay un laminado del MCP, las esperas de `stop()`/`stop_internal()` no tienen límite. Laminar, exportar y cerrar se rechazan sin aviso visible (`show_status_message` solo escribe en el log).
- Si una importación por MCP choca con un diálogo (objeto demasiado grande), el objeto puede quedar en el `Model` sin fila en la lista de objetos.
- Un `project.open` fallido deja el proyecto a medias.
- Varias herramientas (`export.geometry`, `view.capture`, archivos en `%TEMP%`) usan `std::filesystem::path(...).string()`, que en Windows usa la página de códigos ANSI. Pueden fallar con rutas con acentos (p. ej. `C:\Users\José`).
- `mcp_orphans_running()` solo cuenta los hilos huérfanos del último `start()`. Si tras una cancelación forzada se lamina otra vez a mano, el MCP puede empezar a laminar mientras el huérfano antiguo sigue en marcha. La oficial ya permite dos hilos sobre el mismo `Print` en ese caso.
- `ExclusiveJobGroup::start()` llama a `stop_all()` antes de comprobar si hay una tarea del MCP sin terminar. Orientar a mano (tecla R, que no comprueba nada) mientras el MCP organiza cancela la tarea del MCP y no arranca la del usuario.

**Revisado y sin efecto observable:** `BackgroundSlicingProcess::start()` vuelve a instalar el callback de progreso en cada laminado. La oficial ya lo instala en `update_slice_context()` justo antes, y para los laminados normales es el mismo callback.

**Empaquetado:**
- El zip no lleva el runtime de Visual C++ y se compila con MSVC 19.51 (v145); hace falta el redistributable reciente.
- El binario no está firmado.
- Los recursos de texto van con LF (sin efecto conocido).
- El resto del pipeline coincide con `build_bambu.yml` oficial.
