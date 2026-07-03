# Hardening Plan

The honest gap list between "well-built internal tool" and "infrastructure strangers can trust," in priority order. Each item has a concrete implementation sketch and an acceptance test. P0 before promoting the repo widely; P1 within the month; P2 when demand justifies it.

## P0 — trust and safety

### 1. Error writeback to Airtable

**Problem.** When a record fails, the error exists only in server logs. The Airtable-side user sees a trigger that hangs or silently resets — invisible failure for exactly the non-engineers this tool is for.

**Implementation.**
- Add `error_field: str = ""` to `TemplateConfig` and `PdfReportConfig` (`template.py`).
- In `run_pipeline` / `run_pdf_report_pipeline` (`pipeline.py`): on any output failure, PATCH `error_field` with a one-line summary (`"OpenGraph failed: Figma layer 'Speaker Name' not found"`), truncated to ~500 chars. On full success, clear the field.
- When a single-select trigger is used and outputs failed, set the trigger to `airtable_trigger_error_value` (new key, e.g. `"Error"`) instead of the reset value, so failed records are visually distinct and retriable.
- Document in `configuration.md`; add to `templates.yaml.example`.

**Acceptance.** Break a layer name on a test template → the record shows the error message and an `Error` status within one poll cycle; fix the layer → re-trigger → error field clears.

**Effort.** ~half a day. Highest value-per-line in this plan.

### 2. Test suite

**Problem.** Zero tests. The most fragile logic — four variant mechanisms with priority ordering and combined-name composition — has no safety net. Contributors can't touch `template.py` responsibly.

**Implementation.**
- `pytest` + `pytest-cov` as dev dependencies; `tests/` package.
- **Unit tests (pure, no network):**
  - Variant resolution matrix: boolean/field/auto/manual priority, combined names (`sponsored_two_speakers`), fallback-on-missing behavior. Table-driven — one test, ~20 cases.
  - `selected_outputs()`: multi-select as list vs comma string, empty selection, unset field.
  - `resolve_field_mappings()` override merging.
  - `load_templates` / `load_pdf_reports` against `templates.yaml.example` (already validated manually every release — automate it).
  - Node-ID normalisation, `_parse_layer_key` index notation, combo-page require/exclude logic, `_field_has_value` edge cases.
- **Golden-image render test:** fixture Figma frame data (checked-in JSON of text/image nodes) + fixture base image + fake record → `build_jpeg` → compare against a checked-in golden JPEG with a perceptual threshold (e.g. mean pixel delta < 2%), not byte equality — font rasterisation varies slightly across platforms.
- CI: GitHub Actions workflow running `ruff check` + `pytest` on PRs.

**Acceptance.** `pytest` green locally and in CI; a deliberate priority-order bug in `_process_output` fails the matrix test.

**Effort.** 1–2 days for the unit layer; golden-image adds another day.

### 3. Make the public upload fallback opt-in

**Problem.** When the Airtable Content API fails, images are silently uploaded to imgbb (and transfer.sh) — public third-party hosts. Fine for public marketing assets; a surprising data-exfiltration path for anything private (sponsor reports with lead numbers). transfer.sh is also effectively dead.

**Implementation.**
- Delete the transfer.sh, catbox, and telegraph fallbacks from `airtable_client.py` (dead code — only imgbb is reachable anyway).
- imgbb becomes explicit: only used when `AIRTABLE_IMGBB_API_KEY` is set **and** new setting `ALLOW_PUBLIC_IMAGE_HOST=true`. Without both, Content API failure raises with a clear message explaining the trade-off.
- Log a WARNING every time a public host is used, naming the host.
- Document the privacy implication in `configuration.md` and `.env.example`.

**Acceptance.** With no flag set and the Content API mocked to fail, the pipeline errors loudly (and, with item 1, writes the error to Airtable) instead of uploading anywhere public.

**Effort.** ~2 hours.

## P1 — reliability and adoption

### 4. Retry queue with backoff

Failed records currently need a human to notice and re-trigger. Track `{record_id: (attempt, next_retry_at)}` in memory; retry at 2m / 10m / 30m; after final failure, write the error field (item 1) and stop. Skip records already in the queue during polling so they aren't double-processed.

**Acceptance.** Kill the network mid-render → record retries automatically and succeeds; three consecutive failures → error field set, no further retries.

### 5. Demo kit — cut time-to-first-render from ~45 min to ~5

The biggest adoption blocker: a new user needs a base, a Figma file, two tokens, and correctly named layers before seeing *anything*.

- Publish a **Figma community file** with one ready-made Speaker Card template (named layers, 0%-opacity placeholders, Space Grotesk).
- Publish an **Airtable base template** (Speakers table with trigger/attachment fields and two sample records with headshots).
- `templates-demo.yaml` in the repo pre-wired to both — user duplicates each, pastes two IDs, runs `--run-once`.
- Add a "5-minute demo" section at the top of the README; teach `AGENTS.md` to offer the demo kit path first.

**Acceptance.** A fresh user (or their agent) goes from clone to a rendered JPEG in one sitting using only duplicated templates.

### 6. Webhook triggers (keep polling as fallback)

Airtable webhooks → a small FastAPI `/webhook` endpoint → dispatch to the pipeline; poll loop stays as a catch-all at a longer interval. Cuts trigger latency from 0–30s to ~1s and drops steady-state API usage. Requires exposing an HTTP service in `fly.toml` (currently a background worker).

### 7. Figma cache invalidation

Frame assets cache for process lifetime; design edits require a restart. Poll each cached file's `last_modified` (one cheap API call) every N minutes and evict changed entries. Kills the "restart the machine after editing Figma" ritual documented in the deployment guide.

### 8. Text fidelity: warn, fit, and document

The Pillow renderer will never be pixel-identical to Figma. Manage the expectation instead of losing the argument one issue at a time:

- **Auto-fit:** shrink font size stepwise (PIL `textbbox`) when a value exceeds its node width — the most common visible defect.
- **Warn on unsupported features** when reading the frame: gradient text fills, strokes on text, auto-resize text nodes, rotated nodes — log exactly what will differ.
- **`docs/rendering-limits.md`:** a short honest page on what reproduces faithfully and what doesn't, linked from the README and the issue template.

## P2 — scale and product

### 9. Dry-run mode

`--dry-run`: validate every template — Figma node IDs resolve, all mapped layers exist in the frame, all Airtable fields exist on a sample record, fonts in `font_map` are present — and print a report without rendering. Pairs perfectly with the AGENTS.md setup flow.

### 10. Parallel record processing

Records process serially; one slow Figma export blocks the batch. `ThreadPoolExecutor`, concurrency 3–5 (Figma rate limits), per-record error isolation already exists.

### 11. S3 / R2 upload target

Airtable attachments aren't publicly linkable and have size limits. `upload_target: airtable | s3 | r2` per output; store the public URL in a URL field. Unlocks direct embedding of OG images from records.

### 12. Positioning decision (not code)

The repo says *tool*, the landing page says *product*, the case study says *consulting lead-gen*. All fine for launch week — but within a month, pick the primary:

| Path | What it implies building next |
|---|---|
| OSS tool | Demo kit, docs, community issues — items 5, 8, 9 |
| Hosted SaaS | Multi-tenancy, auth, billing, webhooks — item 6 first, then a control plane |
| Case study / services | Nothing here; effort goes to content and the PlatformEngineering.org funnel |

The plan above is sequenced to keep all three doors open through P1.

---

## Already fixed (for the record)

- ✅ Hardcoded PDF filename → `filename_label_field` / `filename_suffix` config
- ✅ Missing `PdfPage` import; all ruff findings
- ✅ Secrets/gitignore hygiene; public repo has no internal IDs
- ✅ Landing page: OG meta + image, real production numbers, case study linked, CTA de-duplicated, contrast raised to WCAG AA
