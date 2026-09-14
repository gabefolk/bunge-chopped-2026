# Bunge Chopped 2026 — Fantasy Dashboard

Live dashboard: https://claude.ai/code/artifact/5a8ee8d7-edb6-4c55-9107-1e3e4ee545e7

Sleeper league ID: `1398014426640044032`

## Files

- `dashboard.html` — the exact file published as the live artifact (bootstrap data baked in).
- `dashboard.template.html` — same page with `__LEAGUE_JSON__` / `__TEAMS_JSON__` / etc.
  placeholders instead of baked-in data, for regenerating `dashboard.html` from fresh data.
- `data/` — raw JSON snapshots pulled from the Sleeper API when this was last built
  (`league`, `teams`, `players`, `matchups_week_1`, `transactions_week_1`).

## How it works

The published artifact can't call the Sleeper API directly (artifacts are sandboxed
from arbitrary external network requests). Instead: Claude fetches from Sleeper,
writes the results into the artifact's own database, and the page reads from there
— with the `data/*.json` files here baked into the page as a fallback so it still
renders even if the database is unavailable.

## Refreshing

Not yet automated. To refresh manually, ask Claude to:
1. Re-pull `league`, `users`, `rosters`, `matchups/<week>`, `transactions/<week>` from
   the Sleeper API for league `1398014426640044032`.
2. Rebuild the slim JSON files in `data/` (see "Rebuilding dashboard.html" below).
3. Republish `dashboard.html` to the same artifact URL (pass `url:` so it updates
   in place instead of creating a new artifact).
4. Also write the same 5 JSON docs into the artifact's own database (`league/current`,
   `league/teams`, `league/players`, `league/matchups_week_N`, `league/transactions_week_N`)
   via the Artifact tool's `write_db` batch action — the page reads from there live,
   the baked-in copy in `dashboard.html` is just the fallback shown on first paint.

## Rebuilding dashboard.html from data/

`dashboard.template.html` has `__LEAGUE_JSON__` / `__TEAMS_JSON__` / `__PLAYERS_JSON__` /
`__MATCHUPS_JSON__` / `__TXNS_JSON__` placeholders. After editing the template or
refreshing `data/*.json`, regenerate `dashboard.html` with:

```python
import json
def esc(obj): return json.dumps(obj, ensure_ascii=True).replace('</', '<\\/')
tpl = open('dashboard.template.html').read()
for name, file in [('LEAGUE','league'),('TEAMS','teams'),('PLAYERS','players'),
                    ('MATCHUPS','matchups_week_1'),('TXNS','transactions_week_1')]:
    tpl = tpl.replace(f'__{name}_JSON__', esc(json.load(open(f'data/{file}.json'))))
open('dashboard.html','w').write(tpl)
```

Then validate before publishing: check `<div>`/`<section>`/`<script>` tag counts balance,
and run `node --check` on the extracted inline `<script>` block.

## Session log

**2026-09-12/13 — built and iterated on v1.**
- Pulled live league/roster/matchup/transaction/player data from the Sleeper API
  (no auth needed) for league `1398014426640044032` ("Bunge Chopped 2026", 18 teams, PPR).
- Built the dashboard as a single-file HTML artifact using the `db` capability
  (artifacts can't fetch external APIs directly — Claude fetches, writes to the
  artifact's db, the page reads from there).
- Sections, in order: Trash Talk Stats, League Insights, Bench Points Left on the
  Table, Live Scoreboard, Standings, Roster Breakdown.
- Stat tiles expand into popover-style detail panels (position: absolute, so they
  overlay content instead of pushing the page down — this was a real bug we fixed).
  "Trigger Finger" rows expand a second level to show the actual transactions.
- Added a "League Insights" section that mines the full dataset for non-obvious
  findings: Lineup Landmines (starters who are Out/Inactive/IR), Bye Week Landmines
  (rosters with 3+ players from one NFL team), Position Power Rankings.
- Redesigned visually (Fredoka/Nunito type, warm coral/lavender palette, rounder
  cards) then rolled back emojis and playful section renames per feedback — kept
  the visual redesign, reverted the copy/emoji.
- Known data quirk: Sleeper hasn't paired head-to-head matchups for this league yet
  (every roster has its own unique `matchup_id`), so "Live Scoreboard" ranks by
  points instead of showing real matchups. Worth checking league settings in Sleeper
  if that doesn't resolve once games start.

**2026-09-13 — live refresh during Week 1 Sunday games, automated refresh schedule.**
- Confirmed live scoring was in progress (points moved significantly between two
  manual pulls a few hours apart); refreshed `data/matchups_week_1.json` and
  `data/league.json` (`last_synced`), rebuilt `dashboard.html` from the template,
  validated tag balance + JSON + inline JS (`node --check`), and republished to
  the same artifact URL. Also wrote all 5 docs into the artifact's live db
  (`write_db` batch, pinned with `if_version`) so viewers get the live overlay
  without waiting for a fresh page load of the baked-in bootstrap.
- Re-confirmed matchups are still **not** paired (`matchups_paired: false`,
  18 distinct `matchup_id`s for 18 rosters) — still a leaderboard, not real
  head-to-head. This looks like a league-schedule setting the commissioner
  hasn't turned on, not something fixable from our side.
- **Set up the refresh schedule** as two cloud routines (via the `schedule` skill
  → `RemoteTrigger`), since cloud routines can't touch this local machine — they
  write straight into the artifact's live db instead of rebuilding local files:
  - `Bunge Chopped - Sunday day games refresh` (`trig_01DCR5teHpg2C8x6HkjVw3fp`) —
    hourly, `0 17-23 * * 0` UTC (Sun ~1pm–7pm ET).
  - `Bunge Chopped - SNF-MNF-TNF refresh` (`trig_012AwGQDa2GSi2f7kDHtHXd1`) —
    hourly, `0 0-4 * * 1,2,5` UTC (covers Sunday/Monday/Thursday night games,
    which land on the *next* UTC calendar day since kickoff is ~8:15–8:25pm ET).
  - Both routines only refresh `league/current`, `league/teams`,
    `league/matchups_week_N`, `league/transactions_week_N` in the artifact's db —
    they deliberately skip `league/players` (large, mostly-static; Sleeper asks
    that its bulk player endpoint not be hit too often) and never touch local
    files or republish the HTML — that stays a manual/periodic step (see
    "Refreshing" above).
  - **Caveat:** the UTC hours above assume EDT (UTC-4). Once DST ends (~Nov 1,
    2026) the windows will run an hour early relative to actual ET kickoffs.
    Worth nudging both cron expressions back an hour around then.
  - Routines are listed at https://claude.ai/code/routines.
- Asked about sharing the artifact with the league — staying private for now.

## TODO / next session

- [ ] Once more weeks of data exist, the "League Insights" and "Trash Talk Stats"
  sections should extend past week 1 (currently hardcoded to
  `league.current_week` for matchups/transactions doc lookups — this already reads
  the week dynamically, just needs multi-week history if we want season-long trends
  like luckiest/unluckiest record).
- [ ] Decide whether to share the artifact with the league (still private to
  Gabriel as of 2026-09-13 — share via the artifact's share menu when ready).
- [ ] Around Nov 1, 2026 (DST ends), shift both refresh routines' cron expressions
  back one hour (EST is UTC-5, not UTC-4).
- [ ] The two refresh routines intentionally never touch `data/*.json` or
  `dashboard.html` locally, and never republish the page — only a manual session
  does that. Consider whether a periodic "resync the local files + republish"
  step is worth its own routine, or stays manual.
