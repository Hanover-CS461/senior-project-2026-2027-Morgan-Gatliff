# 5. Recipes

This section collects four practical patterns you'll reuse throughout the project.
Each one builds on the earlier sections and operates on the **parsed model**, not raw
dictionaries.

---

## Recipe 1: Resolve `$ref` links automatically

The Core API nests links inside links. This recursive helper walks a JSON object and
fetches every `$ref` it finds, replacing it with the actual data.

```python
import requests

session = requests.Session()  # reuse the connection


def resolve(obj, depth=2):
    """Recursively replace {'$ref': url} objects with their JSON (to a depth)."""
    if depth < 0:
        return obj
    if isinstance(obj, dict):
        if "$ref" in obj:
            return resolve(session.get(obj["$ref"]).json(), depth - 1)
        return {key: resolve(value, depth - 1) for key, value in obj.items()}
    if isinstance(obj, list):
        return [resolve(item, depth - 1) for item in obj]
    return obj
```

> **Warning:** follow `$ref`s **selectively**. A single event can link to hundreds of
> players, each of which links to more data. Unbounded recursion will hammer the API
> and slow your program to a crawl. Prefer addressing the exact `$ref` you need.

> **When to use this:** collection endpoints (`calendar`, `athletes`, `events`) return
> `count` + `items[]` of links. Resolving the whole collection is usually wasteful —
> pick the item you need. `resolve` is a *debugging* tool for exploring an unfamiliar
> response, not a production strategy.

### The `.pvt` gotcha

Some Core API responses contain `$ref` URLs on ESPN's **internal** domain,
`sports.core.api.espn.pvt`, which is not publicly reachable. Replace `.pvt` with
`.com` before requesting:

```python
def fix_pvt(url):
    return url.replace("sports.core.api.espn.pvt", "sports.core.api.espn.com")
```

---

## Recipe 2: A week tracker

Fetch the calendar once, cache it, and compute the current week from the date. This
drives a "Week 6" header and can trigger logic when the week rolls over.

```python
from datetime import datetime, timezone

BASE = "https://sports.core.api.espn.com/v2/sports/football/leagues/nfl"


def load_calendar():
    """Fetch the calendar once and parse it into a Calendar object."""
    return Calendar.from_raw(get_json(f"{BASE}/calendar/ondays"))


def _parse(ts):
    return datetime.fromisoformat(ts.replace("Z", "+00:00"))


def current_week(calendar, now=None):
    now = now or datetime.now(timezone.utc)
    for section, entry in calendar.entries():
        if _parse(entry.start_date) <= now < _parse(entry.end_date):
            return {
                "phase": section.label,
                "label": entry.label,
                "value": entry.value,
                "ends": entry.end_date,
            }
    return None  # Off Season


calendar = load_calendar()
print(current_week(calendar))
```

Exercise: store the last-seen `value` and print a message when it changes.

> **Cache this.** The calendar is static. One request per day is plenty; one per
> poll is wasteful.

---

## Recipe 3: Find your favorite team's live game

Because `Scoreboard` already knows how to walk `competitors`, the lookup is a single
method call. Team ids are strings (`"7"` = Denver), so compare as strings.

```python
def find_team_game(board, team_id):
    return board.game_for_team(team_id)


board = Scoreboard.from_raw(get_json(SCOREBOARD_URL))
game = find_team_game(board, "7")

if game is None:
    print("Denver has no game today.")
else:
    print(game.name, "—", game.status.label)
```

> **Id vs abbreviation:** the API keys teams by numeric id (`"7"`), not by
> abbreviation (`"DEN"`). If your input is an abbreviation, map it first — see the
> `TEAM_IDS` table in the project's `request.py`.

### Parsing a team's leaders

Once you have the game, the leaders are already parsed:

```python
for category in game.leaders:
    top = category.leaders[0] if category.leaders else None
    if top:
        print(f"{category.name}: {top.name} — {top.display_value}")
```

---

## Recipe 4: A polling loop

An always-on display checks the API on an interval and reacts when a game goes live.
Poll every **30–60 seconds** — frequent enough to feel live, gentle enough on the
API.

```python
import time

POLL_SECONDS = 45
last_state = None


while True:
    try:
        board = Scoreboard.from_raw(get_json(SCOREBOARD_URL))
        game = board.game_for_team("7")
    except Exception as exc:          # network error, bad JSON, etc.
        print("poll failed:", exc)
        time.sleep(POLL_SECONDS)
        continue

    if game:
        state = game.status.state
        if state != last_state:
            print(f"Game state changed: {last_state} -> {state}")
            last_state = state
        if game.status.is_live:
            print(summarize(game))

    time.sleep(POLL_SECONDS)
```

> **Best practice:** parse first, then compare model fields. Comparing raw JSON on
> every poll is error-prone; comparing `game.status.state` is not.

> **Failure mode to watch:** if parsing raises inside the loop, you'll lose the whole
> iteration. Keep the request **and** the parse inside `try`/`except`, as above.

---

### See also

- [`requests.Session`](https://requests.readthedocs.io/en/latest/api/#requests.Session)
- [Python `time.sleep`](https://docs.python.org/3/library/time.html#time.sleep)
- [Python `datetime`](https://docs.python.org/3/library/datetime.html)
- [Python `dataclasses`](https://docs.python.org/3/library/dataclasses.html)

---

**← Previous: [4. Past Weeks](historical.md)** &nbsp;|&nbsp; **Next → [6. Exercises](exercises.md)**
