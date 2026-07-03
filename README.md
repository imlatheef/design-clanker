# 🤖 Design Clanker

**The tireless design robot.** Design Clanker watches your Airtable base, and every time a record is ready it renders a pixel-perfect image from your Figma template — real data, real fonts, real photos — and uploads it straight back to the record. No exports, no copy-paste, no designer on call at 2am.

Check a box in Airtable → get finished social cards, OG images, speaker graphics, and multi-page PDF reports seconds later.

```
        ┌────────────┐        ┌────────────┐        ┌────────────┐
        │  Airtable  │  poll  │   Design   │ export │   Figma    │
        │   record   │───────▶│  Clanker   │◀───────│  template  │
        └────────────┘        └─────┬──────┘        └────────────┘
              ▲                     │
              │   upload JPG/PDF    │  overlay text, photos,
              └─────────────────────┘  logos onto the frame
```

## Features

- **Config, not code** — every workflow lives in `templates.yaml`. Map Airtable fields to Figma layer names and you're done.
- **Multi-output templates** — one trigger renders many assets (OG image + LinkedIn card + speaker social) into separate attachment fields.
- **Smart variants** — switch Figma frames automatically based on record data: a checkbox (`Is Sponsored` → sponsored layout), a select value (`Location` → London/NYC/Virtual designs), or field presence (second speaker photo filled → two-speaker layout). Variants compose, so `sponsored` + `two_speakers` resolves to `sponsored_two_speakers`.
- **True-to-design text** — reads position, size, font family/weight, color, and alignment straight from the Figma file and re-renders text with your actual TTF fonts.
- **Photo compositing** — pulls attachments from Airtable, optionally removes backgrounds with AI (rembg/U2Net), and clips to the exact corner radius (or circle) defined in Figma.
- **Multi-page PDF reports** — assemble static pages, data-overlay pages, and conditionally included pages (based on which field groups have data) into one branded PDF. Built for sponsor recap reports.
- **Status progression** — optionally writes `Pending` → `Working` → `Done` back to the trigger field so your team can watch progress inside Airtable.
- **Ops-friendly** — resilient poll loop, Figma export retries with backoff, per-frame asset caching, graceful SIGTERM handling, colorized logs, and a `--run-once` mode for cron/GitHub Actions.

## Quick start

Requires Python 3.12+ and [uv](https://docs.astral.sh/uv/) (or plain pip).

```bash
git clone https://github.com/imlatheef/design-clanker
cd design-clanker
uv sync

# 1. Credentials
cp .env.example .env          # add your Airtable + Figma tokens

# 2. Templates
cp templates.yaml.example templates.yaml   # map your fields to your layers

# 3. Run
uv run airtable-to-figma                   # polls every 30s
uv run airtable-to-figma --run-once        # single pass, e.g. for cron
```

You'll need:

| Credential | Where to get it |
|---|---|
| `AIRTABLE_API_KEY` | [airtable.com/create/tokens](https://airtable.com/create/tokens) — scopes `data.records:read` + `data.records:write`, with your base added |
| `FIGMA_API_KEY` | Figma → Account Settings → Personal Access Tokens |
| `AIRTABLE_IMGBB_API_KEY` (optional) | [imgbb.com/api](https://imgbb.com/api) — fallback image host if the Airtable Content API isn't available on your plan |

## Your first template

1. **Design a frame in Figma.** Name the layers you want filled (`Title`, `Speaker Name`, `Photo`, …). See the [Figma template guide](docs/figma-templates.md) for tips (0%-opacity placeholders, static fonts, node IDs).
2. **Add a trigger field in Airtable** — a checkbox like `Ready for Design`, or a single-select status field.
3. **Describe the workflow in `templates.yaml`:**

```yaml
templates:
  - name: "Speaker Card"
    airtable_base_id:          "appXXXXXXXXXXXXXX"
    airtable_table_name:       "Speakers"
    airtable_trigger_field:    "Ready for Design"
    airtable_attachment_field: "Generated Design"

    figma_file_key:      "AbCdEfGhIjKlMnOp"
    figma_frame_node_id: "12:345"

    field_mappings:            # Airtable field → Figma layer
      "Full Name": "Speaker Name"
      "Talk Title": "Title"
    image_field_mappings:      # Airtable attachment → Figma layer
      "Headshot": "Photo"
```

4. **Run the poller and check the box.** Seconds later the finished JPEG lands in `Generated Design`, and the checkbox is unchecked so it won't run twice.

That's the simple case. `templates.yaml.example` walks through every feature — multi-output, all four variant mechanisms, fonts, background removal, and PDF reports — and the [configuration reference](docs/configuration.md) documents every key.

## Documentation

| Doc | What's in it |
|---|---|
| [Configuration reference](docs/configuration.md) | Every key in `templates.yaml` and `.env`, with the variant-resolution rules |
| [Figma template guide](docs/figma-templates.md) | How to structure Figma files so they render perfectly |
| [Deployment guide](docs/deployment.md) | Docker, Fly.io, and GitHub Actions cron setups |
| [templates.yaml.example](templates.yaml.example) | Annotated sample config covering every feature |
| [ROADMAP.md](ROADMAP.md) | Where this is headed |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Dev setup and how to send changes |

## How it works

```
poller.py            polls each template's table with a filterByFormula
  └─ pipeline.py     per record: resolve variant → fetch Figma assets (cached)
       ├─ figma_client.py     export frame image/PDF, read text & image nodes
       ├─ image_renderer.py   Pillow overlay: erase placeholders, draw text,
       │                      composite photos with radius clipping
       ├─ background_remover.py  rembg/U2Net, skips already-transparent photos
       └─ airtable_client.py  upload attachment (Content API → imgbb fallback)
```

Figma's REST API can't edit files, so Design Clanker exports the frame as a base image, reads the text/image node geometry from the file JSON, and redraws your data on top with Pillow — matching font, size, color, and alignment. Frame assets are cached per `(file, frame)` so a batch of 50 records hits Figma once.

## Deployment

Runs anywhere Python runs. The included `Dockerfile` builds a slim image with the background-removal model pre-baked, and `fly.toml` deploys it as a Fly.io background worker:

```bash
fly launch && fly secrets set AIRTABLE_API_KEY=pat... FIGMA_API_KEY=figd_...
fly deploy
```

Prefer zero servers? Run `--run-once` on a GitHub Actions schedule. Details in the [deployment guide](docs/deployment.md).

## Contributing

PRs welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). The [roadmap](ROADMAP.md) has a prioritized list of good first projects (error writeback to Airtable, text auto-fit, webhook triggers).

## License

[MIT](LICENSE)
