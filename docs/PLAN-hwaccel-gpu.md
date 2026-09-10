# Plan — Aceleración por hardware (GPU) de la decodificación de vídeo

**Estado: FASES 1–3 Y 5 HECHAS y verificadas en vivo (2026-09-10/11). VAAPI
decodifica el wallpaper en la GPU. Fases 4 (seguridad) y 6 (docs README)
pendientes.**

**Objetivo:** que el wallpaper en vídeo se decodifique en la GPU (no en CPU),
detectando el tipo de gráfica del usuario (NVIDIA / AMD / Intel) y aplicando la
configuración de aceleración correspondiente, de forma que funcione en todo el
hardware del mercado sin romper el fallback a CPU.

---

## 1. Qué hemos averiguado (verificado en esta máquina)

### 1.1 El vídeo NO lo reproduce ningún `.sh`
El wallpaper lo reproduce el **`MediaPlayer` de `Background.qml`**
(`QtMultimedia`, backend **FFmpeg** de Qt). Los scripts del plugin
(`video-add`, `video-theme`, …) solo usan `ffmpeg` para trabajo puntual
(poster frame, remux sin audio, downscale del poster del TUI). Por lo tanto
**"acelerar por hardware" significa hacer que el reproductor Qt elija VAAPI,
no cambiar flags de ffmpeg en un script.**

> Consecuencia: el lugar donde se decide la decodificación es el entorno del
> proceso `quickshell` (el que carga `Background.qml`), vía variables de
> entorno del backend FFmpeg de Qt.

### 1.2 VAAPI ya funciona en esta máquina
- GPU: AMD **HawkPoint1** (iGPU APU, NPU XDNA), driver kernel `amdgpu` cargado.
- `/dev/dri/renderD128` existe y es `0666` (cualquier usuario puede abrirlo →
  el acceso NO es el obstáculo).
- `libva`, `libva-drm` presentes; `ffmpeg -hwaccels` lista `vaapi`.
- **Prueba real:** `ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128
  -i <clip> -frames:v 1 -f null -` → **`VAAPI_DECODE_OK`** (decodificó un frame
  en HW). El decode por software también pasa.
- `vainfo` **no está instalado** (no hay herramienta de diagnóstico de VAAPI;
  hay que añadirlo como dep de verificación o usar `ffmpeg` como sonda).

### 1.3 El backend FFmpeg de Qt 6.11 trae VAAPI y lee variables de entorno
`strings` sobre `/usr/lib/qt6/plugins/multimedia/libffmpegmediaplugin.so`
(confirma lo que el plugin instalado realmente entiende):
- Decodificadores HW soportados: `h264_vaapi`, `hevc_vaapi`, `mjpeg_vaapi`,
  `mpeg2_vaapi`, `vp8_vaapi`, `vp9_vaapi` (+ `VAAPITextureConverter`).
- Variables de entorno que **sí lee**:
  - `QT_FFMPEG_DECODING_HW_DEVICE_TYPES`  → lista de prioridades de backend de
    decode HW (p.ej. `vaapi,cuda,vdpau`).
  - `QT_FFMPEG_ENCODING_HW_DEVICE_TYPES`  → idem para encode (irrelevante: el
    wallpaper no codifica).
  - `QT_FFMPEG_HW_ALLOW_PROFILE_MISMATCH=1` → permite decodificar si el perfil
    no coincide (útil p. ej. H.264 baseline sobre VAAPI).
  - `QT_DISABLE_HW_TEXTURES_CONVERSION=1` → desactiva la conversión de
    texturas en GPU (fallback si hubiera artefactos).
- Nota del doc de Qt: con el backend **VAAPI la conversión de texturas por HW
  está desactivada por defecto** (se activa con `QT_XCB_GL_INTEGRATION=xcb_egl`,
  solo en X11). En Wayland esto no aplica → no es un bloqueante.

### 1.4 Por qué "no cargaba VAAPI por defecto" (root cause real, verificado)
**No era que la variable no estuviera.** El usuario ya había intentado forzarlo
vía `hl.env("QT_FFMPEG_DECODING_HW_DEVICE_TYPES","vaapi")` en
`~/.config/hypr/autostart.lua` (el comentario de ese fichero lo documenta). El
problema real era el **orden de sondeo por defecto de Qt**: al no fijar la
prioridad, el backend FFmpeg de Qt probaba `vdpau → vulkan → cuda` (todos
fallando en esta AMD integrada) antes de llegar a `vaapi`, llenando el journal
con "Invalid setup for format vdpau/vulkan/cuda" en cada arranque de clip.
Fijar `QT_FFMPEG_DECODING_HW_DEVICE_TYPES=vaapi` **soluciona** el sondeo y
VAAPI decodifica (verificado en vivo, §2.5).

> Consecuencia: la solución es **forzar la prioridad explícitamente** en el
> entorno de la sesión, de forma **portable** (no un `hl.env` a mano que
> `omarchy refresh hyprland` pisa y que no cubre NVIDIA).

### 1.5 Dónde inyectar el entorno
- `quickshell` (pid 256768) es hijo de `omarchy-launch-` (256766) y está bajo el
  unit de la sesión del WM `wayland-wm@hyprland.desktop.service`, que ya carga
  `EnvironmentFile=-%t/uwsm/env_session.conf` (mecanismo de **uwsm** para inyectar
  entorno en la sesión, heredado por todo lo que arranca dentro).
- **Punto de inyección candidato (a confirmar en la fase de impl.):** un fichero
  de entorno de la sesión que `quickshell` herede. No hay ningún unit dedicado
  a `omarchy-shell` al que ponerle drop-in; la variable tiene que llegar al
  entorno de la sesión del WM.

### 1.6 Qué NO hay en esta máquina (para el mapeo multi-vendor)
- Sin `nvidia-smi` / `cuda` (no hay NVIDIA aquí) → la rama NVIDIA no se puede
  probar en esta máquina de test; se documenta y se valida por inspección.
- Sin GStreamer **VA-API** (`vaapih264dec` no existe, no hay `gst-vaapi`) → el
  único backend multimedia de Qt disponible es **FFmpeg** (solo
  `libffmpegmediaplugin.so`). Bien: nos centraremos en FFmpeg, no en GStreamer.

---

## 2. Diseño

### 2.1 Un nuevo script: `bin/video-hwaccel.sh`
Detección + aplicación de la aceleración. Sin dependencias nuevas (usa `lspci`,
`lsmod`, `nvidia-smi`, `ffmpeg` y el propio nodo `/dev/dri/*`).

**Salida:** un fichero de entorno (p.ej.
`~/.config/omarchy/plugins/p3lu.video-background/hwaccel.env`) con las variables
`QT_FFMPEG_*` apropiadas para la gráfica detectada, **y** una línea de log
legible (`hwaccel.log`) con lo detectado y lo aplicado, para diagnóstico.

### 2.2 Detección de la gráfica (orden de decisión)
El script **enumera TODAS** las GPUs de vídeo (integradas y dedicadas) vía
`lspci -nnk` (PCI + vendor + driver in-use) y **mapea cada nodo de render
`/dev/dri/by-path/*-render` a su GPU física** por PCI. Luego decide:
1. **NVIDIA:** `nvidia-smi` responde **y/o** `lspci` muestra NVIDIA con driver.
   → `QT_FFMPEG_DECODING_HW_DEVICE_TYPES=cuda` (NVDEC vive en la dGPU).
2. **Intel (iGPU) / AMD (iGPU o dGPU) / otras:** hay `/dev/dri/renderD*` y
   `lspci`/`lsmod` confirma `i915`/`amdgpu`. → `vaapi` +
   `QT_FFMPEG_HW_ALLOW_PROFILE_MISMATCH=1`.
   **Preferencia de nodo en híbrido:** si hay varios `renderD*`, elige el de la
   dGPU no-Intel (así un portátil iGPU+dGPU apunta el decode a la dGPU).
   (VAAPI cubre Intel y AMD; es el camino único para los iGPU del mercado.)
3. **Nada de lo anterior** (o sin `renderD*`): **no forzar nada** → CPU.
   `hwaccel.env` con solo un comentario. **Nunca romper el wallpaper.**

> **Dedicada vs integrada:** se DISTINGUIEN en el log (cada GPU con su PCI,
> vendor y nodo). En híbrido se prefiere el nodo de la dGPU. Limitación real:
> el backend FFmpeg de Qt **no permite fijar un nodo concreto** (no hay env
> var), así que en la práctica VAAPI usa el nodo por defecto de la sesión; la
> enumeración + el `hwaccel.log` muestran qué nodo corresponde a qué chip, y la
> preferencia de nodo queda documentada para el caso en que Qt lo permita.

### 2.3 Mecanismo de aplicación (ELEGIDO: drop-in de systemd user)
Se implementó el **drop-in de systemd-user sobre la plantilla del WM**:
- `video-hwaccel.sh --apply` escribe
  `~/.config/systemd/user/wayland-wm@.service.d/10-video-hwaccel.conf` con
  `[Service] EnvironmentFile=-<plugin>/hwaccel.env` y lanza
  `systemctl --user daemon-reload`.
- Es **portable** (funciona en cualquier Omarchy/Hyprland: el unit del WM es
  `wayland-wm@.service`), **idempotente** (re-escrivo el fichero y el
  `EnvironmentFile=-` con `-` no falla si no existe) y **no pelea** con el
  `env_session.conf` de uwsm (que sigue cargando primero).
- **No se tocó** `hl.env()` en `autostart.lua`: ese `hl.env` del usuario sigue
  funcionando y el drop-in lo reemplaza de forma portable; al reiniciar sesión
  ambos apuntan al mismo valor (`vaapi`), así que no hay conflicto.
- La variable entra en vigor en el **siguiente arranque de la sesión**
  (logout/login o `systemctl --user restart wayland-wm@.service`); la sesión
  actual ya tenía su entorno y no se toca.

> Por qué no los otros dos: el `env_session.conf` de uwsm lo **sobrescribe**
> `omarchy refresh`/`omarchy update`, y `hl.env` en `autostart.lua` lo pisa
> `omarchy refresh hyprland`. El drop-in de systemd es el único que sobrevive a
> ambas operaciones y es el que el plugin puede gestionar de forma idempotente.

### 2.4 Fallback y "no romper"
- Si la detección falla → entorno vacío → Qt usa CPU (comportamiento actual,
  seguro).
- Si VAAPI se fuerza pero la GPU no lo soporta bien → Qt **recaerá a software
  automáticamente** (el backend FFmpeg de Qt hace fallback); además la tarjeta
  de fallback del plugin (imagen estática) sigue cubriendo cualquier error.
- El script es **idempotente** y no toca `Background.qml` (coherente con la
  restricción del proyecto).

### 2.5 Punto de verificación empírico (cómo saber que funciona)
Tras aplicar y reiniciar el shell, comprobar en vivo:
- **GPU:** el proceso de decodificación abre `/dev/dri/renderD*` **además** de
  lo que ya abre para compositar (3 fds hoy) → más fds = VAAPI activo.
  (`/proc/<quickshell>/fd | grep -c renderD`)
- **Carga de CPU:** `quickshell` en `top`/`ps` con CPU sensiblemente menor que
  el decode por software.
- **Log de Qt:** `QT_FFMPEG_DEBUG=1` o la categoría
  `qt.multimedia.ffmpeg.hwaccelvaapi` → "Creating VAAPI HW accelerator".
- **Sonda VAAPI:** `vainfo` (añadido como dep opcional) lista el perfil
  H.264/HEVC del driver.
- El `video-hwaccel.sh` escribirá `hwaccel.log` con vendor, backend elegido,
  nodo `renderD*` y el estado del chequeo, para que el usuario lo vea de un
  vistazo (y para debug en otros hardware).

---

## 3. Fases de implementación (cada una verificable)

| # | Tarea | Verificación |
|---|-------|--------------|
| 0 | (hecha) Investigación + este plan | — |
| 1 | **(HECHA)** `video-hwaccel.sh`: enumera todas las GPUs + mapeo de nodos, detección NVIDIA/Intel/AMD, preferencia de dGPU en híbrido, `hwaccel.env` + `hwaccel.log`. **Solo detección, sin aplicar.** | ✅ Ejecutado: "AMD → vaapi" (esta máquina); stubs: NVIDIA→cuda, híbrido→vaapi eligiendo nodo de dGPU, sin-GPU→cpu. `bash -n` OK. |
| 2 | **(HECHA)** Mecanismo de inyección: drop-in de systemd-user sobre `wayland-wm@.service` vía `video-hwaccel.sh --apply`. | ✅ Drop-in escrito + `daemon-reload`; el unit del WM incorpora el `EnvironmentFile` (verificado con `systemctl --user cat`). Apunta a la copia instalada del plugin. |
| 3 | **(HECHA)** Verificar que VAAPI **de hecho** decodifica el wallpaper en vivo. | ✅ Con el wallpaper activo, `quickshell` pasa de 3 a **6 fds** de `renderD128` (composición + VAAPI), **CPU 0.1%**, journal muestra `h264 High 1920x1080` reproduciéndose. Root cause del fallo original documentado (§1.4). |
| 4 | Revisión de seguridad del nuevo script (mismo estándar que los 13 actuales: `set -euo pipefail`, quoting, sin `eval`, `rm` acotado). | `bash -n` + el checklist de seguridad. |
| 5 | **(HECHA)** Integración: `video-add.sh` llama a `video-hwaccel.sh --apply` al añadir un clip (si el plugin está activo), y `hooks/post-update` lo re-aplica tras `omarchy update`. | ✅ Ambos hooks wired; `bash -n` OK. Auto-config idempotente. |
| 6 | Docs (README: sección "GPU acceleration" + tabla de vendors) + commit + push. | — |

---

## 4. Riesgos

- **Mecanismo de inyección (fase 2)** es lo más incierto: depende de cómo cada
  instalación de Omarchy/uwsm monta el entorno de la sesión. Mitigación: el
  script detecta el mecanismo disponible y, si ninguno, da instrucciones
  manuales. Se cierra en la fase 2 con una verificación en vivo.
- **Reinicio del shell para que entre en vigor:** las vars de entorno se leen al
  arrancar `quickshell`. Hacer `omarchy refresh shell` (o logout/login) tras
  aplicar. El `post-update` hook ya existe para re-aplicar tras `omarchy update`.
- **Doble GPU:** elegir el nodo de render correcto (ver §2.2).
- **NVIDIA no testeable aquí:** sin `nvidia-smi` en la máquina de test. La rama
  se escribe y revisa por inspección; se valida cuando aparezca hardware NVIDIA.
- **VAAPI inestable en algún driver:** mitigado por el fallback automático de Qt
  + la tarjeta de fallback del plugin.

---

## 5. Open questions (estado)

1. ~~**¿Auto-configurar o manual?**~~ → **RESUELTO (2026-09-10):** (A)
   auto-config — `video-hwaccel.sh` se ejecuta al añadir el primer clip y en el
   `post-update` hook.
2. ~~**¿Lista corta o larga?**~~ → **RESUELTO (2026-09-10):** lista corta
   específica del vendor (`vaapi` / `cuda`); Qt hace fallback a CPU si falla.
3. **¿Añadir `vainfo` como dep de verificación?** (opcional; hoy no está). Útil
   para el log de diagnóstico, pero `ffmpeg` ya sirve de sonda. **Abierta** —
   se decide en la fase 3 si hace falta más visibilidad.
4. ~~**¿Doble GPU: priorizar dGPU o nodo por defecto?**~~ → **RESUELTO
   (2026-09-10):** el script enumera todas las GPUs y elige el nodo de la dGPU
   no-Intel cuando hay varios; la limitación real (Qt no fija nodo) queda
   documentada en §2.2.

---

## 6. Decisión solicitada

Aprobar el plan y elegir la opción de **§5.1 (A auto-configurar vs. B manual)**
y la de **§5.2 (lista corta vs. larga)**. Con eso arranco la **fase 1**
(detección, sin aplicar aún) y te aviso por aquí cuando el `hwaccel.env` y el
`hwaccel.log` estén listos para revisar.
