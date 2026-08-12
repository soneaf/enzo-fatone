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

Both stat pages read the same database views. `basketball/index.html` is the one
with the sheet integration and the film, news and schedule sections; `stats/index.html`
is stats only.

---

## Where the numbers come from

**The 2026-27 season comes live from the EnzoStats app's Supabase database**
(`~/Server/Developer/EnzoStats`, project `wzmtpvqbngevlcelolor`). Whitney scores a game on
her phone; this page shows it. No export, no spreadsheet to keep tidy.

It reads six read-only views and nothing else:

| View | Feeds |
|---|---|
| `public_season_totals` | the PPG / RPG / APG / FG% strip; the shooting splits and eFG%/TS% on `/stats` |
| `public_game_log` | the season stats table, trends, per-game lines, and every table on `/stats` |
| `public_shots` | the shot chart on both pages |
| `public_schedule` | the Upcoming table on `/stats` |
| `public_live_game` | the live scoreboard while a game is on |
| `public_game_periods` | the By Period section on `/stats` — added 2026-08-11 |

Between them they carry: a full box-score line per game, the four shooting splits,
eFG%/TS%, shot coordinates with the shot's value, and **as of 2026-08-11** the period
breakdown, points-in-paint, minutes played and the game's format. The venue is published
for *upcoming* games only, through `public_schedule`.

### The security model, and why it looks the way it does

The key in the page is the **publishable** key and is meant to be public. The protection
is not the key, it is that the key can only reach those six views:

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

**This is no longer theory.** Adding `public_game_periods` on 2026-08-11 was the first
chance to test it, and it was tested rather than assumed: inside a transaction that rolls
back, with a final game and eleven shots seeded, `SET LOCAL ROLE anon` sees **4 rows** from
`public_game_periods` (built on `game_events`) and **0 rows** from `game_period_scores`
(`security_invoker=on`) for the same game. Nothing was published and no push fired — the
whole probe ends in a deliberate `raise exception`, which is also how the push plumbing was
probed without retiring a real device row.

### The schedule and the live score

Added 2026-08-10. They are the only two views that show anything *before* a game is over —
everything else on the site is finished games only.

`public_schedule` is every non-final game in the active season, with the venue **and the
street address**. Steven's call, made deliberately: it is the same information schools and
MaxPreps already publish, and a coach deciding whether to drive needs it. Worth being
conscious that it puts a minor's location and time on an indexed page — that was weighed,
not overlooked, and it is the reason to think twice before adding anything more personal.

`public_live_game` is the in-progress game with Enzo's line derived as it goes. It
recomputes the box score from `game_events` with the counting maths **copied verbatim from
`public_game_log`**, for exactly the reason that one does it: `game_box_scores` is
`security_invoker=on` and returns nothing to `anon`.

**Polled, not pushed** — every 25 seconds. Realtime would mean opening websocket access to
anonymous visitors, which is far wider exposure than a read-only view, for a number that
moves every few minutes.

The freshness line is the part worth keeping. A score that hasn't moved because nobody
scored has to look different from one that hasn't moved because the scorer's phone lost
signal — the same reasoning as `LinkState` in the app. After four minutes with no event the
card says the score may be behind rather than quietly implying it is current.

**The schedule renders outside `#content`**, deliberately. Before the first game there are
no finals and the load path returns early on the empty state; a page with a full schedule
and no results should show the schedule, which is the most useful thing on it in October.

### The calendar subscription

`webcal://…/functions/v1/schedule-ics` — an edge function that serves the season as an
`.ics` feed, linked from the Upcoming section. A **subscription**, not an export: when a
game moves, the event moves with it, because each VEVENT keys on the game's own id.
That is why `public_schedule` exposes `id` — an opaque UUID for a row `anon` cannot read.

**JWT verification is off, and has to be** — a calendar client cannot authenticate. It is
safe only because the function reads `public_schedule` and `public_game_log` with the
*publishable* key, so the feed exposes exactly what the page already does and nothing more.
If anyone ever points it at a base table or the service key, that stops being true.

Finished games stay in the feed as all-day events carrying the result, so the calendar
doubles as a season record. Upcoming games with no tip-off are all-day too — a guessed
7pm is a time somebody drives to, the same reasoning the app uses for game-day reminders.

Two things that bit during the build, both worth knowing:

- **Postgres returns `tip_off` as `"19:00:00"`**, while the app stores `"19:00"`. A single
  `replace(":", "")` strips only the first colon and produced `T1900:0000` — every timed
  game malformed. Caught only by testing against real rows, not fabricated ones.
- **Lines must be CRLF and folded at 75 octets.** Apple Calendar forgives neither
  consistently and Google forgives less, which shows up as "works on my phone, empty on
  everyone else's".

Timezone is fixed at `America/New_York`. Right for Florida and Georgia; an hour out for the
rare Central-time trip (Jackson County, TN). Deliberate — guessing per venue from an
address would be a worse kind of wrong.

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
recruiting page by the gold "Full Season Stats" bar above the Season Stats panel. It
touches nothing but the public views — the sheet, the YouTube feed and the Twitter widget
are all absent, so this page makes exactly six network requests:

```
GET public_season_totals?select=*
GET public_game_log?select=*&order=played_on.asc
GET public_shots?select=*&order=played_on.asc
GET public_schedule?select=*
GET public_live_game?select=*
GET public_game_periods?select=*&order=played_on.asc,period.asc
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

### The By Period section

Added 2026-08-11, from `public_game_periods`. Average points per period as bars, then a
table with one row per game and the game column frozen.

**It is Enzo's points, and it says so twice** — in the section note and in the app's
matching screens. There is no team score per period anywhere in this system: `games` holds
one running total, and the period column lives on `game_events`, which only ever records
his plays. A row of numbers under a scoreboard reads as the team's line unless something
says otherwise.

Columns are driven by the deepest period anyone actually reached, so a season with no
overtime shows no OT column and a double-overtime game adds two. Labels come from the
game's `format`: quarters unless **every** game was played in halves, since a single header
row over mixed formats is a lie either way. Periods with no plays are absent from the view
rather than zero, so the table fills gaps itself — a quiet quarter is real and common, and
treating a missing row as missing data would leave holes.

The whole section, heading included, hides when there are no period rows. A finished game
with nothing recorded against it is possible — somebody starts the scorer and nobody taps —
and a "By Period" heading over nothing reads as a broken page rather than an empty one.

### Things it deliberately does not do
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

Nothing on this repo, which is worth stating plainly rather than leaving an empty heading.
The site reads five public views and renders them; every open question about *what* it
shows is a decision about which columns to expose, and those live in the EnzoStats handoff.

The period breakdown, minutes played and points-in-paint that used to sit here are all
**shipping** — see "By Period" above and "Recent history" below. The game-day reminder is
built and shipping too.

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

- **By Period added, 2026-08-11**, along with points-in-paint and minutes in the box
  scores. Backed by a new `public_game_periods` view and four columns appended to
  `public_game_log` (`id`, `points_in_paint`, `seconds_played`, `format`) — appended
  because `CREATE OR REPLACE VIEW` may only add at the end and `public_season_totals` is
  defined over it. Verified against the live views (both return `200`, empty, which is
  correct in August), against the role-switching rollback probe described above, and
  rendered against fabricated local data that was never committed — eight games, one of
  them into overtime, so the OT column could be seen appearing.
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
