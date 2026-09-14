# FIBA Tournament Dashboard

Self-updating dashboard for FIBA competitions, built for Canada Basketball.
Pure Python — no R.

**Live hub:** https://jordanngo205.github.io/Olympic-Pre-Qualifying-Tournament-Tracker/

One GitHub Pages site now serves multiple tournaments. The site root is a
static hub page (`docs/index.html`, hand-authored — the scraper never
touches it) with a card linking out to each tournament's own dashboard,
which lives in its own `docs/<slug>/` subfolder:

| Tournament | Dashboard |
|---|---|
| FIBA Women's Basketball World Cup 2026 Qualifying Tournament — Istanbul (Istanbul, Türkiye — 11–17 Mar 2026) | https://jordanngo205.github.io/Olympic-Pre-Qualifying-Tournament-Tracker/wc-qualifying-istanbul-2026/ |
| FIBA U18 Women's AmeriCup 2026 (Irapuato, Mexico — 9–15 Jun 2026) | https://jordanngo205.github.io/Olympic-Pre-Qualifying-Tournament-Tracker/u18-americup-2026/ |
| FIBA U17 Women's Basketball World Cup 2026 (Brno, Czechia — 11–19 Jul 2026) | https://jordanngo205.github.io/Olympic-Pre-Qualifying-Tournament-Tracker/u17-world-cup-2026/ |
| FIBA Women's Olympic Pre-Qualifying Tournament 2026 (Guadalajara, Mexico — 17–23 Aug 2026) | https://jordanngo205.github.io/Olympic-Pre-Qualifying-Tournament-Tracker/olympic-pre-qualifying-2026/ |

Adding another tournament later just means: run `fiba_scrape.py` with a new
`--publish-slug`, then add one more card to `docs/index.html`.

## How it works

No URLs are entered by hand anywhere. The scraper matches a tournament by name
against FIBA's own event index, then walks down to the games:

```
--event "olympic pre-qualifying guadalajara"
   → /en/events              event index, matched by name → slug
   → /en/events/<slug>/games full schedule, final games only
   → /games/<id>-<A>-<B>     box scores + play-by-play
   → CSVs → dashboard_template.html → docs/<slug>/index.html
```

Every page's data comes out of the Next.js hydration payload that
fiba.basketball ships inside its HTML.

For a **live** tournament, a GitHub Action re-runs this every 5 minutes,
scrapes any game that has gone final since the last run, rebuilds the
dashboard, and pushes. GitHub Pages serves the result, so the public link
updates itself. A **finished** tournament instead gets a `workflow_dispatch`
(manual-trigger) workflow — there is nothing left to poll for, so a 5-minute
cron would just be wasted Actions minutes forever.

`--publish-slug SLUG` controls where a dashboard lands: `docs/<SLUG>/index.html`
instead of `docs/index.html`. This is what lets several tournaments share one
Pages site — the Olympic Pre-Qualifying dashboard (the original, no slug)
still publishes to the site root.

## Usage

```bash
pip install -r requirements.txt

python3 fiba_scrape.py --list women olympic        # browse events
python3 fiba_scrape.py --event olympic pre-qualifying guadalajara \
                       --qualify-spots 2 \
                       --name "Olympic Pre-Qualifying 2026"

# a second tournament, published alongside the first instead of overwriting it
python3 fiba_scrape.py --event u17 world cup 2026 \
                       --qualify-spots 4 \
                       --name "U17 World Cup 2026" \
                       --publish-slug u17-world-cup-2026

# stay running until every game is final
python3 fiba_scrape.py --event guadalajara --watch 15
```

Re-runs are incremental — games already in the CSVs are skipped.

Only games FIBA has marked final are scraped (`gameStatisticStatusCode == VALID`
and not `isLive`), because a game in progress has no complete box score.

## Outputs

Written to `<Competition>/data/`:

| File | Contents |
|---|---|
| `game details` | one row per game: teams, scores, round, venue |
| `player box scores` | per-player traditional box |
| `team box scores` | per-team box, with opponent columns joined on |
| `team adv box scores` | possessions, ORTG, DRTG, eFG%, TO/poss, DREB rate, AST/FG% |
| `pbp` | play-by-play with shot zones, distances, and seconds elapsed |
| `standings` | W/L and point differential per team per game |
| `player enriched` | box + PBP-derived stats + offensive/defensive net points |
| `daily awards` | 20 per-day superlatives |
| `participant log` | rosters |

## Dashboard template

`dashboard_template.html` holds all the layout and styling. The scraper splices
data between the `// %%DATA_START%%` and `// %%DATA_END%%` markers, emitting
`GAME_DETAILS`, `ADV`, `PLAYER_DATA`, `QUALIFIERS`, `QUALIFY_SPOTS`,
`GENERATED_AT` and `FLAG_MAP`.

Because those markers are re-emitted into the output, **any generated dashboard
is itself a valid template** — copy one over `dashboard_template.html` to reuse
its design.

Add a country to `FLAG_MAP` in `fiba_scrape.py` when a new team appears.

## Legacy

`legacy/` holds the original archived copy of the U17 World Cup data — the
reference tournament used to validate this scraper against the Basketball
Canada R pipeline it replaces (team advanced box, enriched player table and
standings matched exactly, with one deliberate difference: `FTP` is computed
as `FTM/FTA` rather than the R script's `FTM/FGA`). Those R scripts have been
removed; the pipeline is entirely Python. They remain in git history at
commit `25c954b`.

That validated data is also the full FIBA U17 Women's Basketball World Cup
2026 dataset (Brno, Czechia — all 56 games through the Final), so it was
copied into `U17 World Cup 2026/data/` to seed the live U17 dashboard above
instead of re-scraping it.
