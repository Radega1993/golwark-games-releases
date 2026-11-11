# Golwark Launcher — MVP 0.1.0

Launcher de escritorio para el universo **Golwark**. El primer juego disponible es **WarpazTCG**. Arquitectura ampliable a futuros juegos por catálogo JSON, sin cambiar el código de Electron.

> **Estado de entrega:** código fuente MVP, probado parcialmente con Node.js (núcleo). **No** se ha compilado el instalador final de Electron porque este entorno no permite descargar dependencias de npm. El ZIP no contiene un Setup.exe ni una AppImage. Consulta las pruebas antes de usarlo en producción.

## Qué funciona

- Interfaz de biblioteca, estado por juego, botones de instalación/juego, panel de configuración y estética Golwark.
- Windows x64 y Linux x64 (plantillas electron-builder: Windows NSIS `.exe` y Linux `.AppImage`).
- Catálogo local incluido y remoto HTTPS configurable (varios juegos).
- **Importar build Unity ya extraída** (copiándola a la carpeta gestionada por Golwark).
- Descargar ZIP, verificar SHA-256, extraer con protección básica frente a traversal/symlinks, instalar/actualizar con staging y rollback.
- Ejecutar juego, evitar actualización/desinstalación cuando fue abierto por el mismo launcher.
- Reparar instalación (reinstalación completa), desinstalar sin tocar `Application.persistentDataPath`.
- Comprobar y aplicar actualizaciones del propio launcher con `electron-updater`, **una vez publicado** un canal generic compatible.
- CLI para generar catálogo real con hashes a partir de ZIPs de Windows y Linux.
- Tests automáticos del catálogo, rutas, importación, actualización y ejecución.

## Arranque en Kubuntu

Necesitas Node.js 22+ y acceso a npm. Desde la carpeta de este proyecto:

```bash
npm install
npm test
npm run dev
```

Primero funciona **sin configuración de servidor remoto**: se muestra WarpazTCG con opción **Importar build**. En la importación selecciona la carpeta extraída que contiene `WarpazTCG.x86_64` y `WarpazTCG_Data/` (Linux) o `WarpazTCG.exe` y `WarpazTCG_Data/` (Windows). Se copiará a la carpeta de juegos del launcher; no se modificará la carpeta origen.

Para crear instaladores:

```bash
npm run dist:linux  # Ejecutar en Linux; obtiene .AppImage
npm run dist:win    # Recomendado ejecutar en Windows; obtiene Setup.exe
```

Es recomendable compilar Windows en Windows y Linux en Linux. Incluyo un workflow GitHub Actions de construcción multiplataforma. La primera vez que hagas `npm install`, versiona `package-lock.json` y cambia el workflow de `npm install` a `npm ci` para builds reproducibles.

## Configurar actualización automática de juegos

Tu carpeta pública de Google Drive **no da por sí sola URLs directas verificables de cada ZIP**, por eso el catálogo incluido deja `url` y `sha256` vacíos: no se promete una actualización que todavía no está configurada. Desde la interfaz se puede abrir la carpeta de Drive o importar una build descargada manualmente.

Cuando puedas alojar ZIPs en HTTPS (por ejemplo Cloudflare R2 con dominio propio), genera el catálogo real:

```bash
node scripts/generate-game-catalog.js \
  --version 1.2.4 \
  --base-url https://downloads.TU-DOMINIO.com/games \
  --windows /ruta/WarpazTCG-v1.2.4-windows-x64.zip \
  --linux /ruta/WarpazTCG-v1.2.4-linux-x64.zip \
  --output publish
```

Resultado:

```text
publish/
├── catalog.json
└── warpaztcg/v1.2.4/
    ├── WarpazTCG-v1.2.4-windows-x64.zip
    └── WarpazTCG-v1.2.4-linux-x64.zip
```

**Nota sobre URL:** el ejemplo genera direcciones `https://downloads.TU-DOMINIO.com/games/warpaztcg/v1.2.4/...`. Coloca `publish/` bajo la ruta HTTP `/games/` o ajusta la URL para coincidir con la estructura de tu alojamiento.

Sube los ZIPs antes que `catalog.json`, para no anunciar un paquete inexistente. No cambies el contenido de una versión ya publicada: publica una versión nueva. Comprueba el SHA-256 y la disponibilidad HTTPS desde otro ordenador.

En **Configuración → URL DEL CATÁLOGO**, introduce el enlace público al `catalog.json`, por ejemplo:

`https://downloads.TU-DOMINIO.com/games/catalog.json`

A partir de ahí, Golwark compara la versión remota con la instalada y habilita **Instalar / Actualizar / Reparar** automáticamente. Si el servidor no responde, sigue mostrando el catálogo local con aviso, pero nunca inventa una descarga válida.

### Añadir un segundo juego

Añade un nuevo objeto a `games[]` siguiendo `catalog/remote-catalog.example.json`, con ID propio (`golwark-rpg`), nombres, URL ZIP por plataforma, SHA-256 y el ejecutable principal. Publica el catálogo; aparecerá en la biblioteca cuando el launcher lo consulte.

## Actualizar el launcher (no los juegos)

El instalador del launcher se empaqueta mediante electron-builder y utiliza electron-updater. El `package.json` define el empaquetado para **Windows NSIS** y **Linux AppImage**. Tras compilar, `dist/` tendrá también manifiestos `latest.yml` y `latest-linux.yml` junto con los paquetes de actualización. Debes subirlos al mismo canal HTTPS, conservar el historial de versiones y probar en una instalación real.

En la UI, configura **CANAL DE ACTUALIZACIONES DEL LAUNCHER** con la URL del directorio HTTPS que contiene esos metadatos. Cuando exista una nueva versión: **Buscar actualización → Descargar → Instalar y reiniciar**.

No actives estas actualizaciones sin completar pruebas de empaquetado, firma de código Windows y recuperación tras fallo. En modo desarrollo, la UI lo indica y no intenta autoactualizar Electron.

## Seguridad y límites

- Descargas de juegos y catálogo: HTTPS; `http://localhost` solamente si ejecutas desarrollo con `GOLWARK_DEV_MODE=1`.
- Archivos ZIP: SHA-256 obligatorio, rutas relativas validadas, límites de tamaño, extracción incremental y rechazo de enlaces simbólicos. **SHA-256 solo garantiza coincidencia con el catálogo; se recomienda firmar los manifiestos y fijar el dominio de descargas antes de producción.**
- Renderer Electron: `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`, CSP restrictiva, IPC limitado y navegación externa controlada.
- Las builds de Unity y los datos persistentes viven fuera del directorio de instalación del launcher.
- El backup de instalación permite rollback ante excepciones durante el cambio de versión. En futuras iteraciones: recuperación de transacciones tras corte eléctrico, descargas reanudables, firma de catálogos y detección de procesos externos incluso después de reiniciar el launcher.
- **No** integra SSO Golwark ni gestiona tokens de Google: WarpazTCG usa su autenticación habitual.
- **No** hay parche delta: cada actualización del juego descarga el ZIP completo.
- No se ha probado un instalador firmado en Windows ni una AppImage en Linux con usuarios externos.

## Desarrollo

| Ruta | Propósito |
| --- | --- |
| `src/main/index.js` | Electron: ventanas, IPC, settings y actualización del launcher |
| `src/main/game-manager.js` | Gestión de juegos: importación, instalación, actualización, rollback, ejecución |
| `src/main/catalog.js` | Catálogo JSON, validaciones, versiones y HTTPS |
| `src/main/downloader.js` | Descarga con streaming y SHA-256 |
| `src/main/zip.js` | Extracción de ZIP segura (yauzl) |
| `src/renderer/` | UI del launcher |
| `catalog/bundled.json` | Catálogo inicial sin descargas automáticas ficticias |
| `scripts/generate-game-catalog.js` | Publicación de manifiestos y hashes |
| `.github/workflows/build-launcher.yml` | CI Windows/Linux |
| `tests/` | Pruebas Node.js |

**Fecha de trabajo:** 11 de noviembre de 2026. El catálogo inicial está basado en `ProjectSettings/ProjectSettings.asset` de `WarpazTCG` (bundleVersion `1.2.4`) y en la estructura real de builds `WarpazPlayerBuild.cs`, consultadas en GitHub.

## Distribución gratuita con GitHub Releases público

Esta variante admite `--github-release-repo OWNER/REPO` en el generador de catálogo, y permite colocar en **Configuración → Canal de actualizaciones del launcher** la URL `https://github.com/OWNER/REPO` para usar el proveedor GitHub de `electron-updater` en un repositorio público. No necesitas exponer el código fuente. Consulta [docs/DISTRIBUCION_GRATIS_GITHUB.md](docs/DISTRIBUCION_GRATIS_GITHUB.md).
# golwark-games-releases
