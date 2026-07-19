# Olivia Rodrigo Production Evolution — Data Engineering Project

This project builds a reproducible data pipeline that tracks how Olivia Rodrigo's production
quality and audio characteristics evolved across her three Dan Nigro-produced studio albums —
*SOUR* (2021), *GUTS* (2023), and *you seem pretty sad for a girl so in love* (2026) — and tests
whether that evolution correlates with audience engagement (Spotify popularity, Billboard chart
performance). It exists to support a Medium article combining data engineering with pop-culture
analysis, and it's built to be honest about a real, current constraint: Spotify killed public
access to its audio-feature data for every app created after November 2024, so this pipeline
treats "get the audio features" as the hard, partially-unsolved problem it actually is, rather
than assuming a clean API exists.

## The pipeline, step by step

**1. Data ingestion.** Five independent sources feed the pipeline, each wrapped behind its own
small class so any one of them can be swapped out later without touching the rest of the code:

- **Track identity** (names, track order, duration, bonus-track flags) comes from Wikipedia,
  hand-verified against each album's page, rather than Spotify's catalog — explained below.
- **Spotify popularity scores** are fetched live, at runtime, via the Spotify Web API's Client
  Credentials flow (app-only, no user login, since this is public catalog data). The same client
  is also used to test — live, against a real track — whether `/audio-features` still works.
- **ReccoBeats**, a free, no-API-key service discovered and live-tested mid-project, supplies the
  *full* Spotify-equivalent audio-feature set — but only for SOUR and GUTS; its catalog predates
  the 2026 album's existence, confirmed by testing real 2026-album track IDs and getting no match.
- **GetSongBPM** was meant to cover tempo/key/time signature for anything ReccoBeats missed. Live
  testing found its endpoint sits behind Cloudflare bot protection that blocks plain server-side
  requests regardless of API key validity, so it currently contributes nothing; the code detects
  this in one request and stops trying, rather than hammering a dead end.
- **Everything ReccoBeats and GetSongBPM don't cover** — currently all 14 tracks of the 2026
  album — is a manual-entry template, the guaranteed-to-work fallback once free options run out.
- **Chart performance** (Billboard Hot 100 peak positions for each album's singles) is compiled
  by hand from Billboard/Wikipedia coverage, the same standard a published article would cite to.

**2. Cleaning / ETL.** A second, independent stage re-reads the CSVs `/data/raw` just produced
(rather than reusing anything held in memory), so it works whether or not ingestion ran in the
same session — including after the manual columns get hand-filled later. It applies one
documented rule for deluxe/bonus tracks (kept and flagged, never dropped, standard-edition treated
as the default comparison set), forces every feature column to the right numeric type, and prints
a validation report: missing values, values outside their expected range, duplicate tracks, and
whether each album's track numbers actually run in sequence. The result is one tidy row per
track, saved to `/data/processed/tracks_clean.csv`.

**3. Exploratory analysis.** For each album: mean/median/std for every audio feature, a trend
chart across all three albums per feature, a closer look at three specific "production
complexity" proxies (loudness variability, energy vs. valence, acousticness vs.
instrumentalness), and a correlation matrix per album testing whether relationships between
features (like loudness and energy) held steady or shifted release to release.

**4. Engagement correlation.** Audio features are tested against two engagement signals — Spotify
popularity and Billboard Hot 100 peak position — using Pearson and Spearman correlation (not a
black-box model, since the goal is a narrative a reader can verify by eye) plus scatter plots with
trend lines. A separate check builds a simple "production polish" score and flags any tracks where
a rawer- or more-polished-sounding track over- or under-performed what its production score would
predict, printing the actual track names and numbers rather than a vague summary.

## Findings

**Real, computed from actual data (SOUR vs. GUTS, standard editions, n=11 and n=12):**

- **Energy rose ~40%**: mean 0.420 (SOUR) → 0.588 (GUTS).
- **Loudness rose ~2.6 dB**: mean −8.86 dB (SOUR) → −6.28 dB (GUTS) — GUTS is audibly louder.
- **Valence (musical positivity) rose ~36%**: mean 0.317 (SOUR) → 0.432 (GUTS).
- **Acousticness fell ~38%**: mean 0.575 (SOUR) → 0.353 (GUTS) — GUTS leans measurably less
  acoustic/more produced, consistent with its rockier, more electric-guitar-driven sound.
- **Tempo actually *fell***, counter to the "bigger, more energetic" story above: mean 141.5 BPM
  (SOUR) → 128.6 BPM (GUTS) — GUTS gets its intensity from loudness/energy, not from being faster.
- **The energy↔acousticness relationship tightened sharply**: correlation −0.65 (SOUR) → −0.94
  (GUTS). Energy and loudness, by contrast, stayed almost exactly as tightly coupled (0.90 → 0.92)
  — so GUTS didn't change *how* energy and loudness relate to each other, but energy and
  acousticness became much more rigidly opposed to one another.
- **Tentative, small-sample signal on chart performance**: among the 6 SOUR+GUTS lead singles with
  both audio-feature and chart data, higher energy correlated with a *worse* Billboard Hot 100
  peak (Pearson r = −0.844, p = 0.035). With n=6 this is fragile — a single outlier could
  overturn it — but it runs counter to a "bigger sound charts better" assumption and is worth
  flagging rather than ignoring.
- **Chart pattern by era**: *SOUR*'s three singles peaked #1 (Drivers License), #3 (Deja Vu), #1
  (Good 4 U) — non-monotonic. *GUTS* declined monotonically: #1 (Vampire) → #7 (Bad Idea Right?)
  → #11 (Get Him Back!). *you seem pretty sad for a girl so in love* broke that pattern and looked
  more like SOUR: #1 (Drop Dead) → #5 (The Cure) → #3 (Stupid Song).

**What isn't available yet, and why:** the 2026 album's 14 tracks have zero audio-feature
coverage (0/14), and Spotify popularity is 0/42 across the board. Not an oversight —
**no free, complete, programmatic source for 2026-album audio features exists** as of this
writing (Spotify's endpoint is dead for any new app; GetSongBPM is Cloudflare-blocked; ReccoBeats'
catalog necessarily predates this album; no Kaggle dataset could possibly contain it either), and
popularity is simply pending `SPOTIFY_CLIENT_ID`/`SECRET` being added to `.env`.

**To complete the picture** (the 2026 album's place in these trends, real popularity
correlations, and the counterintuitive polish-vs-popularity check, which needs both audio features
and popularity together): fill in the 2026 album's manual columns in `/data/raw/..._raw.csv` using
a per-track lookup tool (the `notes` column documents this), add Spotify credentials to `.env`,
and re-run Sections 2–4. Nothing above was estimated or invented to look more complete than the
data actually is — every number here came out of the notebook exactly as printed.

## Data limitations and caveats

- **SOUR/GUTS audio features come from ReccoBeats, not Spotify's own now-inaccessible pipeline.**
  Its response includes an ISRC and a live popularity figure matching Spotify's own, so this data
  clearly derives from Spotify in some way; ReccoBeats' public docs don't state whether it's
  independently computed or a cached mirror, or spell out redistribution terms. Persisting it to
  `/data/raw` was a deliberate decision made with that provenance uncertainty acknowledged.
- **The 2026 album's audio features remain fully manual** once filled in, sourced from third-party
  estimation tools (e.g. tunebat.com, chosic.com) rather than any original analysis pipeline.
- **Spotify popularity is a moving target, not a fixed dataset.** It's fetched live at request
  time and stamped with a `fetched_on` date; re-running the notebook later will very likely return
  different numbers, since Spotify's popularity score is recency-weighted by design.
- **Spotify's own data is never persisted to this repo.** Per Spotify's Developer Terms ("do not
  store Spotify Content indefinitely"), only a live, in-memory popularity lookup is used — nothing
  raw from the Spotify API itself is written to `/data/raw`. (ReccoBeats is a separate third party,
  not fetched through this project's own Spotify agreement — see above.)
- **The "dynamic range" proxy is not real dynamic range.** Spotify-style `loudness` is a single
  averaged dB value per track; within-album standard deviation across tracks is used as a rough
  stand-in for loudness variability, not an actual LUFS/dynamic-range measurement.
- **The "production polish" composite is a self-defined heuristic** (z-scored energy + loudness −
  acousticness), not a validated industry metric — used only to rank tracks against each other
  within this dataset.
- **The n=6 chart-correlation finding above is fragile.** Six data points is not enough to treat
  any correlation as robust; it's reported because it's real and counterintuitive, not because
  it's statistically conclusive.
- **Chart data covers only the 9 released singles**, and only Billboard Hot 100 peak position —
  not the Billboard 200 album chart, non-US charts, or streaming-specific charts.
- **TikTok/social engagement metrics were deliberately scoped out** of this version rather than
  shipped as empty placeholder columns; no public API exists for this and manual estimation was
  judged not worth the fabrication risk for this pass.
- **Deluxe/bonus tracks are kept, not dropped**, flagged via `is_bonus`, and excluded only from the
  default standard-edition comparisons — GUTS's 5 "spilled" deluxe tracks were recorded separately
  from its standard edition and would otherwise blur what "GUTS-era" sound means.

## Reproducing this

**Requirements:** Python 3.12+, and the packages in `requirements.txt`:

```bash
pip install -r requirements.txt
```

**API keys needed** (copy `.env.example` to `.env` and fill in):

- `SPOTIFY_CLIENT_ID` / `SPOTIFY_CLIENT_SECRET` — free, from a Spotify Developer app
  ([developer.spotify.com/dashboard](https://developer.spotify.com/dashboard); Client Credentials
  flow, no special access needed). Powers popularity lookups, and — if set — live Spotify-ID
  resolution for the ReccoBeats step (otherwise a hardcoded SOUR/GUTS ID map is used instead, so
  ReccoBeats works even without this). Audio-features access itself isn't available to any new
  app regardless of credentials.
- `GETSONGBPM_API_KEY` — optional; free registration at [getsongbpm.com/api](https://getsongbpm.com/api).
  As of this writing their endpoint is Cloudflare-blocked for programmatic requests, so this
  currently has no effect, but the code will pick it up automatically if that changes.
- **No key needed for ReccoBeats** — it's called directly, no registration required.

**Running it:**

1. `jupyter notebook notebooks/data_engineering_olivia_rodrigo.ipynb`, then run all cells top to
   bottom — each of the four sections can also be re-run independently once `/data/raw` exists.
2. To add the 2026 album's audio-feature data: open its `_raw.csv`, look up each track on a tool
   like tunebat.com or chosic.com, fill in the blank columns, save, then re-run Sections 2 through 4.
3. Outputs: `/data/raw/*.csv` (one per album, plus `chart_peaks.csv`),
   `/data/processed/tracks_clean.csv`, and every chart/statistic rendered inline in the notebook.

## Data sources

- Track identity: Wikipedia (SOUR, GUTS, and *you seem pretty sad for a girl so in love* album pages)
- Popularity: [Spotify Web API](https://developer.spotify.com/documentation/web-api) (live, not persisted)
- Audio features (SOUR + GUTS): [ReccoBeats](https://reccobeats.com) (free, no key; see caveats)
- Tempo/key (fallback): [GetSongBPM](https://getsongbpm.com) (currently blocked — see caveats)
- Spotify track IDs (fallback, no credentials needed): [kworb.net](https://kworb.net)
- Chart positions: compiled from Billboard/Wikipedia coverage
- 2026 album's audio features: manually compiled (see notebook Section 1.5 for methodology)

## Acknowledgments

Tempo and key data provided by [GetSongBPM](https://getsongbpm.com).
