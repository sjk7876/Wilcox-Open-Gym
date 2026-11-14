# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

No build step. Open `index.html` directly in a browser, or serve it with any static file server:

```bash
python3 -m http.server
```

## Architecture

The entire application lives in a single `index.html` file with inline CSS and JavaScript. There are no external dependencies beyond Google Fonts. Keeping it single-file preserves drag-and-drop simplicity — splitting would require a local server for dev (browsers block local JS module imports via CORS).

### State

All mutable state is held in module-level JS variables:

- `NUM_TEAMS` / `NUM_COURTS` / `NUM_SITTING` — derived by `applyTeamCount(n)` (4–9 teams; courts = min(3, floor(n/2)); the rest sit). User picks the count in Step 1
- `teamColors` — array of 9 color names indexed by team number (0-based); only the first `NUM_TEAMS` are used
- `r1matchups` — the 3 court matchups for Round 1, user-configured before generation
- `r1waiting` — derived: teams not in `r1matchups`, auto-synced via `syncWaiting()`
- `r2benchOrder` — array of `NUM_SITTING` team numbers; `r2benchOrder[i]` is the waiter assigned to Court `i+1` in Round 2, user-configurable
- `numRounds` — constant `7`
- `schedule` — array of round objects `{ matches: [{a, b, winner}], sitting: [] }`
- `playedMatchups` — `Set` of `"min-max"` strings tracking every matchup already played, used to avoid rematches
- `forcedMatches` — array of `{ a, b }` objects for user-specified required matchups
- `forcedMatchSatisfiedRound` — parallel array to `forcedMatches`; `null` = pending, 1-based round number = satisfied, `-1` = failed

### Persistence

All state is saved to `localStorage` under the key `wilcox_state_v1` on every mutation. On page load, `loadState()` restores all variables before the UI is built, so a refresh resumes exactly where the user left off. Clicking **Clear** resets and re-saves. If the state schema changes incompatibly, bump the version suffix in `SAVE_KEY`.

### Schedule generation logic

**Round 1** is configured manually by the user (Step 2). After clicking "Generate", only Round 1 is rendered — subsequent rounds are computed lazily.

**Round 1 → Round 2 transition** (the only round where actual wins matter):
- User pre-assigns which waiter comes to each court via the "Round 2 Bench Assignment" section (Step 4)
- After R1 winners are entered: `winner[i]` plays `r2benchOrder[i]` on Court `i+1`
- Losers sit out Round 2

**Round 2 → Round 3+** (fully mechanical — winner buttons are hidden):
- Teams who played two rounds in a row sit next
- Teams who played one round (came off the bench) stay
- Teams who sat come on

**Pairing and courts (R2+)** — `bestCourtAssignment` scores every pairing of the available teams × every court order: rematches (1000) > forced-match bonus (−5000) > court waits (100) > staying teams changing courts (10), plus a random tiebreak. Used by both the 9-team and fewer-teams paths. The 9-team path withholds the forced bonus when both teams are stayers (they're locked to different courts).

**Court waits** — `isCourtWait` / `courtWaitFor`: a match has to wait if no team in it played on that court last round and at least one of its teams just played on a different court (bench teams are waiting anyway). The schedule highlights these courts with a "⏳ Waits for X vs Y" note. Separately, `courtArrivals(ri, ci)` lists teams arriving on a court from a different court last round; each gets a "↪ X moves from Court N" note on the cell.

**`cascadeFrom(roundIdx)`** — regenerates every later round `CASCADE_TRIES` (40) times and keeps the schedule with the fewest unmet forced matches, then rematches, then court waits. Retries set `forcedDeferChance` so a forced matchup can be held back to a round where it won't cause a wait. Used by `onWinnerSelected` and `swapCourts`. A forced matchup landing in the last round costs `COST.lastRoundForced` (nights usually stop after 6 rounds), ranked just below an unmet one.

**Rotation groups (9 teams)** — `sameRotation(a, b)`: teams rotate in three lockstep groups (R1 bench, R1 winners, R1 losers). A forced matchup inside one group can't happen after R2 without someone changing courts, so the forced list flags it ("maybe" while R1 results are pending for two court teams).

**Fewer than 9 teams** — `computeNextRound` hands off to `computeNextRoundFewerTeams`; the 9-team path keeps its own mechanical sit rotation (only pairing is shared). R1→R2: winners on the first `NUM_SITTING` courts face the assigned waiters, other winners face a loser from a different court, leftover losers sit. R3+: the `NUM_SITTING` teams with the longest play streak (then most games) sit (for 8 teams this caps everyone at 3 games in a row; for 7/6/5/4 teams there aren't enough empty slots for any cap); pairing goes through `bestCourtAssignment`. **Priority team** (`priorityTeam`, fewer than 9 teams only): shielded from sitting until its streak hits `PRIORITY_MAX_RUN` (3), so it gets the most games the cap allows (5 in 6 rounds for 8 teams). It must play R1, and should be on Court 1 or 2 — a Court 3 loser can't be re-paired in R2 and sits. With 4–6 teams rematches are unavoidable over 7 rounds; the scorer just minimizes them.

**Forced matchups** — user can specify matchups that must occur somewhere in the schedule (in practice: the host's team vs 1–4 chosen opponents). They're scored as a large bonus in `bestCourtAssignment`. After all rounds cascade, any unsatisfied forced match is marked `-1` (failed) and the schedule is cleared with an error naming the offending constraint.

**`rerollRound(roundIdx)`** discards that round and everything after it, then re-runs `computeNextRound` with a fresh shuffle to generate different pairings.

**`onWinnerSelected`** truncates `schedule` to `roundIdx + 1`, rebuilds `playedMatchups` from scratch up to that point, then re-cascades all remaining rounds via `cascadeFrom`. This ensures changing a Round 1 result cleanly invalidates any downstream rounds.
