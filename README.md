# Video Wallpaper — Theme for Omarchy

An [Omarchy](https://omarchy.org/) theme for **video wallpapers**: loop
**your own clips** in `videos/` as your desktop background, muted, with
automatic image fallback. The included skin is neon (cyan / magenta / yellow
on black).

> Preview: ![video wallpaper preview](preview.gif)
>
> **Status:** v0.6.0 — theme + video plugin (v0.6.0) verified on real
> hardware. Ships 3 sample clips (H.264, no audio, via Git LFS; free
> material from Wikimedia Commons, see [`ATTRIBUTION.md`](ATTRIBUTION.md)) —
> **add your own clips** to `videos/` (see [Videos](#videos-add-your-own)),
> or turn any clip into a full, palette-matched theme with the Aether helper
> (see [Theme from a clip](#theme-from-a-clip-aether)).

## What's included

- **Video wallpaper**: a looping, muted clip from the active theme as your
  desktop background, with **automatic image fallback** (never a black
  screen).
- **Clip cycling**: `omarchy theme bg next` advances to the next clip —
  image and video advance together.
- **Neon skin** (cyan / magenta / yellow on black) applied to shell, bar,
  notifications, OSD, terminal and apps. Make it yours in `colors.toml`.
- 100% managed with **Omarchy's own commands** (install, update, remove).

## Architecture (two repos)

Omarchy separates **themes** and **plugins**:

| Artifact | Repo | Installed with | What it provides |
|---|---|---|---|
| **Theme** `video-wallpaper` | `github.com/p3lusa/video-wallpaper` | `omarchy theme install` | colors, icons, configs, `backgrounds/`, `videos/` |
| **Plugin** `p3lu.video-background` | `github.com/p3lusa/video-background` | `omarchy plugin add` | the video renderer (background layer) + the library tools (TUI, add/remove, cycler) |

> **Why two repos?** `omarchy theme install` clones the repo at the root of
> `themes/<name>/` (theme files), and `omarchy plugin add` clones the repo at
> the root of `plugins/<id>/` expecting a `manifest.json` at the root. They
> can't share the same root → two repos.

## Requirements

- Omarchy (Hyprland + Quickshell)
- `QtMultimedia` for Quickshell (Qt6) — ships with the distro
- (Recommended) hardware decoding via Vulkan (AMD / NVIDIA / Intel)

## Installation (Omarchy's own commands)

```bash
# 1) Theme
omarchy theme install https://github.com/p3lusa/video-wallpaper.git

# 2) Video plugin
omarchy plugin add https://github.com/p3lusa/video-background.git --enable

# 3) Hand the background layer over to the video renderer (once)
omarchy plugin disable omarchy.background

# 4) Apply the theme
omarchy theme set video-wallpaper
```

## Update / uninstall

```bash
# Update (the clone is pulled; re-applying the theme re-stages the assets)
omarchy theme update
omarchy theme set video-wallpaper
omarchy plugin update p3lu.video-background

# Uninstall (restores the stock image background behavior)
omarchy theme remove video-wallpaper
omarchy plugin remove p3lu.video-background --yes
omarchy plugin enable omarchy.background
```

> **Note:** `omarchy theme update` does a `git pull` on the clone, but the
> active theme is a staged copy at `~/.local/state/omarchy/current/theme/` —
> that's why an update ends with `omarchy theme set video-wallpaper`
> (re-stage + transition, no shell restart).

## Videos: add your own

Clips live in **`videos/`** (theme root), one `*.mp4` per background. Each
clip has a paired PNG in `backgrounds/` **with the same base name** (that PNG
is the fallback, the lock screen, and what `bg next` shows between videos).

This repo ships 3 sample clips (free material, see
[`ATTRIBUTION.md`](ATTRIBUTION.md)). To add your own:

```bash
# 1) Re-encode with this recipe (1440p30, no audio)
ffmpeg -i source.mp4 \
  -vf "scale=2560:1440:flags=lanczos,fps=30" \
  -c:v libx264 -preset medium -crf 23 -profile:v high \
  -an -movflags +faststart -y /tmp/NAME.mp4

# 2) And its PNG (frame at 40%, avoids opening fades)
dur=$(ffprobe -v error -show_entries format=duration -of csv=p=0 /tmp/NAME.mp4)
ffmpeg -ss "$(awk "BEGIN{printf \"%.3f\", $dur*0.4}")" -i /tmp/NAME.mp4 \
  -frames:v 1 -q:v 2 /tmp/NAME.png

# 3) Drop them in the installed clone (untracked: git pull won't touch them)
cp /tmp/NAME.mp4 ~/.config/omarchy/themes/video-wallpaper/videos/
cp /tmp/NAME.png ~/.config/omarchy/themes/video-wallpaper/backgrounds/
omarchy theme set video-wallpaper   # re-stages the whole clone (including yours)
```

> If you don't want them showing in `git status`, add their names to
> `~/.config/omarchy/themes/video-wallpaper/.git/info/exclude` (only affects
> that clone, not the repo).

**Clip spec** (recommended; the bundled loops are 1080p30):

- `H.264` (hardware decoding; HEVC untested on this stack)
- `2560x1440` (16:9 1440p; the reference panel is 2560x1600 and the video is
  cropped with `KeepAspectRatioByExpanding`), `30 fps`
- **No audio track** — *required*: the FFmpeg backend of `QtMultimedia`
  (Qt 6.11) exposes no `muted`/`volume`, so a clip with audio would be heard
- 8–40 s, ~20–40 MB (CRF 23); the loop is plain (`loops: -1`), the seam is
  inherent to real footage
- Measured cost: **~4 % of a core** (AMD 780M, Vulkan H.264) at 1440p30

> **Note:** `omarchy theme set` re-stages *everything* in the clone
> (tracked + untracked), so your clips are picked up in the staged copy
> automatically.

## Library manager (TUI)

`video-manage` is a terminal UI (built on `gum`, styled by the active
theme's palette) for the whole library: browse the clips with live status
(● the playing one, `[own palette]`/`[library]`), play any clip (video +
palette), **add** a new one (native file picker → per-clip theme with an
Aether palette → mirrored into the library with hardlinks), and **remove**
one (with confirmation, from everywhere: per-clip theme, library copies, and
the cycle list). If the clip being removed is the playing one, it switches to
another video first. It never touches your original clip files.

```bash
video-manage              # the TUI
video-add.sh clip.mp4     # add without the TUI (--strip-audio drops the audio track)
video-remove.sh name      # remove without the TUI
```

### Keybindings (self-installing)

The first time any video tool runs, the plugin installs these keybindings
into `~/.config/hypr/bindings.lua` (marked block, never touches your own
lines; `video-bindings.sh --remove` takes them back):

| Key | Action |
|---|---|
| `Super+Ctrl+Space` | **Unified wallpaper switcher** — takes over the stock wallpaper key. On video themes it opens the video switcher (a carousel of your whole library with poster previews); on image themes it opens the stock background picker |
| `Super+Ctrl+Alt+Left` / `Right` | Previous / next video (cycles the whole library, wrap-around) |
| `Super+Ctrl+Alt+W` | **Library manager** — opens the `video-manage` TUI in a terminal window (closes when you quit) |

## Theme from a clip (Aether)

The plugin ships `bin/video-theme.sh`: it turns a clip into a **complete
Omarchy theme** — it extracts a poster frame, [Aether](https://github.com/omacom/aether)
derives the color palette and generates the configs (terminal, bar, lock
screen, …), it installs the theme at `~/.config/omarchy/themes/<name>/` with
the clip in its `videos/`, and activates it. Result: the video wallpaper and
every accent color on the system come from the same clip.

```bash
~/.config/omarchy/plugins/p3lu.video-background/bin/video-theme.sh \
  ~/Videos/my-clip.mp4 my-theme
```

Prerequisites: `aether` in `PATH`, `ffmpeg`/`ffprobe`, and the plugin
enabled (without it the theme shows the poster instead of the video). To
update the clip later: replace the file in the theme's `videos/` and
`omarchy theme set <name>`.

With several clips created this way (one per theme), `video-next` /
`video-prev` and the video switcher (`Super+Ctrl+Space`) cycle through them —
and since each clip is a theme, **every video change also changes the whole
system palette** (terminal, bar, borders, …). Themes created by this tool
clean themselves up when abandoned (neither active nor in the cycle list),
so they don't clutter the theme switcher.

> **No duplicates:** if a clip exists both as its own theme (with a palette)
> and as a loose file in another theme's `videos/` (e.g. `video-wallpaper`),
> the cycler and the switcher visit it only once — always via its own
> theme, the one carrying the palette.

## How the video wallpaper works

- The plugin reads the **active theme**
  (`~/.local/state/omarchy/current/theme`) and, when that theme ships
  `videos/*.mp4`, plays the clip looping and muted on the `Background` layer
  (one `MediaPlayer` per panel → multi-monitor).
- **Cycling:** `omarchy theme bg next` (or `bg set`) advances to the next
  clip — the video is **derived from the active background** (same base name
  as the PNG), so image and video advance together and can never desync. The
  active clip survives shell restarts (the background symlink is Omarchy's
  own persisted state); `omarchy theme set` (another theme) returns to clip 1.
- **Power saving:** the video **pauses** while the session is locked or idle
  (the screensaver covers the desktop) and resumes in place when you return.
- **No black flash:** the video only appears after the first decoded frame;
  until then the clip's PNG is shown.
- When a theme does **not** ship `videos/`, the plugin behaves exactly like
  the stock `omarchy.background`: it paints the `current/background` image.
  So `omarchy theme set`, `omarchy theme bg next`, transitions, and the lock
  screen all keep working as usual.
- **Fallback:** a corrupt or undecodable MP4 → the PNG is shown (never a
  black screen). Verified: the shell survives a junk MP4.

## Project layout

```
video-wallpaper/        ← THEME repo (root)
├── README.md  ATTRIBUTION.md  LICENSE
├── colors.toml  icons.theme
├── mako.ini  hyprlock.conf  ...   (per-app configs of the skin)
├── backgrounds/          PNGs (fallback + lock)
└── videos/               MP4s (loops)

video-background/       ← PLUGIN repo (separate)
├── manifest.json
├── Background.qml
└── bin/                  (video-manage, video-add, video-remove, …)
```

## License and third-party content

- **Code and configs:** **MIT** (see [`LICENSE`](LICENSE)).
- **Bundled `videos/` and `backgrounds/`:** free material from
  [Wikimedia Commons](https://commons.wikimedia.org/) (CC0 / CC BY) — full
  attribution in [`ATTRIBUTION.md`](ATTRIBUTION.md).
- **Your clips:** the video files you add for personal use are your
  responsibility. This repo **does not distribute** third-party derivative
  material (series, movies, etc.).

## Credits

- **[Omarchy](https://github.com/omacom/omarchy)** (MIT) — the platform.
  The `p3lu.video-background` plugin is derived from Omarchy's stock
  `omarchy.background` plugin (same MIT license).
- **[Wikimedia Commons](https://commons.wikimedia.org/)** — source of the
  bundled sample clips (see [`ATTRIBUTION.md`](ATTRIBUTION.md)).
- **[moewalls.com](https://moewalls.com/)** — a common source of
  anime/cyberpunk clips for personal use (many from Steam Community). The
  clips you use are your responsibility: this repo does not distribute them.
