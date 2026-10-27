---
layout: default
---
# 1. Setup

In this section you'll set up a project folder, learn how to **read an ESPN endpoint
URL**, meet ESPN's two API domains, and write the two helpers used throughout the
tutorial: one that fetches JSON and one that pretty-prints it.

---

## Anatomy of an endpoint

Almost every ESPN URL has the same four parts. Learn to read them and the rest of
the tutorial is just swapping pieces.

```text
https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard?week=6&seasontype=2
└──┬─┘   └──────┬──────┘└────────┬────────┘└────┬────┘└──────┬──────┘└──┬───┘└──────────┬─────────┘
 scheme       host            API prefix      sport        league   resource    query params
```

| Part | Example | What it tells you |
|---|---|---|
| Scheme | `https` | Always HTTPS for these endpoints |
| Host | `site.api.espn.com` | Which API you're talking to (see below) |
| Path | `/apis/site/v2/sports/football/nfl/scoreboard` | What resource you want |
| Query | `?week=6&seasontype=2` | How to filter/select that resource |

> **Read paths as nouns, queries as filters.** The path names the *thing*
> (`scoreboard`, `calendar`, `events`); the query string narrows it (`week=6`,
> `dates=20261011`). When you're unsure what to call, look at the path first, then
> ask what filter you need.

---

## The two ESPN API domains

ESPN exposes (at least) two different public API hosts, and they behave very
differently. Knowing which is which will save you hours of confusion.

| Domain | Base URL | Best for | Response style |
|---|---|---|---|
| **Site API** | `https://site.api.espn.com/apis/site/v2/...` | Scoreboards, summaries, news, standings | Rich, self-contained objects |
| **Core API** | `https://sports.core.api.espn.com/v2/...` | Athletes, events, plays, odds | `$ref`-heavy ("links to data") |

The rule of thumb:

- Want **scores and game state right now**? Use the **Site API**.
- Want **detailed data** (athletes, play-by-play, win probability)? Use the **Core
  API** and follow its `$ref` links.

> **Key idea:** The Core API often returns a *table of contents* — a small object
> full of `$ref` URLs — instead of the data itself. You must request those links to
> get the actual values. The Site API usually embeds the data directly.

> **Unofficial API:** These endpoints are public but **undocumented and unsupported**.
> Field names can change without notice. That is exactly why the parsing pattern in
> this tutorial matters: when a shape changes, you fix one `from_raw` method instead
> of every place in your app that read a raw dictionary.

---

## Create your project folder

```bash
mkdir espn-tutorial
cd espn-tutorial
python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install requests
```

Create a file called `espn.py`. We'll grow it through the tutorial.

---

## Your first request

Start with the **Core API's league calendar** — a good "hello world" because it's
small.

```python
import requests

CALENDAR_URL = (
    "https://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar"
)

response = requests.get(CALENDAR_URL)
response.raise_for_status()          # fail loudly on 4xx/5xx
print(response.text)
```

Run it:

```bash
python espn.py
```

You'll see one long line of JSON. It's valid but very hard to read. Let's fix that.

> **Why `raise_for_status()`?** Without it, `response.json()` on a 404/500 page will
> fail with a confusing decode error. Checking the status first turns a network
> problem into a clear exception.

---

## Two reusable helpers

Throughout the tutorial we'll use the same two functions. Add them to `espn.py`:

```python
import json
import requests

CALENDAR_URL = (
    "https://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar"
)


def get_json(url, **params):
    """GET a URL (with optional query params) and return decoded JSON."""
    response = requests.get(url, params=params, timeout=10)
    response.raise_for_status()
    return response.json()


def show(data):
    """Pretty-print JSON/structured data in a readable, non-escaped way."""
    print(json.dumps(data, indent=2, ensure_ascii=False))


calendar = get_json(CALENDAR_URL)
show(calendar)
```

Why each argument matters:

- `params={...}` — `requests` encodes the query string for you, so you don't have to
  build `?week=6&seasontype=2` by hand.
- `timeout=10` — a request that hangs forever will hang your whole program.
- `raise_for_status()` — converts HTTP errors into Python exceptions.
- `indent=2` — indents nested objects so the structure is visible.
- `ensure_ascii=False` — prints real characters (e.g. accents) instead of `\uXXXX`
  escapes. See the
  [`json.dumps` docs](https://docs.python.org/3/library/json.html#json.dumps).

Now the output is readable:

```json
{
  "count": 4,
  "pageIndex": 1,
  "pageSize": 25,
  "pageCount": 1,
  "items": [
    { "$ref": "http://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar/ondays?lang=en&region=us" },
    { "$ref": "http://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar/offdays?lang=en&region=us" },
    { "$ref": "http://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar/blacklist?lang=en&region=us" },
    { "$ref": "http://sports.core.api.espn.com/v2/sports/football/leagues/nfl/calendar/whitelist?lang=en&region=us" }
  ]
}
```

That is the **table of contents** mentioned earlier: `count` of `4` items, and four
`$ref` links. There is no actual calendar data here yet — we follow those links in
the next section.

---

## The parsing mindset

`get_json()` gives you plain Python: nested `dict`s and `list`s. You *could* read
values directly with long index chains:

```python
# Works, but brittle and hard to read:
state = event["competitions"][0]["status"]["type"]["state"]
```

Every one of those keys is a chance to crash, and the meaning is buried in brackets.
Instead, we'll follow one pattern for the whole tutorial: **a `from_raw` classmethod
on a small dataclass**.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class GameStatus:
    state: str
    period: int

    @classmethod
    def from_raw(cls, raw: dict) -> "GameStatus":
        status_type = raw.get("type", {})
        return cls(
            state=status_type.get("state", "pre"),
            period=raw.get("period", 0),
        )
```

Three habits to notice, because every parser in this tutorial uses them:

1. **`raw.get(key, default)` instead of `raw[key]`** — a missing key becomes a sane
   default instead of a `KeyError`.
2. **`raw.get("type", {})` then `.get(...)`** — defends against a whole nested
   object being absent.
3. **One class per JSON object** — each `from_raw` only knows about its own slice of
   the response. Nesting is handled by calling the child's `from_raw`.

The payoff: your display code reads `game.status.state` instead of
`game["competitions"][0]["status"]["type"]["state"]`, and when ESPN renames a field
you change exactly one method.

> **Don't write every parser up front.** Parse only the fields your display needs.
> A scoreboard response embeds team logos, tickets, odds, and weather you may never
> touch.

---

## A note on opening a browser JSON viewer

For very large responses (the scoreboard can exceed 250 KB), pretty-printing to the
terminal is still unwieldy. A handy alternative is to view the URL directly in a
browser — modern browsers render JSON with folding. You can also pipe to a tool:

```bash
python espn.py | python -m json.tool
```

> Make sure your script prints **only** JSON for that pipe to work.

---

### See also

- [Python `json` module](https://docs.python.org/3/library/json.html)
- [`requests` Quickstart](https://requests.readthedocs.io/en/latest/user/quickstart/)
- [Python `dataclasses`](https://docs.python.org/3/library/dataclasses.html)
- [ESPN Core API WADL (list of all endpoints)](https://sports.core.api.espn.com/v2/application.wadl?detail=true)

---

**← Previous: [Introduction](index.md)** &nbsp;|&nbsp; **Next → [2. League Calendar](calendar.md)**
