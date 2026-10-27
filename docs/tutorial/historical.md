# 4. Past Weeks

The live scoreboard is convenient, but you will also want to look **backwards** — at
last week's results, a specific matchup, or a date you missed. The same scoreboard
endpoint handles all of it; you just change the query string. Because the JSON shape
does not change, the **same parsers** from Section 3 work unchanged.

---

## Two ways to see a past week

### Option A — by week number

```
https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard?week=5&seasontype=2
```

- `week=5` selects Week 5.
- `seasontype=2` selects the **regular season**.

> **Why `seasontype` is required:** week numbers **reset** for each season phase.
> Preseason Week 1 and Regular Season Week 1 both exist. Without `seasontype`, your
> request is ambiguous. Values: `1`=preseason, `2`=regular, `3`=postseason.

### Option B — by date or date range

```
https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard?dates=20261011
https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard?dates=20260913-20260914
```

Use this when you know a calendar date but not the week number.

| You know | Use | Example |
|---|---|---|
| The week number | `week` + `seasontype` | `?week=5&seasontype=2` |
| A single date | `dates` (`YYYYMMDD`) | `?dates=20261011` |
| A date range | `dates` (`YYYYMMDD-YYYYMMDD`) | `?dates=20260913-20260914` |
| A team's week | Fetch the week, then filter by team id | `board.game_for_team("7")` |

> **Parameter precedence:** if you send `week` and `dates` together, the result can
> be surprising. Send **one** selector at a time and keep the code predictable.

---

## Try it

Reuse `get_json` and `Scoreboard` from the previous sections:

```python
previous_week = Scoreboard.from_raw(
    get_json(SCOREBOARD_URL, week=5, seasontype=2)
)

for game in previous_week.games:
    teams = {t.home_away: t for t in game.teams}
    away, home = teams["away"], teams["home"]
    print(
        f"{away.abbr} {away.score} @ {home.abbr} {home.score} "
        f"[{game.status.label}]"
    )
```

Example output:

```text
CIN 21 @ DET 28  [Final]
MIA 17 @ MIN 24  [Final]
```

Notice there is **no new parsing code here** — `Scoreboard.from_raw` already knows how
to read an `events` array, whether it came from today or Week 5.

---

## What changes between live and past games

The **structure is identical** — your parser works on both — but the values differ:

| Field | Live game | Past game |
|---|---|---|
| `status.type.state` | `"in"` | `"post"` |
| `status.type.completed` | `false` | `true` |
| `status.displayClock` | running, e.g. `"12:34"` | `"0:00"` |
| `status.period` | 1–4 (or 5+ in OT) | 4 (or 5+ in OT) |
| `situation` | present | usually absent |
| `competitors[].score` | current | final |
| `competitors[].linescores` | partial | complete |

> **Practical benefit:** because the shape doesn't change, `summarize(game)` from the
> last section works for history with no modifications. That is the point of putting
> the state logic on `GameStatus` instead of scattering it through the app.

---

## Parsing per-quarter scores

Completed games carry a `linescores` array on each competitor — one entry per
quarter (plus overtime). It is a good example of a small nested list you may want to
parse, because your display likely shows a quarter-by-quarter table.

```json
"linescores": [
  { "value": 7.0, "displayValue": "7", "period": 1 },
  { "value": 3.0, "displayValue": "3", "period": 2 }
]
```

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class LineScore:
    period: int
    display_value: str

    @classmethod
    def from_raw(cls, raw: dict) -> "LineScore":
        return cls(
            period=raw.get("period", 0),
            display_value=raw.get("displayValue", "0"),
        )


def quarter_scores(raw_event: dict) -> dict:
    """Map team abbreviation -> list[LineScore]."""
    competition = raw_event["competitions"][0]
    out = {}
    for competitor in competition.get("competitors", []):
        abbr = competitor.get("team", {}).get("abbreviation", "?")
        out[abbr] = [
            LineScore.from_raw(q) for q in competitor.get("linescores", [])
        ]
    return out


raw_week = get_json(SCOREBOARD_URL, week=5, seasontype=2)
previous_week = Scoreboard.from_raw(raw_week)

for raw_event in raw_week.get("events", []):
    print(quarter_scores(raw_event))

# The parsed Game objects are still available for everything else:
print(previous_week.week, len(previous_week.games))
```

> **Raw vs parsed:** `Scoreboard.from_raw` currently drops `linescores` because the
> live display did not need them. When you add a new field, decide whether it belongs
> on `Team` (parse it in `from_raw`) or should be read from the raw event. Adding it
> to `Team` is cleaner if it is used in more than one place.

> **`displayValue` vs `value`:** `value` is a float (`7.0`); `displayValue` is the
> string ESPN wants you to show (`"7"`). Use `displayValue` for UI text and `value`
> only for arithmetic.

---

## Combining history with your week tracker

Your calendar week tracker (from [Section 2](calendar.md#why-this-matters-a-week-tracker))
tells you the current week number. "The previous week" is then simply a subtraction:

```python
current = 6            # from your week tracker
previous = current - 1

url = f"{SCOREBOARD_URL}?week={previous}&seasontype=2"
```

Because `WeekEntry.value` is a **string**, convert before doing arithmetic:

```python
current = int(entry.value)
previous = current - 1
```

> **Edge case:** at `week=1`, `previous` becomes `0`, which is invalid. Guard against
> it (e.g. switch to the previous season's postseason) if your app allows browsing
> backwards indefinitely.

---

### See also

- [Site API scoreboard endpoint reference (community)](https://github.com/pseudo-r/Public-ESPN-API)
- [Python f-strings](https://docs.python.org/3/reference/lexical_analysis.html#f-strings)
- [Python `int()`](https://docs.python.org/3/library/functions.html#int)

---

**← Previous: [3. Live Scoreboard](scoreboard.md)** &nbsp;|&nbsp; **Next → [5. Recipes](recipes.md)**
