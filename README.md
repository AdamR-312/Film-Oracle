# Adam's Cinema

A personal film site built in raw HTML, CSS, and vanilla JavaScript — no frameworks, no build tools, no dependencies. Seven standalone pages hosted on GitHub Pages, with live data from TMDB and AI-powered recommendations via the Anthropic API. Both API keys are secured through Cloudflare Worker proxies; no secrets appear in client-side code.

**Live site:** [adamr-312.github.io/Film-Oracle](https://adamr-312.github.io/Film-Oracle)

---

## Pages

| File | Room | Description |
|------|------|-------------|
| `index.html` | Grand Lobby | Homepage, room navigation, live filmstrip banner (Now in Theatres) |
| `adams-oracle.html` | Screening Room No. 1 | 14-question AI film recommendation tool |
| `top25.html` | Screening Room No. 2 | Navigation hub for 14 genre list pages |
| `list.html` | Screening Room No. 2 | Universal list page — URL-param driven, renders any genre list via TMDB |
| `collection.html` | Screening Room No. 3 | Personal film archive built from Letterboxd CSV export |
| `indie.html` | Screening Room No. 4 | Resources, forums, and state of independent exhibition |
| `staff.html` | The Mezzanine | Technical documentation — architecture, stack, how the Oracle works |

---

## Architecture

The site is fully static. Every page is a self-contained HTML file that runs directly in a browser — no server, no database, no build step. GitHub Pages serves the files as-is from the main branch.

The two exceptions to full staticity:

**Top 25 Lists + Filmstrip** — fetch live data from [The Movie Database (TMDB)](https://www.themoviedb.org/). All requests route through a Cloudflare Worker proxy (`tmdb-proxy.adamrowe312.workers.dev`) that appends the API key from a Worker secret before forwarding to TMDB.

**Movie Oracle** — submits a structured prompt to Anthropic's `/v1/messages` endpoint. Requests route through a second Cloudflare Worker (`oracle-proxy.adamrowe312.workers.dev`) that holds the Anthropic API key as a Worker secret. The browser POSTs to the Worker; the Worker calls Anthropic.

```
Browser
  ├── Static pages → GitHub Pages (no auth)
  ├── TMDB requests → tmdb-proxy.workers.dev → api.themoviedb.org
  └── Oracle requests → oracle-proxy.workers.dev → api.anthropic.com
```

---

## Tech Stack

- **HTML5 / CSS3** — single-file pages, embedded styles, CSS custom properties for the design system
- **Vanilla JavaScript** — fetch API, sessionStorage, DOM manipulation; no libraries
- **Anthropic API** — `claude-sonnet-4-20250514` via Worker proxy
- **TMDB API** — `discover/movie` and `movie/now_playing` endpoints via Worker proxy
- **Cloudflare Workers** — two Workers handling API key proxying and CORS
- **GitHub Pages** — free static hosting, deploys on push to main
- **Google Fonts** — Playfair Display, EB Garamond, Cinzel
- **Letterboxd Export** — CSV data parsed into static HTML for the Collection page

---

## How the Oracle Works

The Oracle collects 14 answers across four groups:

1. **Taste anchors** — a personal masterpiece and an overrated film. Together these triangulate aesthetic position more efficiently than any genre checkbox.
2. **Emotional state** — weight tolerance, runtime preference, desired ending tone.
3. **Preferences** — genre, themes, subtitles, pace, protagonist type, obscurity, era window.
4. **Open intent** — what to avoid, what they want the film to do to them.

JavaScript assembles the answers into a structured prompt and POSTs to the Cloudflare Worker. Claude responds in a fixed format (`TITLE`, `DIRECTED BY`, `WHY`, `FIND IT`), which is parsed with regex and injected into the result card. The submit button stays locked until all 14 questions are answered.

If the user has already seen the recommendation, an "Already Seen It" button appends the title to a `seenFilms` array that travels with subsequent requests, instructing Claude not to repeat it.

---

## How the Top 25 Lists Work

All 14 genre lists are served by a single page (`list.html`) via a `?list=` URL parameter. On load, the page reads the parameter, looks up the matching TMDB query config, and fires the request through the Worker.

Each list fetches four pages of results simultaneously (~80 candidates), then filters out any film IDs already seen this session using `sessionStorage`. This prevents the same film from appearing on multiple lists during a browsing session. The top 25 unique results are rendered. If fewer than 25 unique results are available after filtering, it falls back to the unfiltered set.

**Genre configs:**

| Key | Title | TMDB Strategy |
|-----|-------|---------------|
| `horror` | Greatest Horror Films | Genre 27, vote_count ≥ 1500 |
| `thriller` | Greatest Thrillers | Genre 53, vote_count ≥ 2000 |
| `comedy` | Greatest Comedies | Genre 35, English-language, sort by vote_count |
| `psych_thriller` | Greatest Psychological Thrillers | Genres 53+18, vote_count ≥ 1500 |
| `action` | Greatest Action Films | Genre 28, vote_count ≥ 2000 |
| `mystery` | Greatest Mystery & Neo-Noir | Genre 9648, vote_count ≥ 1000 |
| `scifi` | Greatest Science Fiction | Genre 878, vote_count ≥ 2000 |
| `historical` | Greatest Historical Films | Genre 36, vote_count ≥ 1000 |
| `drama` | Greatest Dramas | Genre 18, excluding action/horror/scifi/comedy |
| `gangster` | Greatest Gangster Films | Keyword 1562, vote_count ≥ 200 |
| `war` | Greatest War Films | Genre 10752, vote_count ≥ 1000 |
| `romance` | Greatest Romance Films | Genre 10749, vote_count ≥ 1000 |
| `animated` | Greatest Animated Films | Genre 16, excluding family, vote_count ≥ 1000 |
| `western` | Greatest Westerns | Genre 37, vote_count ≥ 500 |

---

## The Collection Page

Built from a real [Letterboxd](https://letterboxd.com) data export (`Settings → Data → Export Your Data`). Four CSVs were used:

- `ratings.csv`
- `profile.csv`
- `the-best-movies-i-have-ever-seen.csv`
- `fever-dream-films.csv`

Data was parsed in Python and hardcoded into the HTML. The page requires no live data connection. To update it: re-export from Letterboxd, re-parse, and push the updated file.

---

## Cloudflare Worker Setup

Both Workers follow the same pattern. To deploy or modify:

1. Install [Wrangler](https://developers.cloudflare.com/workers/wrangler/) (`npm install -g wrangler`)
2. Authenticate: `wrangler login`
3. Add API key as a secret: `wrangler secret put TMDB_KEY` / `wrangler secret put ANTHROPIC_KEY`
4. Deploy: `wrangler deploy`

The Workers validate the request origin against an allowlist before forwarding. CORS headers are set on all responses so the browser accepts them from GitHub Pages.

---

## Deployment

The site deploys automatically. Push to `main` → GitHub Pages serves it.

To deploy locally for development, any static file server works:

```bash
# Python
python -m http.server 8080

# VS Code
# Live Server extension, default port 5500
```

The Cloudflare Workers include `localhost:5500` and `127.0.0.1:5500` in their CORS allowlist for local development.

---

## Design System

All pages share the same CSS custom properties:

```css
--black: #080608;
--crimson: #6b0f1a;
--gold: #c9a84c;
--gold-dim: #7a6130;
--cream: #f0e8d8;
--cream-dim: #c8b898;
```

Typefaces: **Playfair Display** (display headings) · **EB Garamond** (body) · **Cinzel** (labels, structural chrome)

---

## Repository Structure

```
Film-Oracle/
├── index.html          # Grand Lobby
├── adams-oracle.html   # Movie Oracle
├── top25.html          # Top 25 hub
├── list.html           # Universal list page
├── collection.html     # Curated collection
├── indie.html          # Indie & local theatres
├── staff.html          # Technical documentation
└── README.md
```

---

*Built in collaboration with [Claude](https://claude.ai) by Anthropic.*
