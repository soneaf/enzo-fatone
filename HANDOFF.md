# enzo-fatone.com — handoff

Enzo's recruiting site. Static, GitHub Pages, `CNAME` → `enzo-fatone.com`, repo
`soneaf/enzo-fatone`. **Deploying is `git push origin main`** — Pages rebuilds in under a
minute. There is no build step and nothing to run.

```
index.html              landing page
basketball/index.html   the recruiting page
stats/index.html        the public season page — the detail the recruiting page omits
submit/index.html       X-post submission form
```

It is one self-contained HTML file per page: styles, markup and script inline. That is
deliberate for a site with no toolchain, but it means `basketball/index.html` is ~1450
lines and the script sits at the bottom.

Both stat pages read the same three database views. `basketball/index.html` is the one
with the sheet integration and the film, news and schedule sections; `stats/index.html`
is stats only.

---

## Where the numbers come from

**The 2026-27 season comes live from the EnzoStats app's Supabase database**
(`~/Server/Developer/EnzoStats`, project `wzmtpvqbngevlcelolor`). Whitney scores a game on
her phone; this page shows it. No export, no spreadsheet to keep tidy.

It reads exactly three read-only views and nothing else:

| View | Feeds |
|---|---|
| `public_season_totals` | the PPG / RPG / APG / FG% strip; the shooting splits and eFG%/TS% on `/stats` |
| `public_game_log` | the season stats table, trends, per-game lines, and every table on `/stats` |
| `public_shots` | the shot chart on both pages |

Between them they carry: a full box-score line per game, the four shooting splits,
eFG%/TS%, and shot coordinates with the shot's value. **They carry no period breakdown,
no minutes played, no points-in-paint and no venue.** Those exist in the app but were not
put in the public views, so anything needing them is a new column and therefore a
security decision — see below.

### The security model, and why it looks the way it does

The key in the page is the **publishable** key and is meant to be public. The protection
is not the key, it is that the key can only reach those three views:

- Base tables stay closed. RLS on them is authenticated-only and `anon` is granted nothing.
- Verified by trying, with data present: reading `games`, `game_events`, `players`,
  `seasons`, `recipients` and `devices` as `anon` returns `[]` every time while all of them
  hold rows. Writes were tested too — `DELETE` and `UPDATE` come back `204` having matched
  zero rows, with the data unchanged afterwards.
- The views expose only stat columns. No user ids, no device tokens, no email addresses.
  **Column choice is the security boundary**, so think before adding one.

**The trap, if you ever add another public view:** the app's own `season_totals`,
`game_box_scores` and `game_period_scores` are declared `security_invoker=on`, so they
execute as the caller and re-apply RLS. A public view built on top of them returns nothing
to `anon` no matter that the public view is itself a definer view — the inner view puts the
policy back. **Read the base tables directly.** That is why `public_game_log` recomputes
the box score instead of selecting from `game_box_scores`, with the counting maths copied
across verbatim so the site and the app cannot drift.

### The sheet is still there, for two things

`API_URL` still points at the Google Apps Script wrapping the old sheet, and `SHEET_DATA`
supplies the **news features** and the **subscriber list** — things the tracker doesn't
know about. Only the stats moved.

### The adapter

`adaptGame()` reshapes a database row into the column names the rendering was written
against, so four working functions did not need rewriting. One trap in there:

> **`FGM`/`FGA` in the sheet vocabulary mean TWO-pointers.** `loadStats` adds `FGM + 3PM`
> to get total field goals, which only balances if threes are excluded. Confirmed against a
> real sheet row. The database means the opposite — `fgm` there counts every shot worth 2
> or 3 — hence `FGM: g.fgm - g.fg3m` in the adapter.

### Shot chart

NFHS half court in feet: 50 wide, 47 deep, baseline at the top, hoop at `x=0.5`,
`y=5.25/47`. Identical to `CourtView.swift` in the app, so a shot's stored 0…1 coordinates
map straight onto the `viewBox` with no conversion. Section hides itself when nothing has
been charted.

---

## `stats/index.html` — the public season page

Built for family and for recruiters who want more than a summary. Linked from the
recruiting page by the gold "Full Season Stats" bar above the Season Stats panel. Reads
the same three views and touches nothing else — the sheet, the YouTube feed and the
Twitter widget are all absent, so this page has exactly three network requests:

```
GET public_season_totals?select=*
GET public_game_log?select=*&order=played_on.asc
GET public_shots?select=*&order=played_on.asc
```

### What it adds over the recruiting page

The recruiting page answers "is this player worth a call" in one strip and one table.
This one answers "what actually happened this season":

- **Shooting** — all four splits (FG, 2PT, 3PT, FT) as makes-over-attempts with bars,
  plus eFG% and TS% each explained in a sentence. The recruiting strip shows FG% alone.
- **Season totals** — every counting stat as a total *and* a per-game figure, including
  the offensive/defensive rebound split, steals, blocks, turnovers and fouls. Some of
  these are nowhere on the recruiting page.
- **How he scores** — one stacked bar per game, split into free throws / twos / threes.
  Deliberately not the recruiting page's plain points bar chart.
- **Game log** — twenty columns rather than eighteen, with the game column frozen, and a
  *totals* row as well as an averages row.
- **Box scores** — one expandable card per game with the full line and that game's own
  splits, eFG% and TS%. Newest first.
- **Shot chart** — the same court, but filterable to a single game, and with a
  paint / mid-range / three zone breakdown underneath.

**There is honest overlap.** The season stats table and the shot chart cover much of what
the recruiting page's collapsed panel already shows. That was accepted rather than
avoided: the recruiting page has to stand alone for a coach who never clicks through, so
duplicating the summary is the point of it.

### Things it deliberately does not do

- **No period-by-period breakdown.** It was asked for and it is not possible today: the
  public views carry no period data, and `game_period_scores` is `security_invoker=on`, so
  building on it returns nothing to `anon` (the trap above). It would need a new public
  view — a security decision, and Steven's to make.
- **No `SEASON_START` date gate.** The recruiting page carries one because it inherited
  the vocabulary of a sheet that holds summer games. This page does not need it:
  `public_game_log` joins `seasons.is_active`, so it returns the 2026-27 season and
  nothing else, and `public_season_totals` is defined as `SELECT … FROM public_game_log`,
  so the two can never disagree. **Adding a client-side date filter here would break that
  guarantee**, because the totals row cannot be filtered to match. The season boundary
  lives in the database, which is a stronger gate than a date string.
- **Rates are never averaged.** One `rate(made, attempts)` helper handles every percentage
  on the page, and the totals rows divide summed makes by summed attempts. Percentages
  that arrive already computed in SQL (`fg_pct`, `ts_pct`, …) are passed through, not
  recomputed.
- **Database FG semantics, not the sheet's.** `fgm`/`fga` here count every shot worth 2
  or 3, so the table's FGM column includes threes and the twos are derived by subtraction
  (`twos()`). This is the opposite convention to `adaptGame()` next door — the two pages
  disagree on the *label* and agree on the *arithmetic*, which is why the FG% they show
  is the same number.

### Three states, and why they are three

`sb()` throws on any non-2xx, so "the season hasn't started" and "the request failed" are
different screens. An empty season is the correct state for most of the year, and it gets
a written panel rather than a page of `0.0%` and `NaN`. Getting this wrong would mean the
page looked broken from August to November every year.

---

## Open — for the next session

### 1. Game-day reminder (app-side, not this repo)

Agreed but not built. See the EnzoStats handoff — the reasoning is that the biggest risk
to the whole system is nobody starting the scorer, and a game that isn't scored produces
no events, no email and no season line, with no way to recover it afterwards.

---

## Decided — do not reopen

### The summer-2026 games stay off the site

**Decided by Steven, 2026-08-09: the site is senior-season only.** This was the previous
session's open question; it is closed.

The sheet holds 22 games from June–July 2026 (16 CFCA, 3 AAU, 3 with no team set). None of
them have ever appeared on the page — `SEASON_START = '2026-09-01'` filtered them out
before any of this work, so nobody was ever looking at them.

The earlier instinct was to show the AAU games separately under a neutral label ("Club" /
"AAU"), but that answer was given on the mistaken understanding that they were already
being counted. They weren't, so there was nothing to correct — only a new section to add,
and Steven decided not to add it. The reasoning:

- The site's job is to present the **senior season** to recruiters. A summer block sitting
  next to it is a second, weaker dataset competing for attention with the one that matters.
- Those games came from the old spreadsheet, whose numbers are the reason this system was
  rebuilt. The corresponding decision on the app side is already recorded as final: *"Do
  not migrate the old summer-2026 stats."* Publishing them on the website would be that
  migration by another route.
- Nothing needed undoing to make this true — it was already the behaviour.

Practically: the sheet data is still fetched and still feeds the news features and the
subscriber list; it is simply never used for stats. `stats/index.html` cannot show the
summer games at all, because they are not in the database. **If this is ever reopened, it
means adding a section, not removing a filter.**

---

## Recent history

- **`stats/index.html` added** — the public season page, linked from the recruiting page.
  Verified against the live views (all three return `200`; the season is empty, which is
  correct in August) and against fabricated local data, which was never committed. Three
  states checked in the browser: loading, empty season, and request failure.
- John Elgani's coach card removed entirely, and every mention of Sunshine Elite taken out
  of both pages — vitals bar, subhead, both `og:description` tags, the team-filter button
  and the code comments. Verified against the rendered DOM rather than the source.

  **One mention survives on purpose.** An embedded tweet in "In The News" quotes Sunshine
  Elite in its own text. Steven's call on 2026-08-09 was to leave it: it is someone else's
  words in approved third-party content, not site copy, and editing a quote to remove a
  team name is not a thing you do. So a `grep` for the phrase still hits — that is not a
  leftover to clean up. Site copy stays clear of it; quoted material is not site copy.
- The team filter still exists and still works; it reveals itself only when the data holds
  more than one team, which it no longer does, so it stays hidden.
