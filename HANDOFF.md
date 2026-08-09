# enzo-fatone.com — handoff

Enzo's recruiting site. Static, GitHub Pages, `CNAME` → `enzo-fatone.com`, repo
`soneaf/enzo-fatone`. **Deploying is `git push origin main`** — Pages rebuilds in under a
minute. There is no build step and nothing to run.

```
index.html              landing page
basketball/index.html   the recruiting page — everything below is about this file
submit/index.html       X-post submission form
```

It is one self-contained HTML file per page: styles, markup and script inline. That is
deliberate for a site with no toolchain, but it means `basketball/index.html` is ~1400
lines and the script sits at the bottom.

---

## Where the numbers come from

**The 2026-27 season comes live from the EnzoStats app's Supabase database**
(`~/Server/Developer/EnzoStats`, project `wzmtpvqbngevlcelolor`). Whitney scores a game on
her phone; this page shows it. No export, no spreadsheet to keep tidy.

It reads exactly three read-only views and nothing else:

| View | Feeds |
|---|---|
| `public_season_totals` | the PPG / RPG / APG / FG% strip |
| `public_game_log` | the season stats table, trends, per-game lines |
| `public_shots` | the shot chart |

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

## Open — for the next session

### 1. The summer-2026 games: show them, or not?

The sheet holds **22 games from June–July 2026** (16 CFCA, 3 AAU, 3 with no team set).
**None of them have ever appeared on the page** — `SEASON_START = '2026-09-01'` filters
them all out, and it did so before any of this work.

Steven's instinct when asked was to show the AAU games separately under a neutral label
("Club" / "AAU") rather than the organisation's name. That answer was given on the
mistaken understanding that the games were already being counted. They weren't. So the
decision is genuinely open:

- leave the page as senior-season-only, or
- add a "Summer 2026" section fed from `SHEET_DATA`, labelled neutrally.

Nothing needs undoing either way — the sheet data is still fetched and simply unused for
stats.

### 2. Game-day reminder (app-side, not this repo)

Agreed but not built. See the EnzoStats handoff — the reasoning is that the biggest risk
to the whole system is nobody starting the scorer, and a game that isn't scored produces
no events, no email and no season line, with no way to recover it afterwards.

---

## Recent history

- John Elgani's coach card removed entirely, and every mention of Sunshine Elite taken out
  of both pages — vitals bar, subhead, both `og:description` tags, the team-filter button
  and the code comments. Verified against the rendered DOM rather than the source.
- The team filter still exists and still works; it reveals itself only when the data holds
  more than one team, which it no longer does, so it stays hidden.
