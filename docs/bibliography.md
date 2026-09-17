---
---
# Bibliography — Live Football Desk Display

## 1. ESPN Public API (site.api.espn.com)

**Kind of source:** Primary. This is ESPN's own server serving ESPN's own data — the original source itself. It's technically undocumented ("hidden API") but openly accessible with no key or authentication.

**Relation to your app:** Resource you will use. This is the engine of your whole project — the live data pipeline.

**Description:** An undocumented JSON API that powers ESPN's own website and app. It exposes endpoints for scoreboards, game summaries, play-by-play, win probability, box scores, team rosters, and schedules across dozens of sports. The NFL endpoints live under `site.api.espn.com/apis/site/v2/sports/football/nfl/...`.

**Relevance to your goal:** It covers *everything* you need: `scoreboard` returns all games with live scores, status, quarter, and clock; `summary?event={id}` returns play-by-play with yard line and down/distance (your field marker), a `winprobability` feed, and a `boxscore` with player stats (your top-performers panel). One API, no key, no payment — the entire feature list maps onto it. **Comment:** the risk is that it's unofficial — ESPN could change it without notice. You should design your app so the data layer is one small, swappable piece, not woven through the whole codebase.

## 2. espn-scores Python package (PyPI)

**Kind of source:** Secondary. A third-party wrapper written by another developer (not affiliated with ESPN — it says so in its own disclaimer) that consumes ESPN's API and repackages it into clean Python functions.

**Relation to your app:** Resource you *could* use in the solution — a convenience layer. Whether you use it or write your own fetcher is a real design decision.

**Description:** A pip-installable Python library that wraps ESPN's API. It supports NFL (weeks, preseason, playoffs), NBA, NHL, MLB, college sports, and more, returning simple dictionaries. One-liners like `nfl.current_week()` and `nfl.final_games()` replace raw HTTP calls and manual JSON parsing.

**Relevance to your goal:** It would save you time on the boring parts (HTTP, parsing, status filtering) and let you focus on your display and logic. **Comment:** but there's a tradeoff. You're building this as a senior project — writing your own small fetcher means you understand the data layer deeply and control it if the API changes. A wrapper is a dependency someone else maintains; if it breaks mid-season, you're stuck. My leaning: write your own thin fetcher, and keep this package in your back pocket as a reference for how the JSON is shaped.

## 3. Public-ESPN-API documentation (GitHub, pseudo-r)

**Kind of source:** Secondary. Third-party documentation written by community members who reverse-engineered ESPN's hidden API. Not affiliated with ESPN.

**Relation to your app:** Resource for your solution — your map of the territory. This is your "documentation," since ESPN never wrote any.

**Description:** A community-maintained GitHub repository and gists documenting ESPN's undocumented endpoints: URLs, parameters, and JSON response shapes across 17+ sports, 139 leagues, 370+ v2 endpoints. Includes curl examples for every major call you'd make.

**Relevance to your goal:** This is the reference you'll live in while building the data layer — it tells you exactly what fields to expect for scoreboard, summary, win probability, and box scores, so you know your data model before you write code. **Comment:** it's community-maintained, so treat it as a starting map and verify each endpoint yourself with a quick `curl` before trusting it.

## 4. ESPN website — NFL scoreboard (espn.com/nfl/scoreboard)

**Kind of source:** Primary — ESPN's own consumer-facing site, the same company that serves the API.

**Relation to your app:** Both. It's your data's origin, *and* it's a competitor — ESPN already offers a full live-scoreboard experience.

**Description:** The consumer web page for live NFL scores: scoreboards, quarter-by-quarter breakdowns, gamecasts with live field position and win probability, box scores, and player stat leaders, all updated in real time.

**Relevance to your goal:** Every feature you're building exists here — scoreboard, ball position, win %, top stats. That's a good sign (proves demand and that the data supports it) and also the honest version of your "competition" question. **Comment:** your differentiator isn't *more* data — it's *fewer* things, in one glance, always on. ESPN's page is a firehose with menus, ads, and navigation. Yours is a fixed, no-interaction desk display that shows one team's game and a ticker. The competitor proves the data; your product proves the *presentation*.

## 5. Yahoo Sports app (Google Play)

**Kind of source:** Primary — Yahoo's own mobile app.

**Relation to your app:** Competitor. A mainstream live-scores consumer app.

**Description:** A mobile app covering live scores, news, and alerts across NFL, NBA, MLB, NHL, and more. Features real-time score updates, personalized alerts for favorite teams and players, lock-screen Live Activities, play-by-play, and betting-related features.

**Relevance to your goal:** It's the closest mainstream "competitor" to what you're building, and it's worth studying for what it does well — real-time updates and follow-your-team features. **Comment:** it's also proof of what you're *not* doing: it's a phone app with notifications and constant engagement designed to keep you scrolling. Yours is a passive desk display — no notifications, no engagement loop, just information at a glance. That contrast is your pitch: "the scoreboard you don't have to open."


# Technologies

## Platform Decision

**Target platform:** Web display, with a desktop wrapper.

The display is built as a web page (HTML/CSS/JS) that runs in a browser. The same page can be wrapped in a native desktop window using pywebview, and later served on a Raspberry Pi in kiosk mode. This means one display layer works on a laptop, a desktop window, and a Pi — the "standalone base for a webapp" idea made real. Command-line and native Android were rejected as wrong shapes for an always-on visual display.

## 1. Python (core language)

**Kind of source:** Primary — the language itself.

**Relation to your app:** The core of the solution — all fetching, parsing, and state computation happens here.

**Description:** A general-purpose interpreted language. The developer's strongest language, used for the entire data layer: calling the ESPN API, parsing JSON, polling for updates, and computing state (current ball position, win %, top player stats).

**Relevance:** Python is the developer's home turf, so the "hard parts" of the project (data pipeline, state logic) are done in the language where the developer is most productive. It also has mature libraries for HTTP (`requests`) and lightweight web serving (Flask).

## 2. JavaScript / React (display layer)

**Kind of source:** Primary — the language and the library.

**Relation to your app:** The face of the solution — everything the user sees.

**Description:** React is a JavaScript library (not a framework) for building component-based user interfaces. The display is a tree of components: Scoreboard, Field, Win Probability, StatsPanel, Ticker. React's component model matches the layout one-to-one, and its diffing engine handles live updates (fetch → re-render what changed) without hand-writing DOM update logic.

**Relevance:** The developer knows HTML/CSS/JS but finds it difficult; React is preferred because it structures the UI cleanly. Note the library/framework distinction: React is a library — the developer calls it and keeps control of structure. A framework (like Django) would call the developer's code and dictate structure.

**Key tools that come with it:** Node.js (runtime), npm (package manager), Vite (build tool that bundles React into browser-loadable files). This is the largest "new stuff" load in the project.

## 3. Flask (web server)

**Kind of source:** Primary — the framework itself.

**Relation to your app:** The middleman that makes the architecture work.

**Description:** A Python micro-framework for serving web pages and APIs. It handles routing and serves the React display plus one JSON endpoint that feeds live data to the frontend.

**Relevance:** Sized exactly right — one page, one endpoint. Critically, Flask sits between the browser and the ESPN API because the API blocks direct browser requests (CORS). The browser cannot call ESPN itself; it must go browser → Flask → ESPN. This makes the Python core a required part of the architecture, not an optional choice.

## 4. pywebview (desktop wrapper)

**Kind of source:** Primary — the library itself.

**Relation to your app:** What turns the web display into a desktop app.

**Description:** A lightweight library that opens a native window and renders HTML/CSS/JS inside it — "lightweight Electron for Python." No bundled browser engine; it uses the operating system's own renderer, keeping the app small.

**Relevance:** This is how the project achieves "desktop now, web later" without writing the UI twice. The display is built once as a web page; pywebview wraps it for desktop today, and the same page is served by Flask for browser/Pi use later.

## 5. requests (Python HTTP library)

**Kind of source:** Primary — the library itself.

**Relation to your app:** The tool that talks to the ESPN API.

**Description:** The standard Python library for making HTTP requests.

**Relevance:** Small, likely already familiar. The developer's own thin fetcher (rather than the espn-scores wrapper) keeps the data layer under the developer's control — the key risk (ESPN changing its undocumented API) stays manageable because only one small module touches the network.

## Storage

**No external database needed.** All data is live — fetched from ESPN on a polling interval, never stored long-term. The only persisted value is the user's favorite team, which lives in `localStorage` (browser) or a small config file (desktop). No cloud database, no schema design, no ORM.

## Hosting / Deployment

- **Desktop now:** pywebview opens a native window — nothing to host, run locally.
- **Web/Pi later:** the Flask server runs locally on the Pi; the Pi's browser opens it in kiosk mode. Still nothing external to host.
- **Public web (stretch):** a free-tier host (PythonAnywhere, Render, Railway) serves the Flask app — only if others need to reach it.

## "New Stuff" Load Summary

| Category | Status |
|---|---|
| New language | **No** — Python is home turf; JS is known but difficult |
| New frameworks/libraries | **Yes** — React, Flask, pywebview |
| New dev tools | **Yes** — Node.js, npm, Vite build tooling |
| External storage | **No** — no database; just localStorage/config file |
| Hosting complexity | **Low** — desktop first, local Pi server later, free-tier public only as stretch |

## 6. NFL App (official)

**Kind of source:** Primary — NFL's own app.

**Relation to your app:** Competitor. A full game-day hub from the league itself.

**Description:** The official NFL mobile app: up-to-the-minute scoring, drive charts, live stat trackers, breaking news, and streaming via the NFL+ subscription. Includes advanced analytics (NFL Pro) with 95+ player/team performance stats, including win-probability-style data.

**Relevance to your goal:** It confirms the data you're building on exists and is valued, but it's the opposite shape of your product — a subscription-based firehose on a phone covering every team. **Comment:** it also shows the NFL treats its premium data as a paid product, which strengthens the case for using ESPN's free public feed instead.

## 7. theScore (Score Media and Gaming)

**Kind of source:** Primary — Score Media's own app.

**Relation to your app:** Competitor. Your closest rival on presentation.

**Description:** A mobile sports app with real-time scores, a favorites feed, deep in-game stats, live play-by-play, and NFL-specific Live Activities that pin down & distance, timeouts, and possession to the lock screen. Also includes chat, betting features, and news.

**Relevance to your goal:** They've thought hard about *glanceable* football information — the lock-screen Live Activity is the closest thing in the market to your desk display. **Comment:** but it's still an interactive phone app fighting for your attention, with ads and a sportsbook. None of the five competitors is a passive, always-on, single-team display. That empty space is your product.
