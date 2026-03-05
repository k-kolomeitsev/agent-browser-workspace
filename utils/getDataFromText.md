# get-data-from-text

Cascade extractor for structural blocks from HTML pages: navigation, main content, forms. Extracted blocks are converted to Markdown via [Turndown](https://github.com/mixmark-io/turndown).

> **CLI-first.** Prefer using the CLI (`node utils/getDataFromText.js ...`) for tasks. The API is intended for tool authors and advanced integrations. See `AGENTS.md` for the policy.

## Install

```bash
npm install
```

## Quick start

### CLI

```bash
# Extract with Markdown conversion (default)
node utils/getDataFromText.js page.html

# Save result to a file
node utils/getDataFromText.js page.html output.json

# Raw HTML only, without Markdown conversion
node utils/getDataFromText.js page.html output.json --raw
```

### API (advanced)

The module can also be imported from Node.js, but the recommended interface is the CLI. See the **API options** section below.

## Output format

```javascript
{
  metadata: {
    title: "Page title",
    lang: "en",
    description: "...",
    jsonLd: { /* Schema.org entity */ },
    jsonLdContent: "articleBody text from JSON-LD"
  },

  navigation: [
    {
      html: "<nav>...</nav>",
      markdown: "- [Home](/)\n- [About](/about)\n...",
      tag: "nav",
      selector: "nav",
      cssSelector: "nav",
      tier: 1,
      confidence: 0.95,
      type: "primary_nav",
      evidence: ["semantic:nav", "high_link_density:0.88", "link_count:5"],
      features: {
        textLength: 45,
        wordCount: 6,
        linkDensity: 0.882,
        linkCount: 5,
        headingCount: 0,
        paragraphCount: 0,
        listItemCount: 5,
        ttr: 3.75
      }
    }
  ],

  content: [
    {
      html: "<article>...</article>",
      markdown: "# Title\n\nArticle text...",
      tag: "article",
      selector: "article",
      cssSelector: "article",
      tier: 2,
      confidence: 0.95,
      type: "main_content",
      evidence: ["semantic:article", "low_link_density", "headings:3", "paragraphs:5"],
      features: { /* same shape as navigation.features */ }
    }
  ],

  forms: [
    {
      html: "<form action='/search'>...</form>",
      tag: "form",
      selector: "[role=\"search\"]",
      cssSelector: "[role=\"search\"]",
      tier: 1,
      confidence: 0.95,
      type: "search",
      evidence: ["role:search", "has_search_input"],
      features: {
        inputCount: 1,
        selectCount: 0,
        textareaCount: 0,
        buttonCount: 1,
        checkboxCount: 0,
        radioCount: 0,
        hasPasswordInput: false,
        hasSearchInput: true,
        hasEmailInput: false,
        hasFileInput: false,
        hasSubmitButton: true,
        labelCount: 0,
        isWrappedInForm: true,
        action: "/search",
        method: "GET"
      }
    }
  ]
}
```

`markdown` is present in `navigation` and `content` blocks by default. With `--raw` / `{ raw: true }` it is omitted. `forms` blocks are always returned as raw HTML.

### `selector` vs `cssSelector`

- **`selector`** — “origin label” (which rule/heuristic found the block). It may not be valid CSS (e.g. `scoring`, `baseline:body`, `regex:...`).
- **`cssSelector`** — a valid CSS selector (when it can be safely determined). Used by higher-level tools (e.g. `getContent`) to scope DOM operations.

## Pipeline

Execution order:

```
HTML ─→ Parse (cheerio)
     ─→ Extract metadata (title, lang, JSON-LD)
     ─→ Detect forms (BEFORE cleanup — inputs/selects/textarea still in DOM)
     ─→ Remove junk (script, style, hidden, empty nodes)
     ─→ Detect navigation (5-tier cascade)
     ─→ Detect content (5-tier cascade + scoring fallback)
     ─→ Clean content (remove noise inside content blocks)
     ─→ Convert to Markdown (Turndown)
     ─→ Result
```

## Detection cascades

### Navigation (5 tiers)

| Tier | Method | Examples |
| ---- | ------ | -------- |
| 1 | Semantic tags / ARIA | `<nav>`, `[role="navigation"]` |
| 2 | CSS classes | `.nav`, `.menu`, `.navbar`, `.main-menu` |
| 3 | CSS IDs | `#nav`, `#navigation`, `#menu` |
| 4 | Regex class/id + structural | `header > ul` with many links |
| 5 | Statistical | link density > 0.5, link count > 3, avg anchor ≤ 4 words |

Subtypes: `primary_nav`, `footer_nav`, `breadcrumbs`, `toc`, `pagination`.

### Content (5 tiers + fallback)

| Tier | Method | Examples |
| ---- | ------ | -------- |
| 1 | Semantic tags / ARIA | `<main>`, `[role="main"]` |
| 2 | Article | `<article>` |
| 3 | Microdata | `[itemprop="articleBody"]` |
| 4 | CSS classes | `.content`, `.entry-content`, `.post-content` |
| 5 | CSS IDs | `#content`, `#main-content`, `#main` |
| 6 | Readability scoring | heuristics: text, commas, link density, class weights |

The scoring fallback is a simplified Readability.js-like algorithm: tag base score, class weights (+25/-25), link density penalty (`score *= 1 - ld`), parent/grandparent propagation.

If scoring doesn’t reach `MIN_EXTRACTED_SIZE` (250 chars) — baseline fallback: JSON-LD `articleBody` → `<article>` → `<p>` → `<body>`.

### Forms (4 tiers)

| Tier | Method | Examples |
| ---- | ------ | -------- |
| 1 | Semantic tags / ARIA | `<form>`, `[role="search"]`, `[role="form"]` |
| 2 | CSS classes / IDs | `.login-form`, `.search-box`, `.filters`, `.contact-form` |
| 3 | Regex class/id | boundary-aware regex for form patterns |
| 4 | Structural | containers with `input` + `button` without `<form>` |

Form types: `auth`, `search`, `filter`, `contact`, `subscribe`, `generic`.

Classification priority: ARIA role → class/id patterns → `action` URL → input types (password → auth, search → search) → placeholder text.

## Content cleaning

Inside extracted content blocks, the following are removed:

- elements matched by “noise selectors” (`.social`, `.share`, `.ad`, `.related`, `.comments`, `.cookie`, ...)
- elements with class/id matching a noise regex (sidebar, widget, promo, sponsor, ...)
- blocks with link density > 0.8 and link count > 2
- forms, empty elements

## Turndown configuration

Settings from `config-turndown`:

| Setting | Value |
| ---------------- | ---------------------- |
| Heading style | `atx` (`#`, `##`, ...) |
| Horizontal rule | `---` |
| Bullet | `-` |
| Code block style | `fenced` (````` ``) |
| Em delimiter | `_` |
| Strong delimiter | `**` |
| Link style | `inlined` |

## API options

| Option | Type | Default | Description |
| ------------ | --------- | ------------ | ------------ |
| `inputType` | `'file'` | auto | Force reading input as a file path |
| `outputFile` | `string` | — | Path to save JSON output |
| `raw` | `boolean` | `false` | Do not convert to Markdown (HTML blocks only) |

## Dependencies

- [cheerio](https://github.com/cheeriojs/cheerio) — server-side DOM parser
- [turndown](https://github.com/mixmark-io/turndown) — HTML → Markdown converter

