# BambuStudio-MCP

Bambu Studio oficial con un **servidor MCP integrado**, para que Claude (u otro cliente MCP) pueda controlar la aplicación abierta: abrir y guardar proyectos, importar modelos, mover objetos, gestionar placas, cambiar perfiles y ajustes, organizar, orientar, laminar, previsualizar y exportar.

Este repositorio **no contiene el código de Bambu Studio**. Solo guarda:

| Archivo | Qué es |
|---|---|
| `UPSTREAM_TAG` | Versión oficial de Bambu Studio que se compila (ahora `v02.08.04.61`, línea 2.8.4 beta). |
| `patches/0001-native-mcp-server.patch` | El servidor MCP del [PR #12485](https://github.com/bambulab/BambuStudio/pull/12485) de otarkhan (90 herramientas), sin cambios. |
| `patches/0002-windows-fixes-and-upstream-behaviour.patch` | Nuestras correcciones (ver abajo). |
| `patches/0003-fix-mcp-tests-windows-build.patch` | Hace que los tests del servidor MCP compilen en Windows. |
| `patches/0004-upstream-parity-without-mcp.patch` | Quita cuatro cambios del PR que también afectaban a quien no usa el MCP (ver abajo). |
| `.github/workflows/build-windows.yml` | Descarga Bambu Studio, aplica los parches y compila la versión portable de Windows. |
| `.github/workflows/check-upstream.yml` | Cada día comprueba si Bambu ha publicado una versión nueva y abre un issue. |

## Correcciones sobre el PR #12485

1. **Compilación en Windows:** `export.geometry` pasaba una ruta `std::filesystem::path` (texto ancho en Windows) a una función que espera `const char*`.
2. **Cancelar un laminado normal ya no puede congelar la app:** el PR quitaba la cancelación forzada con tiempo límite que trae Bambu Studio. Se recupera para los laminados normales; los lanzados por el MCP mantienen el comportamiento del PR.
3. **Cerrar la ventana:** el PR impedía cerrar Bambu Studio durante cualquier laminado. Ahora solo lo impide mientras hay una operación del MCP en marcha.
4. **Tests en Windows:** el ejecutable de tests no compilaba (`<windows.h>` incluía `winsock.h` antes de Boost.Asio). Además MSVC intentaba enlazar automáticamente una `.lib` de Boost con un nombre que no existe. Se definen `WIN32_LEAN_AND_MEAN` y `BOOST_ALL_NO_LIB` y se enlaza `crypt32`, como ya hace la app, solo para los tests; la app no cambia.
5. **Igual que la oficial cuando no se usa el MCP** (`0004`):
   - Cambiar el idioma o activar el LOD en Preferencias funciona mientras se lamina u organiza. El PR lo impedía sin avisar.
   - Organizar, orientar o rellenar la placa arrancan aunque la tarea anterior tarde en pararse, como en la oficial. El PR las ignoraba sin avisar.
   - Los avisos de un laminado normal (progreso, terminado, cancelado) llegan por el mismo camino que en la oficial. El PR los hacía pasar por el control del MCP.
   - Si Windows no puede crear el hilo de una tarea, se muestra el mensaje de la oficial. Con el PR salía otro error y, si pasaba al descargar el plugin de red, la app se cerraba de golpe. Es un caso muy raro.
6. **"Reparar modelo" en Windows:** el CI de Bambu fija un Windows SDK (10.0.22000.0) que ya no está en los runners de GitHub, y sin él la opción se compila fuera sin avisar. El workflow busca el SDK disponible y falla si la opción no se va a compilar.

## Instalar (Windows x64)

1. Descarga el `.zip` de la última versión en [Releases](../../releases) y, si quieres, comprueba el `.sha256`.
2. Descomprímelo en una carpeta (por ejemplo `C:\BambuStudio-MCP`) y abre `bambu-studio.exe`.
   - No está firmado: Windows SmartScreen avisará. Pulsa **Más información → Ejecutar de todas formas**.
   - Necesita el Visual C++ Redistributable x64 **más reciente** ([descarga de Microsoft](https://learn.microsoft.com/cpp/windows/latest-supported-vc-redist)). Se compila con Visual Studio 2026; con una versión antigua puede no arrancar o cerrarse al empezar.
3. **Comparte la carpeta de datos con Bambu Studio oficial** (`%APPDATA%\BambuStudio`): perfiles, sesión e impresoras. Haz una copia de esa carpeta antes de la primera vez.
4. Si la aplicación ofrece actualizarse, **no aceptes**: instalaría la versión oficial, sin MCP. Las actualizaciones llegan por este repositorio.

## Activar el MCP y conectar Claude

1. En Bambu Studio: **Preferencias → Local MCP server** → activar. Copia la dirección (por defecto `http://127.0.0.1:27183/mcp`) y el token.
2. Deja **desactivados** los permisos de acciones de impresora y de lectura de la cuenta en la nube hasta haber probado todo lo demás.
3. **Claude Code:**
   ```
   claude mcp add --transport http bambu-studio http://127.0.0.1:27183/mcp --header "Authorization: Bearer TU_TOKEN"
   ```
4. **Claude Desktop** (necesita un puente, [mcp-remote](https://www.npmjs.com/package/mcp-remote), y Node.js instalado). En `claude_desktop_config.json`:
   ```json
   {
     "mcpServers": {
       "bambu-studio": {
         "command": "npx",
         "args": ["mcp-remote", "http://127.0.0.1:27183/mcp", "--header", "Authorization:${AUTH}"],
         "env": { "AUTH": "Bearer TU_TOKEN" }
       }
     }
   }
   ```

El servidor solo escucha en el propio ordenador (127.0.0.1) y exige el token. Trata el token como una contraseña.

## Avisos

- **Es una beta.** El parche solo encaja en la línea 2.8.4, que Bambu publica como beta.
- **Imprimir no está probado.** Una compilación propia sin firmar puede tener limitaciones con el plugin de red de Bambu en Windows (cámara remota, impresión por la nube). El modo LAN con Developer Mode es la alternativa segura. Prueba en este orden: iniciar sesión, imprimir por LAN, imprimir por la nube, cámara y, por último, imprimir desde el MCP.
- **El PR #12485 no está revisado por Bambu Lab** y solo lo había compilado su autor en macOS.

## Actualizar a una versión nueva de Bambu Studio

1. El workflow **Check upstream** abre un issue cuando hay versión nueva y dice si los parches se aplican.
2. Si se aplican: cambia `UPSTREAM_TAG` y lanza **Actions → Build Windows → Run workflow**. Tarda unas 1,5–2,5 h (la primera vez algo más, por las dependencias).
3. Si no se aplican: hay que regenerar los parches sobre la nueva versión.

GitHub desactiva los workflows programados de los repos públicos tras 60 días sin actividad; si pasa, reactívalo desde la pestaña Actions.

## Licencia

Bambu Studio y los parches de este repositorio se distribuyen bajo la [GNU AGPL v3](LICENSE). El código fuente correspondiente a cada versión publicada es la etiqueta indicada en `UPSTREAM_TAG` más los parches de `patches/` en el commit indicado en la release.
