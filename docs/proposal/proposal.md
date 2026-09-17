---
layout: default
---
# Live Football Display

## Project Description and Main Features

The goal of this project is to build a live football display that runs on a desktop computer. It shows a single team's game as it happens — no opening an app, no notifications, just information at a glance. The application is built for a personal reason: to keep the Sunday ritual alive with my dad, who took me to every home game growing up, after I move away. While the first version targets the desktop, the underlying design — a Python core serving a web page — means it could later grow into a dedicated desk display or a Raspberry Pi-powered product.

The application is built as three layers: a Python core that fetches and parses live game data, a Flask server that sits between the browser and the data source, and a React display that renders the scoreboard. The Python core polls ESPN's public API on a regular interval and computes the state of the featured game. The display shows:

- A **scoreboard** for the featured game with score, quarter, and game clock
- A **football field graphic** with the current ball position, down, and distance
- A **win percentage** readout for each team
- **Top player statistics** for both teams
- A **ticker** of all other games with live scores

The user picks a favorite team when the app starts. That team's game is featured while it is live; when the game ends, the display automatically switches to the next most relevant game — the team's division or conference, upcoming opponents, or another game the user selects. Every feature pulls from ESPN's live data feed; no code modification or database is required. The only stored value is the user's favorite team, kept in the browser's local storage.

## Similar Existing Solutions

Five existing products offer live NFL scores: the ESPN scoreboard website, the official NFL app, Yahoo Sports, and theScore. All of them share the same shape: an interactive mobile or web application that the user must open, navigate, and engage with. ESPN's site is a firehose of menus, ads, and every game in the league. The NFL app is a subscription hub covering all teams with streaming and paid analytics. Yahoo Sports and theScore push notifications and betting features designed to keep the user engaged.

None of them is a passive, always-on display focused on one team. theScore's lock-screen Live Activity — which pins down and distance to the phone's lock screen — is the closest existing idea, but it still lives on a phone and requires setup and interaction. This project occupies the empty space those products leave: a dedicated display that needs no opening, no scrolling, and no interaction, focused on one team at a time.

## References

[1] ESPN, "ESPN Public API — NFL Scoreboard," site.api.espn.com. [Online]. Available: https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard

[2] D. Hantzmon, "espn-scores," PyPI, 2026. [Online]. Available: https://pypi.org/project/espn-scores/

[3] pseudo-r, "ESPN Public API Documentation," GitHub. [Online]. Available: https://github.com/pseudo-r/Public-ESPN-API

[4] ESPN, "NFL Scores," espn.com. [Online]. Available: https://www.espn.com/nfl/scoreboard

[5] Yahoo, "Yahoo Sports: Scores and News," Google Play. [Online]. Available: https://play.google.com/store/apps/details?id=com.yahoo.mobile.client.android.sportacular

[6] NFL Enterprises LLC, "NFL App," App Store. [Online]. Available: https://apps.apple.com/us/app/nfl/id389781154

[7] Score Media and Gaming Inc., "theScore: Sports News & Scores," App Store. [Online]. Available: https://apps.apple.com/us/app/thescore-sports-news-scores/id285692706
