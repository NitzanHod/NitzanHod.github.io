# Maintenance Guide — NitzanHod.github.io

This file is for Claude (or any future maintainer) reopening this repo with no prior context. The site is built on al-folio v1.x; the maintainer-supplied context lives in `CLAUDE.md`, `AGENTS.md`, and `docs/`. Read those if you need to understand the plugin/gem architecture. **What follows is what's specific to *this* personalized site** and the recipes for the most common tasks.

This file is excluded from the Jekyll build via the `exclude:` block in `_config.yml`, so it will never be published.

---

## Site essentials

- **Live URL:** https://nitzanhod.github.io/
- **GitHub repo:** https://github.com/NitzanHod/NitzanHod.github.io
- **Local clone:** `~/Desktop/claude_workspace/NitzanHod.github.io/`
- **Owner inputs / source assets:** `~/Desktop/claude_workspace/nitzan_info/` — CV-style RTFs, paper PDFs, and prepared `*_image` files for paper previews. The user drops new material here.
- **Style reference:** https://yoadtew.github.io/ — also al-folio. Match its look when in doubt.

## Architecture in one paragraph

al-folio v1 is a thin starter. Almost all rendering lives in versioned plugin gems (`al_folio_core`, `al_citations`, `al_search`, …). This repo owns only `_config.yml`, `_data/*`, content collections (`_pages`, `_news`, `_projects`, `_bibliography`), and `assets/`. Two **local layout overrides** in `_layouts/` (about.liquid and bib.liquid) shadow gem files because the home-page reorder and the publication-card style cannot be expressed via config alone. CLAUDE.md says local overrides are allowed but tracked — keep them small and only change what config can't.

## What's already personalized

| Concern | File |
|---|---|
| Site identity (name, url, description, favicon emoji) | `_config.yml` (top section) |
| Author-name highlighting in bib | `_config.yml` → `scholar.last_name: [Hodos]`, `scholar.first_name: [Nitzan, N.]` |
| Show all authors (no "and N more" collapse) | `_config.yml` → `max_author_limit:` (blank) |
| Disable demo blog/external posts in Cmd-K search | `_config.yml` → `posts_in_search: false`, `external_sources:` empty |
| Bio text + page order | `_pages/about.md` (body) + `_layouts/about.liquid` (override) |
| Profile photo | `assets/img/prof_pic.jpg` |
| Social icons (no RSS) | `_data/socials.yml` |
| Publications list | `_bibliography/papers.bib` |
| Paper preview images (1.36:1 white-padded) | `assets/img/publication_preview/*.png` |
| Venue chip text/color/link | `_data/venues.yml` (key matches each entry's `abbr` field) |
| Coauthor links (e.g. Schuster) | `_data/coauthors.yml` |
| Publication card layout (image left, name in `<strong>`, no periodical line, venue chip in buttons row) | `_layouts/bib.liquid` (override) |
| Publication card style for `/publications/` page | shares the same `_layouts/bib.liquid` |

## How the site goes live

`.github/workflows/deploy.yml` runs on every push to `main`, builds Jekyll, and pushes the `_site/` to a `gh-pages` branch. **GitHub Pages is configured to serve from `gh-pages`**, not main. After a push, build takes ~2 min; do a hard-refresh (Cmd+Shift+R) before troubleshooting because GitHub edge caching is aggressive.

To check status:

```bash
# Replace TOKEN with a fresh PAT (Contents: read+write).
curl -s -H "Authorization: token TOKEN" \
  "https://api.github.com/repos/NitzanHod/NitzanHod.github.io/actions/runs?per_page=2" \
  | python3 -c "import json,sys; [print(r['name'], r['status'], r['conclusion']) for r in json.load(sys.stdin)['workflow_runs']]"
```

If a deploy fails: the most useful logs are in the GitHub Actions UI (Settings → Actions → workflow run). Don't bypass hooks or force-push to main; if the build is broken, fix the cause and add another commit.

> ⚠️ **Tokens.** Treat any PAT pasted in chat as compromised — push, then immediately revoke at https://github.com/settings/tokens. Prefer asking the user to push from their own authenticated terminal (`! cd ~/Desktop/claude_workspace/NitzanHod.github.io && git push`) over taking tokens.

## Recipes

### Add a new publication

The user typically drops a paper PDF and a prepared figure at `~/Desktop/claude_workspace/nitzan_info/`. Steps:

1. **Confirm metadata** — title, full author list (in order), venue + year, arxiv ID, code URL. Don't read the PDF; ask if it's not already in the user's notes file (e.g., `format and content.rtf`).
2. **Image** — convention: name it `<key>.png` and put it in `assets/img/publication_preview/`. Pad it to **1418×1042 (≈1.36:1) with white background** so it matches the others. Quick recipe (Pillow is already on the user's machine):
    ```python
    from PIL import Image
    target_ratio = 1418/1042
    img = Image.open(src).convert('RGB')
    w,h = img.size; cur = w/h
    if cur < target_ratio:
        new_w, new_h = int(round(h*target_ratio)), h; off = ((new_w-w)//2, 0)
    else:
        new_w, new_h = w, int(round(w/target_ratio)); off = (0, (new_h-h)//2)
    canvas = Image.new('RGB', (new_w,new_h), (255,255,255)); canvas.paste(img, off)
    canvas.save(dst, 'PNG')
    ```
    If the source is a PDF figure: `qlmanage -t -s 1600 -o . source.pdf` produces `source.pdf.png`.
3. **Bib entry** — append to `_bibliography/papers.bib` as `@misc` (NOT `@inproceedings` — that triggers the "In <booktitle>, <year>" periodical line which we suppress). Required keys: `abbr` (matches a `_data/venues.yml` entry), `title`, `author`, `year`, `preview`, plus optional `arxiv`, `code`, `website`, `pdf`. Equal-contribution authors get an asterisk after the **last name** (`Hodos*, Nitzan`); the layout renders it as a superscript.
4. **Venue chip** — if the venue isn't already in `_data/venues.yml`, add an entry. Use the existing **purple `#5a3eaf`** unless the user explicitly asks for a different color. Key must equal the bib `abbr` exactly (e.g., `"NeurIPS 2026"`).
5. **Coauthor link** (optional) — to make a co-author's name a clickable link, add them to `_data/coauthors.yml`. Key is **lowercased last name without accents**; value is a list of `{firstname: [list], url: ...}` entries.
6. Commit + push. Wait for the deploy and ask the user to hard-refresh.

### Update profile photo

1. User drops a new image somewhere. Copy it to `assets/img/prof_pic.jpg` (overwrite — JPG keeps the existing references valid).
2. Commit + push. The al-folio cache-bust plugin appends a hash, so no manual cache invalidation is needed.

### Edit bio / about-page text

`_pages/about.md` body — Markdown with HTML allowed. The wrapping layout (`_layouts/about.liquid`) handles the header, profile-image float, social-icons row, and inlined publications. **Don't put `{% bibliography %}` inside about.md** — the layout already injects it after a `clear:both` so the floated photo can't bleed into the publications grid.

### Edit social links

`_data/socials.yml`. Plugin: [jekyll-socials](https://github.com/george-gca/jekyll-socials). Display order = file order. Supported keys (currently used): `email`, `scholar_userid`, `semanticscholar_id`, `github_username`, `linkedin_username`, `x_username`. **Do not add `rss_icon`** — there is no blog. Adding any key, even with a falsy value, makes it render. To remove an icon, delete the line.

### Change a component's appearance

The right place depends on what you're changing:

| Change | Where |
|---|---|
| Bio text | `_pages/about.md` body |
| Order of bio / image / socials / publications on home | `_layouts/about.liquid` |
| Header name bolding | `_layouts/about.liquid` (`<span class="font-weight-bold">`) |
| Publication card structure (image size, button order, name styling, what fields render) | `_layouts/bib.liquid` |
| Venue chip color/text/link | `_data/venues.yml` |
| Coauthor link (clickable name) | `_data/coauthors.yml` |
| Site-wide colors / fonts | gem-owned (`al_folio_core`). Per `docs/CUSTOMIZE.md`, override locally with `_sass/_themes.scss` + `_sass/_variables.scss`. Don't touch tailwind tokens. |
| Navbar items | `_pages/*.md` front-matter (`nav: true`, `nav_order: N`). Currently only `about` (implicit) and `publications`. |

When in doubt: **prefer config + data + content edits over layout overrides**. Each new override increases drift risk when the upstream gem updates. CLAUDE.md spells this out — the maintainer's `bundle exec al-folio upgrade overrides audit` tracks every override in `.al-folio-overrides.yml`.

### Tweak the publications page

`/publications/` and the home page share `_layouts/bib.liquid`, so any visual tweak (image col width, button order, periodical line, etc.) propagates to both. The `/publications/` page additionally enables `bib_search` (the inline search box).

## Known intentional deviations from al-folio defaults

- **`booktitle` removed from bib entries**, paper type set to `@misc`. Reason: user wants the venue chip alone, not the "In <full venue name>, <year>" periodical line.
- **`max_author_limit` blanked.** Reason: user wants the full author list visible (4–5 names per paper).
- **`<strong>` not `<em>` for the user's own name.** Reason: aesthetic choice — the user explicitly asked for bold over italic-underline.
- **Venue chip moved out of the image column** into the buttons row. Reason: ICLR/NAACL/ICML chip on top of the figure was a visual header the user didn't ask for.
- **Publications inlined on home page** under `<h2 id="publications">Publications</h2>`. The separate `/publications/` page also exists (with bib_search enabled). Navbar link goes to `/publications/`.
- **All venue chips use one purple `#5a3eaf`.** Reason: a single conference shouldn't stand out by color.
- **Image padding to 1.36:1.** Reason: paper figures vary widely in aspect; cfnn is the reference.

## Things to NEVER do without explicit user approval

- Touch files outside `~/Desktop/claude_workspace/` — this is a personal Mac with unrelated work.
- Force-push to main, rewrite history, or delete branches.
- Add tracking (Google Analytics etc.) — `analytics:` block in `_config.yml` is intentionally empty.
- Re-enable `external_sources` or any `posts_in_search` content; we explicitly purged the demo blog noise.
- Touch gem-owned files via the `gems/` or `vendor/` paths. Stick to `_layouts/` overrides for visual changes.

## Reference reading inside the repo

- `CLAUDE.md` — full v1 architecture, plugin map, gating model.
- `AGENTS.md` — short routing rules + validated command set.
- `docs/CUSTOMIZE.md` — comprehensive customization guide (bib fields, theme colors, navbar mechanics).
- `docs/INSTALL.md` — deploy mechanics (we use the `gh-pages` branch model documented under "For personal and organization webpages").
- `docs/BOUNDARIES.md` — area→gem ownership table; consult before any layout edit.
- `.agents/skills/al-folio-bootstrap/SKILL.md` — the bootstrap workflow we already followed.
