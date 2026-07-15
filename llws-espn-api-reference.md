# Little League World Series — ESPN Unofficial API Reference

Compiled July 15, 2026, for use building a live scores / bracket tracker for the 2026 LLWS
(tournament runs mid-to-late August; regional qualifying games are already showing on
ESPN as of this writing).

## TL;DR

There is no official, dedicated "Little League World Series API." What we're actually
using is ESPN's **unofficial, undocumented** site API — the same JSON backend that powers
espn.com's scoreboards for every sport. ESPN happens to carry Little League Baseball and
Softball as their own "leagues" within the `baseball` sport, which is why this works at
all. No API key or auth is required.

**Confirmed league slug: `llb`** (Little League Baseball World Series)
Softball equivalent, unverified but documented the same way: `lls`

Verified independently across two separate community-maintained docs
([pseudo-r/Public-ESPN-API](https://github.com/pseudo-r/Public-ESPN-API/blob/main/docs/sports/baseball.md),
a GitHub gist of NFL/MLB endpoints), and confirmed live by fetching
`https://www.espn.com/little-league-world-series/scoreboard` on 2026-07-15, which was
already populated with regional championship games (e.g. "Southeast Regional
Championship," "Southwest Regional Championship" on Aug 11, 2026).

---

## Caveats — read before building on this

- **This is not an official API.** ESPN does not publish or support it. It can change or
  disappear without notice. There's no SLA, no versioning guarantee, nothing.
- **No auth needed**, which is both the appeal and part of why it could get locked down
  later — treat it as "works until it doesn't."
- **No dedicated "bracket" endpoint.** The LLWS single/double-elimination bracket
  structure isn't returned as structured bracket data — you'll need to infer it from
  each event's `name`/`shortName`/`notes` fields (e.g. "US Bracket," "International
  Bracket," "Championship"). See the JSON shape notes below.
- **Historical depth is not guaranteed.** This is a live/current-season scoreboard API.
  For past years (2025 and earlier), Wikipedia's year-by-year LLWS results pages are a
  more stable source than trying to query old dates through this endpoint.
- Standard advice for any unofficial API: cache responses, don't hammer it, and build in
  a fallback (e.g. graceful "data unavailable" state) in case ESPN changes something
  mid-tournament.

---

## Core Endpoints

Base pattern:
```
https://site.api.espn.com/apis/site/v2/sports/baseball/llb/{resource}
```

| Resource | Purpose |
|---|---|
| `scoreboard` | Today's games — scores, status, teams |
| `scoreboard?dates=YYYYMMDD` | Games for a specific date |
| `scoreboard?dates=YYYYMMDD-YYYYMMDD` | Games across a date range (confirmed working pattern on other ESPN leagues, e.g. World Cup: `?dates=20260611-20260719`) |
| `summary?event={eventId}` | Full box score / game detail for one game |
| `teams` | List of teams (populated once tournament teams are seeded) |
| `teams/{id}` | Single team detail |
| `teams/{id}/schedule` | A specific team's schedule |
| `news` | LLWS-related news items |

### Example calls

```bash
# Today's Little League Baseball games
curl "https://site.api.espn.com/apis/site/v2/sports/baseball/llb/scoreboard"

# Games on a specific day (e.g. LLWS opening day 2026 — confirm actual date closer to time)
curl "https://site.api.espn.com/apis/site/v2/sports/baseball/llb/scoreboard?dates=20260813"

# Whole tournament window in one call
curl "https://site.api.espn.com/apis/site/v2/sports/baseball/llb/scoreboard?dates=20260813-20260823&limit=100"

# Full box score for a specific game (event ID comes from a scoreboard response)
curl "https://site.api.espn.com/apis/site/v2/sports/baseball/llb/summary?event=EVENT_ID_HERE"
```

### Useful query params (seen across ESPN's scoreboard endpoints generally)

| Param | Notes |
|---|---|
| `dates` | Single date (`YYYYMMDD`) or range (`YYYYMMDD-YYYYMMDD`) |
| `limit` | Cap on number of events returned — bump this up (e.g. `limit=100`) if a date range seems to be cutting off games |
| `groups` | Used on some ESPN leagues (college sports especially) to pull specific divisions/brackets. Unconfirmed whether LLWS needs this since the field only has ~20 teams total — start without it, add only if games seem to be missing |

---

## JSON Response Shape (scoreboard)

This is the general shape ESPN uses across all its sport scoreboards — baseball, in this
case. Field names are consistent with MLB/college baseball, which is a good sign for
stability since it's shared plumbing rather than something LLWS-specific that could be
deprioritized.

```
{
  "leagues": [ ... ],          # league metadata, mostly boilerplate
  "season": { "year": 2026, "type": ... },
  "day": { "date": "2026-08-13" },
  "events": [
    {
      "id": "401234567",       # <-- this is the eventId used in the `summary` endpoint
      "date": "2026-08-13T17:00Z",
      "name": "Team A vs Team B",
      "shortName": "TMA @ TMB",
      "competitions": [
        {
          "id": "401234567",
          "date": "...",
          "notes": [ { "headline": "..." } ],   # sometimes carries round/bracket info
          "status": {
            "type": {
              "state": "pre" | "in" | "post",   # game phase
              "completed": true/false,
              "description": "Final" | "In Progress" | "Scheduled"
            }
          },
          "competitors": [
            {
              "id": "...",
              "homeAway": "home" | "away",
              "team": {
                "id": "...",
                "displayName": "...",
                "abbreviation": "...",
                "logo": "https://..."
              },
              "score": "4",
              "linescores": [ { "value": 1 }, { "value": 0 }, ... ]  # per-inning runs
            }
          ]
        }
      ]
    }
  ]
}
```

### Parsing notes

- `events[]` is the top-level list of games — iterate this for a scoreboard/bracket view.
- Each event nests exactly one `competitions[0]` in practice (the array wrapper is ESPN's
  generic schema, not something LLWS-specific).
- `competitors[]` always has exactly 2 entries (home/away) for baseball. Use `homeAway`
  to tell which is which, not array order.
- `status.type.state` is the reliable field for "is this game live/upcoming/done" — don't
  rely on `completed` alone; check `state` first.
- There is no `round` or `bracket` field. If you want to label games (e.g. "US
  Championship" vs "International Championship" vs "World Series Final"), you'll need to
  parse it out of `notes[].headline` or `name`/`shortName`, and it may take a look at a
  live response mid-tournament to see exactly how LLWS phrases these once games start.

---

## Tournament Structure (known in advance — hardcode this)

Since 2022, the LLWS has run 20 teams total, split into two completely separate
**modified double-elimination** brackets: **10 US teams** and **10 International teams**.
The two sides never play each other until the very end. This structure has been stable
since 2011 (aside from expanding from 8 to 10 teams per side in 2022), so it's safe to
bake the shape of the tournament into the tracker rather than trying to infer it from
the API.

**Within each 10-team side (US and International, independently):**

1. Teams start in the **Winners' bracket**. A loss there doesn't eliminate a team — it
   drops them into the **Elimination (losers') bracket**.
2. A second loss (i.e., a loss while already in the elimination bracket) eliminates a
   team from the tournament entirely.
3. This plays out until each side is down to exactly two teams:
   - The **Winners' bracket finalist** — still undefeated (0 losses)
   - The **Elimination bracket finalist** — has exactly 1 loss, and fought back through
     the loser's side to get here
4. Those two meet in the **US Championship** game (or **International Championship** on
   the other side). This game is **single-elimination** — importantly, even if the
   Elimination bracket team wins, there's no "if necessary" rematch like you'd see in the
   College World Series. One game, winner takes the side.
5. There's also typically a **third-place consolation game** on each side between the
   two semifinal losers.

**Once both sides have a champion:**

6. The **US Champion** and **International Champion** meet in the single-elimination
   **World Series Championship** game — this is the final, and the only point at which
   the two sides of the bracket ever intersect.

### What this means for building the tracker

- You can hardcode the shape above (winners bracket → elimination bracket → side
  championship → world championship) as your data model, independent of the API.
- What you **can't** derive from formulas alone is the exact **initial pairings** —
  i.e., which of the 10 regional champs plays which in round 1, and exactly which
  elimination-bracket slot a round-1 loser drops into. Little League sets this
  explicitly each year (they draw/announce "opening-round matchups" roughly two months
  before the tournament) specifically to avoid early rematches and to seed things
  fairly. That draw is published on littleleague.org, not derivable — grab it fresh
  each year rather than assuming last year's shape repeats.
- Practical approach: hardcode the bracket *logic* (win → stay in winners bracket /
  advance; lose once → drop to elimination bracket; lose twice → eliminated; last two
  standing per side → single-elim side championship; two side champs → single-elim
  final). Then feed it the actual team names/pairings from the published opening-round
  matchups once they're announced, and update team positions in the bracket as
  `scoreboard` results come in — matching games by team name/date rather than expecting
  the API to tell you what round a game belongs to.

---

## Suggested approach when building the tracker (not prescriptive — just a starting point)

1. Hit `scoreboard` with no `dates` param during the tournament window to get "what's on
   today."
2. Cache the `id` of each event as it's discovered — that's your key for pulling
   `summary?event={id}` later for box scores.
3. Since there's no real bracket endpoint, consider maintaining your own small mapping of
   "expected games" (US bracket, International bracket, semis, final) based on the
   published LLWS schedule, and match incoming events to it by team names / date rather
   than trying to parse a bracket out of the API.
4. Poll politely — this is someone's reverse-engineered scoreboard backend, not a
   dedicated sports-data product built to handle high query volume.

---

## Sources

- https://github.com/pseudo-r/Public-ESPN-API/blob/main/docs/sports/baseball.md (documents `llb`/`lls` slugs explicitly)
- https://gist.github.com/nntrn/ee26cb2a0716de0947a0a4e9a157bc1c (independently lists `llb` in ESPN's baseball league slug list)
- https://gist.github.com/akeaswaran/b48b02f1c94f873c6655e7129910fc3b (general ESPN hidden API patterns — `dates` ranges, `groups`/`limit` behavior)
- https://www.espn.com/little-league-world-series/scoreboard (live-checked 2026-07-15 to confirm the league is active and populated ahead of the tournament)
