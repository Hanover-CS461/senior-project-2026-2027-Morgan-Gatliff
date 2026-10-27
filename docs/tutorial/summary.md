---
layout: default
---
# 7. Summary

Congratulations — you've gone from a single line of JSON to a working, **typed data
layer** for a live NFL display. The biggest change from "just fetch the data" is that
you now own a model: raw JSON goes in one side, small objects come out the other.

---

## What you learned

- **Endpoint anatomy.** Every URL is `host + path + query`. The path names the
  resource; the query selects/filters it. You can swap `scoreboard` for `calendar`
  or a game id without relearning the API.
- **Two API domains.** `site.api.espn.com` returns rich, self-contained objects
  (scores, game state); `sports.core.api.espn.com` returns `$ref`-heavy data you must
  follow links to expand.
- **The calendar endpoints** describe the season structure — game days and week
  ranges. Parsing them into `Calendar → SeasonSection → WeekEntry` makes a **week
  tracker** a short loop instead of nested indexing.
- **The live scoreboard** is your primary live source. You learned exactly where
  score, clock, quarter, down, distance, and field position live, and why the
  `situation` object only appears during a live game.
- **The `from_raw` parsing pattern.** One dataclass per JSON object, `.get(key,
  default)` for every field, optional nested objects become `Optional[...]`, and
  lists become tuples. A missing field yields a default, not a `KeyError`.
- **Historical weeks** use the same endpoint with `week` / `seasontype` / `dates`
  parameters — same shape, same parsers, different values.
- **Practical patterns:** selective `$ref` resolution, favorite-team lookup, and a
  polling loop that compares parsed state.

## The model you built

```text
Scoreboard
├── week, season
└── games: tuple[Game]
    ├── id, name, date
    ├── status: GameStatus
    │   ├── state       "pre" | "in" | "post"
    │   ├── period / display_clock
    │   └── is_live / is_final  (properties)
    ├── teams: tuple[Team]
    │   ├── id, abbr, name, score, record, winner
    │   └── home_away
    ├── situation: Optional[Situation]     <- live only
    │   ├── down, distance, yard_line
    │   ├── possession, is_red_zone
    │   └── last_play: Optional[Play]
    └── leaders: tuple[LeaderCategory]
        └── leaders: tuple[Leader]
```

## The mental model

```text
Calendar (static)  ──►  parse  ──►  week tracker  ──►  "which week am I in?"
Scoreboard (live)  ──►  parse  ──►  game state     ──►  "what's happening now?"
```

Keep the **static** calendar cached and poll the **live** scoreboard on an interval.
That split keeps your app fast and gentle on the API. And keep parsing and display
separate: `from_raw` is the only place that knows ESPN's field names.

## Endpoint quick reference

| Domain | Endpoint | Returns | Use it for |
|---|---|---|---|
| Site | `/apis/site/v2/sports/football/nfl/scoreboard` | `events[]`, `week`, `season` | Live + historical scores |
| Site | `/apis/site/v2/sports/football/nfl/summary?event={id}` | `boxscore`, plays, etc. | Full game detail |
| Core | `/v2/sports/football/leagues/nfl/calendar` | `items[]` of `$ref` | Discovery: list calendar sub-resources |
| Core | `/v2/sports/football/leagues/nfl/calendar/ondays` | `eventDate.dates[]`, `sections[]` | Game days + week ranges |
| Core | `/v2/sports/football/leagues/nfl/events/{eventId}` | `$ref`-heavy event | Detailed event data |
| Core | `/v2/sports/football/leagues/nfl/events/{eventId}/competitions/{id}/...` | `$ref`-heavy sub-resources | Plays, odds, win probability |

> **Verify, don't assume.** These paths are undocumented. The WADL (below) is the
> closest thing to a spec, and even it lags reality. When a path 404s, fall back to
> the parent resource and follow its `$ref`s.

## Where to go next

- Build the display layer on top of this model (the project's next step).
- Add the Core API `winprobability` and `plays` endpoints and parse them into new
  dataclasses, reusing the same `from_raw` pattern.
- Extend `Team.from_raw` to include `linescores` so quarter-by-quarter tables come
  straight from the model.
- Add a `to_dict()` method to each dataclass so the Flask layer can `jsonify` the
  model without exposing raw JSON.

---

## See also

- [ESPN Core API WADL — full endpoint list](https://sports.core.api.espn.com/v2/application.wadl?detail=true)
- [Community ESPN API documentation (pseudo-r/Public-ESPN-API)](https://github.com/pseudo-r/Public-ESPN-API)
- [Site API scoreboard endpoint](https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard)
- [Python `requests` documentation](https://requests.readthedocs.io/en/latest/)
- [Python `json` module](https://docs.python.org/3/library/json.html)
- [Python `dataclasses`](https://docs.python.org/3/library/dataclasses.html)
- [Python `datetime` module](https://docs.python.org/3/library/datetime.html)


---

**← Previous: [6. Exercises](exercises.md)** &nbsp;|&nbsp; **Back to [Introduction](index.md)**
