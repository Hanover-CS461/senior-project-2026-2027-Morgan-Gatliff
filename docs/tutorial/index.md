# ESPN Public API: Pull Live NFL Data with Python

Welcome! This tutorial teaches you how to pull **live NFL data** — scores, game
clock, field position, and schedules — from ESPN's public API using Python, and how
to **parse that nested JSON into small, typed Python objects** you can actually use.
It is the data foundation for the Live NFL Football Display project, but everything
here stands on its own if you just want to fetch and structure NFL data.

---

## Learning objectives

By the end of this tutorial you will be able to:

1. **Read an ESPN endpoint URL** — its host, path, and query parameters — and choose
   the right domain for a task.
2. Query the **league calendar** endpoints, follow their `$ref` links, and parse the
   season structure into objects.
3. **Parse nested JSON into dataclasses** using the `from_raw` pattern, including
   missing keys, optional nested objects, and lists of child objects.
4. Parse the **live scoreboard** into `Scoreboard`, `Game`, `Team`, `GameStatus`, and
   `Situation` objects, and read score, quarter, clock, down, distance, and field
   position from them.
5. Reuse the same parsers for **past weeks** and a specific **favorite team's** game.
6. Write a small **week tracker** and a **polling loop** on top of the parsed model.

## Target audience

This tutorial is for:

- A programmer who **knows the basics of Python** (variables, functions,
  `for` loops, dictionaries, classes) but has **never used ESPN's API**.
- Someone comfortable running commands in a terminal.
- Anyone building a sports dashboard, scoreboard, or notification tool.

You do **not** need to know Flask, React, or web development — this tutorial is
purely the data layer.

## Prerequisites

| Requirement | Why | Check |
|---|---|---|
| Python 3.9+ | The examples use modern Python and dataclasses | `python3 --version` |
| `requests` library | Sends the HTTP requests | `pip show requests` |
| An internet connection | We query a live API | — |
| A text editor or IDE | To write and run the code | — |

If `pip show requests` returns nothing, install it:

```bash
pip install requests
```

> **No API key needed.** ESPN's public API is undocumented but does not require
> authentication for the endpoints we use.

---

## How to use this tutorial

Work through the sections in order. Each page ends with a **Next →** link, and you
can always jump back with **← Previous**. The concepts build on each other: we fetch
a response, look at its shape, then write a parser for it, then use the parsed
objects.

| # | Section | What you'll learn |
|---|---|---|
| 1 | [Setup](setup.md) | Endpoint URL anatomy, the two domains, and the parsing mindset |
| 2 | [League Calendar](calendar.md) | `$ref` links, season structure, parsing sections, week tracker |
| 3 | [Live Scoreboard](scoreboard.md) | The full model: status, teams, situation, plays, leaders |
| 4 | [Past Weeks](historical.md) | Query parameters and reusing the same parsers |
| 5 | [Recipes](recipes.md) | `$ref` resolution, week tracker, team lookup, polling loop |
| 6 | [Exercises](exercises.md) | Practice writing parsers on your own |
| 7 | [Summary](summary.md) | Endpoint catalog, model recap, where to go next |

Let's begin.

---

**Next → [1. Setup](setup.md)**
