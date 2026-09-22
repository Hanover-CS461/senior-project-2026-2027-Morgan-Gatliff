---
marp: true
theme: gaia
size: 16:9
paginate: true
---

# Live Scoreboard Display
### A senior project presentation

---

# Why I Built This
### The Sunday ritual with my dad


The reason I chose to build this app is to keep a tradition with my dad alive even after college.

We all have personal traditions or rituals, right?

I had this idea four years ago and wanted to make it a reality.

---

# The Problem
### Scores live in apps on our phones

The main purpose is to bring a personal scoreboard to everyone. 

#### Why?
It gives a more personal relationship between the user and their favorite sport. It provides all the stats you might expect to see during a sporting event. It brings the user to the game.

---

# What It Does
### Scoreboard, Field position, Win %, Top stats, Ticker

- **Scoreboard** — the featured game's score, quarter, and game clock
- **Field graphic** — where the ball is, with down and distance
- **Win percentage** — which way the game is leaning
- **Top player stats** — the names doing the work for both teams
- **Ticker** — every other game, live, at a glance

All of it updates automatically from live data.

---

# The Feature: Follow a Team
### Pick a team — the display follows the game

- Choose your favorite team when the app starts
- That team's game is featured while it's live
- The display stays locked on them — score, field, stats

---

# The Feature: Auto-Switch
### When the game ends, the display moves on

- When your team's game finishes, the display doesn't sit idle
- It switches to the next most relevant game
- Division/conference rivals, upcoming opponents, or your own pick

The display keeps itself useful with zero interaction.

---

# How It Works
### ESPN API → Python core → Flask → React

- **Python core** fetches and parses live data, polls on a schedule
- **Flask** sits between the browser and ESPN (browsers can't call ESPN directly — CORS)
- **React** renders everything you see
- **pywebview** wraps the same web page into a desktop app

---

# The Data
### ESPN's public API: scoreboard, summary, win probability

- Free, no API key, no authentication
- `scoreboard` — all games, scores, status, quarter, clock
- `summary` — play-by-play with yard line, down, and distance
- `winprobability` — live win percentage feed
- `boxscore` — player stats

---

# Similar Solutions
### ESPN, NFL App, Yahoo, The Score — and what's missing

- ESPN site — a firehose of menus, ads, and every game
- NFL app — subscription hub: streaming, paid analytics
- Yahoo / theScore — notifications, betting, engagement loops
- theScore's lock-screen live activity is the closest idea

All of them are apps you *open*. 

---

# Why This One Is Different
### A display you don't have to open

- No opening, no scrolling, no notifications
- One team, focused, at a glance
- Built to sit on a desk and just be there
- The scoreboard you don't have to open

---

# Challenges
### Live data, polling, and the API that could change overnight

- Polling without hammering the API
- Computing "current" state from play-by-play data
- ESPN's API is undocumented — it could change with no warning

---

# Thank You
### Questions?