# AGENTS.md — Design Clanker setup playbook

You are helping a user set up **Design Clanker**: a Python poller that watches Airtable trigger fields, renders Figma template frames with live record data, and uploads the finished JPEG/PDF back to the record.

Your job is to make setup **conversational**: interview the user for the few things only they know, discover everything else through the Airtable and Figma APIs, write the config files yourself, verify with a real test record, and (if they want) deploy to Fly.io. The user should never have to hand-edit YAML or hunt for IDs in URLs.

## Project map

```
src/airtable_to_figma/   the package — poller, pipeline, clients, renderer
templates.yaml           user's config (gitignored; you will create it)
templates.yaml.example   annotated sample of every feature
.env                     secrets (gitignored; you will create it)
fonts/                   TTF files referenced by font_map
docs/configuration.md    every config key, variant rules, trigger lifecycle
docs/figma-templates.md  how the Figma file must be structured
docs/deployment.md       Docker / Fly.io / GitHub Actions
```

Run modes: `uv run airtable-to-figma` (poll loop) · `uv run airtable-to-figma --run-once` (single pass — use this for testing).

## Ground rules

- **Never print, echo, or commit secrets.** Tokens go straight into `.env`. Both `.env` and `templates.yaml` are gitignored — keep it that way unless the user explicitly wants their (private) fork to carry `templates.yaml` (`git add -f`).
- Validate token shape early: Airtable tokens start with `pat`, Figma tokens with `figd_`. The app refuses to start otherwise.
- Prefer **discovering via API** over asking the user for IDs. Ask for choices ("which base? which table?"), not for strings like `appXXXXXXXXXXXXXX`.
- After every config change, run the validation snippet (below) before running the app.
- Work through steps one at a time and confirm before moving on; don't dump the whole interview at once.

## Setup interview

### 0 · Prerequisites

Check `python3 --version` (needs 3.12+) and `uv --version`. Install deps with `uv sync` (fallback: `pip install -e .`).

### 1 · Credentials

Ask the user to create two tokens (send them these exact links, then have them paste tokens directly to you or into `.env` themselves):

- **Airtable**: <https://airtable.com/create/tokens> — scopes `data.records:read`, `data.records:write`, `schema.bases:read` (this last one lets you discover their tables and fields for them), with their base(s) added to the token's access list. If you'll need to create missing fields (step 3), also request `schema.bases:write`.
- **Figma**: Figma → Settings → Security → Personal access tokens — File content **read** is enough.
- Optional: **imgbb** key from <https://api.imgbb.com/> — fallback upload host; skip unless uploads fail later.

Write `.env` (copy `.env.example`):

```
AIRTABLE_API_KEY=...
FIGMA_API_KEY=...
AIRTABLE_IMGBB_API_KEY=
POLL_INTERVAL=30
```

### 2 · Discover their Airtable

Don't ask for base IDs — list what the token can see and let them pick:

```bash
# List bases (name + id)
curl -s -H "Authorization: Bearer $AIRTABLE_API_KEY" \
  https://api.airtable.com/v0/meta/bases | python3 -m json.tool

# List tables + fields (names, types) for the chosen base
curl -s -H "Authorization: Bearer $AIRTABLE_API_KEY" \
  https://api.airtable.com/v0/meta/bases/BASE_ID/tables | python3 -m json.tool
```

From the chosen table's field list, identify:

- **Trigger field** — a `checkbox` (simplest) or `singleSelect`. For single-select, also collect the option names to use for `airtable_trigger_value` / `_pending_value` / `_working_value` / `_reset_value` — they must match the select options **exactly**.
- **Attachment field(s)** — type `multipleAttachments`, one per output; this is where the finished design lands.
- **Data fields** — the text fields to render, and any attachment fields holding photos.

### 3 · Missing fields

If there's no trigger or output field, either walk the user through adding them in the Airtable UI, or create them yourself (needs `schema.bases:write`):

```bash
curl -s -X POST -H "Authorization: Bearer $AIRTABLE_API_KEY" -H "Content-Type: application/json" \
  https://api.airtable.com/v0/meta/bases/BASE_ID/tables/TABLE_ID/fields \
  -d '{"name": "Ready for Design", "type": "checkbox", "options": {"icon": "check", "color": "greenBright"}}'

curl -s -X POST -H "Authorization: Bearer $AIRTABLE_API_KEY" -H "Content-Type: application/json" \
  https://api.airtable.com/v0/meta/bases/BASE_ID/tables/TABLE_ID/fields \
  -d '{"name": "Generated Design", "type": "multipleAttachments"}'
```

### 4 · Discover their Figma template

Ask only for the **Figma file URL** (any link to the file). Extract the file key: `figma.com/design/<FILE_KEY>/...`. Then discover the frames and layers yourself:

```bash
# Top-level pages and frames (id + name) — enough to ask "which frame is the template?"
curl -s -H "X-Figma-Token: $FIGMA_API_KEY" \
  "https://api.figma.com/v1/files/FILE_KEY?depth=2" \
  | python3 -c "
import json,sys
doc=json.load(sys.stdin)['document']
for page in doc['children']:
    for f in page.get('children',[]):
        print(f\"{f['id']:>10}  {f['type']:<10} {page['name']} / {f['name']}\")"

# All TEXT layers + image placeholders inside the chosen frame (uses the project's own client)
PYTHONPATH=src uv run python -c "
from airtable_to_figma.figma_client import FigmaClient
import os
c = FigmaClient(api_key=os.environ['FIGMA_API_KEY'], file_key='FILE_KEY')
print('TEXT LAYERS:');  [print(' ', n['name'], '| font:', n['font_family'], n['font_weight']) for n in c.get_text_nodes('NODE_ID')]
print('IMAGE SLOTS:'); [print(' ', n['name'], '|', n['shape']) for n in c.get_image_nodes('NODE_ID') if n['name']]"
```

(Export `AIRTABLE_API_KEY`/`FIGMA_API_KEY` into the shell from `.env` for these one-offs, or use `uv run --env-file .env`.)

Now **propose the mappings**: match Airtable field names to Figma layer names by meaning (e.g. `Full Name` → `Speaker Name`, `Headshot` → `Photo`) and show the user the proposed table for confirmation. Flag Figma layers that got no mapping and Airtable fields that look renderable but unmapped.

Also check from the text-node output which **font families/weights** the frame uses. For each one not already in `fonts/` + covered by `font_map`, tell the user to download the **static** TTFs (Google Fonts → Download → `static/` folder — variable fonts don't work with Pillow) and drop them in `fonts/`. Space Grotesk (Light→Bold) ships with the repo.

Remind them of the one Figma-side change that most improves quality: set placeholder text layers to **0% opacity** and use `erase_placeholders: false`. (Details in `docs/figma-templates.md`.)

### 5 · Write and validate `templates.yaml`

Write the file yourself from the interview answers — start from the "Speaker Card" shape in `templates.yaml.example` and only add features they need (multi-output, variants, PDF reports — see `docs/configuration.md`). Then validate:

```bash
uv run python -c "
from airtable_to_figma.template import load_templates, load_pdf_reports
load_templates('templates.yaml'); load_pdf_reports('templates.yaml')
print('templates.yaml is valid')"
```

### 6 · End-to-end test

1. Pick (or have the user pick) a real record with data in the mapped fields, and set its trigger (check the box / set the select to the trigger value). You can do this via API:
   ```bash
   curl -s -X PATCH -H "Authorization: Bearer $AIRTABLE_API_KEY" -H "Content-Type: application/json" \
     "https://api.airtable.com/v0/BASE_ID/TABLE_NAME/RECORD_ID" \
     -d '{"fields": {"Ready for Design": true}}'
   ```
2. Run `uv run airtable-to-figma --run-once` and read the logs with the user.
3. Success looks like: `Found 1 record(s)` → `Exported frame … size=…` → `Rendered JPEG size=…` → `✓ Uploaded`. Have the user open the record — the design should be in the attachment field and the trigger reset.
4. If the render looks wrong (font, position, missed layer), fix `font_map` / layer names / mappings and re-trigger. Common errors are tabled in `docs/deployment.md`.

### 7 · Deploy (offer, don't assume)

Ask how they want to run it:

- **Fly.io** (always-on, ~30s latency): needs `flyctl` and a Fly account.
  ```bash
  fly launch --no-deploy        # accept the existing fly.toml; pick an app name + region
  fly secrets set AIRTABLE_API_KEY=... FIGMA_API_KEY=... AIRTABLE_IMGBB_API_KEY=...
  fly deploy
  fly logs                      # confirm 'Loaded N template(s)' and a clean poll
  ```
  Notes: the Docker build COPYs `templates.yaml` from the local directory (works fine even though it's gitignored — Fly builds from the working dir, not git). Keep the VM at **2 GB** if `remove_background: true` (the model OOMs 1 GB machines). After any `templates.yaml` change: `fly deploy` again.
- **GitHub Actions cron** (free, no server, 5–15 min latency): copy the workflow from `docs/deployment.md`, add the two tokens as repo secrets. ⚠️ Since `templates.yaml` is gitignored, this path needs it committed — only appropriate in a **private** fork.
- **Local / other**: `uv run airtable-to-figma` under any process manager; Docker instructions in `docs/deployment.md`.

### 8 · Wrap up

Tell the user how to extend from here: add more templates to `templates.yaml` (multi-output, variants), the config reference is `docs/configuration.md`, and re-run step 6 whenever they add one. If they change the Figma design later, the poller must be restarted/redeployed — frame exports are cached per process.

## Gotchas you should know before debugging

- Trigger single-select values must match Airtable options **exactly** (`"Ready for publication"` ≠ `"Ready for publishing"` — this has bitten before).
- Table names are case-sensitive; node IDs accept `12-345` or `12:345`.
- A single-select trigger with no `airtable_trigger_reset_value` gets re-processed every poll.
- Figma exports can 400 under load; the client already retries 4× with backoff — don't add your own retry loop.
- The pipeline caches Figma assets per `(file_key, node_id)` per process; stale designs mean the process needs a restart, not a config change.
- Attachment upload tries Airtable's Content API first, then imgbb (if key set), then transfer.sh. If uploads fail, the fix is almost always adding `AIRTABLE_IMGBB_API_KEY`.
- Lint: `uvx ruff check src/` must pass if you touch code (line length 100).
