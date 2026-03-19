# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Pipeline

All improvements to Adam's Cinema follow this seven-stage pipeline using custom skills. **Do not skip stages.**

```
/scout → /architect → approve idea → /builder → /reviewer → /archivist → /ruca
```

| Stage | Skill | Purpose |
|---|---|---|
| 1 | `/scout` | Researches the topic via authoritative sources; produces a Research Brief for /architect |
| 2 | `/architect` | Reads the full codebase + Scout's brief, proposes 5–8 ranked feature ideas |
| 3 | *(approval)* | User selects an idea before any code is written |
| 4 | `/builder` | Implements the feature with a TodoWrite checklist, following all conventions |
| 5 | `/reviewer` | Audits new/modified code for security, structure, and design — outputs PASS/FAIL |
| 6 | `/archivist` | Updates staff.html, README.md, CLAUDE.md, and MEMORY.md to reflect the new feature |
| 7 | `/ruca` | Confirms branch, verifies edits, pushes to GitHub only if all checks pass |

Skills are defined in [.claude/skills/](.claude/skills/). `/reviewer` must PASS before `/archivist` runs. `/ruca` must PASS before any push occurs.

---

## Project Overview

**Adam's Cinema** — a static, no-build HTML/CSS/JS movie website with a vintage cinema aesthetic. There is no package manager, build step, or framework. Open any `.html` file directly in a browser.

## Architecture

All pages share one stylesheet ([style.css](style.css)) for the cinema theme (velvet dark palette, gold accents, crimson, curtain animations). Each HTML file has a `<style>` block for page-specific overrides.

### Pages

| File | Purpose |
|---|---|
| [index.html](index.html) | Grand Lobby — navigation hub linking all rooms |
| [adams-oracle.html](adams-oracle.html) | AI movie recommender — 14-question quiz → Claude API |
| [collection.html](collection.html) | Adam's curated personal film collection |
| [top25.html](top25.html) | Ranked genre/era/director lists |
| [list.html](list.html) | Universal list renderer — loaded via URL params from top25 |
| [indie.html](indie.html) | Independent & local theatre guide |
| [classic.html](classic.html) | Internet Archive public domain film browser |
| [concession.html](concession.html) | Fill-in-the-blank famous movie quotes game via Claude |
| [sixdegrees.html](sixdegrees.html) | Four Degrees of Cinema — Claude-propose + TMDB-verify chain builder; 1-hop, 2-hop, or 3-hop paths |
| [doublefeature.html](doublefeature.html) | Double Feature — enter one film, Claude suggests a paired double bill with programmer's note |
| [staff.html](staff.html) | Technical showcase / about page |

### External Services

Two Cloudflare Workers proxy all external API calls (no keys exposed client-side):

- **`https://oracle-proxy.adamrowe312.workers.dev`** — proxies Anthropic Claude API (`claude-sonnet-4-20250514`). Used by: Oracle (recommendation), Concession Stand (in-character quote responses), Six Degrees (chain building), Oracle Dossier (viewing guide second call), Double Feature (pairing suggestion).
- **`https://tmdb-proxy.adamrowe312.workers.dev`** — proxies TMDB API. Used for: trending/upcoming filmstrips (index), movie poster + backdrop (oracle result), similar films grid (oracle result), collection posters, Six Degrees film/credits resolution, Double Feature film poster resolution.
- **Wikipedia REST API** — `en.wikipedia.org/api/rest_v1/page/summary/{name}` — called directly from browser (open CORS, no key). Used for director bio panel in Oracle.
- **Internet Archive API** — `archive.org/advancedsearch.php` — called directly from browser (no credentials). Used by classic.html for public domain film browsing.

### Design System (style.css)

CSS custom properties define the full palette:
- `--gold / --gold-dim / --gold-pale / --gold-bright` — accent colors
- `--velvet / --velvet-mid / --velvet-light` — background layers
- `--crimson / --crimson-bright` — accent reds
- `--cream / --cream-dim / --cream-muted / --cream-ghost` — text scale

Shared structural elements (ceiling bar, floor stripe, curtains, scroll-rail) are defined in `style.css` and used across all pages.

### Oracle Flow (adams-oracle.html)

1. User answers 14 questions (multi-select, scale, text, dropdowns)
2. JS collects answers → POST to `oracle-proxy` Worker with a detailed system prompt
3. Worker calls Claude → returns `{ title, director, year, why, find }` JSON
4. JS fetches TMDB for poster + backdrop image, renders result card
5. JS fetches TMDB recommendations for a 3-card "similar films" grid
6. Second Claude call fires (max_tokens:180) → generates "On This Film" viewing dossier
7. Wikipedia fetch → renders director bio panel

### Filmstrip (index.html)

Fetches TMDB `now_playing` → builds a CSS-animated scrolling film strip. Duplicates the list for a seamless loop. Scroll speed is dynamically set from content width.
