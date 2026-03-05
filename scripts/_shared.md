# _shared

Shared browser initialization module for all scripts in `scripts/`. Auto-detects the connection mode: CDP attach to an already-running Chrome, or launching a new instance. Also parses common CLI flags.

It also contains the **site profile registry** (`scripts/sites/*.json`) — configs that “remember”:

- rules/selectors for scraping specific sites (e.g. Google SERP);
- UI controls (selectors + action descriptions) that are useful to include in an agent response when working with the given host.

> **Internal / advanced.** This module is meant for authors of new CLI tools in `scripts/`. For agent tasks, do not use it directly — run the existing CLI scripts instead (see `AGENTS.md`).

## API

### `initBrowser(options)`

Initializes a `BrowserUse` instance and returns `{ browser, ownsInstance }`.

`ownsInstance` indicates that the instance was created by this call (not passed in) — it should be closed after use via `releaseBrowser()`.

```javascript
const { initBrowser, releaseBrowser } = require('./scripts/_shared');

const { browser, ownsInstance } = await initBrowser();
// ... work with browser ...
await releaseBrowser(browser, ownsInstance);
```

#### Connection priority

| Priority | Condition | Mode |
|---|---|---|
| 1 | `options.browser` is passed | Reuse (`ownsInstance: false`) |
| 2 | `options.launch === true` | Launch a new Chrome (`close()` terminates it) |
| 3 | `options.cdp === true` or a string | Force CDP connect |
| 4 | Nothing specified | Auto-detect: CDP probe → on failure `launchCDP` |

Auto-detect sends a CDP probe to `http://localhost:9222` with a 3-second timeout. If Chrome with CDP is available — it connects. Otherwise it starts Chrome as a background process with `--remote-debugging-port` via `launchCDP()` and connects via CDP. Chrome continues running after `close()`, and subsequent calls attach instantly.

#### Options

| Option | Type | Default | Description |
|---|---|---|---|
| `browser` | `BrowserUse` | — | Existing instance to reuse |
| `launch` | `boolean` | `false` | Force launch of a new Chrome |
| `cdp` | `boolean\|string` | — | Force CDP (`true` or endpoint URL) |
| `headless` | `boolean` | `false` | Headless mode (launch) |
| `profile` | `string` | `'AgentProfile'` | Chrome profile name |
| `endpointURL` | `string` | `http://localhost:9222` | CDP endpoint |
| `timeout` | `number` | `30000` | Connection / navigation timeout (ms) |

#### Examples

```javascript
// Auto-detect (CDP → launchCDP)
const { browser, ownsInstance } = await initBrowser();

// Force CDP on a custom port
const { browser, ownsInstance } = await initBrowser({
  cdp: 'http://localhost:9333',
});

// Force launch in headless mode
const { browser, ownsInstance } = await initBrowser({
  launch: true,
  headless: true,
});

// Pass an existing instance
const myBrowser = await BrowserUse.launch();
const { browser, ownsInstance } = await initBrowser({ browser: myBrowser });
// ownsInstance === false → releaseBrowser() will NOT close it
```

### `parseBaseFlags(argv)`

Parses shared CLI flags from `process.argv.slice(2)`.

```javascript
const { parseBaseFlags } = require('./scripts/_shared');

const flags = parseBaseFlags(process.argv.slice(2));
// flags = { cdp, launch, headless, url, timeout }
```

| Flag | Type | Description |
|------|-----|-------------|
| `--cdp [endpoint]` | `true\|string` | Connect via CDP |
| `--launch` | `boolean` | Launch a new browser |
| `--headless` | `boolean` | Headless mode |
| `--profile <name>` | `string` | Chrome profile name (default: AgentProfile) |
| `--url <url>` | `string` | URL to navigate to |
| `--timeout <ms>` | `number` | Timeout |
| `--shutdown` | `boolean` | Shut down background Chrome |

### `flagsToBrowserOptions(flags)`

Converts `parseBaseFlags()` output into options for `initBrowser()`.

```javascript
const { parseBaseFlags, flagsToBrowserOptions, initBrowser } = require('./scripts/_shared');

const flags = parseBaseFlags(process.argv.slice(2));
const { browser, ownsInstance } = await initBrowser(flagsToBrowserOptions(flags));
```

### `releaseBrowser(browser, ownsInstance)`

Safely closes the browser only if `ownsInstance === true`.

```javascript
const { initBrowser, releaseBrowser } = require('./scripts/_shared');

const { browser, ownsInstance } = await initBrowser();
try {
  // ...
} finally {
  await releaseBrowser(browser, ownsInstance);
}
```

### `shutdownBrowser(options)`

Stops the background Chrome process on the CDP port. Delegates to `BrowserUse.shutdown()`.

```javascript
const { shutdownBrowser } = require('./scripts/_shared');
await shutdownBrowser();             // port 9222 by default
await shutdownBrowser({ port: 9333 });
```

## Site profiles (`scripts/sites/*.json`)

Site profiles let you move site-specific selector/control “hardcode” into JSON and reuse it across agents and scripts.

### Functions

- `loadSiteProfiles()` — load all JSON profiles from `scripts/sites/` (cached).
- `getSiteProfileById(id)` — get a profile by `id` (throws if not found).
- `getSiteProfileForHost(host)` — select a profile by hostname (supports exact matches and masks like `*.example.com` / `.example.com`).
- `getResolvedSiteInfoForUrl(url)` — return compact “site info” for a URL: `{ id, name, host, controls[] }`, where `controls[].selector` is resolved from `selectorKey` into an actual CSS selector.

### Recommended profile structure

```json
{
  "id": "google-search",
  "name": "Google Search",
  "hosts": ["google.com", "www.google.com"],
  "baseUrl": "https://www.google.com",
  "scraping": {
    "selectors": {
      "searchInput": "textarea[name=\"q\"], input[name=\"q\"]"
    }
  },
  "controls": {
    "items": [
      {
        "name": "Search input",
        "selectorKey": "searchInput",
        "description": "Primary query input field",
        "actions": ["fill", "press:Enter"]
      }
    ]
  }
}
```

## Export

```javascript
module.exports = {
  initBrowser,
  parseBaseFlags,
  flagsToBrowserOptions,
  releaseBrowser,
  shutdownBrowser,
  DEFAULT_CDP_ENDPOINT,   // 'http://localhost:9222'
  loadSiteProfiles,
  getSiteProfileById,
  getSiteProfileForHost,
  getResolvedSiteInfoForUrl,
};
```

## Dependencies

- [browserUse](../utils/browserUse.md) — control Chrome via Playwright
