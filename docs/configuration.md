# Configuration reference

Design Clanker is configured in two places:

- **`.env`** — secrets and runtime settings (never commit this)
- **`templates.yaml`** — templates and PDF reports (no secrets, but contains your base IDs and file keys)

## `.env`

| Variable | Required | Default | Description |
|---|---|---|---|
| `AIRTABLE_API_KEY` | ✅ | — | Airtable personal access token. Must start with `pat`. Needs `data.records:read` + `data.records:write` scopes and access to your base(s). Create at [airtable.com/create/tokens](https://airtable.com/create/tokens). |
| `FIGMA_API_KEY` | ✅ | — | Figma personal access token. Must start with `figd_`. Create in Figma → Account Settings → Personal Access Tokens. |
| `AIRTABLE_IMGBB_API_KEY` | — | `""` | [imgbb.com](https://imgbb.com/api) API key. Used as a fallback image host when the Airtable Content API upload fails (some plans restrict it). |
| `POLL_INTERVAL` | — | `30` | Seconds between Airtable polls. |

Configuration is validated on startup — a missing or malformed key fails fast with a message telling you how to fix it.

## `templates.yaml`

Top-level structure:

```yaml
templates:    # list of image templates
  - ...
pdf_reports:  # list of multi-page PDF reports (optional)
  - ...
```

---

### Template keys

#### Identity & Airtable

| Key | Required | Default | Description |
|---|---|---|---|
| `name` | ✅ | — | Human-readable label, used in logs. |
| `airtable_base_id` | ✅ | — | `appXXXXXXXXXXXXXX` — first path segment of your base's URL. |
| `airtable_table_name` | ✅ | — | Exact table name, case-sensitive. |
| `airtable_trigger_field` | — | `"Ready for Design"` | The field the poller watches. |
| `airtable_trigger_value` | — | `""` | If set, the trigger field is treated as a **single-select** and fires when it equals this value. If blank, it's treated as a **checkbox** and fires when checked. |
| `airtable_trigger_pending_value` | — | `""` | Optional status written when the record is picked up (single-select mode). |
| `airtable_trigger_working_value` | — | `""` | Optional status written when rendering starts. |
| `airtable_trigger_reset_value` | — | `""` | Status written after processing (single-select mode). Checkbox triggers are simply unchecked. If blank in single-select mode, the status is left unchanged — **beware:** the record will be picked up again on the next poll unless the value differs from `airtable_trigger_value`. |
| `airtable_attachment_field` | single-output only | `""` | Attachment field the JPEG is uploaded to. Ignored when `outputs` is defined. |

#### Figma (single-output mode)

| Key | Required | Default | Description |
|---|---|---|---|
| `figma_file_key` | ✅* | `""` | From the file URL: `figma.com/design/<FILE_KEY>/...` |
| `figma_frame_node_id` | ✅* | `""` | From "Copy link to selection": `?node-id=12-345` → `"12:345"`. Dashes are normalised to colons automatically. |
| `figma_export_scale` | — | `2.0` | Export scale, `0.5`–`4.0`. |

\* Required unless `outputs` is defined.

#### Mappings

| Key | Description |
|---|---|
| `field_mappings` | `"Airtable field" → "Figma text layer"`. The layer's position, size, font, color, and alignment are read from Figma and the value is drawn to match. |
| `image_field_mappings` | `"Airtable attachment field" → "Figma layer"` (rectangle, ellipse, frame, component…). The first attachment is downloaded, aspect-fill cropped, and clipped to the layer's corner radius (ellipses become circles). |

#### Rendering

| Key | Default | Description |
|---|---|---|
| `erase_placeholders` | `true` | Paints a background-coloured box over each mapped text layer before drawing, to hide placeholder text. Causes visible boxes on non-uniform backgrounds — the cleaner approach is to set placeholder text layers to **0% opacity in Figma** and set this to `false`. |
| `remove_background` | `false` | Run AI background removal (rembg / U2Net-lite) on mapped photos. Photos that already contain transparency are skipped. |

#### Fonts

| Key | Description |
|---|---|
| `font_map` | `"Figma family name" → "file.ttf"` (relative to `fonts/`). Figma reports weight as a suffix on the family name — map `"Space Grotesk"`, `"Space Grotesk Bold"`, `"Space Grotesk Medium"`, etc. separately. **Use static TTFs, not variable fonts** — Pillow can't load variable fonts. |
| `font_overrides` | Per-layer overrides: `{"Layer Name": {font_path, font_size, color}}`. Use when auto-detection needs a nudge. |

If no mapping matches, the renderer falls back to system fonts (Arial/Helvetica/DejaVu), then Pillow's built-in font.

#### Multi-output

| Key | Description |
|---|---|
| `outputs` | List of outputs (below). When defined, the template-level `airtable_attachment_field` / `figma_*` keys are ignored. |
| `selection_field` | Optional Airtable **multi-select** field naming which outputs to render. Empty selection = nothing runs (with a warning). Unset = all outputs run. |

### Output keys

Each entry in `outputs` renders one image to one attachment field:

| Key | Required | Description |
|---|---|---|
| `name` | ✅ | Label, also used in the uploaded filename. |
| `attachment_field` | ✅ | Airtable attachment field for this output. |
| `figma_file_key`, `figma_frame_node_id`, `figma_export_scale` | ✅ | Default frame for this output. |
| `selection_value` | — | If set, this output only runs when the value is selected in the template's `selection_field`. |
| `field_mapping_overrides`, `image_field_mapping_overrides` | — | Merged **over** the template-level mappings (add or replace entries). |
| variant keys | — | See below. |

### Variants

A variant is an alternative Figma frame with the **same layer names**. Four mechanisms select one, checked in priority order:

1. **`boolean_variant_field`** — if this checkbox field is truthy, use variant `boolean_variant_name` (default `"sponsored"`).
2. **`field_variant_field`** + **`field_variant_map`** — map the field's value to a variant name, e.g. `{"LiveDay LDN": "liveday_ldn"}`.
3. **`auto_variant_on_field`** — if this field has any content (e.g. a second speaker photo), use variant `auto_variant_name` (default `"two_speakers"`).
4. **`variant_field`** — use the field's raw value as the variant name (`"London"` → `variants["London"]`).

Mechanisms 1 and 2 **combine** with the auto-variant: when the auto field also has content, the combined name `"<base>_<auto_variant_name>"` (e.g. `sponsored_two_speakers`) is tried first, then the base name.

Unmatched or missing variants log a warning and fall back to the default frame — a bad value never fails the record.

`variants` is a map of name → `{figma_file_key, figma_frame_node_id, figma_export_scale}`.

Variants work at the template level (single-output mode: `variant_field` + `variants`) and per output (all four mechanisms).

---

### PDF report keys (`pdf_reports`)

A PDF report assembles Figma frames into a single multi-page PDF:
**static pages** (always included, cached across records) → **combo pages** (included conditionally, with data overlay and logo compositing) → **closing page**.

| Key | Required | Description |
|---|---|---|
| `name` | ✅ | Label used in logs. |
| `airtable_base_id`, `airtable_table_name` | ✅ | Source table. |
| `airtable_trigger_field`, `airtable_trigger_value`, `airtable_trigger_reset_value`, `airtable_trigger_pending_value`, `airtable_trigger_working_value` | trigger field ✅ | Same semantics as templates. |
| `attachment_field` | ✅ | Where the merged PDF is uploaded. |
| `figma_file_key` | ✅ | One file containing all report frames. |
| `static_page_ids` | ✅ | Frame node IDs always included first, in order, exported as-is. |
| `closing_page_id` | — | Frame appended as the last page. |
| `logo_attachment_field` | — | Airtable attachment field holding a logo to composite onto every combo page. |
| `logo_layer_name` | — (`"Logo"`) | Figma layer to place the logo on (aspect-fit, centred in the layer's box). |
| `filename_label_field` | — | Airtable field whose (slugified) value prefixes the PDF filename. Falls back to `report`. |
| `filename_suffix` | — (`"Report"`) | Appended to the filename: `<label>-<suffix>.pdf`. |
| `field_groups` | — | Named groups of Airtable fields. A group is **active** when any of its fields is non-empty and non-zero. |
| `combo_pages` | — | Conditional pages, evaluated in order (below). |
| `font_map` | — | Same as templates; used for the PDF text overlay. |

#### Combo pages

| Key | Description |
|---|---|
| `figma_frame_node_id` | The frame to include. |
| `require` | Group names that must ALL be active. |
| `exclude` | Group names that must ALL be inactive. |
| `field_mappings` | ⚠️ **Inverted vs. templates:** `"Figma layer name" → "Airtable field name"`. Duplicate layer names within a frame are disambiguated with an index suffix: `"Leads[0]"`, `"Leads[1]"`. Empty/zero values render as `—`. |

---

## Trigger lifecycle

```
checkbox mode:       ☑ checked ──▶ render ──▶ ☐ unchecked
single-select mode:  "Generate" ──▶ "Pending" ──▶ "Working" ──▶ "Ready for publication"
                                   (optional)    (optional)     (trigger_reset_value)
```

The poller queries each table with `filterByFormula` every `POLL_INTERVAL` seconds (max 50 records per poll per template). If any output of a multi-output template fails, the others still render, the trigger is still reset, and the failure is logged.
