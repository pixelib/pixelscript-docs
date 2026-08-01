# Branding assets

The three SVGs below are **copies**. The source of truth is `branding/` in the
[pixelscript](https://github.com/pixelib/pixelscript) repository — change them there and copy them back
in, do not edit them here.

| File | Source | Use |
|------|--------|-----|
| `pixelscript-logo.svg` | copied | Full lockup: mark + wordmark + tagline. Inherits `currentColor` for the "Pixel" half and the tagline, so it only looks right when inlined or given a `color`. |
| `pixelscript-mark.svg` | copied | Square mark, fully self-coloured. Safe as an `<img>` anywhere. Sidebar lockup and SVG favicon. |
| `pixelscript-mark-mono.svg` | copied | Single-colour mark for contexts that need one ink. Currently unused by the site; kept so the set stays complete. |
| `pixelscript-logo-on-dark.svg` | **derived** | `pixelscript-logo.svg` with `style="color:#EAFBEE"` on the root, so it renders correctly as an `<img>` on the dark site. Used as the home page hero. |
| `apple-touch-icon.png` | **derived** | 180×180 render of the mark on `#12181e`. |
| `social-card.png` | **derived** | 1200×630 Open Graph card, the on-dark lockup centred on `#12181e`. |
| `../../favicon.ico` | **derived** | 16/32/48/64 render of the mark, in the site root where just-the-docs looks for it. |

## Regenerating the derived files

Run from the repository root, after copying fresh SVGs in:

```bash
sed '1s|<svg |<svg style="color:#EAFBEE" |' \
  assets/branding/pixelscript-logo.svg > assets/branding/pixelscript-logo-on-dark.svg

for n in 16 32 48 64; do
  rsvg-convert -w $n -h $n assets/branding/pixelscript-mark.svg -o /tmp/icon-$n.png
done
magick /tmp/icon-16.png /tmp/icon-32.png /tmp/icon-48.png /tmp/icon-64.png favicon.ico

rsvg-convert -w 180 -h 180 -b '#12181e' \
  assets/branding/pixelscript-mark.svg -o assets/branding/apple-touch-icon.png

rsvg-convert -w 880 assets/branding/pixelscript-logo-on-dark.svg -o /tmp/logo-880.png
magick -size 1200x630 xc:'#12181e' /tmp/logo-880.png -gravity center -composite \
  -depth 8 -strip assets/branding/social-card.png
```

## Palette

Taken from the marks; mirrored in `_sass/color_schemes/pixelscript.scss`.

| Colour | Hex | Where |
|--------|-----|-------|
| Leaf green | `#56D964` | Top face of the block |
| Mid green | `#2E9F4A` | Left face, primary buttons |
| Deep green | `#1C7038` | Right face |
| Light green | `#7CF089` | Pixel dust, links |
| Off-white | `#EAFBEE` | Code texture, headings |
| Amber | `#FFB020` | "Script" in the wordmark, base pixels |
| Brown | `#8C5A1E` | Shadowed base pixels |
| Site background | `#12181e` | Page background, icon backdrops, `theme-color` |
