---
name: docs-as-static-html
description: Generate self-contained, terminal-retro static HTML documentation pages — single-file or multi-page, with Datatype variable font for inline charts, optional Mermaid diagrams, and browser-side syntax highlighting. Each output .html is fully self-contained (CSS + JS inlined; font and Mermaid and highlight.js loaded from CDNs at page open). Use this skill whenever the user asks for documentation, runbooks, README-style pages, internal wikis, API docs, knowledge base entries, release notes, design specs, project handoff docs, or any "static HTML doc" — including when they just say "make me a doc page for X", "write up [thing] as HTML", "I need a one-pager I can email", or hand you content and ask for a published-looking page. Prefer this skill over plain Markdown, over a Word .docx, or over inline chat output any time the deliverable is a standalone documentation artifact the user will save, share, or host.
---

# docs-as-static-html

Generate self-contained terminal-retro HTML documentation pages. Each output is a single `.html` file that works opened from disk, served from any static host, attached to an email, or dropped on a NAS. There is no build script — Claude composes the HTML directly using the design system in `assets/`.

The aesthetic is "`man` rendered to web": monospace-first, high-contrast, phosphor-green-on-near-black by default (with paper-and-ink light mode and a system-auto toggle), and ASCII flourishes like `▌` heading bars and `[ NOTE ]` / `[ WARN ]` / `[ DANGER ]` callout labels. Charts and sparklines are drawn by the Datatype font itself directly inside running text. Diagrams render client-side via Mermaid. Code blocks highlight client-side via highlight.js.

## When to use this skill

Use this skill any time the deliverable is a **standalone documentation artifact** — something the user will save, share, host, or hand off. Strong signals:

- Words like _documentation, runbook, README, wiki page, knowledge base, API docs, design doc, handoff, release notes, postmortem, spec, one-pager_
- "Make me a doc / page / write-up for X"
- "Generate HTML docs for ..." or "build me a static page for ..."
- "I need a single file I can email / drop on S3 / put on my NAS / share"
- The user pastes content and asks for it "rendered nicely" or "as a page"

Do **not** use this skill for:

- Conversational answers, summaries, or analyses the user reads inline and doesn't save (use normal chat output)
- Word documents — when the user explicitly wants `.docx`, use the `docx` skill instead
- Slide decks — use `pptx`
- Spreadsheets — use `xlsx`
- Interactive web apps, dashboards, or React components (this skill is doc-focused — use a different skill for app UI)

## Workflow

The skill bundles three asset files and one HTML template. To generate a page, Claude:

1. Reads `assets/template.html`, `assets/styles.css`, `assets/runtime.js`.
2. Composes the body HTML for the page using the patterns below.
3. Substitutes placeholders in the template (a handful of string replacements).
4. Writes the result as a `.html` file and presents it.

No external tools required. See `references/content-patterns.md` for a longer walkthrough of how to shape good doc content for this format.

### Step 1: Decide page structure (single-page vs multi-page)

- **Single-page** if: under ~3,000 words AND at most ~6 top-level (`##`) sections AND the content reads as one continuous narrative.
- **Multi-page** if: over 3,000 words OR more than 6 top-level sections OR the content is clearly chaptered (e.g., multiple distinct topics that wouldn't share a TOC well).

When in doubt, prefer single-page — the sticky search and TOC keep long single pages navigable. Split only when the user implicitly expects chapters.

### Step 2: Compose the body HTML

Use these patterns (full reference in `references/content-patterns.md`):

**Headings** — every `<h2>` and `<h3>` needs an `id` for TOC linking. Slugify normally:

```html
<h2 id="architecture">Architecture</h2>
<h3 id="components">Components</h3>
```

**Code blocks** — use `data-filename` for the header strip; the runtime wraps it in the `.code-block` frame automatically:

```html
<pre><code class="language-python" data-filename="api.py">def hello():
    return "world"</code></pre>
```

**Callouts** — four variants: `note`, `tip`, `warn`, `danger`. The CSS prepends `[ NOTE ]` / `[ TIP ]` / `[ WARN ]` / `[ DANGER ]` labels in the appropriate accent colour, so just write the body:

```html
<div class="callout note">
  <p>Informational context the reader will appreciate.</p>
</div>

<div class="callout danger">
  <p>An operation that can lose data or cause downtime.</p>
</div>
```

**Datatype charts** — wrap chart expressions in `<span class="chart">` for the font to render them as inline visuals:

```html
<p>
  Memory pressure <span class="chart">{l:10,20,35,42,48,55,72,85}</span> has
  been climbing.
</p>
```

See `references/datatype-syntax.md` for the full expression reference.

**Mermaid diagrams** — wrap the source in a `.diagram.mermaid` div; Mermaid (loaded from CDN, see step 4) replaces it with rendered SVG:

```html
<div class="diagram mermaid">graph LR A --> B B --> C</div>
```

**Tables** — standard HTML tables. The CSS styles header rows in accent colour with uppercase headers and dashed row separators. Datatype charts in cells render beautifully.

**Keyboard keys** — `<kbd>Cmd</kbd>` + `<kbd>K</kbd>` for keyboard shortcuts.

### Step 3: Compose the structural pieces

These are HTML fragments that fill placeholders in the template.

**Title block** (`{{TITLE_BLOCK}}`):

```html
<section class="title-block">
  <h1>Screeps Private Server<span class="cursor"></span></h1>
  <p class="tagline">Self-hosted Screeps on a UGREEN NAS via Docker</p>
  <div class="meta">
    <span>v1.0</span>
    <span>2026-05-18</span>
    <span>by glen</span>
  </div>
</section>
```

The `<span class="cursor"></span>` is a blinking phosphor-green cursor at the end of the title. Always include it on h1.

**Table of contents** (`{{TOC_HTML}}`) — built from your h2/h3 ids:

```html
<nav class="toc" aria-label="On this page">
  <div class="toc-title">Contents</div>
  <ul>
    <li>
      <a href="#architecture">Architecture</a>
      <ul>
        <li><a href="#components">Components</a></li>
      </ul>
    </li>
    <li><a href="#operations">Operations</a></li>
  </ul>
</nav>
```

If the page is short (1-2 sections), the TOC adds noise — leave `{{TOC_HTML}}` empty.

**Man-page header bar** — set `{{MANHEAD_LEFT}}` / `{{MANHEAD_MID}}` / `{{MANHEAD_RIGHT}}` as plain text. Convention is `SHORTNAME(N)` on left/right and a category on the middle:

```
{{MANHEAD_LEFT}}  → SCREEPS(7)
{{MANHEAD_MID}}   → OPERATIONS
{{MANHEAD_RIGHT}} → SCREEPS(7)
```

Pick the short name from the doc's brand: keep the brand if it's ≤10 chars; otherwise use initials of multi-word titles (e.g. "Screeps Private Server" → "SPS"). Section number is `man`-style: 1 (general), 5 (config files), 7 (operations/runbooks), 8 (admin).

**Pagenav** (`{{PAGENAV_HTML}}`) — only for multi-page builds. Each page gets a prev/index/next row:

```html
<nav class="pagenav">
  <a class="prev" href="encounters.html">Encounter Mechanics</a>
  <a class="home" href="index.html">index</a>
  <a class="next" href="loot.html">Loot Budget</a>
</nav>
```

For single-page builds, leave it empty.

### Step 4: Decide which CDN scripts to include (`{{SCRIPTS}}`)

The template loads Datatype from Google Fonts unconditionally (it's how chart expressions render). Two more CDN scripts are optional, included only when needed:

**highlight.js** — include if the page has any `<pre><code>` blocks. The CSS already styles `.hljs-*` token classes; the script just applies them:

```html
<script src="https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11/build/highlight.min.js"></script>
<script>
  hljs.highlightAll();
</script>
```

**Mermaid** — include if the page has any `<div class="diagram mermaid">` blocks. Picks `dark` or `default` theme to match the page's resolved theme at load time:

```html
<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  const t = document.documentElement.getAttribute('data-theme');
  const isDark =
    t === 'dark' ||
    (!t && window.matchMedia('(prefers-color-scheme: dark)').matches);
  mermaid.initialize({
    startOnLoad: true,
    theme: isDark ? 'dark' : 'default',
    themeVariables: { fontFamily: '"JetBrains Mono", monospace' },
    securityLevel: 'strict',
  });
</script>
```

Concatenate both into `{{SCRIPTS}}` (with a blank line between) if the page needs both. Leave it empty if neither.

### Step 5: Assemble and save

Read the template and asset files, do the substitutions, write the result.

**OUTPUT LOCATION — this is a hard rule, not a default.**

All generated `.html` files go in a `docs/` directory:

- **Single-page**: `<project-root>/docs/<slug>.html`
- **Multi-page**: `<project-root>/docs/<slug>/index.html` plus sibling `<page>.html` files in that same subdirectory

`<project-root>` is the nearest ancestor of the current working directory that contains a `.git/`, `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, or `pom.xml`. If none is found, use the current working directory. Create `docs/` if it doesn't exist. Don't write to `/tmp`, `~/Downloads`, the project root itself, or anywhere else — even if the user doesn't explicitly say "put it in `docs/`", that is still where it goes.

`<slug>` is a kebab-case version of the page title — letters and digits only, lowercased, joined by single hyphens. `Screeps Private Server` → `screeps-private-server`. For multi-page, the directory takes the slug of the overall doc set and each page file takes its own slug.

Quick shell snippet to find the project root:

```bash
root=$(git rev-parse --show-toplevel 2>/dev/null) || \
root=$(pwd)
mkdir -p "$root/docs"
```

Then write to `"$root/docs/<slug>.html"`.

For the substitution itself (any tool that does string replacement is fine — Python, sed, or just composing the entire HTML inline with `create_file`):

```python
from pathlib import Path

SKILL = Path("<wherever the skill lives>")
template = (SKILL / "assets/template.html").read_text()
css = (SKILL / "assets/styles.css").read_text()
js = (SKILL / "assets/runtime.js").read_text()

html = template
for placeholder, value in {
    "{{TITLE}}": "Screeps Private Server",
    "{{DESCRIPTION}}": "Self-hosted Screeps on a UGREEN NAS via Docker",
    "{{BRAND}}": "Screeps",
    "{{MANHEAD_LEFT}}": "SCREEPS(7)",
    "{{MANHEAD_MID}}": "OPERATIONS",
    "{{MANHEAD_RIGHT}}": "SCREEPS(7)",
    "{{TITLE_BLOCK}}": title_block_html,
    "{{TOC_HTML}}": toc_html,
    "{{CONTENT}}": body_html,
    "{{PAGENAV_HTML}}": "",
    "{{FOOTER_LEFT}}": "docs-as-static-html",
    "{{FOOTER_RIGHT}}": "built 2026-05-18",
    "{{SCRIPTS}}": scripts_html,
    "{{CSS}}": css,
    "{{JS}}": js,
}.items():
    html = html.replace(placeholder, value)

# Always docs/ — never elsewhere.
import subprocess, os
root = subprocess.run(
    ["git", "rev-parse", "--show-toplevel"],
    capture_output=True, text=True
).stdout.strip() or os.getcwd()
out = Path(root) / "docs" / "screeps-private-server.html"
out.parent.mkdir(parents=True, exist_ok=True)
out.write_text(html)
```

If `docs/<slug>.html` already exists, overwrite it (the assumption is that the user is iterating on the same doc). If the user clearly means a _new_ doc with the same title, ask before clobbering.

### Step 6: Present

Use `present_files` with the generated `.html` file(s). The tool copies the file from `<project-root>/docs/` into the user-visible outputs directory, so the user sees the doc immediately — but the canonical, on-disk copy lives in `docs/`, where it stays after the conversation ends.

For multi-page output, present `index.html` first so the user lands on the entry page.

## A complete worked example

See `examples/single-page.html` for a fully composed page. Open it in a browser to see what the design system produces; read its source to see the exact HTML patterns to follow.

## A note on browser-side dependencies

Three CDNs are involved in the live rendering:

| Resource      | CDN                                              | When loaded            |
| ------------- | ------------------------------------------------ | ---------------------- |
| Datatype font | Google Fonts (`fonts.googleapis.com`)            | Always                 |
| highlight.js  | jsDelivr (`cdn.jsdelivr.net/gh/highlightjs/...`) | Pages with code blocks |
| Mermaid       | jsDelivr (`cdn.jsdelivr.net/npm/mermaid@11`)     | Pages with diagrams    |

If any CDN is unreachable, the page degrades gracefully:

- No Datatype → chart expressions appear as literal monospace text
- No highlight.js → code blocks render plain (still readable, just no colour)
- No Mermaid → diagram source displays as monospace inside the diagram container

For pages that truly need offline rendering, the user can self-host the assets and tweak the template — but for typical doc-on-the-web use, the CDN setup is what makes the skill setup-free.
