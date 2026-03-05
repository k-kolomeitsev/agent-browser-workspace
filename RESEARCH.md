# Deep Research — in-depth research methodology

Instructions for running a high-quality research process via Google Search using the tools from [AGENTS.md](AGENTS.md).

Browser constraints (single-threaded, sequential access) are described in [AGENTS.md](AGENTS.md) and fully apply here.

> **CLI-first.** Prefer running the repository tools via CLI (`node scripts/...`, `node utils/...`). Avoid writing custom JavaScript scripts; only do it as a last resort when the CLI tools cannot perform a required step.

---

## File structure

Each research produces two artifacts:

```
archive/<YYYY>-<MM>-<DD>-<slug>/          ← working folder
  insights.md                              ← running draft of insights
  links.json                               ← stable snapshot of Google SERPs (across all queries)
  *.md, images/                            ← downloaded pages (getContent)
  any intermediate files and artifacts

researches/<YYYY>-<MM>-<DD>-<slug>.md     ← final report
```

**Slug** — a short topic identifier: lowercase Latin letters, dashes instead of spaces.

Example for the research “Where to monitor AI updates”, done on Feb 28, 2026:

- `archive/2026-02-28-ai-monitoring/` — working folder
- `archive/2026-02-28-ai-monitoring/insights.md` — insights
- `researches/2026-02-28-ai-monitoring.md` — final report

Create the working folder **at the start of Phase 1**, before the first browser call. Create `researches/` if needed.

---

## Phase 1: Planning

Before opening the browser, analyze the topic and make a plan.

### 1.1 Decompose the topic

Break the topic into **5–8 aspects** (dimensions / subtopics). For each aspect, define what kind of information is needed (facts, lists, comparisons, opinions, statistics).

Example for “Where to monitor AI updates”:

- Research paper aggregators
- Model release trackers
- Email newsletters
- Social networks and communities (Reddit, X, Discord, Bluesky)
- YouTube channels
- Developer platforms (GitHub, HF Spaces, daily.dev)
- AI company blogs (primary sources)
- Analytics / summary resources

### 1.2 Tie it to the task context

If the research request mentions **specific examples** (products, models, projects, tools, companies), list them and plan dedicated queries for each to trace the **appearance chain**:

author → primary source publication → distribution channels → where it could be discovered earliest

Outcome: a **mapping table** in the final report (see Phase 5) linking each example to its type, primary source, and first discovery channel. This gives the reader an actionable model: “a novelty of type X → watch channel Y”.

Example: if the task includes “Qwen3.5, KittenTTS, AnchorWeave, Cohere Tiny Aya”, determine for each whether it is an arXiv preprint, a corporate announcement, or an open-source project — and where it appeared first.

### 1.3 Search queries

Form **15–20 queries** (minimum 12). Queries must:

- **Cover different aspects** from 1.1 (do not restate the same thing in different words).
- **Target different source types**: overview articles, “best of” lists, comparisons, forum discussions, official sites.
- **Use English as the primary query language**. Add queries in the report language only if needed.
- **Include the year** (`2025`, `2026`) to get current results.
- **Include discovery queries** (2–4) — intentionally looking for niche/less-known/alternative resources that don’t show up in standard top results. Examples:
  - `"underrated tools for tracking AI"`,
  - `"lesser-known alternatives to [main resource]"`,
  - `"hidden gems AI monitoring"`,
  - `"what AI tools are people missing"`.
- **Include example-specific queries** from 1.2 (if any) — to trace their appearance chain.

Example query set (same topic):

```
Core (by aspects):
1. best AI newsletters 2026
2. how to track new AI model releases real-time
3. hugging face daily papers how it works
4. best AI news aggregators 2026
5. AI research papers tracking tools arxiv
6. reddit communities for AI news LLM
7. best twitter accounts for AI news 2026
8. AI YouTube channels research papers review
9. AI company blogs to follow 2026 OpenAI Google Anthropic
10. developer platforms for AI news daily.dev github trending
11. AI discord servers for researchers 2026
12. AI model benchmarks leaderboards comparison sites
13. Hacker News AI coverage statistics
14. Bluesky ML AI community
15. best RSS feeds for AI news monitoring

Discovery (niche/alternative):
16. underrated tools for tracking AI research 2026
17. hidden gem AI news sources developers don't know about
18. AI product launch platforms product hunt alternatives

Example-specific (from the task):
19. "KittenTTS" OR "AnchorWeave" where first published
20. Qwen3.5 release announcement original source
```

### 1.4 Initialize files

> **⚠️ CRITICAL.** Proper initialization ensures the final report path is never lost during long runs and context compression.

1. Define paths:
   - Working folder: `archive/<YYYY>-<MM>-<DD>-<slug>/`
   - Final report: `researches/<YYYY>-<MM>-<DD>-<slug>.md`

2. Create the working folder and `researches/` (if they don’t exist).

3. **Create the final report file** `researches/<YYYY>-<MM>-<DD>-<slug>.md` with a stub:

   ```markdown
   # [Research topic]

   > Research in progress. The report will be written here when finished.
   ```

4. **Create `insights.md`** in the working folder. The first lines must be an **anchor header** with the final report path:

   ```markdown
   > **FINAL REPORT →** `researches/<YYYY>-<MM>-<DD>-<slug>.md`

   ---
   ```

   This anchor is your main “safety rope”. The agent reads `insights.md` at each phase and always sees where to write the final output. **Never delete it and never push this line down.**

5. All subsequent `getContent` calls must use the working folder as `dir`.

---

## Phase 2: Data collection

### 2.1 Process

Take one query at a time. For each query:

1. Run the search and save a stable SERP snapshot (`links.json`) via CLI:

   ```bash
   node scripts/googleSearch.js "<query>" --links --dir archive/<YYYY>-<MM>-<DD>-<slug>/
   ```

   - **Why this matters:** `links.json` is a stable artifact. It lets you continue research without re-running Google Search (which may change results). You can simply open the desired URL from the file.
   - **Saving rule:** always pass `--dir archive/<...>/` even when you only print links (`--links`) so results are appended into `archive/<...>/links.json`.

   The `links.json` format is documented in [`scripts/googleSearch.md`](scripts/googleSearch.md).

2. For each relevant result index `i`, open it and save content to Markdown via CLI:

   ```bash
   node scripts/googleSearch.js "<query>" --open <i> --dir archive/<YYYY>-<MM>-<DD>-<slug>/ --name <page-slug>.md
   ```

3. Immediately after each saved page, append insights to `insights.md` **on disk** (NOW, not later).

**Writing insights is part of the link-processing loop, not a separate later step.** Do not open the next link until you have written insights from the current page to disk. This guarantees that if the agent gets interrupted, everything already processed is preserved.

4. **Empty or minimal content.** If `getContent` returns empty Markdown or only header/menu — the page likely renders via JavaScript. Do not skip it: apply the **escalation strategy** from “Reliable content extraction” in [AGENTS.md](AGENTS.md):

   - Try a CLI “wait + extract” run: `node utils/browserUse.js <url> --wait "<selector>" --extract out.json`
   - Take a screenshot for visual verification: `node utils/browserUse.js <url> --screenshot debug.png --full-page`
   - If the page requires scrolling to load content — automated scrolling is an advanced / last-resort case (API).

   Record whatever you obtained in `insights.md` (text, screenshot summary). Only if none works — record the failure + URL and move on.

5. **Infinite-scroll pages.** If the page loads content on scroll (feeds, forums, trending pages), fully automated scrolling requires programmatic control (API) and should be treated as a last resort. If the page is critical, capture what you can (content/screenshot), record the limitation in `insights.md`, and only then consider a minimal script.

6. For each query, review **5–10 relevant links**. If page 1 is weak — collect links from additional pages via CLI (example):

   ```bash
   node scripts/googleSearch.js "<query>" --page 2 --links --dir archive/<YYYY>-<MM>-<DD>-<slug>/
   ```

**Target volume: 40–60 pages reviewed** across the entire research.

> It is **forbidden** to keep insights “in memory” and write them in bulk later (after multiple pages or at the end of the query). Each page = immediate write to disk.

### 2.2 Cross-link harvesting (snowball sampling)

While reading each page, record in `insights.md` all **mentioned resources, tools, and platforms** that you don’t already have in your notes — even if they are only mentioned briefly. Mark them as `[candidate]`:

```markdown
### Candidates from [Page title](URL)

- [candidate] daily.dev — mentioned as a developer-focused aggregator
- [candidate] Product Hunt AI — mentioned as a launch platform
- [candidate] LLM Changelog — release-notes tracker
```

These candidates are inputs for Phase 3 (gap analysis). This is how you discover niche resources that don’t rank high for your main queries but are referenced inside good overview pages. **Do not evaluate candidates in this phase** — only capture them. Evaluation/filtering happens in Phase 3.

### 2.3 Insights format (`insights.md`)

Each insight is a separate block:

```markdown
### [Short insight title]

**Source:** [Page title](URL)
**Query:** the search query that found the page

- Extracted fact 1
- Extracted fact 2
- Concrete numbers, dates, names — everything that might be useful later
```

Must-have:

- **URL and source title** are written immediately, not “later”. Each fact is tied to a concrete source.
- **Numeric data** — capture exact values from the source, not a memory-based paraphrase.
- **Don’t only paraphrase** — write down facts, numbers, names, URLs. You can add your own interpretation, but mark it as `[my take]`.

---

## Phase 3: Gap analysis

After finishing the main collection:

1. **Review** `insights.md` end-to-end.
2. **Compare** your collected data against the plan from Phase 1. List aspects that are weakly covered or not covered at all.
3. **Process snowball candidates** from Phase 2.2: collect all `[candidate]` entries, deduplicate, and for each new candidate do a quick validation (open its page, assess relevance, write an insight or discard).
4. **Form 5–8 additional targeted queries** to close the gaps.
5. **Repeat Phase 2** for these queries.

Typical gaps to check:

- Did you cover **all source types** (not only newsletters, but also forums, dev platforms, company blogs)?
- Did you cover **adjacent categories** — resources not answering the question directly but useful in context? Examples: developer aggregators (daily.dev), product launch platforms (Product Hunt), datasets/databases (Kaggle), changelog trackers, chronological timelines.
- Did you capture **alternative/niche resources** that didn’t show up in top Google results but were referenced inside the pages you read (snowball candidates)?
- Do you have **quantitative data** where you currently only have qualitative descriptions?
- Did you cover **recent ecosystem changes** (shutdowns, acquisitions, migrations)? For each resource you recommend as active, confirm it actually works at the time of research.

---

## Phase 4: Cross-verification

Before writing the report, verify key data:

- **Numeric data** (audience, subscribers, number of models, dates) — confirmed by **at least two independent sources**. If sources disagree, take a **conservative** value and note the discrepancy if needed.
- **Resource status** — for each resource recommended in the report, **open its URL in the browser** and verify it is active and matches the description. Do not rely on secondary descriptions. If the resource shut down, migrated, was acquired, or changed substantially — record it as a fact in the report (example: “Papers With Code was shut down by Meta in 2025; trending papers moved to Hugging Face”).
- **Claims like “first”, “only”, “largest”** — require evidence or must be softened to “one of…”.
- **People/company names and social handles** — verify spelling.

---

## Phase 5: Writing the report

> **⚠️ Before you start:** open `insights.md` and read the **first line** — it contains the final report path (written in Phase 1.4). The report file already exists — overwrite its contents with the finished report.

### 5.1 Structure

The final report must be saved to the file path in the first line of `insights.md` (format: `researches/<YYYY>-<MM>-<DD>-<slug>.md`).

1. **Title and metadata**
   - Report topic
   - Research date
   - Method: number of queries, pages reviewed, sources used

2. **Introduction** — 2–4 sentences of context. Include 1–2 current facts with links to justify relevance.

3. **Mapping table** (if the task included specific examples) — a table linking each example to its type, primary source, and earliest discovery channel. Place it right after the introduction (before the main sections). It gives the reader a “map” that the sections then expand.

   Example format:

   | Item | Type | Primary source | Where to spot it first |
   |---|---|---|---|
   | Qwen3.5 | Model (open-source) | HF Model Hub | HF Daily Papers, X (@akhaliq) |
   | AnchorWeave | Research | arXiv + GitHub Pages | arXiv cs.CV, HF Papers |
   | Cohere Tiny Aya | Corporate blog | cohere.com/blog | Newsletters, X |

4. **Main sections** — grouped by thematic categories (not by the search order). Each section includes:
   - **A descriptive paragraph** — what this category is, why it matters, how it fits the landscape.
   - **Concrete resources/facts** — with footnotes `[^N]` to sources.
   - **Comparison tables** — where appropriate for structured presentation.
   - **Practical recommendations** — what to do with it, which resource fits which use.

5. **Strategy / recommendations** — the practical end result: a concrete plan of action for the reader.

6. **References section** — a numbered list of **all** sources used:

   ```
   1. [Page title](URL) - Short note...
   2. [Page title](URL) - Short note...
   ```

### 5.2 Sources and footnotes

- **Every non-trivial fact** must have a footnote `[^N]`.
- Place the footnote **at the first mention** of a fact from a given source. One source can back multiple facts.
- The References section at the end must include **all** sources — both those cited via footnotes and additional verification sources.

### 5.3 Resource descriptions

For each resource you describe, provide:

- **Name and URL** — exact and verified.
- **What it is** — 1–3 sentences with specifics (not “a good resource”, but what it does and how it differs).
- **Concrete data** — audience, update frequency, scope, authors — with sources.
- **Context** — how it fits the ecosystem, what it is connected to, who uses it.

Bad: `TLDR AI is a good daily AI digest.`

Good: `TLDR AI (tldr.tech/ai) is a daily digest with an audience of 500K+ subscribers[^25], covering research, news, and tools as ultra-short summaries. It targets developers and data scientists.[^26]`

### 5.4 Visual elements

Where appropriate, use:

- **Process diagrams** (ASCII) — for chains, flows, dependencies.
- **Tables** — for comparing similar objects across multiple dimensions.
- **Grouped lists** — for categorization.

Visual elements complement the text but **do not replace** descriptive paragraphs.

---

## Quality checklist

Go through every item before finalizing the report:

| # | Criterion | Threshold |
|---|---|---|
| 1 | Search queries | 15+ |
| 2 | Pages reviewed | 40+ |
| 3 | Unique sources in References | 30+ |
| 4 | Facts with numbers have footnotes | All |
| 5 | Numeric data verified by 2+ sources | Key facts |
| 6 | No uncovered aspects from the plan (Phase 1) | All covered |
| 7 | Each category has a descriptive paragraph | All |
| 8 | Date and method are stated | Yes |
| 9 | Practical recommendations are present | Yes |
| 10 | No claims without sources | Yes |
| 11 | Snowball candidates from Phase 2.2 are processed | All |
| 12 | Recommended resources verified as active (Phase 4) | All |
| 13 | Mapping table (if examples exist) | Yes |
| 14 | Report saved to the file from the first line of `insights.md` | Yes |

If at least one criterion is not met — improve the report before finalization.

---

## Common mistakes

| Mistake | How to avoid |
|--------|-------------|
| Too few queries (3–5) | Plan 15–20 before you start searching |
| Too few pages (10–20) | Aim for 40–60, track the count |
| Missing References | Write URLs into `insights.md` for each extraction — assembling References later becomes trivial |
| Inflated numbers (audience, ratings) | Verify via 2+ sources, take a conservative estimate |
| Entire categories missing | Compare against Phase 1 plan, do gap analysis in Phase 3 |
| Only tables, no explanations | Every table must have a context paragraph |
| Source title without URL | Write `[Title](URL)` at the moment of extraction |
| Missing resources mentioned inside pages | Snowball sampling (Phase 2.2): capture all `[candidate]`, process in Phase 3 |
| Recommending a shut-down/migrated resource | Live-visit each resource URL in Phase 4; record status changes |
| No link to example items from the task | Mapping table (Phase 1.2 + Phase 5): examples → type → primary source → channel |
| Too few references for a large list of resources | Each resource needs at least 1 reference; discovery queries + snowball add extra sources |
| Lost the final report path after context compression | The path is in the first line of `insights.md` (Phase 1.4) — read it before Phase 5 |

