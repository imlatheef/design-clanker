# Changelog

## [0.1.0] — 2026-07-03

Initial open-source release. 🤖

### Features
- **Poll-based pipeline** — watches Airtable trigger fields (checkbox or single-select), renders Figma templates with live record data, uploads results back to attachment fields.
- **Multi-output templates** — one trigger renders multiple assets, each to its own attachment field, optionally filtered by a multi-select field.
- **Variant selection** — four mechanisms (boolean field, field-value map, field presence, manual select), composable (e.g. `sponsored_two_speakers`).
- **True-to-design text rendering** — position, size, font family/weight, color, and alignment read from the Figma file, drawn with your own TTF fonts.
- **Photo compositing** — attachment download, optional AI background removal (rembg/U2Net-lite), corner-radius and circular clipping matched to Figma.
- **Multi-page PDF reports** — static pages, data-overlay pages, conditional pages driven by field-group presence, logo compositing, configurable output filename.
- **Status progression** — optional Pending → Working → Done writeback to the trigger field.
- **Ops** — `--run-once` mode for cron, Figma export retries with backoff, per-frame asset caching, graceful SIGTERM handling, Dockerfile with the background-removal model pre-baked, Fly.io config.
