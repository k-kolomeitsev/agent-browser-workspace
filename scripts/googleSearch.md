# googleSearch

`GoogleSearch` is a class for step-by-step control of Google Search. Each operation is a separate method: run a search, collect links (without ads), open a result in a new tab, extract content, return to results, paginate.

> **CLI-first.** Use the CLI interface for agent tasks. The API is intended for tool authors and advanced integrations. See `AGENTS.md` for the policy.

## Quick start

### CLI

```bash
# Search and print links
node scripts/googleSearch.js "node.js best practices" --links

# Search, open the first link, save content
node scripts/googleSearch.js "playwright tutorial" --open 0 --dir ./output --name article.md

# Search on page 2
node scripts/googleSearch.js "web scraping" --page 2 --links

# Search and save a stable snapshot of results (links.json)
node scripts/googleSearch.js "web scraping" --links --dir ./archive/my-research
```

### API (advanced)

The class can also be used programmatically from Node.js, but the recommended interface is the CLI. See the **API** section below for options and behavior.

## API

### Constructor

```javascript
const google = new GoogleSearch(options);
```

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `browser` | `BrowserUse` | `null` | Existing browser instance |
| `cdp` | `boolean\|string` | — | CDP mode |
| `launch` | `boolean` | — | Force launch mode |
| `headless` | `boolean` | `false` | Headless |
| `timeout` | `number` | `30000` | Timeout |
| `linksDir` | `string` | `null` | If set — `getLinks()` appends results to `<linksDir>/links.json` |

### Properties

| Property | Type | Description |
|----------|-----|-------------|
| `browser` | `BrowserUse` | Current browser instance |
| `links` | `Array` | Last parsed list of links |

### Methods

#### `init()`

Initializes the browser. Call before any other methods.

```javascript
const google = new GoogleSearch({ launch: true, headless: true });
await google.init();
```

#### `search(query, options?)`

Opens Google, types the query, presses Enter. Waits for the results page to load.

```javascript
await google.search('playwright tutorial');
```

| Parameter | Type | Description |
|----------|-----|-------------|
| `query` | `string` | Search query |
| `options.timeout` | `number` | Navigation timeout (ms) |

#### `getSearchForm()`

Gets the search form from the current page via [getForms](./getForms.md). Returns a form object with `type: "search"` or `null`.

```javascript
const form = await google.getSearchForm();
// {
//   type: 'search',
//   selector: '...',
//   fields: [
//     { name: 'q', selector: 'textarea[name="q"]', ... },
//   ],
// }
```

#### `getLinks(options?)`

Parses organic results (without ads) using an anchor-first strategy. Collects all `<a>` with an `<h3>` inside the results area while filtering ad containers. Results are cached in `google.links`.

If `linksDir` was set in the constructor, `getLinks()` **automatically** saves results to `links.json` (append/create).

```javascript
const links = await google.getLinks();
// [
//   { index: 0, title: 'Best Practices...', url: 'https://...', snippet: '...' },
//   { index: 1, title: 'Node.js Guide',     url: 'https://...', snippet: '...' },
// ]
```

Result item format:

| Field | Type | Description |
|------|-----|-------------|
| `index` | `number` | Index (0-based) |
| `title` | `string` | Result title |
| `url` | `string` | Target URL |
| `snippet` | `string` | Snippet / description |

##### `links.json` (stable SERP snapshot)

The file is stored at `linksDir/links.json` and contains an **array of objects**:

```json
[
  {
    "query": "playwright tutorial",
    "url": "https://example.com/article",
    "title": "Playwright Tutorial",
    "description": "…snippet…",
    "type": "html"
  },
  {
    "query": "playwright tutorial",
    "url": "https://example.com/paper.pdf",
    "title": "Paper PDF",
    "description": "",
    "type": "pdf"
  }
]
```

- One output directory = one `links.json` file.
- If the file already exists, new items are **appended** (deduplicated by `query + url`).
- `type` is derived from the URL: for regular sites — `html`, for PDFs — `pdf`, for other extensions — the extension without the dot (e.g. `docx`).

#### `openLink(n, options?)`

Opens the n-th link from `getLinks()` in a new tab and switches to it. Remembers the results tab so `closeTab()` can return to it. PDF links (`.pdf`) are detected automatically — `getContent()` will extract text via `getPdfText()`.

```javascript
const { url, isPdf } = await google.openLink(0);
if (isPdf) console.log('PDF will be handled via getContent()');
```

| Parameter | Type | Description |
|----------|-----|-------------|
| `n` | `number` | Link index (0-based) |
| `options.timeout` | `number` | Navigation timeout |

Returns: `{ url: string, isPdf: boolean }`

Throws `Error` if `getLinks()` was not called, and `RangeError` if out of bounds.

#### `getContent(options)`

Extracts Markdown content for the current active tab. Delegates to [getContent](./getContent.md) with the current browser.

```javascript
const result = await google.getContent({
  dir: './output',
  name: 'article.md',
  imageSubdir: 'images',
});
// result.markdown, result.images, result.savedTo, result.metadata
```

| Parameter | Type | Description |
|----------|-----|-------------|
| `options.dir` | `string` | Output directory |
| `options.name` | `string` | Markdown filename |
| `options.imageSubdir` | `string` | Images subdirectory (default: `images`) |
| `options.minWidth` | `number` | Minimum image width (default: `100`) |
| `options.minHeight` | `number` | Minimum image height (default: `100`) |

#### `closeTab()`

Closes the current tab and returns to the Google results tab.

```javascript
await google.openLink(0);
// ... work with the page ...
await google.closeTab();
// Now the Google results tab is active
```

#### `goToPage(n, options?)`

Navigates to the n-th Google pagination page. If `n` is omitted or equals 0 — clicks “Next” (`#pnnext`).

```javascript
await google.goToPage(2);   // go to page 2
await google.goToPage();    // next page
```

| Parameter | Type | Description |
|----------|-----|-------------|
| `n` | `number` | Page number (1-based). `0` / `undefined` = next page |
| `options.timeout` | `number` | Timeout |

After navigation, the cached link list is cleared — call `getLinks()` again.

#### `close()`

Closes the browser (only if the instance was created by this object).

```javascript
await google.close();
```

## Google selectors

Google selectors are stored in the profile `scripts/sites/google-search.json` and exported as `GOOGLE_SELECTORS` for easy updates when Google changes markup:

Export: `GOOGLE_SELECTORS` from `scripts/googleSearch.js`.

| Key | Selector | Description |
|------|----------|-------------|
| `url` | `https://www.google.com` | Google base URL |
| `searchInput` | `textarea[name="q"], input[name="q"]` | Search input |
| `resultAnchor` | `#search a[href], #rso a[href], .MjjYud a[href], .tF2Cxc a[href]` | All anchors in the results area |
| `resultTitle` | `h3` | Result title (inside/near the anchor) |
| `resultBlock` | `.MjjYud, .tF2Cxc, div.g, [data-hveid]` | Wrapper block for a single result |
| `resultSnippet` | `[data-sncf], [style*="-webkit-line-clamp"], .VwiC3b, .lEBKkf` | Snippet |
| `adContainers` | `#tads, #bottomads, [data-text-ad], [data-ad-slot]` | Ad containers |
| `nextPage` | `#pnnext, a[aria-label="Next"]` | “Next” link |
| `paginationLink` | `a[aria-label="Page {n}"]` | Link to page N |

Note: Google localizes some `aria-label` values depending on UI language. The `scripts/sites/google-search.json` profile contains additional locale-specific selector variants — treat it as the source of truth.

### How link extraction works (anchor-first strategy)

Google frequently changes SERP markup. Until 2025 each result lived in `div.g`, but by 2026 Google moved to `.MjjYud` (result group) and `.tF2Cxc` (single result container). `div.g` often returns 0 elements now.

The current `getLinks()` algorithm uses an **anchor-first** approach that is resilient to wrapper changes:

1. **Collect all `<a href>` inside the results area** — using `resultAnchor` to cover `#search`, `#rso`, `.MjjYud`, `.tF2Cxc`.
2. **Filter ads** — any `<a>` inside `adContainers` (`#tads`, `#bottomads`, `[data-text-ad]`, `[data-ad-slot]`) is dropped.
3. **Keep only anchors with `<h3>`** — an organic result contains `<h3>` either inside the anchor (`a > h3`) or within the nearest wrapper block (`resultBlock`). Links without `<h3>` (navigation, related searches, “People also ask”) are dropped.
4. **Unwrap Google redirects** — for `https://www.google.com/url?...`, the real `url/q` is extracted and saved.
5. **Deduplicate** by final URL.
6. **Snippet** — extracted from the nearest `resultBlock` via `resultSnippet`.

If Google changes markup again and `getLinks()` returns 0 — update `resultAnchor` and/or `resultBlock`.

## CLI

```
Usage: node scripts/googleSearch.js "<query>" [options]

Options:
  --page <n>              Go to pagination page n after search
  --links                 Print organic links as JSON
  --open <n>              Open the n-th link (0-based) and get content
  --dir <dir>             Output directory for content (required with --open). If set, also appends results to <dir>/links.json
  --name <file>           Filename for content (required with --open)
  --image-subdir <dir>    Subdirectory for images (default: images)
  --profile <name>        Chrome profile name (default: AgentProfile)
  --cdp [endpoint]        Connect via CDP (default: http://localhost:9222)
  --launch                Force launch a new browser
  --headless              Run headless
  --timeout <ms>          Timeout (default: 30000)
  --help                  Show this help
```

Without `--links` and `--open`, it prints links to stdout by default.

## Dependencies

- [_shared](./_shared.md) — browser initialization
- [getContent](./getContent.md) — page content extraction
- [getForms](./getForms.md) — search form extraction
- [browserUse](../utils/browserUse.md) — Chrome control

## Site profile (selectors/controls)

Google selectors and UI controls are stored in `scripts/sites/google-search.json`. If Google changes markup again — update it **there**, not in code.

