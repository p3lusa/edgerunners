# Plan — Correcciones de add/remove (5 puntos)

**Estado: ✅ implementado y verificado (2026-09-10).** Cambios en
`bin/video-theme.sh`, `bin/video-add.sh`, `bin/video-remove.sh`,
`bin/video-manage.sh`, `bin/video-cleanup.sh`. Verificación: sandbox con
`HOME` fake + stubs de `omarchy`/`aether` (add con/sin audio, con/sin
`--no-activate`, remove del clip activo con fallback, poda de `.meta`,
TUI bajo PTY).

**Objetivo:** que la adición y el borrado de clips desde el TUI funcionen de forma
predecible y sin efectos colaterales.

**Contexto:** el TUI `plugin/bin/video-manage.sh` delega las operaciones en
`video-add.sh` / `video-remove.sh` (y estos en `video-theme.sh`). Cinco flancos
identificados en la revisión. Los cambios son localizados en `bin/*.sh`; **no**
toca el servicio `Background.qml`.

---

## Punto 1 — `do_add` no ofrece quitar el audio

- **Problema.** `do_add` llama a `video-add.sh "$f"` sin `--strip-audio`. Un clip
  con pista de audio falla (`video-add.sh` → exit 1) y el TUI muestra "add failed"
  sin ofrecer solución. El plugin no tiene control de volumen, así que el audio es
  siempre indeseado.
- **Fix.** En `do_add` (video-manage.sh), precomprobar con `ffprobe -select_streams a`
  si hay pista de audio. Si la hay, `gum confirm` *"El clip tiene audio. ¿Quitarlo?
  (remux lossless)"* y pasar `--strip-audio` a `video-add.sh` si el usuario acepta.
  - Alternativa más simple: auto-`--strip-audio` siempre que haya audio, con un aviso.
- **Archivo.** `plugin/bin/video-manage.sh` → `do_add`.
- **Verificar.** Añadir un clip con audio → prompt → queda añadido sin pista de audio;
  `ffprobe` del espejo confirma ausencia de stream de audio.

## Punto 2 — Race del loop (verificado: OK)

- **Análisis.** `do_add`/`do_remove` bloquean el TUI (`gum spin` síncrono, fzf
  abortado) y `scan_library` se re-ejecuta al top del loop → la lista se refresca
  sola. No hay acceso concurrente. **No requiere cambio funcional.**
- **Opcional (cosmético).** `clip_meta` deja archivos `.meta` huérfanos tras un
  remove (inofensivos). Poda opcional de metas sin clip en `video-cleanup.sh`.
- **Verificar.** Añadir y luego borrar un clip → lista consistente; sin metas
  huérfanos (si se aplica la poda).

## Punto 3 — `r` depende del estado `hl` del preview

- **Problema.** La tecla `r` borra el clip "destacado" leyendo `$SESSION/hl`, que el
  preview escribe. Si el preview no se ha disparado (usuario rápido, primer render),
  `hl` está vacío → `r` no hace nada (silencio). UX inconsistente.
- **Fix (recomendado).** `r` = abrir el **picker de borrado** (mismo que la entrada
  "Remove a video"): lista solo clips, con preview, el usuario elige qué borrar. Sin
  dependencia de `hl`, consistente. Refactor: extraer `remove_picker()` y usarlo en la
  entrada de menú y en la tecla `r`.
  - Alternativa: mantener "borrar el destacado" con guard — si `hl` vacío, cae al picker.
- **Archivo.** `plugin/bin/video-manage.sh` → `main`, bindings, `do_remove`.
- **Verificar.** Pulsar `r` al instante (antes de que el preview pinte) → abre el
  picker; elegir → confirma y borra.

## Punto 4 — Añadir un clip cambia el wallpaper actual

- **Problema.** `video-add.sh` → `video-theme.sh` termina con `omarchy theme set`,
  **activando** el nuevo clip. Al añadir desde el TUI, el wallpaper salta al clip
  nuevo (efecto colateral sorpresa). El TUI ya tiene "play" (Enter) para eso.
- **Fix.** Añadir `--no-activate` a `video-theme.sh` (salta el paso 4,
  `omarchy theme set`; sigue generando el tema + espejo + registro en ciclo). Hilo a
  través de `video-add.sh` (parsea y reenvía). `do_add` pasa `--no-activate`. El CLI
  por defecto sigue activando (comportamiento actual).
- **Archivos.** `plugin/bin/video-theme.sh`, `plugin/bin/video-add.sh`,
  `plugin/bin/video-manage.sh` → `do_add`.
- **Verificar.** Añadir un clip desde el TUI → el wallpaper **no cambia**; el clip
  aparece en la lista; `Enter` lo reproduce.

## Punto 5 — Fallback de `video-remove.sh` no exige el emparejamiento

- **Problema.** Al proteger el estado activo, el fallback elige otro tema con
  `find "$tdir/videos" -name '*.mp4'` (cualquier mp4, sin poster emparejado). Un clip
  sin poster emparejado no se reproduce (el plugin exige el emparejamiento), así que
  podría pasar a un tema donde nada se reproduce. `video-cycle.sh` /
  `video-switcher.sh` sí exigen el emparejamiento.
- **Fix.** Alinear el criterio: solo considerar temas con al menos un clip
  **emparejado** (`backgrounds/<base>.png` + `videos/<base>.mp4`). Extraer un helper
  `paired_clip_exists <theme_dir>` y reutilizarlo (candidato a unificar con la lógica
  de cycle/switcher).
- **Archivo.** `plugin/bin/video-remove.sh` → bloque "protect the active state".
- **Verificar.** Borrar el clip activo de un tema que solo tiene clips sin emparejar →
  no elige ese tema como fallback.

---

## Orden de implementación

`1, 3, 4` (funcionales del TUI) → `5` (correctitud de remove) → `2` (opcional,
cosmético). Cada punto es independiente y testeable por separado.

## Riesgo / rollback

Cambio localizado en `bin/*.sh`; sin cambios al servicio QML. Reversible por punto.
El punto 4 cambia el comportamiento del add (ya no activa) — documentar en README.
