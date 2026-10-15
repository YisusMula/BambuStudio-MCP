# BambuStudio-MCP

Objetivo: que Claude controle Bambu Studio mediante MCP. Para ello compilamos Bambu Studio oficial con el servidor MCP del PR bambulab/BambuStudio#12485 más nuestras correcciones. Plataforma: **Windows**. Repo **público**. El usuario escribe en español.

- Antes de responder sobre opciones, alternativas o riesgos, lee `docs/investigacion.md`: toda la investigación ya está hecha y verificada; no la repitas.
- Este repo no contiene el código de Bambu Studio. `UPSTREAM_TAG` indica la versión oficial y `patches/*.patch` se aplican en orden.
- La compilación la hace `.github/workflows/build-windows.yml` (lanzamiento manual). `check-upstream.yml` abre un issue cuando sale una versión nueva.
- Para actualizar a una versión nueva o regenerar parches, sigue `docs/actualizar.md`.
- No edites `patches/0001-*` a mano: es el PR original. Nuestras correcciones van en `0002` o en parches nuevos numerados.
- Opciones de compilación obligatorias: `-DBBL_RELEASE_TO_PUBLIC=1 -DBBL_INTERNAL_TESTING=0`, en modo Release.
