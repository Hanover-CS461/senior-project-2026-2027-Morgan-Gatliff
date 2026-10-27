# 2. League Calendar

The calendar endpoints describe the **shape of the NFL season**: which days have
games, and how the season is divided into preseason, regular season, postseason,
and offseason. This is static data — it changes once a year — which makes it perfect
for a "week tracker".

This section does two things: it shows you how to **navigate the Core API's `$ref`
links**, and it shows how to **parse the resulting nested lists into objects**.

---

## The base calendar endpoint

```
https://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar
```

Read that path left to right: `sports/football/leagues/nfl` is the league, and
`/calendar` is the resource. As you saw in Setup, this returns four `$ref` links.
Those four links point to the **sub-endpoints** of the calendar:

| Sub-endpoint | Path | Meaning |
|---|---|---|
| `ondays` | `/calendar/ondays` | Days on which games are played |
| `offdays` | `/calendar/offdays` | Days with no games |
| `blacklist` | `/calendar/blacklist` | A second "no games" list |
| `whitelist` | `/calendar/whitelist` | A second "games" list |

> **A real observation:** `ondays` and `whitelist` return **identical** data, and
> `offdays` and `blacklist` return **identical** data (apart from a `type` label and
> a self-referencing `$ref`). In practice you only need `ondays` to know the game
> days. We keep all four here because the API offers them.

> **Endpoint rule of thumb:** when a Core API response has `count` + `items` of
> `$ref`s, you are looking at a *collection*. Pick the child you need and request it
> directly — you never have to resolve all of them.

---

## Following a `$ref`

Fetch the `items` from the base calendar, then request the link you need. This is
the general pattern for **every** Core API `$ref`:

```python
import json
import requests

BASE = "https://sports.core.api.espn.com/v2/sports/football/leagues/nfl"


def get_json(url, **params):
    response = requests.get(url, params=params, timeout=10)
    response.raise_for_status()
    return response.json()


def show(data):
    print(json.dumps(data, indent=2, ensure_ascii=False))


calendar_index = get_json(f"{BASE}/calendar")

for item in calendar_index["items"]:
    link = item["$ref"]
    detail = get_json(link)
    print("=" * 60)
    print(detail["type"])
    print("=" * 60)
    show(detail)
```

Notice that `get_json(link)` works on the `$ref` string directly: it is a full URL,
including `?lang=en&region=us`, so `requests` can use it as-is.

> **Cost warning:** each `$ref` is a separate HTTP request. The base calendar has
> only four, so resolving all of them is fine. Do not carry this habit over to
> events, where one response can link to hundreds of players.

---

## What a sub-endpoint returns

Here is a trimmed `ondays` response. The interesting fields are `eventDate.dates`
(the list of game days) and `sections` (the season structure):

```json
{
  "$ref": "http://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar/ondays?lang=en&region=us",
  "type": "ondays",
  "startDate": "2026-08-06T07:00Z",
  "endDate": "2027-02-16T07:59Z",
  "eventDate": {
    "type": "ondays",
    "dates": ["2026-08-06T07:00Z", "2026-08-13T07:00Z"]
  },
  "sections": [
    {
      "label": "Preseason",
      "value": "1",
      "startDate": "2026-08-06T07:00Z",
      "endDate": "2026-09-06T06:59Z",
      "entries": [
        { "label": "Preseason Week 1", "value": "2", "detail": "Aug 13-19",
          "startDate": "2026-08-13T07:00Z", "endDate": "2026-08-20T06:59Z" }
      ]
    },
    {
      "label": "Regular Season",
      "value": "2",
      "startDate": "2026-09-06T07:00Z",
      "endDate": "2027-01-13T07:59Z",
      "entries": [
        { "label": "Week 1", "value": "1", "detail": "Sep 6-15",
          "startDate": "2026-09-06T07:00Z", "endDate": "2026-09-16T06:59Z" }
      ]
    }
  ]
}
```

### The shape, in words

```text
ondays                     <- one JSON object
├── type                   <- "ondays"
├── eventDate.dates[]      <- flat list of game days (strings)
└── sections[]             <- season phases
    ├── label              <- "Preseason" / "Regular Season" / ...
    ├── value              <- season type: 1, 2, 3, 4
    └── entries[]          <- weeks/rounds
        ├── label          <- "Week 1", "Wild Card", ...
        ├── value          <- week number ("1".."18", "1".."4")
        ├── detail         <- human text ("Sep 6-15")
        └── startDate/endDate
```

That is a **list of lists**: sections contain entries. Parsing it means flattening
that two-level structure into something your code can loop over.

### Fields worth knowing

| Field | Type | Meaning |
|---|---|---|
| `type` | string | Which sub-endpoint this is (`ondays`, `offdays`, ...) |
| `eventDate.dates` | list | Individual game days (or off days) |
| `sections[].label` | string | Season phase: `Preseason`, `Regular Season`, `Postseason`, `Off Season` |
| `sections[].entries[].label` | string | The week/round name, e.g. `Week 1`, `Wild Card` |
| `sections[].entries[].value` | string | The week number **as a string** |
| `sections[].entries[].startDate` / `endDate` | string | Exact date range of that week |

> **Type gotcha:** `value` is a **string**, not an integer. If you need a number,
> call `int(entry.value)`. Keep the raw string if you plan to pass it back as a query
> parameter (`?week=6`).
>
> **Tip:** Prefer `startDate` / `endDate` for any calculation. The human-readable
> `detail` field (e.g. `"Sep 6-15"`) does not always exactly match the real range.

---

## Parsing the calendar into objects

Two nested dataclasses mirror the two nested JSON levels. Each `from_raw` handles
only its own object and delegates downward.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class WeekEntry:
    label: str
    value: str          # keep as string: it is a query param value
    start_date: str
    end_date: str
    detail: str = ""

    @classmethod
    def from_raw(cls, raw: dict) -> "WeekEntry":
        return cls(
            label=raw.get("label", ""),
            value=raw.get("value", ""),
            start_date=raw.get("startDate", ""),
            end_date=raw.get("endDate", ""),
            detail=raw.get("detail", ""),
        )


@dataclass(frozen=True)
class SeasonSection:
    label: str
    value: str
    entries: tuple      # of WeekEntry

    @classmethod
    def from_raw(cls, raw: dict) -> "SeasonSection":
        return cls(
            label=raw.get("label", ""),
            value=raw.get("value", ""),
            entries=tuple(WeekEntry.from_raw(e) for e in raw.get("entries", [])),
        )


@dataclass(frozen=True)
class Calendar:
    sections: tuple     # of SeasonSection

    @classmethod
    def from_raw(cls, raw: dict) -> "Calendar":
        return cls(
            sections=tuple(
                SeasonSection.from_raw(s) for s in raw.get("sections", [])
            )
        )

    def entries(self):
        """Flatten sections -> (section, entry) pairs."""
        for section in self.sections:
            for entry in section.entries:
                yield section, entry
```

Why the `entries()` generator instead of building a big list? It saves you from
writing the same nested `for` loop everywhere, and it keeps the section context
(you often want to know that `Week 6` belongs to `Regular Season`).

> **Parsing note:** `Off Season` has **no** `entries` key. The
> `raw.get("entries", [])` default makes that section parse to an empty tuple
> instead of raising a `KeyError`. Look for the missing key in the sample — it is
> the most common source of crashes in this data.

---

## Why this matters: a week tracker

Because every week has a `startDate` and `endDate`, you can determine **the current
week** purely from today's date — no polling required. A date falls inside exactly
one entry's range.

```python
from datetime import datetime, timezone


def _parse(ts):
    return datetime.fromisoformat(ts.replace("Z", "+00:00"))


def current_week(calendar, now=None):
    """Return (phase, week_label, week_value) for the current date, or None."""
    now = now or datetime.now(timezone.utc)
    for section, entry in calendar.entries():
        if _parse(entry.start_date) <= now < _parse(entry.end_date):
            return section.label, entry.label, entry.value
    return None  # e.g. during the Off Season
```

Running this on a mid-October day returns something like
`("Regular Season", "Week 6", "6")`. We'll put this to work in the
[Recipes](recipes.md#recipe-2-a-week-tracker) section.

> **Why `replace("Z", "+00:00")`?** ESPN timestamps end in `Z` (UTC). Older Python
> versions' `fromisoformat` rejects `Z`, so we rewrite it to the equivalent offset.
> Modern Python (3.11+) accepts `Z`, but this form works everywhere.

> **Design note:** Fetch the calendar **once** (at startup or once a day) and cache
> it. It is static data — do not poll it every minute.

---

## Practice: inspect a sub-endpoint directly

1. Copy an `ondays` URL from the output.
2. Open it in your browser.
3. Find the `Off Season` section and confirm it has no `entries` array.
4. Confirm that `whitelist` returns the same `dates` array as `ondays`.

---

### See also

- [Python `datetime`](https://docs.python.org/3/library/datetime.html)
- [`str.isoformat` / `fromisoformat`](https://docs.python.org/3/library/datetime.html#datetime.datetime.fromisoformat)
- [Python `dataclasses`](https://docs.python.org/3/library/dataclasses.html)
- [Community ESPN API documentation](https://github.com/pseudo-r/Public-ESPN-API)

---

**← Previous: [1. Setup](setup.md)** &nbsp;|&nbsp; **Next → [3. Live Scoreboard](scoreboard.md)**
