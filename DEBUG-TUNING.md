# Debug y Tuning — Tema Video Wallpaper

## 1. Observabilidad (dónde mirar)
- **Diagnóstico general:** `omarchy debug --no-sudo --print`
  (escribe `/tmp/omarchy-debug.log`).
- **Shell (Quickshell):** proceso largo (gestionado por uwsm). Logs vía
  `journalctl` (buscar `omarchy-shell` / `quickshell`). Si no hay journal,
  revisar el stderr del proceso o `~/.local/state/omarchy/`.
  - Descubrir la fuente: `systemctl --user list-units | grep -iE 'shell|omarchy'`
  - `journalctl --user -n 50 | grep -iE 'shell|quickshell|background'`
- **IPC / estado del plugin:**
  - `omarchy-shell shell ping`
  - `omarchy-shell shell listPlugins`
  - `omarchy plugin list --json`
- **Estado del fondo (symlinks):**
  - `readlink -f ~/.local/state/omarchy/current/theme`
  - `readlink -f ~/.local/state/omarchy/current/background`
  - `omarchy theme current`
- **Hyprland:** `hyprctl reload`, `hyprctl configerrors`, `hyprctl monitors`.
- **Decodificación de video (GStreamer):**
  - `gst-inspect-1.0 | egrep -i 'vulkan|h264|h265'`  (¿HW decode disponible?)
  - `GST_DEBUG=3 gst-launch-1.0 filesrc location=<mp4> ! decodebin ! autovideosink`

## 2. Fallo clásico: conflicto de capa (pantalla negra)
- **Síntoma:** escritorio negro tras activar el plugin.
- **Causa:** **dos** clientes en `WlrLayer.Background` (stock `omarchy.background`
  + `p3lu.video-background`).
- **Diagnóstico:**
  `omarchy plugin list --json | jq '.[]|select(.id|test("background"))'`
  → si ambos `enabled:true`, ese es el problema.
- **Fix:** `omarchy plugin disable omarchy.background` y `omarchy restart shell`.
- **Prevenir:** el hook `post-update.d` (Fase 6) re-asegura el estado.

## 3. Decodificación: HW (Vulkan) vs SW
- **Síntoma:** video "lento", CPU alta, calentamiento.
- **Diagnóstico:**
  - `top` / `htop` con el video activo (¿un núcleo al 100%?).
  - `GST_DEBUG` para ver si usa `vulkanh264dec` (HW) o `avdec_h264` (SW).
- **Fix:**
  - Asegurar los plugins Vulkan de GStreamer instalados.
  - Bajar resolución/bitrate del MP4 (ver tuning).
  - Usar **H.264** (mejor soporte HW) en vez de HEVC si el decode de HEVC no va.

## 4. El video no aparece / no cambia de tema
- ¿El tema activo trae `videos/`?
  `ls "$(readlink -f ~/.local/state/omarchy/current/theme)/videos/"`
- ¿El symlink `current/theme` apunta al tema correcto?
- ¿El `FileSystemWatcher` reaccionó? (revisar logs del shell al hacer `theme set`).
- Fallback: `omarchy restart shell` fuerza la recarga.

## 5. Tabla de fallos comunes
| Síntoma | Causa probable | Fix |
|---|---|---|
| Pantalla negra | 2 clientes en capa Background | deshabilitar stock; `restart shell` |
| Video con sonido | falta `muted: true` | `MediaPlayer.muted: true` |
| Video saltarín | decode SW / bitrate alto | HW decode (Vulkan) + bajar bitrate |
| No cambia al cambiar tema | watcher/IPC sin reaccionar | forzar `restart shell`; revisar logs |
| Se ve imagen, no video | tema sin `videos/` o MP4 inválido | añadir MP4; validar con `ffprobe` |
| Crash del shell con MP4 raro | codec no soportado | re-encodar a H.264; fallback a imagen |
| `bg next` no cicla en tema con video | el tema usa video, no imágenes | esperado; ciclar video = Fase 4 (opcional) |

## 6. Tuning

### 6.1 Encodado del video (loop seamless + calidad)
- **Codec:** H.264 (máx. compatibilidad HW) o HEVC (menos tamaño, HW menos universal).
- **Resolución:** 1080p por defecto; 1440p si el panel lo soporta y sobra GPU.
- **FPS:** 24–30 (suficiente para fondo; 60 si el clip es "action").
- **Bitrate:** ~2–6 Mbps (1080p). No más de lo necesario.
- **Sin pista de audio.**
- **Loop seamless:** cortar en un punto donde el bucle no se nota; keyframe en el
  corte. Probar el bucle con `mpv --loop-file <mp4>`.
- Ejemplo:
  ```bash
  ffmpeg -i in.mp4 -t 20 -an -c:v libx264 -crf 20 -preset slow \
    -pix_fmt yuv420p -r 30 -movflags +faststart out.mp4
  ```

### 6.2 HW decode (Vulkan)
- Verificar `vulkanh264dec` / `vulkanh265dec` presentes.
- Confirmar que el pipeline de GStreamer los elige (`GST_DEBUG`).

### 6.3 Multi-monitor
- Un `MediaPlayer` por pantalla (simple) vs uno compartido (menos CPU).
- Ajustar `fillMode` según la relación de aspecto (`PreserveAspectCrop`).

### 6.4 Batería / idle / lock
- **Pausar** el video: al **bloquear** (recomendado siempre), en **batería**
  (opcional), tras N s de **idle** (opcional). Toggles en el plugin.
- Fallback "reducir movimiento": congelar en un frame (usar el PNG de
  `backgrounds/`).

### 6.5 Medir rendimiento
- Con el video activo vs imagen: CPU (`top`), GPU (`radeontop` / `nvtop`),
  batería (drain por hora).
- Objetivo: CPU de reposo con video **< ~15–20%** de un núcleo (con HW decode).

## 7. Regresión (probar en cada cambio)
1. `omarchy theme set video-wallpaper` → video, colores OK, sin `configerrors`.
2. `omarchy theme set <otro>` → imagen.
3. `omarchy theme bg next` → cicla imagen; en tema con videos **también
   avanza el clip** (1/N → N/N → 1/N).
4. Lock (`omarchy system lock`) → PNG, sin video.
5. `omarchy refresh shell` → degradación a imagen (no negro) → re-aplicar → video.
6. MP4 corrupto → imagen, shell vivo.
7. `omarchy debug --no-sudo --print` limpio (si existe en la versión; aquí
   se usa `journalctl --user -t omarchy-shell | grep -E 'ERROR|FATAL'` + ping).

## 8. Publicación (Git LFS, GitHub)
- **Regla:** el repo público **no** contiene clips/frames de terceros
  (copyright CDPR/Aniplex/Trigger). Solo los loops synth originales.
  Los clips personales viven fuera del git (no-versionados en el clone o en
  `~/Videos/clips/`).
- **Pitfall — LFS GC tras force-push:** si reescribiste el historial
  (`git push --force`), GitHub puede hacer GC de los objetos LFS y dejar
  `404` en `media.githubusercontent.com` (clones nuevos → pointers rotos).
  **Fix:** desde el repo local, `git lfs push origin <branch> --all`
  (re-sube todos los objetos referenciados) y volver a probar con un clone
  anónimo (sin credenciales):
  ```bash
  env -i HOME=/tmp/x PATH=$PATH git clone https://github.com/<user>/video-wallpaper.git /tmp/x
  cd /tmp/x && git lfs install --local && git lfs pull && file videos/*.mp4
  ```
- **Verificación pública:** `gh repo view <repo> --json visibility -q .visibility`.
- **Clips personales en el clone instalado:** copiar a
  `~/.config/omarchy/themes/video-wallpaper/videos|backgrounds/` + nombres en
  `.git/info/exclude` → sobreviven a `theme update` (git pull) y `theme set`
  los incluye en el re-stage.
