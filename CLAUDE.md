# League League Draft — Fantasy Football Draft Order Tracker

Single-file app (`index.html`) for determining the 2026 fantasy football draft order across three live sporting events. All state lives in `localStorage`. No build step, no dependencies. Deployed via GitHub Pages at **https://stephenzott.github.io/league_league_draft/**.

## Concept

Eight fantasy team owners are each assigned competitors in three real-world sporting events. Performance in each event earns points on an 8-down-to-1 scale. The owner with the most total points across all three events earns the **first pick** in the fantasy football draft; the owner with the fewest points earns the **last pick**.

**Draft order logic:** best cumulative score → picks first. Worst cumulative score → picks last.

---

## Scoring System

Each event awards points based on finish/rank among the 8 owners:

| Finish | Points |
|--------|--------|
| 1st    | 8      |
| 2nd    | 7      |
| 3rd    | 6      |
| 4th    | 5      |
| 5th    | 4      |
| 6th    | 3      |
| 7th    | 2      |
| 8th    | 1      |

Points from all three events sum to produce the overall draft order.

**Overall tiebreaker:** TBD.

---

## Owners

Eight owners, hardcoded by ID:

| ID | Name    |
|----|---------|
| 1  | Matt    |
| 2  | Keanan  |
| 3  | Zach    |
| 4  | Patrick |
| 5  | Derek   |
| 6  | Cody    |
| 7  | Josh    |
| 8  | Stephen |

---

## Events

### Event 1 — The Open Championship ✅ BUILT

- **Dates:** July 16–20, 2026
- **Live data source:** `https://site.api.espn.com/apis/site/v2/sports/golf/leaderboard?league=pga` — ESPN identifies the event by name (`/the open/i`).
- **Scoring metric:** Sum of finishing positions for each owner's 4 golfers. Lower sum = better (like golf scoring). Owner with lowest sum gets 8 draft points; highest sum gets 1.
- **Tiebreaker:** Compare all individual round stroke counts across all 4 golfers, sorted ascending. Owner with the lower best single round wins; if still tied, compare next-best round, etc.

#### Golfer assignments (randomly drawn, fixed)

Groups were formed via snake draft of the top 32 pre-tournament odds (picks 1,16,17,32 as one group; 2,15,18,31 as the next; etc.), then randomly assigned to owners.

| Owner   | Golfers |
|---------|---------|
| Matt    | Scottie Scheffler, Si Woo Kim, Shane Lowry, Harris English |
| Keanan  | Xander Schauffele, Ludvig Åberg, Patrick Reed, Alex Fitzpatrick |
| Zach    | Viktor Hovland, Chris Gotterup, Joaquin Niemann, Patrick Cantlay |
| Patrick | Jon Rahm, Tyrrell Hatton, Tom Kim, Bryson DeChambeau |
| Derek   | Matt Fitzpatrick, Wyndham Clark, Sam Burns, JJ Spaun |
| Cody    | Robert MacIntyre, Justin Rose, Min Woo Lee, Aaron Rai |
| Josh    | Tommy Fleetwood, Collin Morikawa, Justin Thomas, Brooks Koepka |
| Stephen | Rory McIlroy, Cameron Young, Russell Henley, Hideki Matsuyama |

#### MC / WD / DQ handling
- Top 70 plus ties make the cut. MC players are ranked by their score-to-par starting at position (cutCount + 1); ties share the same position number.
- WD **before** the cut (period < 3): treated identically to MC.
- WD **after** making the cut (period ≥ 3): placed last among MIF players (position = cutCount).
- DQ: treated as MC.

#### Name normalization
ESPN uses accents and dots our hardcoded names don't (`Joaquín Niemann`, `J.J. Spaun`). `normName()` strips NFD accent marks and periods before comparing.

---

### Event 2 — MLB ✅ BUILT

- **Dates / window:** August 1–14, 2026.
- **How it works:** Each owner is assigned one MLB team. Best win-loss record over the window earns 8 draft points; worst record earns 1.
- **Tiebreaker:** Total runs scored by the team over the window.
- **Sorting:** Win percentage (W / (W+L)) as primary sort; handles unequal game counts from rainouts/doubleheaders. Runs scored breaks ties.
- **Live data source:** `https://site.api.espn.com/apis/site/v2/sports/baseball/mlb/scoreboard?dates=20260801-20260814&limit=200`

#### MLB team assignments (randomly drawn, fixed)

| Owner   | Team                    | ESPN Abbr |
|---------|-------------------------|-----------|
| Matt    | San Francisco Giants    | SF        |
| Keanan  | Philadelphia Phillies   | PHI       |
| Zach    | Boston Red Sox          | BOS       |
| Patrick | New York Yankees        | NYY       |
| Derek   | Tampa Bay Rays          | TB        |
| Cody    | Milwaukee Brewers       | MIL       |
| Josh    | Los Angeles Dodgers     | LAD       |
| Stephen | Atlanta Braves          | ATL       |

---

### Event 3 — Little League World Series (LLWS) 🟡 SCAFFOLDED

- **Dates:** Opening Round Aug 19, 2026 → Championship Aug 30, 2026, in Williamsport, PA. (Confirmed directly off ESPN's live schedule — Regional Championships, the state/country qualifiers that decide the 20-team field, run Aug 10–15 and are excluded from scoring.)
- **How it works:** Each owner is assigned exactly **one** LLWS team (`OWNERS[].llwsTeam`, singular — like `mlbTeam`, not an array).
- **Scoring metric — hybrid of MLB's approach and bracket-aware tiebreaking:**
  1. **Win percentage** (primary sort, same as MLB) — teams that advance further in a double-elim bracket generally accumulate more wins, so win% still tracks performance despite unequal game counts.
  2. **Head-to-head** — if two tied teams played each other during the window, whoever won more of those meetings ranks higher. LLWS splits into separate US/International brackets that only meet in the World Championship, so most ties will have no head-to-head game — falls through to the next tiebreaker when that happens.
  3. **Total wins** — rewards a team that advanced via the elimination (losers') bracket over one that lost early at a higher win%.
  4. **Runs scored** — final tiebreaker, same as MLB.
- **Elimination tracking:** Each team's `eliminated` status (true/false) is tracked explicitly, not inferred from win-loss count alone — a loss doesn't always end a team's tournament (see below).
- **Assignments:** TBD.
- **Live data source:** `site.api.espn.com/apis/site/v2/sports/baseball/llb/scoreboard?dates=20260819-20260830&limit=200` — confirmed working, live-tested against the real 2026 tournament schedule. See `llws-espn-api-reference.md` for general API notes.
- **Bracket round classification (`classifyLLWSHeadline()` in index.html):** ESPN has no structured "round" field, but each game's `notes[].headline` names the round in plain text — confirmed live against the real schedule:
  | Headline contains | Category | Losing eliminates you? |
  |---|---|---|
  | "Opening Round" | `winners` | No — demoted to elimination bracket |
  | "Double Elimination" | `winners` | No — demoted to elimination bracket |
  | "Elimination Game" | `elimination` | **Yes** — this is your 2nd loss |
  | "United States Championship" / "International Championship" | `side-championship` | **Yes** — single-elim, no rematch even on a 1st loss |
  | "Consolation Game" | `consolation` | Already eliminated (3rd-place game between semifinal losers) |
  | "Championship" (World Final) | `world-championship` | **Yes** — losing means runner-up |
  | "Regional Championship" | `regional` | N/A — pre-LLWS qualifier, excluded from scoring entirely |
- **Scaffold state (current):** Tab is enabled and fully wired — `fetchLLWSGames()` pulls the Aug 19–30 window every sync, `computeLLWSScores()` runs the real hybrid scoring + elimination logic, and `renderLLWS()` renders one card per owner (record, win%, runs, alive/eliminated badge, current-game line) mirroring `renderMLB()`. The only missing piece is `OWNERS[].llwsTeam` — it's `null` for every owner, so every card currently shows "Team not yet assigned" and all pts show `—`. Filling in the 8 team assignments (same shape as `mlbTeam`: `{ name, abbr }`, using the abbreviation ESPN returns in its `team.abbreviation` field) is the only step left to make this fully live.

---

## Tab Structure

### Tab 1 — Overall Standings ✅ BUILT
- Ranked table of all 8 owners by total points across all three events.
- Columns: Rank · Owner · The Open · MLB · LLWS · Total Pts · Draft Pick.
- MLB and LLWS show `—` until those events are built.
- Draft pick = rank (most total points picks 1st).

### Tab 2 — The Open ✅ BUILT
- One card per owner, sorted by current position sum (lower = better).
- Each card: rank badge · owner name · sum of positions · draft pts badge.
- 2×2 grid of golfer tiles, each showing position badge (color-coded by tier), golfer name, score-to-par, and thru info.
- Banner describes tournament state (pre / live / final).

### Tab 3 — MLB ✅ BUILT
- One card per owner, sorted by win percentage (best record first).
- Each card: rank badge · owner name · team name + abbr · draft pts badge.
- Stats row: W–L record · win % · runs scored (tiebreaker).
- Banner describes window state (pre / live / final / dev test mode).

### Tab 4 — LLWS 🟡 SCAFFOLDED
- One card per owner, sorted by the hybrid win% → head-to-head → wins → runs rule (see Event 3 above).
- Each card: rank badge · owner name · team name + abbr (or "Team not yet assigned") · alive/eliminated badge · draft pts badge.
- Stats row: W–L record · win % · runs scored. Game line shows the team's current/next/last game, same as MLB.
- Fully functional pipeline (fetch → score → render) — just waiting on `OWNERS[].llwsTeam` assignments to have real data to display.

---

## File Structure

```
index.html                      — entire app: HTML, CSS (<style>), JS (<script>)
CLAUDE.md                       — this file
TheLeagueintertitle.png         — wood-panel background image (referenced in CSS)
league.jpeg                     — "The League" poster (design reference, not used in app)
League League Teams.rtf         — original owner list
League League MLB TEAMS.rtf     — MLB team assignments
llws-espn-api-reference.md      — ESPN unofficial API notes for the LLWS
```

---

## Technical Approach

- **Single `index.html`** — no build toolchain, no npm, no frameworks.
- **`localStorage` key:** `leagueLeague2026_v1` — caches the last successful ESPN fetch so page reloads show data instantly.
- **Polling:** 90s normally; drops to 20s when at least one golfer is actively playing holes (`hasLive = true`). Pause/resume button in the header.
- **Sync pill:** shows `Syncing` / `Live` / `Synced HH:MM` / `Error` states.

---

## Visual Design

- **Background:** `TheLeagueintertitle.png` — wood-panel from the show's intertitle card, `background-size: cover; background-attachment: fixed`.
- **Cards/tables:** Dark warm-brown (`rgba(22,16,10,0.95)`) so they're readable against the wood but tonally matched.
- **Header/tabs:** Slightly transparent dark (`rgba(14,9,4,0.92)`) with `backdrop-filter: blur(2px)`.
- **Accent:** Red `#cc0000` — tab underlines, banner left border, draft pts badges.
- **Gold:** `#c9a84c` — #1 rank, points values, section labels.
- **Text:** Warm cream `#f5f0e8`; muted warm gray-brown `#9a8870`.
- **Title motif:** Red vertical bar left of "LEAGUE LEAGUE DRAFT" (mirrors the show's logo).

---

## Open Questions / TBD

- [ ] LLWS team assignments per owner (one team each — see Event 3 above)
- [ ] Overall tiebreaker rule when two owners have equal total points across all three events
- [x] Whether LLWS data is available via ESPN API — confirmed yes, `llb` scoreboard endpoint is live and working (see Event 3 above)
- [x] Exact LLWS scoring rule — resolved: win% → head-to-head → wins → runs scored, with explicit elimination tracking (see Event 3 above)
