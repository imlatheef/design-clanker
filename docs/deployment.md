# Deployment guide

Design Clanker is a single long-running process (or a one-shot cron job) — no database, no queue, no inbound traffic. State lives entirely in Airtable, so restarts are always safe: an interrupted record just gets picked up on the next poll.

## Local

```bash
uv sync
cp .env.example .env            # fill in tokens
cp templates.yaml.example templates.yaml
uv run airtable-to-figma        # poll loop
uv run airtable-to-figma --run-once   # single pass (great for testing a template)
```

Without uv: `pip install -e .` then `airtable-to-figma`.

## Docker

The multi-stage `Dockerfile` produces a slim runtime image with the rembg background-removal model (~170 MB) **pre-baked at build time**, so containers start instantly and never download at runtime.

```bash
docker build -t design-clanker .
docker run --env-file .env design-clanker
```

Note: `templates.yaml` is copied into the image at build time — rebuild after config changes, or bind-mount it for faster iteration:

```bash
docker run --env-file .env -v ./templates.yaml:/app/templates.yaml design-clanker
```

## Fly.io

The included `fly.toml` runs the poller as a background worker (no HTTP service). The rembg model needs memory: **2 GB is the tested size**; 1 GB machines get OOM-killed when background removal runs.

```bash
fly launch --no-deploy          # creates the app, keeps fly.toml
fly secrets set AIRTABLE_API_KEY=pat... FIGMA_API_KEY=figd_... AIRTABLE_IMGBB_API_KEY=...
fly deploy
fly logs                        # watch it work
```

Deploys after any code or template change:

```bash
git commit -am "Update templates" && fly deploy
```

The poller handles SIGTERM gracefully, so Fly deploys/stops are clean.

## GitHub Actions (serverless cron)

No server at all: run a single poll pass on a schedule. Latency = your cron interval (GitHub schedules are best-effort, typically 5–15 min).

```yaml
# .github/workflows/poll.yml
name: Design Clanker poll
on:
  schedule:
    - cron: "*/10 * * * *"
  workflow_dispatch: {}       # manual trigger button

jobs:
  poll:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync
      - run: uv run airtable-to-figma --run-once
        env:
          AIRTABLE_API_KEY: ${{ secrets.AIRTABLE_API_KEY }}
          FIGMA_API_KEY: ${{ secrets.FIGMA_API_KEY }}
          AIRTABLE_IMGBB_API_KEY: ${{ secrets.AIRTABLE_IMGBB_API_KEY }}
```

Heads-up: with `remove_background: true`, the first run downloads the U2Net model each time (no cache between runs) — add an `actions/cache` step for `~/.u2net` if that gets slow.

## Sizing & limits

| Concern | Detail |
|---|---|
| Memory | ~256 MB base; **~1.5–2 GB** with background removal (U2Net-lite). |
| Airtable API | One `filterByFormula` list call per template per poll + a few calls per processed record. Airtable's limit is 5 req/s per base — fine at default settings. |
| Figma API | Frame exports are the slow call (seconds for big frames) and are cached per `(file, frame)` for the process lifetime. Exports retry up to 4× with backoff when Figma's render queue is busy. |
| Poll frequency | `POLL_INTERVAL` (default 30 s). Records are claimed at up to 50 per template per poll. |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `AIRTABLE_API_KEY must start with 'pat'` | You're using a legacy API key — create a personal access token. |
| `Figma returned no image URL for node …` | Wrong `figma_frame_node_id`, or the token lacks access to the file. Use `12:345` form (dashes are auto-normalised). |
| Text renders in the wrong font | The Figma family+weight name isn't in `font_map`, or the TTF is a variable font — use static files. |
| Attachment upload fails | Set `AIRTABLE_IMGBB_API_KEY` — the Content API isn't available on all plans; imgbb is the fallback host. |
| Record processed repeatedly | Single-select trigger with no `airtable_trigger_reset_value` — set one that differs from the trigger value. |
| OOM on small machines | Background removal needs ~2 GB, or set `remove_background: false`. |
| Design changes not showing | Frame assets are cached per process — restart/redeploy after editing the Figma file. |
