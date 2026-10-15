# Actualizar a una versión nueva de Bambu Studio

Cuándo usarlo: hay un issue "Nueva versión de Bambu Studio", falla "Apply patches" en el CI o el usuario pide pasar a otra versión.

1. Comprueba la etiqueta nueva y si es beta o estable:
   - `git ls-remote --tags --refs https://github.com/bambulab/BambuStudio 'v*' | awk -F/ '{print $3}' | sort -V | tail -5`
   - Página de releases de bambulab/BambuStudio.
2. Clona esa etiqueta: `git clone --depth 1 --branch <tag> https://github.com/bambulab/BambuStudio`. Prueba los parches en orden con `git apply --check` y después `git apply`.
3. **Si todos se aplican:** cambia `UPSTREAM_TAG`, haz commit y push, y lanza el workflow **Build Windows**.
4. **Si alguno falla:**
   - Aplica con `git apply -3`, o a mano, sobre la etiqueta nueva.
   - Resuelve los conflictos conservando la intención del parche.
   - Regenera el parche con `git diff` respecto a la etiqueta.
   - Mira también si el PR #12485 tiene commits nuevos (`git fetch https://github.com/bambulab/BambuStudio pull/12485/head`).
   - Anota en el README qué se adaptó.
5. Usa siempre un número de versión publicado. El plugin de red se descarga por la serie `AA.BB.CC`.
6. Si Bambu fusiona el PR #12485, quita el `0001` y conserva solo las correcciones que sigan siendo necesarias.
7. Anota en `docs/investigacion.md` cualquier hallazgo nuevo.
