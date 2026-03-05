# Agent Browser Workspace

**Local deep research + browser use for any agent.**  
Run everything on your machine with a real Chrome profile (CDP) — no Docker, no remote browsers, no platform lock-in.

This repo is a small, production-ready toolkit for agent-driven browsing and reproducible deep research:
control Chrome, extract content/forms, save pages to Markdown (with images), and automate Google Search — all as **CLI scripts** and **Node.js APIs**.

> “Local-first” here means: your browsing session lives in an isolated Chrome profile on your computer.

> **CLI-first.** For agent tasks, prefer running the existing tools via CLI (`node scripts/...`, `node utils/...`). Avoid writing new JavaScript scripts; only do it as a last resort when the CLI tools cannot perform a required step.

## Why it competes

- **Local-first browser automation**: real Google Chrome + isolated profile (`AgentProfile`) + CDP.
- **Extensible by design**: add/maintain site-specific selectors & controls via `scripts/sites/*.json` (no hardcoding in prompts or code).
- **Reproducible deep research workflow**: stable SERP snapshots (`links.json`), downloaded sources in Markdown, and a disciplined research loop.
- **Cost-efficient deep research**: strong DeepResearch Bench results with a much cheaper model tier.

## Features

- **Browser control (low level)**: `utils/browserUse.js`
  - `launchCDP` (recommended), `launch` (persistent), `connectCDP`
  - navigation, clicks, form fill, scrolling (incl. infinite scroll), screenshots
  - download images/files, evaluate JS, basic network tools
  - **PDF auto-handling**: extract text even when Chrome PDF extensions intercept navigation
- **HTML → structured data (no browser)**: `utils/getDataFromText.js`
  - extract navigation blocks, main content blocks (with Markdown), and forms (classified)
- **High-level scripts (CLI + API)**:
  - `scripts/getContent.js` — save current/target page to Markdown + download images + rewrite links
  - `scripts/getForms.js` — detect forms and generate ready-to-use CSS selectors for fields
  - `scripts/getAll.js` — content + forms in one call (single HTML snapshot)
  - `scripts/googleSearch.js` — step-by-step Google Search: search → collect links → open → extract → paginate
- **Site profiles** (`scripts/sites/*.json`)
  - central place to maintain selectors and “controls” (actionable UI elements)
  - scripts return a `site` field so an agent can reuse selectors without guessing

## DeepResearch Bench results (cost-efficient)

We use **DeepResearch Bench (DRB)** as the main external benchmark for deep research agents ([official site](https://deepresearch-bench.github.io/), [repo](https://github.com/Ayanami0730/deep_research_bench), [leaderboard](https://huggingface.co/spaces/Ayanami0730/DeepResearch-Leaderboard)).

As measured on DRB (RACE overall), the reference points are:

| System | DRB overall |
|---|---:|
| Gemini-2.5-Pro Deep Research | **48.88** |
| OpenAI Deep Research | **46.98** |
| **Agent Browser Workspace (this repo)** | **44.37** *(submission pending review)* |
| Perplexity Deep Research | **42.25** |

**Important context:**
- Our **44.37** result was produced using **Claude Haiku 4.5** (significantly cheaper than frontier “deep research” stacks) and a **Cursor-based agent**. Results were submitted to the leaderboard and are currently under review.
- Claude Haiku 4.5 is designed for low-latency, cost-effective agentic workloads (see [Anthropic announcement](https://www.anthropic.com/news/claude-haiku-4-5)).

## Install (local)

### Requirements

- Node.js 18+ (20+ recommended)
- Google Chrome (desktop)

### Setup

```bash
npm install
npx playwright install chrome
```

### Start/stop background Chrome (recommended)

```bash
npm run chrome
```

Stop:

```bash
npm run chrome:stop
```

For full setup details (profiles, Windows shortcut, verification), see [INSTALLATION.md](INSTALLATION.md).

## Quickstart (CLI)

### 1) Save a page to Markdown (with images)

```bash
node scripts/getContent.js --url https://example.com --dir ./output --name page.md
```

### 2) Extract content + forms in one pass

```bash
node scripts/getAll.js --url https://example.com --dir ./output --name page.md --forms-output ./output/forms.json
```

### 3) Deep research loop via Google Search

```bash
# Save a stable SERP snapshot (links.json)
node scripts/googleSearch.js "playwright tutorial" --links --dir ./archive/my-research

# Open a result and save content
node scripts/googleSearch.js "playwright tutorial" --open 0 --dir ./archive/my-research --name article.md
```

`links.json` is intentionally stable: you can resume research later without re-running Google Search.

## Usage (advanced: Node.js API)

The Node.js API exists primarily for **authoring/extending tools**. For normal tasks, prefer the CLI quickstart above.

- Low-level browser control: see [`utils/browserUse.md`](utils/browserUse.md)
- Google Search + content extraction: see [`scripts/googleSearch.md`](scripts/googleSearch.md)

## Extending: add a site profile

Site profiles live in `scripts/sites/*.json`. A profile can define:
- `scraping.selectors` — named selectors used by scripts
- `controls.items` — “UI controls” (selectorKey + human-readable actions) that scripts expose back as `site.controls`

Minimal example:

```json
{
  "id": "my-site",
  "name": "My Site",
  "hosts": ["example.com", "www.example.com"],
  "scraping": {
    "selectors": {
      "searchInput": "input[name=\"q\"]"
    }
  },
  "controls": {
    "items": [
      {
        "name": "Search input",
        "selectorKey": "searchInput",
        "description": "Main search field",
        "actions": ["fill", "press:Enter"]
      }
    ]
  }
}
```

When the page host matches a profile, scripts (`getContent`, `getForms`, `getAll`) return:
`site: { id, name, host, controls[] }` — so your agent can act on the page without guessing selectors.

## Important: single-threaded browser access

Chrome/profile is a shared resource. **Use browser tasks sequentially, from a single agent/process at a time.**  
Parallelize only non-browser work (e.g., parsing raw HTML via `getDataFromText`).

## Docs

- [AGENTS.md](AGENTS.md) — tools overview, usage rules, escalation strategy for JS-heavy pages, PDF handling
- [INSTALLATION.md](INSTALLATION.md) — setup, Chrome profile, QoL shortcuts
- [RESEARCH.md](RESEARCH.md) — deep research methodology (artifacts, phases, quality checklist)
- `scripts/` — high-level scripts documentation:
  - [getContent](scripts/getContent.md)
  - [getForms](scripts/getForms.md)
  - [getAll](scripts/getAll.md)
  - [googleSearch](scripts/googleSearch.md)
- `utils/` — low-level utilities documentation:
  - [browserUse](utils/browserUse.md)
  - [getDataFromText](utils/getDataFromText.md)

## Contributing

PRs are welcome — especially:
- new/updated site profiles in `scripts/sites/`
- extractor improvements for tricky pages (SPA, lazy-load, paywalls/overlays)
- better heuristics for form classification and content block selection

## License

Apache-2.0. See [LICENSE](LICENSE).

