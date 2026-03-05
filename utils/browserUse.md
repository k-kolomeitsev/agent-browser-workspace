# browser-use

Control a real Google Chrome instance via [Playwright](https://playwright.dev/). Supports three modes: auto-start Chrome with CDP (`launchCDP`), launch a persistent profile (`launch`), or connect to an existing Chrome via CDP (`connectCDP`). Includes navigation, form filling, screenshots, image/file downloads, and PDF text extraction. Integrates with `getDataFromText` for extracting structured data from live pages.

> **CLI-first.** Prefer using `node utils/browserUse.js ...` for tasks. The API sections below are for tool authors and advanced integrations. See `AGENTS.md` for the policy.

## Install

```bash
npm install
npx playwright install chrome
```

## Quick start

### CLI

```bash
# Get page HTML
node utils/browserUse.js https://example.com --html page.html

# Screenshot (full page)
node utils/browserUse.js https://example.com --screenshot shot.png --full-page

# Download content images
node utils/browserUse.js https://example.com --images ./img

# Extract structured data via getDataFromText
node utils/browserUse.js https://example.com --extract output.json

# Multiple actions in one run
node utils/browserUse.js https://example.com --html page.html --screenshot shot.png --images ./img

# Connect via CDP (Chrome is already running)
node utils/browserUse.js https://example.com --cdp --html page.html

# Headless mode
node utils/browserUse.js https://example.com --headless --html page.html

# Wait for an element before running actions
node utils/browserUse.js https://example.com --wait ".content" --html page.html
```

### API (advanced)

The module can also be used programmatically from Node.js, but the recommended interface is the CLI. See the sections below for details.

## Three operating modes

### Mode 1: `launchCDP` — auto-start Chrome with CDP (recommended)

Starts Chrome as a background process with `--remote-debugging-port`, then connects via CDP. If Chrome is already running on the port, it simply connects without launching another instance.

Key advantage: `close()` only disconnects; the Chrome process keeps running. The next call attaches instantly without recreating the browser.

```javascript
const browser = await BrowserUse.launchCDP();
await browser.goto('https://example.com');
await browser.close(); // Chrome stays alive

// Later, in another script:
const browser2 = await BrowserUse.launchCDP(); // attaches to the already-running Chrome
await browser2.goto('https://other.com');
await browser2.close();

// When you are completely done:
await BrowserUse.shutdown(); // terminates the background Chrome process
```

This is the default mode for auto-detection in `scripts/_shared.js`.

### Mode 2: `launch` — launch a persistent Chrome profile

Uses `chromium.launchPersistentContext()` with `channel: 'chrome'`. Cookies, localStorage, history, and extensions persist across sessions. Chrome is fully closed by `close()`.

```javascript
const browser = await BrowserUse.launch({
  headless: false,
  viewport: { width: 1920, height: 1080 },
});
```

The profile is stored in a separate directory (not your main Chrome profile). Default paths:

| OS | Path |
|----|------|
| Windows | `%LOCALAPPDATA%\Google\Chrome\AgentProfile` |
| macOS | `~/Library/Application Support/Google/Chrome/AgentProfile` |
| Linux | `~/.config/google-chrome/AgentProfile` |

The directory is created automatically on first run.

### Mode 3: `connectCDP` — connect to an existing Chrome

Connects to Chrome started with `--remote-debugging-port` via Chrome DevTools Protocol. The Chrome process stays alive after `close()`.

Start Chrome manually first:

```bash
# Windows (PowerShell)
& "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir="%LOCALAPPDATA%\Google\Chrome\AgentProfile"

# macOS
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --remote-debugging-port=9222 --user-data-dir="$HOME/Library/Application Support/Google/Chrome/AgentProfile"

# Linux
google-chrome --remote-debugging-port=9222 --user-data-dir="$HOME/.config/google-chrome/AgentProfile"
```

Then connect:

```javascript
const browser = await BrowserUse.connectCDP({
  endpointURL: 'http://localhost:9222',
});
```

## API

### Initialization

```javascript
const BrowserUse = require('./utils/browserUse');

// Static factories (create and initialize an instance)
const browser = await BrowserUse.launchCDP(options); // recommended
const browser2 = await BrowserUse.launch(options);
const browser3 = await BrowserUse.connectCDP(options);

// Or via an instance
const instance = new BrowserUse();
await instance.launchCDP(options);
// ...
await instance.close();
```

#### `launchCDP(options)`

Launches Chrome with CDP and connects. If Chrome is already listening on the port, it connects without launching.

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `port` | `number` | `9222` | CDP port |
| `profile` | `string` | `'AgentProfile'` | Chrome profile name |
| `userDataDir` | `string` | per-OS | Full profile directory path (overrides `profile`) |
| `headless` | `boolean` | `false` | Headless |
| `timeout` | `number` | `30000` | Startup + connection timeout (ms) |
| `args` | `string[]` | `[]` | Additional Chromium flags |

#### `launch(options)`

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `profile` | `string` | `'AgentProfile'` | Chrome profile name |
| `userDataDir` | `string` | per-OS | Full profile directory path (overrides `profile`) |
| `headless` | `boolean` | `false` | Headless |
| `channel` | `string` | `'chrome'` | Browser channel (`chrome`, `chrome-beta`, `msedge`, ...) |
| `viewport` | `{width, height}` | `1280×720` | Viewport size |
| `acceptDownloads` | `boolean` | `true` | Allow file downloads |
| `args` | `string[]` | `[]` | Additional Chromium flags |

#### `connectCDP(options)`

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `endpointURL` | `string` | `http://localhost:9222` | CDP endpoint |
| `timeout` | `number` | `30000` | Connection timeout (ms) |

### Accessors

```javascript
browser.page     // currently active page (Page)
browser.context  // current browser context (BrowserContext)
browser.browser  // Browser object (CDP mode only; null in persistent mode)
browser.mode     // 'persistent' | 'cdp' | null
```

### Navigation

```javascript
await browser.goto(url, options)    // navigate to URL
await browser.goBack()              // back
await browser.goForward()           // forward
await browser.reload()              // reload

browser.getUrl()                    // current URL (sync)
await browser.getTitle()            // page title
```

`goto` options:

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `waitUntil` | `string` | `'load'` | `'load'` \| `'domcontentloaded'` \| `'networkidle'` \| `'commit'` |
| `timeout` | `number` | `30000` | Navigation timeout (ms) |
| `referer` | `string` | — | HTTP Referer |

### Reliable navigation (JS rendering)

For pages that render content via JavaScript (SPAs, dynamic pages), `goto()` may return empty HTML. Use `gotoAndWaitForContent()`:

```javascript
// Navigation with wait: load → networkidle → DOM stabilization
const { response, contentReady } = await browser.gotoAndWaitForContent('https://spa-app.com');
console.log(`Content stable: ${contentReady.stable}, ${contentReady.contentLength} chars`);

// With a required element
await browser.gotoAndWaitForContent('https://example.com', {
  waitForSelector: '.article-body',
});

// With increased timeouts for slow pages
await browser.gotoAndWaitForContent('https://slow-page.com', {
  timeout: 60000,
  networkIdleTimeout: 10000,
  contentTimeout: 20000,
});
```

`gotoAndWaitForContent` options:

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `timeout` | `number` | `30000` | Navigation timeout (ms) |
| `networkIdleTimeout` | `number` | `5000` | Max wait for networkidle after load (ms) |
| `contentTimeout` | `number` | `15000` | Max wait for content stabilization (ms) |
| `waitForSelector` | `string` | — | CSS selector: wait for the element before checking stabilization |
| `pollInterval` | `number` | `300` | Stabilization polling interval (ms) |
| `stableCount` | `number` | `3` | Number of consecutive equal readings to consider content stable |

Result format:

```javascript
{
  response: Response,      // Playwright navigation response
  contentReady: {
    stable: true,          // whether content stabilized before timeout
    contentLength: 14523,  // document.body.innerText length
    elapsed: 2150,         // stabilization wait time (ms)
  },
}
```

### Getting page content

```javascript
const html = await browser.getHtml();                    // full page HTML
const inner = await browser.getElementHtml('.article');  // innerHTML for an element
const text = await browser.getText('.title');            // textContent for an element
const href = await browser.getAttribute('a.link', 'href'); // attribute value
```

### Element interactions

```javascript
await browser.click(selector, options)     // click
await browser.dblclick(selector, options)  // double click
await browser.hover(selector, options)     // hover
await browser.fill(selector, value)        // fill (clears previous value)
await browser.type(selector, text, opts)   // character-by-character typing
await browser.select(selector, value)      // select option in <select>
await browser.check(selector)              // check checkbox/radio
await browser.uncheck(selector)            // uncheck checkbox
await browser.press(selector, key)         // press key ('Enter', 'Tab', 'Control+a')
await browser.uploadFile(selector, paths)  // upload file via <input type="file">
```

### Form filling

`fillForm` accepts `{ selector: value }` and auto-detects field type:

```javascript
await browser.fillForm({
  '#email':    'user@example.com',   // input[type="text/email"] → fill
  '#password': 'secret',             // input[type="password"]   → fill
  '#country':  'US',                 // <select>                 → selectOption
  '#agree':    true,                 // input[type="checkbox"]   → check / uncheck
  '#avatar':   './photo.jpg',        // input[type="file"]       → setInputFiles
});
```

| Element type | Action |
|-------------|--------|
| `<input>` (text, email, password, ...) | `fill(value)` |
| `<textarea>` | `fill(value)` |
| `<select>` | `selectOption(value)` |
| `<input type="checkbox/radio">` | `check()` / `uncheck()` |
| `<input type="file">` | `setInputFiles(value)` |

### Scrolling

```javascript
// Scroll one viewport down
await browser.scroll();

// Scroll up
await browser.scroll({ direction: 'up' });

// Jump to top/bottom
await browser.scroll({ direction: 'top' });
await browser.scroll({ direction: 'bottom' });

// Scroll by a specific pixel distance
await browser.scroll({ distance: 500 });
await browser.scroll({ direction: 'up', distance: 300 });

// Scroll to an element (scrollIntoView)
await browser.scroll({ selector: '#comments' });

// Infinite scroll — repeated scrolling with delays for loading
const result = await browser.scroll({ times: 20, delay: 1500 });
console.log(`Iterations: ${result.iterations}, reached bottom: ${result.reachedBottom}`);

// Infinite scroll to the very bottom (scroll until it stops)
const result2 = await browser.scroll({
  direction: 'bottom',
  times: Infinity,
  delay: 2000,
  timeout: 60000,  // stop after 60 seconds regardless
});
```

Options:

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `direction` | `string` | `'down'` | `'down'` \| `'up'` \| `'top'` \| `'bottom'` |
| `distance` | `number` | viewport height | Pixels to scroll (overrides viewport step) |
| `selector` | `string` | — | CSS/XPath selector: scroll to the element |
| `times` | `number` | `1` | Iterations (for infinite scroll) |
| `delay` | `number` | `1000` | Pause between iterations (ms) to allow content to load |
| `timeout` | `number` | `30000` | Maximum total time (ms); stops early if exceeded |

Result format:

```javascript
{
  scrollTop: 4520,       // current scroll position (px from top)
  scrollHeight: 12800,   // full page height
  reachedBottom: false,  // whether the bottom is reached
  iterations: 20,        // how many iterations were performed
}
```

When `times > 1`, the method stops early if after a scroll down the page height and scroll position do not change anymore (no more content is loading). `reachedBottom` is set to `true`.

### Waiting

```javascript
await browser.waitFor(selector, options)              // wait for element
await browser.waitForUrl(pattern, options)            // wait for URL
await browser.waitForLoadState(state, options)        // 'load' | 'domcontentloaded' | 'networkidle'
await browser.waitForResponse(urlPattern, opts)       // wait for network response
await browser.waitForContentReady(options)            // wait for DOM stabilization after JS rendering
await browser.wait(ms)                                // fixed delay
```

`waitForContentReady` waits for DOM stabilization after JS rendering:

```javascript
// After a normal navigation, wait for client-side rendering
await browser.goto('https://spa-app.com');
const result = await browser.waitForContentReady({ timeout: 10000 });
if (!result.stable) {
  console.log('Content did not fully stabilize, but text exists:', result.contentLength);
}
```

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `pollInterval` | `number` | `300` | Polling interval (ms) |
| `stableCount` | `number` | `3` | How many equal readings in a row = stable |
| `timeout` | `number` | `15000` | Max wait time (ms) |

Result: `{ stable: boolean, contentLength: number, elapsed: number }`

`waitFor` options:

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `state` | `string` | `'visible'` | `'attached'` \| `'detached'` \| `'visible'` \| `'hidden'` |
| `timeout` | `number` | `30000` | Timeout (ms) |

### Screenshots

```javascript
// Visible viewport
await browser.screenshot({ path: 'screen.png' });

// Full scrollable page
await browser.screenshot({ path: 'full.png', fullPage: true });

// To a buffer (no file)
const buffer = await browser.screenshot();

// Screenshot a specific element
await browser.screenshotElement('.header', { path: 'header.png' });

// JPEG with quality
await browser.screenshot({ path: 'screen.jpg', type: 'jpeg', quality: 80 });

// Clip a region
await browser.screenshot({ path: 'area.png', clip: { x: 0, y: 0, width: 500, height: 300 } });
```

The target directory is created automatically.

### Download images from content

```javascript
const results = await browser.downloadImages({
  outputDir: './images',       // output directory (created automatically)
  selector: 'img',             // CSS selector to find images
  minWidth: 100,               // skip icons smaller than 100px wide
  minHeight: 100,              // skip icons smaller than 100px tall
  concurrency: 5,              // parallel download limit
});

// Only from the content area
const results2 = await browser.downloadImages({
  selector: 'article img',
  minWidth: 200,
});
```

Result format:

```javascript
[
  {
    src: 'https://example.com/photo.jpg',
    alt: 'Description',
    savedAs: '/abs/path/images/photo.jpg',
    size: 145832,           // bytes
    success: true,
  },
  {
    src: 'https://example.com/broken.png',
    alt: '',
    success: false,
    error: 'HTTP 404',
  },
]
```

Notes:

- Supports lazy-loaded images (`data-src`, `data-lazy-src`, `data-original`, `currentSrc`)
- Skips `data:` URIs
- Duplicate filenames are auto-numbered (`photo.jpg`, `photo_1.jpg`, ...)
- Extension is inferred from Content-Type if missing from the URL

### Download files

```javascript
const result = await browser.downloadFile('a.download-btn', {
  outputDir: './downloads',
  filename: 'report.pdf',  // optional — otherwise uses the suggested name
});

// result = { filename: 'report.pdf', path: '/abs/path/downloads/report.pdf', url: '...' }
```

### Tab management

```javascript
await browser.newPage();            // open a new tab (becomes active)
await browser.switchToPage(0);      // switch tab by index
const pages = browser.getPages();   // list all open tabs
await browser.closePage();          // close current tab
```

### JavaScript evaluation

```javascript
const title = await browser.evaluate(() => document.title);

const links = await browser.evaluate(() =>
  Array.from(document.querySelectorAll('a'), a => ({ href: a.href, text: a.textContent }))
);

const result = await browser.evaluate(([x, y]) => x * y, [7, 8]);
```

### Network

```javascript
// Block resource types (speed up page loads)
await browser.blockResources(['image', 'stylesheet', 'font', 'media']);

// Monitor requests
browser.onRequest(req => console.log('>>', req.method(), req.url()));
browser.onResponse(res => console.log('<<', res.status(), res.url()));
```

### Working with PDF

Chrome extensions (Adobe Acrobat, Google PDF Viewer) can intercept navigation to PDF files, making them unavailable through `getHtml()`. Use the PDF-specific methods:

```javascript
// Extract PDF text from a URL
const { text, totalPages } = await browser.getPdfText('https://example.com/paper.pdf');
console.log(`${totalPages} pages, ${text.length} chars`);

// Save the PDF to disk and extract text
const result = await browser.getPdfText('https://example.com/paper.pdf', {
  saveTo: './downloads/paper.pdf',
});

// Per-page text (not merged)
const { text: pages } = await browser.getPdfText('https://example.com/paper.pdf', {
  mergePages: false,
});
pages.forEach((pageText, i) => console.log(`Page ${i + 1}: ${pageText.length} chars`));
```

#### `getPdfText(url, options)`

| Option | Type | Default | Description |
|-------|-----|---------|-------------|
| `mergePages` | `boolean` | `true` | Merge all pages into a single string |
| `saveTo` | `string` | — | Save the PDF binary to this path |

Result: `{ text: string|string[], totalPages: number, savedTo?: string }`

#### `gotoOrPdf(url, options)`

Universal navigation: automatically detects PDFs (by URL extension or via extension interception) and extracts text instead of navigating.

```javascript
const result = await browser.gotoOrPdf('https://example.com/paper.pdf');
if (result.isPdf) {
  console.log(`PDF: ${result.totalPages} pages, ${result.text.length} chars`);
} else {
  const html = await browser.getHtml();
}
```

| Option | Type | Description |
|-------|-----|-------------|
| `mergePages` | `boolean` | For PDFs: merge pages |
| `savePdfTo` | `string` | For PDFs: save the file |
| *...rest* | — | All `goto()` options for non-PDF URLs |

Result: `{ isPdf: boolean, response?, text?, totalPages?, savedTo? }`

### Closing and shutdown

```javascript
await browser.close();    // close/disconnect (Chrome stays alive in CDP/launchCDP modes)
await browser.shutdown(); // disconnect AND terminate Chrome (only if started via launchCDP)

// Static shutdown — kill Chrome on the CDP port (no instance required)
await BrowserUse.shutdown();
await BrowserUse.shutdown({ port: 9333 });
```

| Mode | `close()` | `shutdown()` |
|-------|-----------|-------------|
| `launchCDP` | Disconnects; Chrome stays alive | Disconnects and kills the Chrome process |
| `persistent` | Closes the context/browser; profile remains on disk | Same as `close()` |
| `cdp` | Disconnects; Chrome stays alive | Same as `close()` |

## Integration with getDataFromText

Via CLI — one command:

```bash
node utils/browserUse.js https://example.com/article --extract output.json
```

## CLI

```
Usage: node utils/browserUse.js <url> [options]
       node utils/browserUse.js --start [--profile <name>]
       node utils/browserUse.js --shutdown

Actions:
  --start               Start Chrome with CDP (stays running in background)
  --shutdown            Shut down background Chrome on CDP port and exit

Options:
  --profile <name>      Chrome profile name (default: AgentProfile)
  --cdp [endpoint]      Connect via CDP (default: http://localhost:9222)
  --headless            Run headless
  --html [file]         Get page HTML (save to file or print to stdout)
  --screenshot [file]   Take screenshot (default: screenshot.png)
  --full-page           Full-page screenshot
  --images [dir]        Download content images (default: ./images)
  --extract [file]      Extract data via getDataFromText (save or print)
  --wait <selector>     Wait for selector before performing actions
  --timeout <ms>        Navigation timeout (default: 30000)
  --help                Show this help
```

Operational logs go to stderr, data goes to stdout. This allows piping:

```bash
node utils/browserUse.js https://example.com --html > page.html
node utils/browserUse.js https://example.com --extract | jq '.content[0].markdown'
```

## Static utilities

```javascript
const BrowserUse = require('./utils/browserUse');

// Default profile path for the current OS
BrowserUse.getDefaultUserDataDir();             // AgentProfile
BrowserUse.getDefaultUserDataDir('Work');       // Work

// Shell command for manually launching Chrome with CDP
BrowserUse.getCdpLaunchCommand();
BrowserUse.getCdpLaunchCommand({ profile: 'Work' });
BrowserUse.getCdpLaunchCommand({ port: 9223, userDataDir: './my-profile' });

// Kill Chrome on a CDP port
await BrowserUse.shutdown();
await BrowserUse.shutdown({ port: 9333 });
```

## Limitations

- **One profile — one process.** Chrome does not allow multiple instances using the same `userDataDir`.
- **Do not use your main profile.** Automating the main Chrome profile is restricted by Chromium policy (Chromium 136+).
- With `connectCDP`, Playwright connects with reduced fidelity compared to `launch`.
- Corporate browser policies may limit automation capabilities.

## Dependencies

- [playwright](https://playwright.dev/) — cross-browser automation
- Google Chrome — must be installed (`npx playwright install chrome`)

