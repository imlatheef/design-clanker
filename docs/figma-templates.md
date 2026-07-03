# Figma template guide

How Design Clanker renders: the Figma REST API can't edit files, so the frame is exported once as a base image, the text/image node geometry is read from the file JSON, and your Airtable data is redrawn on top with Pillow — matching each node's position, size, font, color, and alignment. That means the quality of the output depends on how the Figma file is set up.

## Layer naming

Layer names are the contract between Figma and `templates.yaml`. For every mapped field:

```yaml
field_mappings:
  "Full Name": "Speaker Name"   # ← a TEXT layer named exactly "Speaker Name"
image_field_mappings:
  "Headshot":  "Photo"          # ← a shape/frame layer named exactly "Photo"
```

- Names are **case-sensitive** and matched exactly.
- The whole frame tree is walked, so mapped layers can be nested inside groups and auto-layout frames.
- Avoid duplicate names among mapped layers within one frame — the first match wins (PDF combo pages can disambiguate with `"Name[1]"`; image templates cannot).

## Text layers

- **Set placeholder text layers to 0% opacity** and use `erase_placeholders: false`. The exported base image then has clean empty space and the renderer draws your data into it. This is the single biggest quality win.
  - Alternative: leave `erase_placeholders: true` (the default) and the renderer paints a background-coloured box over each text node before drawing. This only looks right on solid backgrounds.
- Keep placeholder content roughly the same length as real data so you design realistic bounding boxes. Long values wrap within the node's width.
- Text is drawn with the node's **font size, fill color, and horizontal alignment** as set in Figma. Left/center/right all work.
- Fixed-size text boxes behave more predictably than auto-width ones.

## Fonts

- Put the TTF files in the repo's `fonts/` directory and map them in `font_map`.
- Figma reports the family plus weight as one name: `"Space Grotesk Bold"`, `"Space Grotesk Medium"`. Map each weight you use.
- **Variable fonts don't work with Pillow.** Download the *static* files from Google Fonts (the `static/` folder inside the download) — one file per weight.
- Unmapped fonts fall back to system Arial/Helvetica/DejaVu, which will look noticeably wrong — map everything you use.

## Image placeholders

- Any non-text layer can be a photo slot: rectangle, ellipse, frame, component instance.
- Photos are **aspect-fill cropped** to the placeholder's box.
- **Corner radius is respected** — a rectangle with 24px radius clips the photo at 24px; an ellipse produces a circular crop automatically.
- With `remove_background: true`, headshots get AI background removal before compositing (photos that already have transparency are skipped), so design the placeholder as if the subject sits directly on your artwork.
- Order layers so the placeholder sits where the photo should appear visually — the photo is composited *on top* of the exported base image at the placeholder's position.

## Frames and node IDs

- Each output/variant points at one **top-level frame**.
- Get the node ID via right-click → **Copy link to selection**; the URL ends with `?node-id=12-345` → use `"12:345"` (or `12-345`, both accepted).
- Frame size = output size. Export scale multiplies it: a 1200×630 frame at `figma_export_scale: 2.0` renders a 2400×1260 JPEG.
- Common sizes: OG image 1200×630, LinkedIn 1200×627, square social 1080×1080, Instagram story 1080×1920.

## Variants

Variant frames (two-speaker layout, sponsored layout, per-city designs) must use the **same layer names** as the default frame — mappings are shared. Extra layers that only exist in a variant (e.g. `Speaker 2 Name`) are fine: map them via `field_mapping_overrides` on the output, and they're simply not found (warning, not error) when the default frame renders.

Keep all variants of an output in the same Figma file, side by side, named clearly (`OG / Single`, `OG / Two Speakers`, `OG / Sponsored`) — future you will thank you.

## Editing a live template

Frame exports and node data are **cached per process**. After changing the Figma file, restart the poller (or redeploy) to pick up the changes.

## Checklist

- [ ] Every mapped text layer exists, named exactly as in `templates.yaml`
- [ ] Placeholder text at 0% opacity, `erase_placeholders: false`
- [ ] All used font weights present as static TTFs in `fonts/` and listed in `font_map`
- [ ] Image placeholders sized/rounded as the photo should appear
- [ ] Frame is top-level and its node ID is in the config
- [ ] Variant frames share layer names with the default frame
