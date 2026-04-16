# Omarchy Theme Backgrounds — Session Walkthrough

A trace of a single Claude Code session in which the `/omarchy` system-level
skill was invoked to replace every pre-installed theme's wallpaper with a
solid-color image derived from that theme's own background color.

The session prompt was:

> /omarchy
>
> I don't particularly care for the background images that come with the
> pre-installed themes. For every pre-installed theme, I'd like to replace
> the default wallpaper with a single solid-color image that uses that
> theme's own background color as a flat fill. Name the file `0.default.png`
> so it sorts first and becomes the theme's default wallpaper.

---

## 1. Initial Skill Load

When the user typed `/omarchy`, Claude Code loaded the system-level skill
at `~/.claude/skills/omarchy/SKILL.md` before responding to the prompt.
The skill supplied domain context that Claude would otherwise have had to
infer or discover the hard way:

- **Scope rules.** The skill's first load-time message is a safety rule:
  `~/.local/share/omarchy/` is managed by git and will be overwritten on
  `omarchy-update`, so it is read-only. All customization belongs under
  `~/.config/`. Reading the source tree is encouraged; editing it is not.
- **Directory map.** The skill enumerates where Hyprland, Waybar, Walker,
  the terminals, and the theme templates live, along with what reloads
  automatically and what must be restarted manually.
- **Theme architecture.** It explains that stock themes live at
  `~/.local/share/omarchy/themes/<name>/`, user themes at
  `~/.config/omarchy/themes/<name>/`, and that user files are layered on
  top of stock ones when a theme is applied.
- **Command discovery.** It documents the `omarchy-<category>-<action>`
  naming pattern and shows how to enumerate all ~145 user-facing commands
  with `compgen -c | grep omarchy`, plus how to read any command's source
  with `cat $(which omarchy-theme-set)`.
- **Decision framework.** A short checklist routes requests to the right
  mechanism — stock command, config edit under `~/.config/`, custom theme
  directory, or automation hook.

None of the skill content anticipated the specific request about
solid-color wallpapers. What it provided was enough grounding for Claude
to know *where* theme assets live, *what* it can safely modify, and
*which* commands to read if it needed to understand behavior.

---

## 2. Research

With the skill's map in hand, Claude investigated four concrete questions
in the user's system before proposing a change.

### 2.1 What themes exist and what files do they contain?

`~/.local/share/omarchy/themes/` held 19 directories: catppuccin,
catppuccin-latte, ethereal, everforest, flexoki-light, gruvbox,
hackerman, kanagawa, lumon, matte-black, miasma, nord, osaka-jade,
retro-82, ristretto, rose-pine, tokyo-night, vantablack, white.

A representative layout (catppuccin):

```
backgrounds/
  1-totoro.png
  2-waves.png
  3-blue-eye.png
btop.theme
colors.toml
icons.theme
neovim.lua
preview.png
vscode.json
waybar.css
```

### 2.2 Where is each theme's background color defined?

The canonical source is `colors.toml` at the root of each theme
directory. The relevant key is `background`:

```toml
# catppuccin/colors.toml
accent = "#89b4fa"
foreground = "#cdd6f4"
background = "#1e1e2e"
...
```

Sampled values across the 19 themes:

| Theme            | background |
| ---------------- | ---------- |
| catppuccin       | `#1e1e2e`  |
| tokyo-night      | `#1a1b26`  |
| nord             | `#2e3440`  |
| kanagawa         | `#1f1f28`  |
| gruvbox          | `#282828`  |
| rose-pine        | `#faf4ed`  |
| white            | `#ffffff`  |

### 2.3 How does Omarchy pick which wallpaper to show?

Reading `~/.local/share/omarchy/bin/omarchy-theme-bg-next` revealed the
cycler's selection logic:

```bash
THEME_BACKGROUNDS_PATH="$HOME/.config/omarchy/current/theme/backgrounds/"
USER_BACKGROUNDS_PATH="$HOME/.config/omarchy/backgrounds/$THEME_NAME/"

mapfile -d '' -t BACKGROUNDS < <(find -L \
  "$USER_BACKGROUNDS_PATH" "$THEME_BACKGROUNDS_PATH" \
  -maxdepth 1 -type f -print0 2>/dev/null | sort -z)
```

Two relevant facts fell out of this:

1. The cycler searches both the user backgrounds directory *and* the
   active theme's stock backgrounds, then sorts the combined list by full
   path. The user path sorts alphabetically before the stock path, so any
   file the user drops into `~/.config/omarchy/backgrounds/<theme>/`
   naturally precedes stock assets.
2. When no current-background symlink exists (i.e. immediately after
   `omarchy-theme-set`), the script picks `BACKGROUNDS[0]`. This means
   the first alphabetically sorted file becomes the active wallpaper by
   default. A filename prefixed with `0.` is guaranteed to win that
   sort.

### 2.4 Are image tools available?

ImageMagick 7.1.2 was already installed at `/usr/bin/magick`. Generating
a solid-color PNG is a one-liner: `magick -size WxH xc:"#COLOR" out.png`.

---

## 3. Plan

The plan that emerged from this research was deliberately minimal —
small scope, no new long-lived tooling, no edits to protected source
directories.

**Approach:** A single idempotent bash script that, for each directory
under `~/.local/share/omarchy/themes/`:

1. Reads the `background` hex from `colors.toml`.
2. Creates `~/.config/omarchy/backgrounds/<theme>/` if it doesn't exist.
3. Uses ImageMagick to generate a 3840×2160 solid-color PNG at
   `0.default.png` inside that directory. 4K is chosen not because
   resolution matters for solid fills (PNG compresses any uniform color
   to ~1 KB) but so `swaybg -m fill` never has to upscale for an
   unusually large display. The `0.` prefix guarantees it sorts first.
4. Skips themes whose `0.default.png` already exists, so re-running
   after an `omarchy-update` that adds new stock themes is safe.
5. After the loop, re-applies the current theme via
   `omarchy-theme-set "$(< ~/.config/omarchy/current/theme.name)"` so
   the active wallpaper refreshes immediately without a logout.

**Why this location and naming:** Claude deliberately avoided
`~/.local/share/omarchy/themes/` because the skill's safety rules
forbid it and upstream updates would erase the work. The user-level
backgrounds directory is exactly the extension point Omarchy provides,
and the `omarchy-theme-bg-next` logic above turns the `0.` naming
convention into a free "default wallpaper" mechanism without any custom
command or hook. Stock wallpapers remain in the rotation for users who
press the bg-next binding — they're just no longer the first thing
shown when a theme is applied.

**Files created:** 19 PNGs, one per theme, at
`~/.config/omarchy/backgrounds/<theme>/0.default.png`. No files
modified; nothing under `~/.local/share/omarchy/` is touched.

---

## 4. Execution

The script ran once and produced output confirming extraction of each
theme's `background` value:

```
catppuccin (#1e1e2e)       -> .../catppuccin/0.default.png
catppuccin-latte (#eff1f5) -> .../catppuccin-latte/0.default.png
ethereal (#060B1E)         -> .../ethereal/0.default.png
everforest (#2d353b)       -> .../everforest/0.default.png
flexoki-light (#FFFCF0)    -> .../flexoki-light/0.default.png
gruvbox (#282828)          -> .../gruvbox/0.default.png
hackerman (#0B0C16)        -> .../hackerman/0.default.png
kanagawa (#1f1f28)         -> .../kanagawa/0.default.png
lumon (#16242d)            -> .../lumon/0.default.png
matte-black (#121212)      -> .../matte-black/0.default.png
miasma (#222222)           -> .../miasma/0.default.png
nord (#2e3440)             -> .../nord/0.default.png
osaka-jade (#111c18)       -> .../osaka-jade/0.default.png
retro-82 (#05182e)         -> .../retro-82/0.default.png
ristretto (#2c2525)        -> .../ristretto/0.default.png
rose-pine (#faf4ed)        -> .../rose-pine/0.default.png
tokyo-night (#1a1b26)      -> .../tokyo-night/0.default.png
vantablack (#0d0d0d)       -> .../vantablack/0.default.png
white (#ffffff)            -> .../white/0.default.png
```

### Verification

Pixel-level color check against three themes at the corners of the
color space:

```
$ magick ~/.config/omarchy/backgrounds/catppuccin/0.default.png \
    -format "%[pixel:p{0,0}]" info:
srgb(30,30,46)          # = #1e1e2e ✓

$ magick ~/.config/omarchy/backgrounds/nord/0.default.png \
    -format "%[pixel:p{0,0}]" info:
srgb(46,52,64)          # = #2e3440 ✓

$ magick ~/.config/omarchy/backgrounds/white/0.default.png \
    -format "%[pixel:p{0,0}]" info:
gray(255)               # = #ffffff ✓
```

Dimension and format check:

```
$ magick identify ~/.config/omarchy/backgrounds/tokyo-night/0.default.png
PNG 3840x2160 8-bit sRGB
```

File size: roughly 1 KB per image (PNG run-length compression is brutal
on a uniform color), 19 images total, so under 30 KB of disk for the
full set.

### End-to-end

Calling `omarchy-theme-set "$(< ~/.config/omarchy/current/theme.name)"`
triggered `omarchy-theme-bg-next`, which re-scanned the user and stock
background paths together, selected the alphabetically first entry, and
symlinked it:

```
$ readlink ~/.config/omarchy/current/background
/home/jaime/.config/omarchy/backgrounds/kanagawa/0.default.png
```

The desktop wallpaper became a solid `#1f1f28` fill. Switching to any
other theme (e.g. `omarchy-theme-set "Tokyo Night"`) immediately
produced its corresponding solid color with no additional steps.

### Follow-up

Re-running the script after a future `omarchy-update` is idempotent:
themes whose `0.default.png` already exists are skipped, and any new
stock themes introduced by the update get fresh solid-color wallpapers
generated from their `colors.toml`. No on-disk state outside
`~/.config/omarchy/backgrounds/` changed.
