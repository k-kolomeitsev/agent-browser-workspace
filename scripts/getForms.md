# getForms

Extract all forms from the current (or specified) browser page with automatic classification (search, auth, filter, contact, subscribe, generic) and field parsing. Each field gets a ready-to-use CSS selector compatible with `browser.fill()` / `browser.fillForm()`.

> **CLI-first.** Use this tool via CLI unless you are authoring new tooling. See `AGENTS.md` for the policy and escalation guidance.

## Quick start

### CLI

```bash
# Forms from the current page (stdout)
node scripts/getForms.js

# Navigate and save to a file
node scripts/getForms.js --url https://example.com --output forms.json

# CDP to a specific port
node scripts/getForms.js --cdp http://localhost:9333 --output forms.json
```

### API (advanced)

The module can also be imported from Node.js, but the recommended interface is the CLI. See the **API** section below for options and return shape.

## API

### `getForms(options)`

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `url` | `string` | `null` | URL to navigate to (`null` = current page) |
| `browser` | `BrowserUse` | `null` | Existing browser instance |
| `html` | `string` | `null` | Pre-fetched HTML (skips `browser.getHtml()`) |
| `cdp` | `boolean\|string` | — | CDP mode |
| `launch` | `boolean` | — | Force launch mode |
| `headless` | `boolean` | `false` | Headless |
| `timeout` | `number` | `30000` | Timeout |

### Result shape

```json
{
  "forms": [
    {
      "type": "search",
      "selector": "[role=\"search\"]",
      "confidence": 0.95,
      "tier": 1,
      "evidence": ["role:search", "has_search_input"],
      "features": {
        "inputCount": 1,
        "selectCount": 0,
        "textareaCount": 0,
        "buttonCount": 1,
        "hasPasswordInput": false,
        "hasSearchInput": true,
        "hasSubmitButton": true,
        "action": "/search",
        "method": "GET"
      },
      "html": "<form action=\"/search\">...</form>",
      "fields": [
        {
          "tag": "input",
          "type": "text",
          "name": "q",
          "id": "search-input",
          "placeholder": "Search...",
          "value": "",
          "selector": "[role=\"search\"] #search-input"
        },
        {
          "tag": "button",
          "type": "submit",
          "name": "",
          "id": "",
          "placeholder": "",
          "value": "",
          "selector": "[role=\"search\"] button[type=\"submit\"]",
          "text": "Search"
        }
      ]
    }
  ],
  "metadata": {
    "title": "Page title",
    "lang": "en",
    "description": "...",
    "url": "https://example.com"
  },
  "site": {
    "id": "google-search",
    "name": "Google Search",
    "host": "www.google.com",
    "controls": [
      {
        "name": "Search input",
        "selector": "textarea[name=\"q\"], input[name=\"q\"]",
        "actions": ["fill", "press:Enter"]
      }
    ]
  }
}
```

### `fields` structure

Each form field contains:

| Field | Type | Description |
|------|-----|-------------|
| `tag` | `string` | HTML tag: `input`, `select`, `textarea`, `button` |
| `type` | `string` | Field type: `text`, `password`, `email`, `submit`, `select`, `textarea`, ... |
| `name` | `string` | `name` attribute |
| `id` | `string` | `id` attribute |
| `placeholder` | `string` | `placeholder` attribute |
| `value` | `string` | `value` attribute |
| `selector` | `string` | Ready CSS selector for `browser.fill()` |
| `text` | `string` | Button text (only for `button`) |
| `ariaLabel` | `string` | `aria-label` (if present) |
| `options` | `Array` | `<select>` options: `[{ value, text, selected }]` |

Selector generation priority: `#id` → `[name]` → `[aria-label]` → `tag[type]`.

### Form types

| Type | Description | Signals |
|-----|-------------|---------|
| `search` | Search form | `role="search"`, `input[type="search"]`, action contains "search" |
| `auth` | Authentication/login | `input[type="password"]`, class/id contains "login", "auth" |
| `filter` | Filters/sorting | Many `select`, class/id contains "filter", "sort" |
| `contact` | Contact/feedback | `textarea`, class/id contains "contact", "feedback" |
| `subscribe` | Subscription/newsletter | `input[type="email"]`, class/id contains "subscribe", "newsletter" |
| `generic` | Other | Doesn’t match other types |

## CLI

```
Usage: node scripts/getForms.js [options]

Options:
  --url <url>             Navigate to URL before extracting (default: current page)
  --output <file>         Save result to JSON file (default: print to stdout)
  --profile <name>        Chrome profile name (default: AgentProfile)
  --cdp [endpoint]        Connect via CDP (default: http://localhost:9222)
  --launch                Force launch a new browser
  --headless              Run headless
  --timeout <ms>          Timeout (default: 30000)
  --help                  Show this help
```

When printing to stdout, forms are shown in a shortened format (no full HTML, only a preview).

## Examples (CLI)

```bash
# Navigate and save all detected forms to JSON
node scripts/getForms.js --url https://example.com --output ./output/forms.json

# Attach to an existing Chrome (CDP) and extract from the current page
node scripts/getForms.js --cdp http://localhost:9222 --output forms.json
```

## Writing custom scripts (last resort)

If you need to actually fill/submit forms as part of a repeatable automation flow, you will need programmatic browser control (API) via `utils/browserUse`. Do this only when you’re confident about the site structure/selectors. See `AGENTS.md` for the policy.

## Dependencies

- [_shared](./_shared.md) — browser initialization
- [browserUse](../utils/browserUse.md) — Chrome control
- [getDataFromText](../utils/getDataFromText.md) — form detection and classification
- [cheerio](https://github.com/cheeriojs/cheerio) — parse form fields from HTML

