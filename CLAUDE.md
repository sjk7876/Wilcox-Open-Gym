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

**Court stability** — staying teams are sorted by their previous court index before pairing, then all pairs (forced + regular) are slotted back to the court where each pair's staying team played last round. This keeps teams on the same court when they play consecutive rounds.

**Fewer than 9 teams** — `computeNextRound` hands off to `computeNextRoundFewerTeams`; the 9-team logic is deliberately left untouched. R1→R2: winners on the first `NUM_SITTING` courts face the assigned waiters, other winners face a loser from a different court, leftover losers sit. R3+: the `NUM_SITTING` teams with the longest play streak (then most games) sit (for 8 teams this caps everyone at 3 games in a row; for 7/6/5/4 teams there aren't enough empty slots for any cap); all pairings of the available teams are scored (rematch penalty, forced-match bonus, court-stability penalty) and the best wins. **Priority team** (`priorityTeam`, fewer than 9 teams only): shielded from sitting until its streak hits `PRIORITY_MAX_RUN` (3), so it gets the most games the cap allows (5 in 6 rounds for 8 teams). It must play R1, and should be on Court 1 or 2 — a Court 3 loser can't be re-paired in R2 and sits. With 4–6 teams rematches are unavoidable over 7 rounds; the scorer just minimizes them.

**`pairTeams(stayGroup, comeOnGroup, played)`** shuffles all permutations of the incoming group and finds the first pairing against the stay group that avoids repeat matchups. Falls back to index-matched pairing if no valid pairing exists.

**`pairFour(pool, played)`** tries all 3 pairings of 4 teams to find one with no rematches.

**Forced matchups** — user can specify matchups that must occur somewhere in the schedule. During `computeNextRound`, forced pairs are extracted first (skipped if both teams are stayers locked to different courts), then the remaining teams are paired normally. After all rounds cascade, any unsatisfied forced match is marked `-1` (failed) and the schedule is cleared with an error naming the offending constraint.

**`rerollRound(roundIdx)`** discards that round and everything after it, then re-runs `computeNextRound` with a fresh shuffle to generate different pairings.

**`onWinnerSelected`** truncates `schedule` to `roundIdx + 1`, rebuilds `playedMatchups` from scratch up to that point, then re-cascades all remaining rounds. This ensures changing a Round 1 result cleanly invalidates any downstream rounds.
