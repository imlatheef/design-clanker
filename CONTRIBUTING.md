# Contributing to Design Clanker

Thanks for helping make the design robot better. Issues, docs fixes, and PRs are all welcome.

## Dev setup

Requires Python 3.12+ and [uv](https://docs.astral.sh/uv/).

```bash
git clone https://github.com/imlatheef/design-clanker
cd design-clanker
uv sync

cp .env.example .env                       # your Airtable + Figma tokens
cp templates.yaml.example templates.yaml   # point at a test base + test Figma file
```

For development you'll want a **scratch Airtable base** and a **scratch Figma file** of your own — any base with a checkbox trigger field and an attachment field, and any Figma frame with a couple of named text layers, is enough.

## Running

```bash
uv run airtable-to-figma --run-once   # single pass — the main dev loop
uv run airtable-to-figma              # full poll loop
```

Validate a config change without touching any APIs:

```bash
uv run python -c "
from airtable_to_figma.template import load_templates, load_pdf_reports
load_templates('templates.yaml'); load_pdf_reports('templates.yaml')"
```

## Code layout

```
src/airtable_to_figma/
├── poller.py             entry point + poll loop
├── pipeline.py           per-record orchestration (images + PDF reports)
├── template.py           pydantic config models + variant resolution
├── settings.py           .env credentials (pydantic-settings)
├── figma_client.py       Figma REST: export frames, read node geometry
├── airtable_client.py    Airtable REST: records + attachment upload
├── image_renderer.py     Pillow text/photo overlay engine
└── background_remover.py rembg wrapper with skip heuristics
```

Rough shape of a change: config keys go in `template.py` (pydantic models — validation is free), behavior goes in `pipeline.py` or `image_renderer.py`, and every new key gets documented in `docs/configuration.md` and demonstrated in `templates.yaml.example`.

## Style

- `ruff check .` must pass (line length 100, isort rules included).
- Type hints everywhere; the package ships `py.typed`.
- Log generously — this runs unattended, logs are the only debugging surface. Follow the existing `[Template › Output]` prefix convention.
- Config failures should fail fast with a message that says how to fix them; per-record failures should log and never kill the poll loop.

## Pull requests

1. Fork, branch from `main`.
2. Keep PRs focused — one feature or fix per PR.
3. Update `CHANGELOG.md` under `[Unreleased]`.
4. If you add or change config keys: update `docs/configuration.md` + `templates.yaml.example`, and make sure the example still validates (command above, pointed at `templates.yaml.example`).

Looking for something to build? [ROADMAP.md](ROADMAP.md) has a prioritized list — error writeback to Airtable, text auto-fit, and webhook triggers are the most wanted.
