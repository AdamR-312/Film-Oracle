# Adam's Cinema

A personal film site built in raw HTML, CSS, and vanilla JavaScript — no frameworks, no build tools, no dependencies. Ten standalone pages hosted on GitHub Pages, with live data from multiple APIs. API keys are secured through Cloudflare Worker proxies; open APIs (Wikipedia, Internet Archive) are called directly from the browser without a proxy.

**Live site:** [adamr-312.github.io/Film-Oracle](https://adamr-312.github.io/Film-Oracle)

---

## Pages

| File | Room | Description |
|------|------|-------------|
| `index.html` | Grand Lobby | Homepage with dual live filmstrips (trending + upcoming) and room navigation |
| `adams-oracle.html` | Screening Room No. 1 | 14-question AI film recommendation tool |
| `top25.html` | Screening Room No. 2 | Navigation hub for 14 genre list pages |
| `list.html` | Screening Room No. 2 | Universal list page — URL-param driven, renders any genre list via TMDB |
| `collection.html` | Screening Room No. 3 | Personal film archive built from Letterboxd CSV export |
| `indie.html` | Screening Room No. 4 | Resources, forums, and state of independent exhibition |
| `classic.html` | Screening Room No. 5 | Public domain film browser — Internet Archive, 8 curated programmes, lightbox player |
| `concession.html` | The Concession Stand | Fill-in-the-blank famous quotes — Claude responds in character as the original speaker |
| `sixdegrees.html` | The Projection Room | Six Degrees of Cinema — TMDB-verified cast/crew chain between any two films; Claude narrates each link |
| `doublefeature.html` | The Double Bill | Double Feature — enter one film, Claude suggests a paired double bill and writes the programmer's note |
| `staff.html` | The Mezzanine | Technical documentation — architecture, stack, how the Oracle works |

---

## Architecture

The site is fully static. Every page is a self-contained HTML file that runs directly in a browser — no server, no database, no build step. GitHub Pages serves the files as-is from the main branch.

Different APIs are handled differently based on whether they require credentials:

**Proxied (keys must stay server-side):**
- **TMDB** — Top 25 lists, dual filmstrips, Oracle result enrichment, Six Degrees film/credits resolution, Double Feature poster resolution. All requests route through `tmdb-proxy.adamrowe312.workers.dev`.
- **Anthropic** — Oracle recommendations, Concession Stand in-character responses, Six Degrees chain narration, Double Feature pairing suggestions. Requests route through `oracle-proxy.adamrowe312.workers.dev`.

**Called directly from the browser (no credentials):**
- **Wikipedia REST API** — Director biography panel on Oracle results. Open, CORS-enabled, no key required.
- **Internet Archive** — Public domain film search and embed on The Archive page. Fully open, no key required.

```
Browser
  ├── Static pages        → GitHub Pages (no auth)
  ├── TMDB requests       → tmdb-proxy.workers.dev → api.themoviedb.org
  ├── Oracle requests     → oracle-proxy.workers.dev → api.anthropic.com
  ├── Wikipedia requests  → en.wikipedia.org (direct, open API)
  └── Archive requests    → archive.org (direct, open API)
```

---

## Tech Stack

- **HTML5 / CSS3** — single-file pages, embedded styles, CSS custom properties for the design system
- **Vanilla JavaScript** — fetch API, sessionStorage, DOM manipulation; no libraries
- **Anthropic API** — `claude-sonnet-4-20250514` via Worker proxy
- **TMDB API** — `discover/movie`, `trending/movie/week`, `movie/upcoming`, `search/movie`, `movie/{id}?append_to_response=videos`, `movie/{id}/recommendations` — all via Worker proxy
- **Wikipedia REST API** — `page/summary/{name}` — called directly from the browser, no proxy
- **Internet Archive API** — `advancedsearch.php`, `services/img/{id}`, `embed/{id}` — called directly from the browser, no proxy
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

JavaScript assembles the answers into a structured prompt and POSTs to the Cloudflare Worker. Claude responds in a fixed format (`TITLE`, `DIRECTED BY`, `WHY`, `FIND IT`), which is parsed with regex and injected into the result card.

After the Oracle responds, two parallel TMDB requests fire via `append_to_response`:
- `movie/{id}?append_to_response=videos` — fetches the film's details and trailer in one call
- `movie/{id}/recommendations` — fetches similar films for the "Also Consider" grid

The result card is then enriched with a `w342` poster, a backdrop image, a `▶ Trailer` button (YouTube link, if an official trailer exists), and a three-card similar films grid.

Simultaneously, the director's name is sent to the **Wikipedia REST API** (`/api/rest_v1/page/summary/{name}`) — called directly from the browser with no proxy — to fetch a biography extract and portrait thumbnail, displayed beneath the director credit.

After the TMDB enrichment completes, a **second Claude call** fires (`max_tokens: 180`) — a viewing dossier. The prompt passes the film title, year, and director, asking for a 2-3 sentence guide covering: the film's cultural significance, one specific cinematic technique or moment worth noticing, and the emotional register of watching it. The dossier appears in the result card under **On This Film**.

If the user has already seen the recommendation, an "Already Seen It" button appends the title to a `seenFilms` array that travels with subsequent requests, instructing Claude not to repeat it.

---

## How the Filmstrips Work

The Grand Lobby header carries two independent animated filmstrips:

- **Top strip** — fetches `trending/movie/week` from TMDB, scrolls left
- **Bottom strip** — fetches `movie/upcoming` from TMDB, scrolls right

Both strips pause when the cursor enters them. Hovering an individual film unit triggers a JS-positioned popup card showing the poster image, title, year, star rating, and synopsis. The popup opens downward for the top strip and upward for the bottom strip, clamped to the viewport.

---

## How the Top 25 Lists Work

All 14 genre lists are served by a single page (`list.html`) via a `?list=` URL parameter. On load, the page reads the parameter, looks up the matching TMDB query config, and fires the request through the Worker.

Each list fetches four pages of results simultaneously (~80 candidates), then filters out any film IDs already seen this session using `sessionStorage`. The top 25 unique results are rendered. If fewer than 25 unique results are available after filtering, it falls back to the unfiltered set.

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

## Six Degrees of Cinema

The Projection Room (`sixdegrees.html`) takes two film titles and constructs a 4–6 step thematic, directorial, or actor-based chain connecting them.

The prompt instructs Claude to output the chain in a strict alternating `FILM:` / `LINK:` format, parsed by the client with regex. Once the chain is extracted, all film titles — including user-entered films and Claude's intermediate choices — are sent to the TMDB proxy simultaneously via `Promise.all`. Each search returns a poster, rendered as a glass film card. The chain animates in step by step with staggered CSS delays.

---

## The Concession Stand

The Concession Stand (`concession.html`) displays a famous movie quote with the final word blanked. The visitor types their answer, which is sent to the oracle-proxy Worker along with the character name, film, year, and the real answer. Claude responds **in character** as the original speaker in 2-3 sentences, commenting on the visitor's attempt without revealing the correct answer. The real answer then blazes in with a gold animation. 30 quotes cycle in randomised order.

---

## Double Feature

The Double Bill (`doublefeature.html`) takes one film title, resolves it via TMDB, then asks Claude to suggest a double bill pairing — a second film that shares a thematic, tonal, or aesthetic connection with the first.

Claude returns a JSON object with the pairing title, year, and a three-sentence programmer's note. The pairing title and year are resolved through TMDB using the `&year=` query parameter (with a fallback retry without the year constraint if no results are found). Both films render side by side in a "Tonight's Programme" card with the programmer's note below. If TMDB cannot find a poster for the suggested pairing, the title and year still display without a poster image.

---

## The Archive

The Archive page (`classic.html`) fetches public domain films directly from the Internet Archive API — no key, no proxy.

Eight programme cards each map to a distinct Lucene `AND` query against `archive.org/advancedsearch.php`:

| Programme | Query |
|-----------|-------|
| Complete Archive | `collection:(feature_films) AND mediatype:movies` |
| Silent Era | `collection:(silent_films) AND mediatype:movies` |
| Westerns | `collection:(feature_films) AND mediatype:movies AND subject:westerns` |
| Film Noir | `mediatype:movies AND subject:(film noir)` |
| Horror | `collection:(feature_films) AND mediatype:movies AND subject:horror` |
| Science Fiction | `mediatype:movies AND subject:(science fiction) AND year:[1940 TO 1969]` |
| Comedy | `collection:(feature_films) AND mediatype:movies AND subject:comedy` |
| Serials | `mediatype:movies AND subject:serials` |

Film thumbnails come from `archive.org/services/img/{id}`. Selecting a film opens it in a lightbox using `archive.org/embed/{id}`.

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

To run locally, any static file server works:

```bash
# Python
python -m http.server 8080

# VS Code
# Live Server extension, default port 5500
```

The Cloudflare Workers include `localhost:5500` and `127.0.0.1:5500` in their CORS allowlist for local development.

---

## Design System

All pages share the same CSS custom properties defined in `style.css`:

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
├── classic.html        # The Archive (public domain cinema)
├── concession.html     # The Concession Stand (fill-in-the-blank quotes)
├── sixdegrees.html     # The Projection Room (Six Degrees of Cinema)
├── doublefeature.html  # The Double Bill (Double Feature pairing)
├── staff.html          # Technical documentation
├── style.css           # Shared design system
├── CLAUDE.md           # Repository guide for Claude Code
└── README.md
```

---

*Built in collaboration with [Claude](https://claude.ai) by Anthropic.*
