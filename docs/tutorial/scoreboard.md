---
layout: default
---
# 3. Live Scoreboard

This is the heart of the project: the endpoint that returns **every NFL game**
happening on a given day, complete with scores, clock, down and distance, and
player leaders. Unlike the Core API, the **Site API** scoreboard embeds its data
directly — no `$ref` chasing. The trade-off is that the response is large and
deeply nested, which is exactly why we parse it.

---

## The live scoreboard endpoint

```
https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard
```

The path breaks down as `site/v2` → `sports/football/nfl` → `scoreboard`. Fetch it
with the helper from Setup:

```python
import json
import requests

SCOREBOARD_URL = (
    "https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard"
)


def get_json(url, **params):
    response = requests.get(url, params=params, timeout=10)
    response.raise_for_status()
    return response.json()


def show(data):
    print(json.dumps(data, indent=2, ensure_ascii=False))


raw = get_json(SCOREBOARD_URL)
print("Games today:", len(raw["events"]))
```

> **Heads up:** this response is large (often 100–250 KB) because every game embeds
> full team, record, leader, odds, and venue objects. Use `limit` (below) while
> experimenting.

---

## Response shape at a glance

```text
{
  "leagues": [ ... ],       # includes the season structure (calendar)
  "season":  { "type": 2, "year": 2026 },
  "week":    { "number": 6, "teamsOnBye": [ ... ] },
  "events":  [ ... ]        # one entry per game  <-- the good stuff
}
```

Each entry in `events` looks like this (trimmed to the fields you care about):

```json
{
  "id": "401872995",
  "name": "Seattle Seahawks at Denver Broncos",
  "date": "2026-10-16T00:15Z",
  "competitions": [
    {
      "competitors": [
        { "id": "7",  "homeAway": "home", "score": "0", "winner": false,
          "team": { "id": "7", "abbreviation": "DEN", "displayName": "Denver Broncos" },
          "records": [ { "name": "overall", "summary": "3-2" } ] },
        { "id": "26", "homeAway": "away", "score": "0", "winner": false,
          "team": { "id": "26", "abbreviation": "SEA", "displayName": "Seattle Seahawks" },
          "records": [ { "name": "overall", "summary": "3-2" } ] }
      ],
      "status": {
        "clock": 0.0,
        "displayClock": "0:00",
        "period": 0,
        "type": { "state": "pre", "completed": false, "detail": "Thu, October 15th at 8:15 PM EDT" }
      },
      "situation": { "down": 2, "distance": 8, "yardLine": 60, "isRedZone": false },
      "leaders": [ { "name": "passingYards", "leaders": [ { "displayValue": "…", "athlete": { … } } ] } ]
    }
  ]
}
```

### The nesting, in words

```text
response
├── season.year / season.type
├── week.number
└── events[]                  <- one game
    └── competitions[0]       <- usually exactly one
        ├── competitors[]     <- exactly two, one home + one away
        │   ├── homeAway
        │   ├── score
        │   ├── winner
        │   ├── team.abbreviation / displayName
        │   ├── records[].summary
        │   └── linescores[]  <- per-quarter scores
        ├── status            <- clock + quarter + state
        ├── situation         <- down/distance/possession (live only)
        └── leaders[]         <- top performers per category
```

**Every parser in this section is just a walk down that tree.**

---

## Where each live value actually lives

This is the single most useful table in the tutorial. Several values are **not**
where you might first guess.

| What you want | JSON path | Present when |
|---|---|---|
| Score (per team) | `competitions[].competitors[].score` | always |
| Which team is home | `competitions[].competitors[].homeAway` | always |
| Team id / abbrev / name | `competitions[].competitors[].team.{id,abbreviation,displayName}` | always |
| Season record | `competitions[].competitors[].records[].summary` (pick `name == "overall"`) | usually |
| Quarter / period | `competitions[].status.period` | always |
| Game clock | `competitions[].status.displayClock` | always |
| Live / final / not started | `competitions[].status.type.state` (`pre` / `in` / `post`) | always |
| Game finished? | `competitions[].status.type.completed` | always |
| **Down** | `competitions[].situation.down` | **live only** |
| **Distance to first down** | `competitions[].situation.distance` | **live only** |
| **Field position** | `competitions[].situation.yardLine` | **live only** |
| **Team in possession** | `competitions[].situation.possession` | **live only** (may be absent) |
| **Is red zone** | `competitions[].situation.isRedZone` | **live only** |
| **Last play** | `competitions[].situation.lastPlay` | **live only** |
| Per-quarter scores | `competitions[].competitors[].linescores[]` | live/final |
| Top player leaders | `competitions[].leaders[].leaders[]` | always (may be empty) |

> **Critical gotcha:** `status` holds the **clock and quarter only** — *not* the
> score and *not* the field position. And the `situation` object (down, distance,
> possession) **exists only while a game is in progress**. Before kickoff and after
> the final whistle it is absent, so your parser must treat it as optional.

> **Two `leaders` levels:** the competition has `leaders` (categories like
> `passingYards`), and each competitor also has its own `leaders` (named
> `passingLeader`, etc.). They are different lists at different depths. This
> tutorial parses the competition-level one.

---

## Parsing the scoreboard into models

Instead of threading long bracket chains through your display code, define one
dataclass per object in the tree above. Each one implements `from_raw`, and each one
only knows its own slice of the JSON.

### 1. `GameStatus` — clock, quarter, state

```python
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)
class GameStatus:
    state: str
    period: int
    display_clock: str
    completed: bool
    detail: str

    @classmethod
    def from_raw(cls, raw: dict) -> "GameStatus":
        status_type = raw.get("type", {})
        return cls(
            state=status_type.get("state", "pre"),
            period=raw.get("period", 0),
            display_clock=raw.get("displayClock", "0:00"),
            completed=status_type.get("completed", False),
            detail=status_type.get("detail", ""),
        )

    @property
    def is_live(self) -> bool:
        return self.state == "in"

    @property
    def is_final(self) -> bool:
        return self.state == "post"

    @property
    def label(self) -> str:
        if self.is_final:
            return "Final"
        return self.detail
```

### 2. `Play` — a single play (nested inside `situation`)

```python
@dataclass(frozen=True)
class Play:
    id: str
    text: str
    play_type: str
    yardage: int
    team_id: str

    @classmethod
    def from_raw(cls, raw: dict) -> "Play":
        return cls(
            id=raw.get("id", ""),
            text=raw.get("text", ""),
            play_type=raw.get("type", {}).get("text", ""),
            yardage=raw.get("statYardage", 0),
            team_id=raw.get("team", {}).get("id", ""),
        )
```

### 3. `Situation` — down, distance, field position

```python
@dataclass(frozen=True)
class Situation:
    possession: str
    down: int
    distance: int
    yard_line: int
    is_red_zone: bool
    possession_text: str
    home_timeouts: int
    away_timeouts: int
    last_play: Optional[Play] = None

    @classmethod
    def from_raw(cls, raw: dict) -> "Situation":
        last = raw.get("lastPlay")
        return cls(
            possession=raw.get("possession", ""),
            down=raw.get("down", 0),
            distance=raw.get("distance", 0),
            yard_line=raw.get("yardLine", 0),
            is_red_zone=raw.get("isRedZone", False),
            possession_text=raw.get("possessionText", ""),
            home_timeouts=raw.get("homeTimeouts", 0),
            away_timeouts=raw.get("awayTimeouts", 0),
            last_play=Play.from_raw(last) if last else None,
        )
```

> **Defensive default:** `possession` and `possessionText` are not always present in
> live samples. Using `.get(..., "")` means a missing value is an empty string, not a
> crash.

### 4. `Leader` and `LeaderCategory` — top performers

```python
@dataclass(frozen=True)
class Leader:
    athlete_id: str
    name: str
    display_value: str

    @classmethod
    def from_raw(cls, raw: dict) -> "Leader":
        athlete = raw.get("athlete", {})
        return cls(
            athlete_id=athlete.get("id", ""),
            name=athlete.get("fullName", "") or athlete.get("displayName", ""),
            display_value=raw.get("displayValue", ""),
        )


@dataclass(frozen=True)
class LeaderCategory:
    name: str
    leaders: tuple          # of Leader

    @classmethod
    def from_raw(cls, raw: dict) -> "LeaderCategory":
        return cls(
            name=raw.get("name", ""),
            leaders=tuple(Leader.from_raw(l) for l in raw.get("leaders", [])),
        )
```

> **Why `or`?** `raw.get("fullName", "") or raw.get("displayName", "")` falls back to
> `displayName` when `fullName` is present but empty. `.get` only protects against a
> *missing* key, not an empty value.

### 5. `Team` — one competitor

```python
@dataclass(frozen=True)
class Team:
    id: str
    abbr: str
    name: str
    score: str
    home_away: str
    record: str
    winner: bool

    @classmethod
    def from_raw(cls, raw: dict) -> "Team":
        team = raw.get("team", {})
        records = raw.get("records", [])
        overall = next(
            (r.get("summary", "") for r in records if r.get("name") == "overall"),
            "",
        )
        return cls(
            id=raw.get("id", ""),
            abbr=team.get("abbreviation", ""),
            name=team.get("displayName", ""),
            score=raw.get("score", "0"),
            home_away=raw.get("homeAway", ""),
            record=overall,
            winner=raw.get("winner", False),
        )
```

> **`records` is a list, not a string.** The first element is usually `overall`, but
> selecting by `name == "overall"` is safer than assuming order. Also note `score` is
> a **string** (`"24"`), not an int — keep it for display, convert only for math.

### 6. `Game` — one event

```python
@dataclass(frozen=True)
class Game:
    id: str
    name: str
    date: str
    status: GameStatus
    teams: tuple            # of Team
    situation: Optional[Situation]
    leaders: tuple          # of LeaderCategory

    @classmethod
    def from_raw(cls, raw: dict) -> "Game":
        competition = raw["competitions"][0]
        situation = competition.get("situation")
        return cls(
            id=raw.get("id", ""),
            name=raw.get("name", ""),
            date=raw.get("date", ""),
            status=GameStatus.from_raw(competition.get("status", {})),
            teams=tuple(
                Team.from_raw(c) for c in competition.get("competitors", [])
            ),
            situation=Situation.from_raw(situation) if situation else None,
            leaders=tuple(
                LeaderCategory.from_raw(c)
                for c in competition.get("leaders", [])
            ),
        )

    def team_by_id(self, team_id: str) -> Optional[Team]:
        return next((t for t in self.teams if t.id == team_id), None)
```

Here is where the optional `situation` pays off: `Situation.from_raw(situation) if
situation else None` means the rest of the app can simply check `if game.situation:`.

### 7. `Scoreboard` — the whole response

```python
@dataclass(frozen=True)
class Scoreboard:
    games: tuple            # of Game
    week: Optional[int]
    season: Optional[int]

    @classmethod
    def from_raw(cls, raw: dict) -> "Scoreboard":
        return cls(
            games=tuple(Game.from_raw(e) for e in raw.get("events", [])),
            week=raw.get("week", {}).get("number"),
            season=raw.get("season", {}).get("year"),
        )

    def game_for_team(self, team_id: str) -> Optional[Game]:
        for game in self.games:
            if game.team_by_id(team_id):
                return game
        return None

    def live_games(self):
        return [g for g in self.games if g.status.is_live]
```

> **`from_raw({})` should not crash.** With the `.get` defaults above, an empty
> response parses to zero games and `None` for week/season. That is a deliberate
> property — it is what makes the parser safe to reuse for every endpoint variant.

### 8. Fetch + parse together

```python
raw = get_json(SCOREBOARD_URL)
board = Scoreboard.from_raw(raw)

print(f"Week {board.week}, {board.season} — {len(board.games)} games")
```

---

## Using the parsed model

Now the display code is ordinary attribute access:

```python
def summarize(game: Game) -> str:
    teams = {t.home_away: t for t in game.teams}
    away, home = teams["away"], teams["home"]

    line = f"{away.abbr} {away.score} @ {home.abbr} {home.score}"

    if game.status.is_live:
        line += f"  |  Q{game.status.period} {game.status.display_clock}"
        if game.situation:
            line += (
                f"  {game.situation.down} & {game.situation.distance} "
                f"at the {game.situation.yard_line}"
            )
    elif game.status.is_final:
        line += "  |  FINAL"
    else:
        line += f"  |  {game.status.detail}"

    return line


for game in board.games:
    print(summarize(game))
```

Example output:

```text
SEA 0 @ DEN 0  |  Thu, October 15th at 8:15 PM EDT
PHI 24 @ DAL 20  |  Q4 2:11  3 & 7 at the 42
KC 30 @ BUF 27  |  FINAL
```

Notice how the same function handles all three states. That "state machine"
(`pre` → `in` → `post`) is what drives your display. Because the branch lives on
`game.status`, you can add a new state without touching the rest of the parser.

---

## A client that keeps the request details in one place

`from_raw` handles the JSON; a small client handles the HTTP. This mirrors the
`EspnClient` in the project and keeps the base URL, session, and timeout together.

```python
import requests


class EspnClient:
    BASE = "https://site.api.espn.com/apis/site/v2/sports/football/nfl"

    def __init__(self) -> None:
        self._session = requests.Session()

    def scoreboard(self, week=None, seasontype=None) -> dict:
        params = {}
        if week is not None:
            params["week"] = week
        if seasontype is not None:
            params["seasontype"] = seasontype

        response = self._session.get(
            f"{self.BASE}/scoreboard", params=params, timeout=10
        )
        response.raise_for_status()
        return response.json()


board = Scoreboard.from_raw(EspnClient().scoreboard(week=6, seasontype=2))
```

> **Why a `Session`?** It reuses the underlying TCP connection across requests, which
> matters in a polling loop. It does **not** cache responses, so you still get fresh
> scores each call.

---

## Query parameters

The scoreboard accepts several parameters. The ones you'll use:

| Parameter | Example | Effect |
|---|---|---|
| `dates` | `?dates=20261015` | Games on one day, or a range `20260913-20260914` |
| `week` | `?week=5` | Games in a week (pair with `seasontype`) |
| `seasontype` | `&seasontype=2` | `1`=preseason, `2`=regular, `3`=postseason |
| `limit` | `?limit=1` | Limit number of events (great while testing) |

```python
# Today's games, but only the first event (small response while learning):
small = Scoreboard.from_raw(get_json(SCOREBOARD_URL, limit=1))
```

We use `week`, `seasontype`, and `dates` fully in the next section.

---

### See also

- [`requests` Response JSON](https://requests.readthedocs.io/en/latest/user/quickstart/#json-response-content)
- [Python `dict.get`](https://docs.python.org/3/library/stdtypes.html#dict.get)
- [Python `dataclasses`](https://docs.python.org/3/library/dataclasses.html)
- [Community ESPN API documentation](https://github.com/pseudo-r/Public-ESPN-API)

---

**← Previous: [2. League Calendar](calendar.md)** &nbsp;|&nbsp; **Next → [4. Past Weeks](historical.md)**
