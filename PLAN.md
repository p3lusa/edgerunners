# Plan de desarrollo — Tema Edgerunners (wallpaper de video)

**Ruta elegida: Ruta A** (renderizador de video como *plugin del shell*,
superconjunto de `omarchy.background`).

**Prioridades:**
1. **Máxima compatibilidad** con el sistema actual de Omarchy.
2. **Gestión 100% con comandos internos** (install / update / remove).

---

## 1. Objetivos y restricciones

**Objetivos**
- Tema Cyberpunk 2077: Edgerunners completo (colores, iconos, configs, fondos).
- Wallpaper de **video en bucle de alta calidad**, mudo.
- Instalar / actualizar / desinstalar con los comandos internos de Omarchy.
- Que Sigan funcionando: `omarchy theme set`, `omarchy theme bg next`, lock
  screen, transiciones, image-picker.

**Restricciones (verificadas en el sistema)**
- `omarchy theme install <url>` clona el repo en `~/.config/omarchy/themes/<nombre>/`
  y ejecuta `omarchy-theme-set`. Un tema clonado **NO** puede llevar `*.lua`,
  `alacritty.toml`, `foot.ini`, `ghostty.conf`, `kitty.conf` ni `vscode.json`
  (se regeneran de `colors.toml`). Todo lo demás **SÍ** se conserva
  (`colors.toml`, `icons.theme`, `backgrounds/`, `videos/`, `mako.ini`,
  `hyprlock.conf`, `btop.theme`, `shell.toml`, `*.css`, `warp.yaml`,
  `preview.png`, ...).
- `omarchy plugin add <url>` clona el repo en `~/.config/omarchy/plugins/<id>/`
  y exige `manifest.json` en la raíz (`schemaVersion=1, id, name, version, kinds,
  entryPoints`; el id **no** puede ser `omarchy.*`).
- El plugin de fondo stock `omarchy.background` pinta en `WlrLayer.Background`
  (namespace `omarchy-background`) y **poll**ea el symlink `current/background`.
- Quickshell 0.3.1 (Qt6) con `QtMultimedia` disponible → `MediaPlayer` +
  `VideoOutput`.
- HW decode por Vulkan disponible (AMD 780M): `vulkanh264dec`, `vulkanh265dec`, ...

## 2. Decisiones de arquitectura
- **Superconjunto (Ruta A):** el plugin `p3lu.video-background` reproduce el
  comportamiento completo del `omarchy.background` stock (imagen,
  symlink-polling, IPC `themeTransition`, transiciones) y **añade** el video.
  Así nada que hoy funciona deja de funcionar.
- **Dos repos:** tema (raíz) + plugin (anidado en `plugin/`). Ver README.
- **Modelo de capa — Diseño A (reemplazo):** el plugin usa el mismo
  `WlrLayer.Background` / `omarchy-background` y se convierte en el único dueño de
  la capa; `omarchy.background` stock se deshabilita. Determinista (un solo
  cliente).
  - *Alternativa (Diseño B, overlay):* el video como superficie **encima** de la
    imagen stock, sin deshabilitar nada. Más robusto ante `omarchy refresh
    shell`, pero depende del orden de apilado en la capa (hay que probarlo). Se
    considera si el Diseño A da problemas.
- **Fuente del video:** el plugin lee el **tema activo** y, si trae `videos/`, lo
  reproduce; si no, usa la imagen. El video es *aditivo*, no reemplaza la imagen.

## 3. Estructura de repos
Ver README: `edgerunners/` = tema; `edgerunners/plugin/` = plugin.

## 4. Fases

### Fase 0 — Preparación y backups
1. `cp -a ~/.config/omarchy ~/.config/omarchy.bak.$(date +%s)`
2. `cp -a ~/.config/hypr ~/.config/hypr.bak.$(date +%s)`
3. Verificar entorno:
   - `omarchy plugin list --json | jq '.[]|select(.id=="omarchy.background")'`
   - `gst-inspect-1.0 | egrep -i 'vulkan|h264'`  (HW decode)
   - `omarchy-shell shell ping`
4. `omarchy debug --no-sudo --print`  (línea base)

**Verificar:** backups creados; shell responde; HW decode presente.

### Fase 1 — Identidad del tema
1. Ajustar `colors.toml` (paleta neón) — ya hay borrador en el repo.
2. `icons.theme` (ej. `Yaru-red`; alternativas `Yaru-magenta` / `Yaru-purple`).
3. Configs "kept" (se conservan al instalar; **no** se regeneran):
   `mako.ini` (notificaciones neón), `hyprlock.conf` (lock), `btop.theme`,
   `helix.toml`, `warp.yaml`, `waybar.css`, `wofi.css`, `swayosd.css`,
   `aether.override.css`, `walker.css`, `chromium.theme`, `keyboard.rgb`,
   `shell.toml`.
4. **No** incluir `*.lua` / configs de terminal / `vscode.json` (se regeneran).
5. `preview.png` (y `preview-unlock.png`) para el selector de temas.

**Verificar:** en una máquina de prueba, `omarchy theme set edgerunners` aplica
colores a shell/terminal/apps sin errores (`hyprctl configerrors` vacío).

### Fase 2 — Plugin de video (núcleo, Ruta A)
1. El plugin ya está en `plugin/` con la base = `omarchy.background` stock
   (funciona como imagen desde el commit 0).
2. Añadir en `Background.qml` (dentro del `PanelWindow`, por pantalla) la rama de
   video:
   - `import QtMultimedia`
   - Resolver `videoUrl` desde el tema activo:
     `readlink ~/.local/state/omarchy/current/theme` → glob `videos/*.mp4`.
   - `MediaPlayer { source: videoUrl; autoPlay: true; loop: true; muted: true }`
   - `VideoOutput { anchors.fill: parent; source: player; fillMode: PreserveAspectCrop }`
   - Mostrar `VideoOutput` solo si hay video; si no, la `Image` stock (fallback).
3. Reaccionar al cambio de tema: `FileSystemWatcher` sobre el symlink
   `current/theme` (o hook a la IPC de tema) → recargar el video.
4. Multi-monitor: un `MediaPlayer` por `Variants` (pantalla), o uno compartido.
5. Validar el plugin: `omarchy plugin validate plugin/`

**Verificar:** `omarchy plugin validate plugin/` → 0. Con un tema que tenga
`videos/`, el video aparece en todas las pantallas; sin `videos/`, se ve la
imagen.

### Fase 3 — Assets de video
1. Fuente de 2–4 loops (10–25 s), sin audio, que cierren el bucle de forma
   seamless (ciudad, personaje, calle). Uso personal; no redistribuir.
2. Encodiar para bucle: H.264/HEVC, 1080p (o res. del panel), 24–30 fps,
   ~2–6 Mbps, sin pista de audio, keyframe en el punto de corte
   (`ffmpeg` o `omarchy transcode`).
3. Generar un PNG (un frame) por video → `backgrounds/` (fallback + lock).
4. Colocar MP4 en `videos/`, PNG en `backgrounds/`. (Si crecen, activar Git LFS.)

**Verificar:** `ffprobe` de cada MP4 (sin audio, res/fps correctos); los PNG existen.

### Fase 4 — Integración tema → video
1. Instalar tema + plugin (ver README) y `omarchy theme set edgerunners`.
2. `omarchy plugin disable omarchy.background`.
3. Comprobar transición tema→tema (video↔imagen) sin pantallazos.
4. Lock screen: usa PNG (frame) vía `current/background` (sigue funcionando).

**Verificar:** `omarchy theme set edgerunners` → video; `omarchy theme set <otro>`
→ imagen; lock muestra PNG; `omarchy theme bg next` cicla imagen (en temas sin video).

### Fase 5 — Gestión con comandos internos (requisito clave)
Documentar y probar el ciclo completo:
- **Instalar:** `omarchy theme install <url>` + `omarchy plugin add <url> --enable`
  + `omarchy plugin disable omarchy.background`.
- **Actualizar:** `omarchy theme update` + `omarchy plugin update p3lu.video-background`.
- **Desinstalar:** `omarchy theme remove edgerunners` + `omarchy plugin remove
  p3lu.video-background --yes` + `omarchy plugin enable omarchy.background`.

**Verificar:** cada comando ejecutado de principio a fin sin errores; el estado
post-desinstalación == estado inicial (imagen stock, sin video).

### Fase 6 — Endurecimiento de compatibilidad
1. Riesgo `omarchy refresh shell` (resetea `shell.json` → re-activa stock,
   desactiva el plugin → **degrada a imagen**, sin conflicto). Documentar la
   re-aplicación en un paso.
2. (Opcional) Hook `post-update.d` que re-asegura: plugin activo + stock
   desactivado.
3. Fallback robusto: si el video falla (decode/codec), el plugin vuelve a imagen
   sin romperse (probar con un MP4 corrupto / codec raro).

**Verificar:** `omarchy refresh shell` → escritorio en imagen (no negro);
re-aplicar → video de vuelta. MP4 corrupto → imagen, sin crash del shell.

### Fase 7 — Publicar
1. Subir el tema a `github.com/<user>/edgerunners` y el plugin a
   `github.com/<user>/edgerunners-wallpaper`.
2. Etiqueta `v0.1.0`. Añadir `preview.png`.

**Verificar:** en una máquina limpia, instalar desde los URLs públicos funciona.

## 5. Checklist de verificación (consolidado)

Estado: **COMPLETADO** (2025-09). Plugin v0.3.0 (rama de video + ciclo).
Repos publicados (privados, por copyright de los clips):
`github.com/p3lusa/edgerunners` (tema, LFS) + `github.com/p3lusa/edgerunners-wallpaper` (plugin).

- [x] `omarchy plugin validate plugin/` → 0
- [x] `omarchy theme set edgerunners` aplica colores (sin `configerrors`)
- [x] `hyprctl configerrors` → sin errores
- [x] Video en pantalla, mudo (sin pista de audio), en bucle (CPU ~3%)
- [x] Tema sin `videos/` → imagen (fallback) — probado con `cpunk`
- [x] `omarchy theme bg next` cicla imagen (tema sin video)
- [x] Lock screen muestra PNG (por construcción: lee `current/background`)
- [x] Transiciones tema→tema (video↔imagen) sin pantallazos
- [x] Degradación post-`refresh shell` → imagen (no negro) — probado simulando el estado
- [x] MP4 corrupto → imagen, sin crash del shell (probado aislado)
- [x] Ciclo install/update/remove completo sin errores (rutas locales git)
- [x] Shell sin errores en journal + `ping` ok (equivale a `omarchy debug`, inexistente en esta versión)
- [x] Ciclo de videos: `theme bg next` avanza clip (1/9 → 9/9 → 1/9)
- [x] Videos reales (9 loops 1440p30 H.264 sin audio, ~4% CPU) + PNG emparejados
- [x] Fase 7: repos en GitHub (privados) + Git LFS + **install completo desde URL verificado**
  (`theme install <url>` con LFS + `plugin add <url> --enable` + video en vivo)

## 6. Garantías de compatibilidad
| Comportamiento | Estado con la Ruta A |
|---|---|
| `omarchy theme set` (imagen) | ✅ intacto (symlink + IPC, el plugin los consume) |
| `omarchy theme bg next` | ✅ intacto (mismo symlink) |
| Lock screen | ✅ intacto (`current/background`) |
| Transiciones / image-picker | ✅ intacto (IPC `themeTransition`) |
| `omarchy refresh shell` | ⚠️ degrada a imagen (re-aplicar plugin) |
| Video | ➕ aditivo (si el tema trae `videos/`) |

## 7. Puntos de decisión abiertos
1. **Diseño A (reemplazo)** vs **Diseño B (overlay)** — empezar con A; pasar a B
   si el refresh-hazard pesa.
2. ¿**Git LFS** para `videos/`? (recomendado si > ~50 MB totales)
3. ¿**Pausar** el video en batería / lock / idle? (recomendado: lock + idle)
4. Nombre final del plugin id (`p3lu.video-background` por defecto).
