# Video Wallpaper — Tema para Omarchy

Tema para [Omarchy](https://omarchy.org/) cuyo objetivo es **wallpaper de
video**: reproduce en bucle y mudo **los clips que tú pongas** en `videos/`
como fondo de escritorio, con fallback automático a imagen. La skin incluida
por defecto es neón (cian / magenta / amarillo sobre negro).

> Preview: ![preview](preview.png)
>
> **Estado:** v0.4.0 — tema + plugin de video (v0.4.0) verificados en máquina
> real. Incluye 3 clips de ejemplo (H.264, sin audio, vía Git LFS; material
> libre de Wikimedia Commons, ver [`ATTRIBUTION.md`](ATTRIBUTION.md)) —
> **añade tus propios clips** a `videos/` (ver [Vídeos](#vídeos-añade-los-tuyos)),
> o genera un tema completo a partir de un clip con el helper de Aether
> (ver [Tema desde un clip](#tema-desde-un-clip-aether)).

## Qué incluye
- **Wallpaper de video**: un clip del tema activo en bucle y mudo como fondo
  de escritorio, con **fallback automático a imagen** (nunca pantalla negra).
- **Ciclo de clips**: `omarchy theme bg next` avanza al siguiente clip —
  imagen y video avanzan juntos.
- **Skin neón** (cian / magenta / amarillo sobre negro) aplicada a shell,
  barra, notificaciones, OSD, terminal y apps. Cámbiala a tu gusto en
  `colors.toml`.
- Gestión 100% con los **comandos internos de Omarchy** (instalar, actualizar,
  desinstalar).

## Arquitectura (dos repos)
El sistema de Omarchy separa **temas** y **plugins**:

| Artefacto | Repo | Se instala con | Qué aporta |
|---|---|---|---|
| **Tema** `video-wallpaper` | `github.com/p3lusa/video-wallpaper` | `omarchy theme install` | colores, iconos, configs, `backgrounds/`, `videos/` |
| **Plugin** `p3lu.video-background` | `github.com/p3lusa/video-background` | `omarchy plugin add` | el renderizador de video (capa de fondo) |

> **¿Por qué dos repos?** `omarchy theme install` clona el repo en la raíz de
> `themes/<nombre>/` (ficheros de tema), y `omarchy plugin add` clona el repo
> en la raíz de `plugins/<id>/` esperando un `manifest.json` en la raíz. No
> pueden compartir la misma raíz → son dos repos.

## Requisitos
- Omarchy (Hyprland + Quickshell)
- `QtMultimedia` para Quickshell (Qt6) — presente en la distro
- (Recomendado) decodificación por hardware vía Vulkan (AMD / NVIDIA / Intel)

## Instalación (comandos internos de Omarchy)
```bash
# 1) Tema
omarchy theme install https://github.com/p3lusa/video-wallpaper.git

# 2) Plugin de video
omarchy plugin add https://github.com/p3lusa/video-background.git --enable

# 3) Ceder la capa de fondo al renderizador de video (una vez)
omarchy plugin disable omarchy.background

# 4) Aplicar el tema
omarchy theme set video-wallpaper
```

## Actualización / desinstalación
```bash
# Actualizar (el clone se actualiza; re-aplicar el tema re-stagea los assets)
omarchy theme update
omarchy theme set video-wallpaper
omarchy plugin update p3lu.video-background

# Desinstalar (restaura el comportamiento de imagen stock)
omarchy theme remove video-wallpaper
omarchy plugin remove p3lu.video-background --yes
omarchy plugin enable omarchy.background
```

> **Nota:** `omarchy theme update` hace `git pull` en el clone, pero el tema
> activo es una copia staged en `~/.local/state/omarchy/current/theme/` — por
> eso la actualización termina con `omarchy theme set video-wallpaper`
> (re-stage + transición, sin reiniciar shell).

## Vídeos: añade los tuyos
Los clips viven en **`videos/`** (raíz del tema), un `*.mp4` por fondo. Cada
clip tiene su PNG emparejado en `backgrounds/` **con el mismo nombre base**
(ese PNG es el fallback, el lock screen y lo que ve `bg next` entre videos).

Este repo distribuye 3 clips de ejemplo (material libre, ver
[`ATTRIBUTION.md`](ATTRIBUTION.md)). Para añadir los tuyos:

```bash
# 1) Re-encódelo con esta receta (1440p30, sin audio)
ffmpeg -i origen.mp4 \
  -vf "scale=2560:1440:flags=lanczos,fps=30" \
  -c:v libx264 -preset medium -crf 23 -profile:v high \
  -an -movflags +faststart -y /tmp/NOMBRE.mp4

# 2) Y su PNG (frame al 40%, evita fades de apertura)
dur=$(ffprobe -v error -show_entries format=duration -of csv=p=0 /tmp/NOMBRE.mp4)
ffmpeg -ss "$(awk "BEGIN{printf \"%.3f\", $dur*0.4}")" -i /tmp/NOMBRE.mp4 \
  -frames:v 1 -q:v 2 /tmp/NOMBRE.png

# 3) Mételos en el clone instalado (no-versionados: git pull no los toca)
cp /tmp/NOMBRE.mp4 ~/.config/omarchy/themes/video-wallpaper/videos/
cp /tmp/NOMBRE.png ~/.config/omarchy/themes/video-wallpaper/backgrounds/
omarchy theme set video-wallpaper   # re-stagea el clone completo (incluidos los tuyos)
```

> Si no quieres que aparezcan en `git status`, añade sus nombres a
> `~/.config/omarchy/themes/video-wallpaper/.git/info/exclude` (solo afecta
> a ese clone, no al repo).

**Spec de los clips** (la recomendada; los loops incluidos son 1080p30):
- `H.264` (decodificación por hardware; HEVC no probado en este stack)
- `2560x1440` (16:9 1440p; el panel de referencia es 2560x1600 y el video se
  recorta con `KeepAspectRatioByExpanding`), `30 fps`
- **Sin pista de audio** — *obligatorio*: el backend FFmpeg de `QtMultimedia`
  (Qt 6.11) no expone `muted`/`volume`, así que un clip con audio sonaría
  como cualquier otro reproductor
- 8-40 s, ~20-40 MB (CRF 23); el bucle es simple (`loops: -1`), el salto del
  punto de loop es inherente a los clips reales
- Coste medido: **~4 % de un núcleo** (AMD 780M, Vulkan H.264) en 1440p30

> **Nota:** `omarchy theme set` re-stagea *todo* el clone (versionado + no
> versionado), así que tus clips se incluyen solos en la copia staged.

## Gestor de la librería (TUI)
`video-manage` es una interfaz de terminal (con `gum`, con los colores del
tema activo) para gestionar toda la librería: ver los clips con su estado
(● el que está sonando, [own palette]/[library]), reproducir cualquiera
(vídeo + paleta), **añadir** uno nuevo (selector de fichero nativo → tema
per-clip con paleta Aether → espejado en la librería con hardlinks) y
**borrar** uno (con confirmación, de todas partes: tema per-clip, copias de
librería y lista de ciclo). Si el clip que borras está sonando, cambia a otro
antes; nunca toca tus ficheros originales.

```bash
video-manage              # el TUI
video-add.sh clip.mp4     # añadir sin TUI (--strip-audio para quitar el audio)
video-remove.sh nombre    # borrar sin TUI
```

### Keybindings (se instalan solos al usar cualquier herramienta)
| Tecla | Acción |
|---|---|
| `Super+Ctrl+Space` | Selector de wallpaper unificado (carrusel de vídeos en temas de vídeo, selector de imagen en el resto) |
| `Super+Ctrl+Alt+Izq/Der` | Vídeo anterior / siguiente (cicla toda la librería) |
| `Super+Ctrl+Alt+W` | Gestor de la librería (el TUI `video-manage`) en una ventana de terminal |

## Tema desde un clip (Aether)
El plugin incluye `bin/video-theme.sh`: genera un **tema Omarchy completo** a
partir de un clip — extrae un poster, [Aether](https://github.com/omacom/aether)
deriva la paleta de colores y genera las configs (terminal, barra, lock
screen, …), instala el tema en `~/.config/omarchy/themes/<nombre>/` con el clip
en su `videos/`, y lo activa. Resultado: el wallpaper de video y todos los
colores de acento del sistema salen del mismo clip.

```bash
~/.config/omarchy/plugins/p3lu.video-background/bin/video-theme.sh \
  ~/Videos/mi-clip.mp4 mi-tema
```

Requisitos: `aether` en `PATH`, `ffmpeg`/`ffprobe`, y el plugin activo (sin él,
el tema muestra el poster en lugar del video). Para actualizar el clip:
sustituye el fichero en `videos/` del tema y `omarchy theme set <nombre>`.

Con varios clips creados así (uno por tema), `video-next` / `video-prev` y el
selector de vídeo (`Super+Ctrl+Space`) ciclan entre ellos — y como cada clip
es un tema, **cada cambio de vídeo cambia también toda la paleta del sistema**
(terminal, barra, bordes…). Los temas que crea esta herramienta se limpian
solos cuando se abandonan (no están activos ni en la lista de ciclo), así que
no ensucian el selector de temas.

> **Sin duplicados:** si un clip existe a la vez como tema propio (con su
> paleta) y como fichero suelto en `videos/` de otro tema (p. ej.
> `video-wallpaper`), el ciclo y el selector solo lo visitan una vez —
> siempre por su tema propio, el que lleva la paleta.

## Cómo funciona el wallpaper de video
- El plugin lee el **tema activo** (`~/.local/state/omarchy/current/theme`) y,
  si ese tema trae `videos/*.mp4`, reproduce el clip en bucle y mudo en la capa
  `Background` (un `MediaPlayer` por panel → multi-monitor).
- **Ciclo:** `omarchy theme bg next` (o `bg set`) avanza al siguiente clip —
  el video se **deriva del fondo activo** (mismo nombre base que el PNG),
  así imagen y video avanzan juntos y no pueden desincronizarse. El clip
  activo sobrevive a reinicios del shell (el symlink de fondo es el estado
  persistido de Omarchy); `omarchy theme set` (otro tema) vuelve al clip 1.
- **Ahorro de energía:** el video se **pausa** con la sesión bloqueada o en
  idle (el screensaver cubre el escritorio) y reanuda in situ al volver.
- **Sin flash negro:** el video solo se muestra tras el primer frame decodificado;
  hasta entonces se ve el PNG del clip.
- Si el tema **no** trae `videos/`, el plugin se comporta exactamente como el
  `omarchy.background` stock: pinta la imagen de `current/background`. Así
  `omarchy theme set`, `omarchy theme bg next`, las transiciones y el lock
  screen siguen funcionando igual.
- **Fallback:** MP4 corrupto o indecodificable → se ve el PNG (nunca pantalla
  negra). Verificado: el shell sobrevive a un MP4 basura.

## Estructura del proyecto
```
video-wallpaper/        ← repo del TEMA (raíz)
├── README.md  ATTRIBUTION.md  LICENSE
├── colors.toml  icons.theme
├── mako.ini  hyprlock.conf  ...   (configs por app de la skin)
├── backgrounds/          PNG (fallback + lock)
└── videos/               MP4 (loops)

video-background/       ← repo del PLUGIN (separado)
├── manifest.json
└── Background.qml
```

## Licencia y contenido de terceros
- **Código y configs:** **MIT** (ver [`LICENSE`](LICENSE)).
- **`videos/` y `backgrounds/` incluidos:** material libre de
  [Wikimedia Commons](https://commons.wikimedia.org/) (CC0 / CC BY) —
  atribución completa en [`ATTRIBUTION.md`](ATTRIBUTION.md).
- **Tus clips:** los archivos de video que añadas para uso personal son
  responsabilidad tuya. Este repo **no distribuye** material derivado de
  terceros (series, películas, etc.).

## Créditos
- **[Omarchy](https://github.com/basecamp/omarchy)** (MIT) — la plataforma.
  El plugin `p3lu.video-background` se deriva del plugin stock
  `omarchy.background` de Omarchy (misma licencia MIT).
- **[Wikimedia Commons](https://commons.wikimedia.org/)** — fuente de los
  clips de ejemplo incluidos (ver [`ATTRIBUTION.md`](ATTRIBUTION.md)).
- **[moewalls.com](https://moewalls.com/)** — fuente habitual de clips de
  anime/cyberpunk para uso personal (muchos proceden de Steam Community).
  Los clips que uses son responsabilidad tuya: este repo no los distribuye.
