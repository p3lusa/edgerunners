# Plan — Aceleración por hardware (GPU) de la decodificación de vídeo

**Estado: INVESTIGADO, pendiente de aprobación (2026-09-10). No se ha tocado
nada todavía.**

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

### 1.4 Por qué "no cargaba VAAPI por defecto"
El proceso `quickshell` **no tiene ninguna de esas variables en su entorno**
(ningún unit omarchy define `QT_FFMPEG_*`). Qt intenta autodetectar el backend
HW, pero en la práctica no estaba enganchando VAAPI (lo observó el usuario).
La solución robusta es **forzar la prioridad explícitamente** en el entorno de
la sesión, en vez de confiar en la autodetección.

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
1. **NVIDIA:** `nvidia-smi` existe y responde **y/o** `lspci` muestra `NVIDIA`
   y hay nodo `cuda` (`/dev/nvidia0` o `nvidia-smi -L`).
   → Backend de decode: `cuda`. Vars:
   `QT_FFMPEG_DECODING_HW_DEVICE_TYPES=cuda` (+ opcional `qsv` no aplica).
   Fallback en lista: `cuda,vdpau` (vdpau por si cuda falla y el driver lo trae).
2. **Intel (iGPU) / AMD (iGPU o dGPU) / otras:** hay `/dev/dri/renderD*` y
   `lspci`/`lsmod` confirma `i915` (Intel) o `amdgpu` (AMD).
   → Backend de decode: `vaapi`. Vars:
   `QT_FFMPEG_DECODING_HW_DEVICE_TYPES=vaapi` y
   `QT_FFMPEG_HW_ALLOW_PROFILE_MISMATCH=1` (amplía cobertura de perfiles H.264/HEVC).
   (VAAPI cubre Intel y AMD; es el camino único para los iGPU del mercado.)
3. **Nada de lo anterior** (o el nodo `renderD*` no existe / no es accesible):
   → **No forzar nada.** Dejar el backend por defecto (CPU) y dejar
   `hwaccel.env` vacío o con un comentario. **Nunca romper el wallpaper.**

> Nota: en sistemas con **doble GPU** (iGPU + dGPU) la decisión debe elegir el
> nodo de render correcto. Regla: preferir la dGPU para el vídeo si hay
> `renderD*` asociado a la dGPU (por `lspci`/`/sys`); si no es trivial, VAAPI
> sobre el nodo por defecto ya ahorra CPU respecto a software. Esto se refina
> en la fase de implementación si aparece un caso real.

### 2.3 Mecanismo de aplicación (a elegir en impl., por robustez)
Orden de preferencia (el que funcione primero y sea portable gana):
1. **Fichero de entorno de la sesión de uwsm/Omarchy** heredado por
   `quickshell` (el `env_session.conf` que ya carga el unit del WM, o el punto
   equivalente que use la instalación). Es el camino "de serie" de Omarchy.
2. **Drop-in de systemd** sobre el unit que lanza la sesión del WM
   (`wayland-wm@hyprland.desktop.service`) añadiendo
   `EnvironmentFile=~/.config/omarchy/plugins/p3lu.video-background/hwaccel.env`.
3. **`post-update` hook** (ya existe en el tema) que re-aplique el entorno y
   avise de que hay que `omarchy refresh shell` para que entre en vigor.

El script debe **detectar cuál de los tres mecanismos existe** en la instalación
y usarlo; si ninguno, imprimir las instrucciones manuales (`export QT_FFMPEG_…`)
y no fallar en silencio.

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
| 1 | Escribir `video-hwaccel.sh`: detección NVIDIA/Intel/AMD + nodo `renderD*`, generación de `hwaccel.env` + `hwaccel.log`. **Solo detección, sin aplicar todavía.** | Ejecutar → `hwaccel.log` muestra "AMD → vaapi" y el `env` tiene las 3 vars. |
| 2 | Elegir + implementar el mecanismo de inyección (uwsm env / drop-in / post-update) para que `quickshell` herede las vars. | Tras `omarchy refresh shell`, `/proc/<quickshell>/*/environ` contiene `QT_FFMPEG_DECODING_HW_DEVICE_TYPES=vaapi`. |
| 3 | Verificar que VAAPI **de hecho** decodifica el wallpaper (fds de `renderD*`, CPU baja, log Qt). | Comparar CPU/fds con y sin la variable. |
| 4 | Revisión de seguridad del nuevo script (mismo estándar que los 13 actuales: `set -euo pipefail`, quoting, sin `eval`, `rm` acotado). | `bash -n` + el checklist de seguridad. |
| 5 | Integración: `video-add.sh`/`video-theme.sh` (o un `post-update`) llaman a `video-hwaccel.sh` al añadir el primer clip, para autoconfigurar la aceleración la primera vez. | Añadir un clip → `hwaccel.env` generado. |
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

## 5. Open questions (para decidir antes de implementar)

1. **¿Auto-configurar o manual?**
   - (A, recomendado) `video-hwaccel.sh` se auto-ejecuta al añadir el primer
     clip y al `post-update`; el usuario no hace nada.
   - (B) Comando explícito `video-hwaccel.sh` que el usuario lanza cuando quiera.
2. **¿Forzar solo VAAPI (lista corta) o lista de prioridades larga?**
   - Recomendado: lista corta específica del vendor (`vaapi` / `cuda`) para que
     la decisión sea predecible; Qt ya hace fallback a CPU si el backend falla.
3. **¿Añadir `vainfo` como dep de verificación?** (opcional; hoy no está). Útil
   para el log de diagnóstico, pero `ffmpeg` ya sirve de sonda.
4. **Doble GPU:** ¿priorizar dGPU para el vídeo o dejar el nodo por defecto?**
   (Recomendado: nodo por defecto salvo que el caso real lo exija.)

---

## 6. Decisión solicitada

Aprobar el plan y elegir la opción de **§5.1 (A auto-configurar vs. B manual)**
y la de **§5.2 (lista corta vs. larga)**. Con eso arranco la **fase 1**
(detección, sin aplicar aún) y te aviso por aquí cuando el `hwaccel.env` y el
`hwaccel.log` estén listos para revisar.
