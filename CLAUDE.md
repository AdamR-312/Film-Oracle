# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Adam's Cinema** — a static, no-build HTML/CSS/JS movie website with a vintage cinema aesthetic. There is no package manager, build step, or framework. Open any `.html` file directly in a browser.

## Architecture

All pages share one stylesheet ([style.css](style.css)) for the cinema theme (velvet dark palette, gold accents, crimson, curtain animations). Each HTML file has a `<style>` block for page-specific overrides.

### Pages

| File | Purpose |
|---|---|
| [index.html](index.html) | Grand Lobby — navigation hub linking all rooms |
| [adams-oracle.html](adams-oracle.html) | AI movie recommender — 13-question quiz → Claude API |
| [collection.html](collection.html) | Adam's curated personal film collection |
| [top25.html](top25.html) | Ranked genre/era/director lists |
| [indie.html](indie.html) | Independent & local theatre guide |
| [staff.html](staff.html) | About page |

### External Services

Two Cloudflare Workers proxy all external API calls (no keys exposed client-side):

- **`https://oracle-proxy.adamrowe312.workers.dev`** — proxies Anthropic Claude API (`claude-sonnet-4-20250514`). Called by the oracle when the user submits their 13 answers.
- **`https://tmdb-proxy.adamrowe312.workers.dev`** — proxies TMDB API. Used for: now-playing filmstrip (index), movie poster + backdrop (oracle result), similar films grid (oracle result), and collection posters.

### Design System (style.css)

CSS custom properties define the full palette:
- `--gold / --gold-dim / --gold-pale / --gold-bright` — accent colors
- `--velvet / --velvet-mid / --velvet-light` — background layers
- `--crimson / --crimson-bright` — accent reds
- `--cream / --cream-dim / --cream-muted / --cream-ghost` — text scale

Shared structural elements (ceiling bar, floor stripe, curtains, scroll-rail) are defined in `style.css` and used across all pages.

### Oracle Flow (adams-oracle.html)

1. User answers 13 questions (multi-select, scale, text, dropdowns)
2. JS collects answers → POST to `oracle-proxy` Worker with a detailed system prompt
3. Worker calls Claude → returns `{ title, director, year, why, find }` JSON
4. JS fetches TMDB for poster + backdrop image, renders result card
5. JS fetches TMDB recommendations for a 3-card "similar films" grid

### Filmstrip (index.html)

Fetches TMDB `now_playing` → builds a CSS-animated scrolling film strip. Duplicates the list for a seamless loop. Scroll speed is dynamically set from content width.
