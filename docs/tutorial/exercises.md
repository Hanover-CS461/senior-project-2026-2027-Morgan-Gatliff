---
layout: default
---
# 6. Exercises

Try these on your own before moving to the summary. Each one pushes slightly past
what the tutorial showed you. Hints and approaches are included — but try first.
Every exercise is about **parsing**, so expect to write a `from_raw` or a small
helper rather than a new request.

---

## Exercise 1 — Parse the box score statistics

**Goal:** Parse `boxscore.teams[].statistics[]` into a small model, keyed by
statistic name, so you can look up `"totalYards"` or `"possessionTime"` directly.

The raw shape (from the project's `example_data.txt`) is:

```json
{
  "boxscore": {
    "teams": [
      {
        "team": { "abbreviation": "SEA" },
        "homeAway": "away",
        "statistics": [
          { "name": "totalYards", "displayValue": "353", "value": "-", "label": "Total Yards" },
          { "name": "possessionTime", "displayValue": "35:38", "value": 2138, "label": "Possession" }
        ]
      }
    ]
  }
}
```

Build a `TeamBoxScore` dataclass with `abbr`, `home_away`, and a `stats` dict mapping
`name -> displayValue`.

**Approach:**

1. Write `TeamBoxScore.from_raw(raw)` that reads `raw["team"]["abbreviation"]`,
   `raw["homeAway"]`, and builds `{s["name"]: s["displayValue"] for s in raw.get("statistics", [])}`.
2. Write `parse_boxscore(raw)` that returns `{abbr: TeamBoxScore}` from
   `raw["boxscore"]["teams"]`.
3. Print `stats["totalYards"]` for each team.

**Hint:** some stats have `"value": "-"` but a usable `displayValue`. When you only
show stats, prefer `displayValue`; it is already formatted. Watch for **duplicate
`name` keys** (`"interceptions"` appears twice) — the dict keeps the last one.

---

## Exercise 2 — A favorite-team schedule

**Goal:** Print your favorite team's game for **every** week of the regular season,
using the `week` and `seasontype` parameters. Return parsed `Game` objects, not raw
dicts.

Example output:

```text
Week 1:  DEN vs SEA
Week 2:  DEN @ KC
Week 3:  BYE
...
```

**Approach:**

1. Loop `week` from `1` to `18`.
2. For each week, call `Scoreboard.from_raw(get_json(SCOREBOARD_URL, week=w, seasontype=2))`.
3. Use `board.game_for_team("7")`.
4. If no game is found, print `BYE`.

**Hint:** add a small `time.sleep(0.5)` between requests so you don't fire 18
requests instantly. Consider caching results — a completed week's schedule will not
change. Also notice that `board.week` tells you which week the response actually
represents, which is useful for verifying your loop.

---

## Exercise 3 — A "did the score change?" tracker

**Goal:** Extend the polling loop so it prints **only when a score changes**, not on
every poll. Compare parsed fields, not raw JSON.

Example output:

```text
SEA 7 - DEN 3
SEA 7 - DEN 10
SEA 14 - DEN 10
```

**Approach:**

1. Keep a dictionary of the last-seen score for each team, keyed by team id.
2. On each poll, build a `{team.id: team.score for team in game.teams}` dict.
3. Compare it to the stored one; print only if changed, then update.

**Hint:** key by `game.id` first so you can track multiple games at once, then by
`team.id` inside each game. Remember `score` is a **string** — comparing strings is
fine here, but convert with `int(...)` if you need arithmetic.

---

## Exercise 4 — Make a parser total (survive missing data)

**Goal:** Make a parser that never raises on a malformed or partial response. This is
the most important parsing skill: the API will eventually omit something you assumed
was there.

Given the `Game.from_raw` and `Team.from_raw` from Section 3, make each of these
return a sensible default instead of raising:

```python
Scoreboard.from_raw({})                                  # no events at all
Game.from_raw({})                                        # no competitions -> currently a KeyError
Game.from_raw({"competitions": [{}]})                    # competition with no status/competitors
Team.from_raw({"id": "7", "team": {"abbreviation": "DEN"}})  # no score/records
LeaderCategory.from_raw({"name": "sacks", "leaders": []})    # empty category
```

**Approach:**

1. Run each line and print the result.
2. Find the `.get(...)` (or indexing) that still raises, and fix it.
3. Specifically check `raw["competitions"][0]`: a missing key raises `KeyError`, and
   an empty list raises `IndexError`. Decide what a "game with no competition" should
   mean (return `None`, or raise a clear error). There is no single right answer —
   pick one and document it.
4. Check `records[0]["summary"]`-style access — replace it with a search for
   `name == "overall"`.
5. Confirm that `game.situation` is `None` (not a crash) when `situation` is absent.

**Hint:** the usual culprits are `raw["key"]` instead of `raw.get("key", default)`,
and assuming a non-empty list (`records[0]`, `leaders[0]`). Guard both.

> **Why this matters:** a display that runs all season in the background cannot afford
> to crash because one game lacked a `records` array. A total parser turns a network
> data surprise into a blank field instead of a dead process.

---

### See also

- [Python `dict` methods](https://docs.python.org/3/library/stdtypes.html#dict)
- [Python `dataclasses`](https://docs.python.org/3/library/dataclasses.html)
- [Python `typing.Optional`](https://docs.python.org/3/library/typing.html#typing.Optional)
- [Python `timedelta`](https://docs.python.org/3/library/datetime.html#timedelta-objects)

---

**← Previous: [5. Recipes](recipes.md)** &nbsp;|&nbsp; **Next → [7. Summary](summary.md)**
