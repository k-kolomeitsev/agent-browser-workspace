# agent-browser-workspace

Local toolkit for working with web pages: control a real Chrome instance, extract structured data from HTML, save pages as Markdown (with downloaded images), and automate Google Search.

## Policy: CLI-first (default)

Prefer invoking existing tools as **CLI commands** from the terminal (for example: `node scripts/getContent.js --dir ./output --name article.md --url https://example.com`).

- Use the CLI tools in `scripts/` and `utils/` first.
- Avoid writing new JavaScript scripts. Only write code when there is no CLI command/flag that can do the job.
- If you do write code, only compose existing modules from this repo (`scripts/*`, `utils/*`) — do not re-implement browser automation.

## When to use

Use the utilities and scripts in this repository when the task involves:

- controlling a browser (navigation, clicks, form filling, screenshots, file downloads);
- extracting content or forms from a web page (live, or from raw HTML);
- saving a page to Markdown with image downloading;
- automating Google Search (collecting links, iterating results).

Do **not** write new browser automation scripts from scratch. Prefer the CLI tools in this repo; use the Node.js APIs only when you need programmatic control.

## IMPORTANT: single-threaded access to the browser

The browser is a shared resource. It must be used **sequentially, from a single agent at a time**. Do not:

- run multiple parallel agents/subagents where each touches the browser;
- call browser scripts (`getContent`, `getForms`, `getAll`, `googleSearch`, `browserUse`) from parallel tasks/subagents;
- connect to the same Chrome instance from multiple processes at the same time.

One Chrome profile = one process, one session, one sequential command stream. If you need multiple browser tasks, run them **strictly sequentially** in the same agent. You can parallelize only work that does not use the browser (e.g. HTML parsing via `getDataFromText`, file processing, etc.).

## Page blockers (CAPTCHA, login, etc.)

If a page requires actions the agent cannot reliably perform automatically (CAPTCHA, login, age confirmation, paywall, a cookie banner that blocks the content), follow one of these modes:

**Default mode** — stop and inform the user:

- Describe what is required (blocker type + page URL).
- Wait until the user completes the action and confirms readiness.
- Continue from where you paused.

**Autonomous mode** — only if the user explicitly requested uninterrupted work (e.g. “handle it yourself”, “don’t stop”, “work autonomously”):

- Try to resolve the issue yourself (close a banner, dismiss an overlay).
- If it doesn’t work — **skip the page** and move on to the next one.
- Do not pause and do not ask for help.

## Structure

```
utils/
  browserUse.js       — low-level Chrome control (Playwright)
  getDataFromText.js  — HTML parsing → navigation, content, forms

scripts/
  _shared.js          — browser init (CDP auto-detect / launch)
  sites/              — site profiles (selectors + controls in JSON)
  getContent.js       — Markdown + images from the active page
  getForms.js         — all page forms with fields and selectors
  getAll.js           — content + forms in one call
  googleSearch.js     — step-by-step Google search
```

## Utilities (low level)

### browserUse — control Chrome

[Docs](utils/browserUse.md) | [Code](utils/browserUse.js)

Use this when you need **direct** browser control: navigation, clicks, form filling, scrolling (including infinite scroll), screenshots, downloading images/files, running JS on the page, tab management. Main modes: `launchCDP` (recommended; background Chrome + CDP), `launch` (persistent profile), and `connectCDP` (attach to an already-running Chrome).

CLI (recommended):

```bash
node utils/browserUse.js https://example.com --html page.html
```

### getDataFromText — parse HTML

[Docs](utils/getDataFromText.md) | [Code](utils/getDataFromText.js)

Use this when you have **raw HTML** (string or file) and need structured blocks extracted from it: navigation, main content (as Markdown), forms with classification. Works without a browser.

CLI:

```bash
node utils/getDataFromText.js page.html output.json
```

## Scripts (high level)

All scripts can be used as a CLI (recommended) and also as API modules. They auto-detect the connection mode: if Chrome is running with CDP — they connect to it; otherwise they launch a new instance.

### getContent — Markdown content with images

[Docs](scripts/getContent.md) | [Code](scripts/getContent.js)

Use when you need to **save the page content as a Markdown file** with downloaded images and rewritten links to local files.

```bash
node scripts/getContent.js --dir ./output --name article.md --url https://example.com
```

### getForms — page forms

[Docs](scripts/getForms.md) | [Code](scripts/getForms.js)

Use when you need to **find and analyze forms** on a page: search, login, filters, contact forms. Returns classification and ready-to-use CSS selectors for fields compatible with `browser.fill()` / `browser.fillForm()`.

```bash
node scripts/getForms.js --url https://example.com --output forms.json
```

### getAll — content + forms

[Docs](scripts/getAll.md) | [Code](scripts/getAll.js)

Use when you need **both content and forms** in a single call. One HTML snapshot, one browser session — both results.

```bash
node scripts/getAll.js --dir ./output --name page.md --forms-output forms.json
```

### googleSearch — Google Search automation

[Docs](scripts/googleSearch.md) | [Code](scripts/googleSearch.js)

Use when you need to **automate Google Search**: find links for a query, iterate results, extract content from discovered pages. Filters ads. Supports pagination.

If `dir` is passed (CLI: `--dir`, API: `linksDir`), `getLinks()` automatically saves results to **`links.json`** in that directory. This is critical for deep research: you can continue later without repeating the search (and without relying on a changing SERP), by following URLs already saved.

```bash
node scripts/googleSearch.js "playwright tutorial" --links
node scripts/googleSearch.js "node.js" --open 0 --dir ./output --name article.md
```

## Deep research via Google

The deep-research methodology is described in [RESEARCH.md](RESEARCH.md).

## Tool selection

| Task | Tool |
| --- | --- |
| Navigation, clicks, scrolling, screenshots, typing | `utils/browserUse` |
| Parse ready HTML without a browser | `utils/getDataFromText` |
| Save a page to Markdown with images | `scripts/getContent` |
| Extract text from a PDF | `scripts/getContent` (auto-detect) |
| Discover what forms exist on a page | `scripts/getForms` |
| Content + forms in one call | `scripts/getAll` |
| Google Search + iterate results | `scripts/googleSearch` |

## Common patterns

All scripts accept an optional `browser` option — an existing `BrowserUse` instance. If provided, it is reused and **not** closed. If not provided, the script creates and closes a browser instance automatically.

Browser initialization across scripts is centralized in [`scripts/_shared`](scripts/_shared.md):

- If Chrome is running with `--remote-debugging-port=9222` — connect via CDP
- Otherwise — auto-start a background Chrome with CDP (`launchCDP`) and connect to it
- CLI flags `--cdp`, `--launch`, `--headless` allow explicit control of the mode

### Site profile enrichment in responses (`site`)

Scripts `getContent`, `getForms`, `getAll` return a `site` field (if the page host matches a profile in `scripts/sites/*.json`).  
If `site` is present — include `site.controls` (selector + description + actions) in your agent/user response. This speeds up further work with the site and reduces “guessing” selectors.

## Site profiles (`scripts/sites`)

Selectors and “controls” for specific sites live in JSON under `scripts/sites/`. This allows agents to “remember” important UI elements (e.g. Google SERP) and reuse them without hardcoding in code or prompts. See [`scripts/_shared`](scripts/_shared.md) for details.

## Reliable content extraction from pages

Many pages render content via JavaScript (SPAs, dynamic pages, lazy-loading). A plain `goto()` + `getHtml()` may return empty or incomplete HTML. If `getContent()` or `getHtml()` returns an empty/minimal result, use this **escalation strategy** — proceed to the next level until you get what you need.

### Level 1: Standard load (default)

CLI (recommended):

```bash
node scripts/getContent.js --url https://example.com --dir ./output --name page.md
```

Scripts `getContent` and `googleSearch.openLink` wait for `networkidle` by default (5 seconds). For many pages this is enough.

### Level 2: Wait for content stabilization

If the content is empty/minimal — the page likely renders via JS after load. Prefer a CLI “wait + extract” run:

```bash
node utils/browserUse.js https://example.com --wait ".article-body" --extract output.json
```

This waits for the key selector and then extracts structured blocks via `getDataFromText` (including Markdown inside `content[].markdown`).

### Level 3: Raw extraction via `evaluate` + manual parsing

If extraction still returns empty content but the page is visibly loaded, the data may be inside Shadow DOM, iframes, or site-specific containers. This usually requires **programmatic** browser access (API) to extract targeted `innerText` / container HTML and then parse via `getDataFromText`. Treat this as an advanced / last-resort step.

### Level 4: Screenshot + visual analysis

If text content is not programmatically accessible (canvas rendering, anti-scraping measures, complex layouts):

```bash
node utils/browserUse.js https://example.com --screenshot page.png --full-page
```

Analyze the screenshot visually.

### Level 5: Scroll + re-collect for lazy-loaded / infinite-scroll pages

Pages with infinite scroll or lazy-loaded blocks load content only on scroll. Fully automated scrolling currently requires **programmatic** control (API) via `utils/browserUse`. Treat this as an advanced / last-resort step and only automate it when you’re confident about the site structure and selectors.

### Summary table

| Symptom | Action |
| --- | --- |
| Content is empty (0 chars) | Level 2: `node utils/browserUse.js <url> --wait "<selector>" --extract out.json` |
| Content is minimal (header/menu only) | Level 3: advanced API extraction (`evaluate` / container HTML) |
| Text is not accessible programmatically | Level 4: `node utils/browserUse.js <url> --screenshot page.png --full-page` |
| Content loads on scroll | Level 5: advanced API scrolling (`browser.scroll({ times: N })`) |
| Page is blocked / CAPTCHA | See “Page blockers” section |

**Rule:** if a page is critical for the research, **do not skip it silently**. Go through all escalation levels. Only if none works — record in your insights that content could not be extracted, and move on.

## Working with PDF files

PDF links in Google results are common (papers, reports, whitepapers). Chrome extensions (Adobe Acrobat, Google PDF Viewer) can intercept PDF navigation, making standard `goto()` + `getHtml()` useless.

**Automatic handling:** `getContent` and `googleSearch` automatically detect PDF links (by `.pdf` extension) and extract text via `getPdfText()` instead of navigating. No additional actions are required — the standard `openLink → getContent → closeTab` flow works for PDFs unchanged.

**CLI usage (recommended):**

```bash
node scripts/getContent.js --url https://example.com/paper.pdf --dir ./output --name paper.md
```

**Limitations of PDF extraction:**

- Only text is extracted — tables, formulas, and diagrams may lose formatting.
- Scanned PDFs (images instead of text) will return empty text — use screenshots in this case.
- Password-protected PDFs cannot be processed.

## Writing custom scripts (last resort)

If you are confident about a site’s structure/selectors and need multi-step, repeatable automation that the CLI tools do not support (e.g. login flows, infinite scroll, complex UI state), you may write a small script. Keep it minimal: compose existing modules from this repo (`scripts/*`, `utils/*`), and avoid re-implementing browser automation.