# League League Draft — Fantasy Football Draft Order Tracker

Single-file app (`index.html`) for determining the 2025 fantasy football draft order across three live sporting events. All state lives in `localStorage`. No build step, no dependencies. Deployed via GitHub Pages.

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

Points from all three events sum to produce the overall draft order. Tiebreaker rules TBD.

---

## Fantasy Teams

**8 owners.** Names and IDs to be added once the team list is finalized (will be dropped into the project folder). Format: hardcoded `defaultName` per team ID, not user-editable.

---

## Events

### Event 1 — The Open Championship (British Open)
- **Dates:** Starts July 17, 2025 (Royal Portrush)
- **How it works:** Each owner is assigned specific golfers in the field. Owner standing is determined by their assigned golfers' aggregate performance (exact metric TBD — e.g. cumulative score to par, or best-of-assigned finishing position).
- **Assignments:** TBD — will be provided.
- **Live data source:** TBD — likely ESPN golf API. Needs individual golfer scores and leaderboard positions.
- **Duration:** ~4 days (72-hole stroke play). Completes ~July 20.

### Event 2 — MLB Runs Scored
- **Dates / window:** TBD — a defined stretch of the MLB regular season.
- **How it works:** Each owner is assigned one (or more) MLB team(s). The owner whose assigned team(s) score the most total runs over the defined window earns the most event points.
- **Assignments:** TBD.
- **Live data source:** TBD — ESPN MLB API scoreboard.
- **Notes:** Window dates need to be set; not necessarily concurrent with the other events.

### Event 3 — Little League World Series (LLWS)
- **Dates:** Typically mid-to-late August (Williamsport, PA).
- **How it works:** Each owner is assigned one or more LLWS teams. Final tournament placement determines points.
- **Assignments:** TBD.
- **Live data source:** TBD — ESPN Little League API or manual entry (ESPN LLWS coverage is lighter; may need manual fallback).
- **Notes:** Latest of the three events chronologically.

---

## Tab Structure

Four tabs total:

### Tab 1 — Overall Standings (main/default tab)
- Ranked table of all 8 owners by total points across all three events.
- Columns: Rank · Owner · Open Points · MLB Points · LLWS Points · Total Points · Draft Pick #.
- Derived draft pick: 1st overall pick → highest total points. 8th pick → lowest total points.
- In-progress events show live/provisional points. Not-yet-started events show `—`.

### Tab 2 — The Open
- Live leaderboard of each owner's assigned golfer(s) and their current tournament standing.
- Shows provisional event-points each owner would earn at the current moment.
- Ranked by current owner standing in this event.

### Tab 3 — MLB Runs
- Shows each owner's assigned MLB team(s) and total runs scored over the defined window.
- Ranked by runs scored; translates to provisional event-points.

### Tab 4 — Little League World Series
- Shows each owner's assigned LLWS team(s) and tournament progress/finish.
- Ranked by finish; translates to event-points.

---

## File Structure

```
index.html        — entire app: HTML, CSS (<style>), JS (<script>)
CLAUDE.md         — this file
league.jpeg       — design reference image ("The League" poster)
```

Assignment reference files (to be added):
- Owner/team list
- Per-event competitor assignments

---

## Technical Approach

Mirrors the world-cup-draft architecture (`/Users/stephenzott/Documents/world-cup-draft/index.html`):

- **Single `index.html`** — no build toolchain, no npm, no frameworks.
- **`localStorage`** for persisting API-fetched and manually-entered state between page loads.
- **Live API sync** on a polling interval (e.g. 90 seconds) during active events, with a visible sync status pill.
- **Manual entry fallback** (especially for LLWS) exposed via `?admin` query param; public view is read-only.

### Likely API sources (to be confirmed)
- The Open: `site.api.espn.com/apis/site/v2/sports/golf/` leaderboard endpoint
- MLB: `site.api.espn.com/apis/site/v2/sports/baseball/mlb/scoreboard`
- LLWS: `site.api.espn.com/apis/site/v2/sports/baseball/little-league-world-series/scoreboard` (coverage TBD)

---

## Visual Design

Inspired by "The League" TV show promo poster (`league.jpeg`):

- **Background:** Near-black (`#0a0a0f` or similar deep dark)
- **Primary accent:** Bold red (`#cc0000` or `#e31c23`) — used for borders, active tab underlines, rank highlights
- **Secondary accent:** Gold/bronze (`#c9a84c` or similar warm gold) — used for #1 rank, champion/winner highlights, trophy imagery
- **Title text:** White, bold, uppercase with a red vertical bar motif echoing the show's logo
- **Surface/card backgrounds:** Very dark gray (`#1a1a22` or similar) to layer above the page background
- **Muted text:** Mid-gray (`#888888`)
- **Overall feel:** Dramatic, cinematic, stadium-atmosphere — fog/glow effects optional via CSS box-shadow or gradient overlays

Tab nav, table structure, sync pill, and responsive breakpoints follow the same patterns as world-cup-draft but reskinned to this dark/red/gold palette.

---

## Open Questions / TBD

- [ ] 8 owner names and IDs
- [ ] Golfer assignments per owner for The Open
- [ ] MLB team assignments per owner + exact date window for the run-scoring metric
- [ ] LLWS team assignments per owner
- [ ] Tiebreaker rule when two owners have equal total points
- [ ] Whether LLWS data is available via ESPN API or requires manual entry
- [ ] Exact metric for The Open event (best single golfer per owner? aggregate of all assigned golfers?)
- [ ] Whether any event awards partial/provisional points mid-event vs. only on completion
