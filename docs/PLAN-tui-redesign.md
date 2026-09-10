# Plan — Rediseño del TUI (estilo moderno)

**Objetivo:** un TUI atractivo y moderno para `video-manage`, manteniendo el stack
**sin dependencias extra** (fzf + gum, temado por la paleta activa de Omarchy).

**Relación con el otro plan:** el rediseño es visual/UX; la lógica de add/remove
(`PLAN-tui-fixes.md`) es independiente y se puede implementar antes, después o junto.

---

## Principios

- Stack shell (fzf + gum), zero deps por defecto. Nerd Font icons con fallback ASCII.
- Todo temado por la paleta activa (re-temado en vivo al cambiar de clip).
- Jerarquía visual clara: **lo que suena → la librería → las acciones**.
- Feedback inmediato y visible (línea de feedback, spinner, toast).

## Diseño objetivo (mockup)

```
┌  ▣ video library ────────────────────────────────────────────────────────┐
│  ▸ AHORA SUENA                                                           │
│    ● 02-aurora-spruce-woods  [own]  12s · 1920x1080@30 · 9.4MB           │
│                                                                           │
│  ── LIBRERÍA (4) ───────────────────────────────────────────────────────  │
│     01-city-night-ljubljana   [library]  20s · 1920x1080@24 · 14.1MB      │
│     02-aurora-spruce-woods    [own]      12s · 1920x1080@30 ·  9.4MB  ●   │
│     03-rain-window-city       [own]       8s · 1440x 900@30 ·  7.2MB      │
│     07-rebecca-gun            [library]  15s · 1080x1920@24 · 11.0MB      │
│                                                                           │
│  ── ACCIONES ────────────────────────────────────────────────────────────  │
│     + Añadir video                                                        │
│     ✕ Quitar video                                                        │
│     ? Ayuda                                                               │
├───────────────────────────────────────────────────────────────────────────┤
│ filter: ab_                                                                │
├───────────────────────────────────────────────────────────────────────────┤
│  a añadir · r quitar · enter reproducir · ? ayuda · q salir               │
└───────────────────────────────────────────────────────────────────────────┘
```

**Pane derecho (preview):** miniatura del poster (opcional) + tarjeta de metadatos
(theme, kind, status, media, size) + hints de acción (`enter → play · r → remove`).

## Checklist visual

1. **Header "hero".** Tarjeta superior con el clip que suena (marcador ● en accent,
   nombre, tag, mini-meta). Separa "estado" de "lista".
2. **Listas seccionadas.** Secciones `LIBRERÍA (n)` y `ACCIONES` con encabezados
   estilizados (regla + label), en vez de una lista plana que mezcla todo.
3. **Filas ricas y alineadas.** Icono + nombre + tag de tipo + duración/resolución +
   marcador de "sonando". Columnas alineadas; tag de tipo con color
   (`own` = accent, `library` = muted).
4. **Preview pane con miniatura.** El poster del clip como imagen + tarjeta de
   metadatos + hints (ver punto "Miniatura").
5. **Footer persistente.** Barra inferior con atajos y recuento, siempre visible
   (no una pantalla aparte).
6. **Modales gum re-estilizados.** File picker, confirm, spinner y help con bordes
   redondeados, accent y copy consistente.
7. **Theming truecolor pulido.** Gradientes sutiles en el header, bordes redondeados,
   espaciado generoso.
8. **Íconos Nerd Font** con fallback ASCII (ya existe; extender al diseño).

## Miniatura del poster (stretch goal, opcional)

- Renderizar el `backgrounds/<base>.png` en el preview pane.
- Opciones: **protocolo de imagen del terminal** (kitty / i3 / ip6 / sixel) si el
  terminal lo soporta, o `chafa` / `viu` como renderizador (sixel/kitty).
- **Fallback elegante:** si no hay renderizador o el terminal no soporta imagen →
  tarjeta de texto (estado actual). Nunca romper el TUI.
- **Decisión:** ¿`chafa` como dep opcional, o solo protocolo nativo? (Ghostty soporta
  kitty + sixel y es el terminal por defecto de Omarchy.)

## Pasos de implementación (incremental, cada uno verificable)

- **A. Estructura + header/footer** (sin nuevas deps): secciones, header hero, footer
  persistente. → *Verificar:* layout correcto, re-temado en vivo al cambiar clip.
- **B. Filas ricas + íconos + alineación:** columnas, tags de color, marcador de
  sonando. → *Verificar:* alineación estable con nombres largos.
- **C. Tarjeta de metadatos en el preview:** reorganizar el preview actual en tarjeta
  estilizada. → *Verificar:* metadatos correctos por clip.
- **D. Miniatura del poster** (opcional, dep `chafa` o protocolo nativo + fallback).
  → *Verificar:* poster visible en terminals que soportan; fallback en texto en el resto.
- **E. Re-estilo de modales gum + feedback/toast:** picker/confirm/spinner/help; línea
  de feedback tras cada operación. → *Verificar:* consistencia visual y feedback claro.

## Verificación global

- Navegación: `j/k`, filtro, `enter` (play), `r` (picker), `a` (add), `?` (help), `q`.
- Re-temado: cambiar a un clip `own` → el TUI cambia de color en vivo.
- Fallback: sin Nerd Font → íconos ASCII; sin imagen → preview de texto.
- Paridad funcional: add/remove/play siguen funcionando (ver `PLAN-tui-fixes.md`).

## Riesgo

- La miniatura (paso D) es lo único que puede requerir una dep opcional; el resto es
  zero-dep. Si se prefiere estrictamente zero-deps, D se hace solo vía protocolo
  nativo del terminal (sin chafa) o se omite.

## Open questions (para decidir antes de arrancar)

1. ¿Miniatura del poster **sí/no**? ¿Dep opcional `chafa` aceptable o estricto
   zero-deps?
2. ¿`r` = **picker** (como en `PLAN-tui-fixes.md` punto 3) o "borrar el destacado"?
   (afecta los hints del footer)
3. ¿**Hero header** fijo arriba, o layout de **dos paneles** (lista | detalle) tipo
   file manager?
