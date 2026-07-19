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

**1. Data ingestion.** Four independent sources feed the pipeline, each wrapped behind its own
small class so any one of them can be swapped out later without touching the rest of the code:

- **Track identity** (names, track order, duration, bonus-track flags) comes from Wikipedia,
  hand-verified against each album's page, rather than Spotify's catalog — explained below.
- **Spotify popularity scores** are fetched live, at runtime, via the Spotify Web API's Client
  Credentials flow (app-only, no user login, since this is public catalog data). The same client
  is also used to test — live, against a real track — whether `/audio-features` still works.
- **Tempo/key/time signature** were meant to come from the free GetSongBPM API. Live testing
  found its endpoint sits behind Cloudflare bot protection that blocks plain server-side requests
  regardless of API key validity, so this source currently contributes nothing; the code detects
  this in one request and stops trying, rather than hammering a dead end.
- **Everything else** (energy, loudness, valence, danceability, acousticness, instrumentalness,
  speechiness) is a manual-entry template — the one guaranteed-to-work fallback once the free
  options are exhausted.
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

**What's real right now, from data already collected:**

- *SOUR*'s three singles peaked at #1 (Drivers License), #3 (Deja Vu), #1 (Good 4 U) on the
  Billboard Hot 100 — a non-monotonic pattern, i.e. the middle single charted lower than both the
  first and third.
- *GUTS*, by contrast, declined monotonically across its three singles: #1 (Vampire) → #7 (Bad Idea
  Right?) → #11 (Get Him Back!) — each successive single performing worse than the last.
- *you seem pretty sad for a girl so in love* broke that GUTS-era monotonic-decline pattern and
  looked more like SOUR's: #1 (Drop Dead) → #5 (The Cure) → #3 (Stupid Song) — its third single
  outperformed its second, same shape as SOUR's non-monotonic run.
- Across all three albums (42 tracks total), 9 were released as singles; the other 33 have no
  chart-peak data by design (Billboard doesn't track non-single tracks).

**What isn't available yet, and why:** every audio-feature column (energy, loudness, valence,
danceability, acousticness, instrumentalness, speechiness, tempo, key, time signature) is
currently blank for all 42 tracks — 0% filled — and so is Spotify popularity. This is not a bug or
an oversight; **no free, complete, programmatic source for this data exists** for these three
albums as of this writing:

- Spotify's `/audio-features` endpoint returns HTTP 403 for any app created after Nov 27, 2024
  (confirmed live in Section 1.2 of the notebook, against a real track ID, not just asserted).
- GetSongBPM's endpoint is Cloudflare-protected against server-side requests, independent of API
  key validity (also confirmed live, in Section 1.3).
- No Kaggle dataset is dedicated to this artist, and any general pre-2024 Spotify dataset
  structurally cannot contain the 2026 album regardless.
- Spotify popularity itself is fetched live and was 0/42 as of this write-up because
  `SPOTIFY_CLIENT_ID`/`SPOTIFY_CLIENT_SECRET` haven't been added to `.env` yet.

**To turn this into the full production-evolution story** (loudness/dynamic-range trends, energy
vs. valence shifts, whether feature correlations changed with the 2026 album's reported 80s
new-wave sound, and the counterintuitive polish-vs-popularity cases): fill in the manual columns
in `/data/raw/*.csv` using a per-track lookup tool (e.g. tunebat.com, chosic.com — the `notes`
column in each file documents this), add Spotify credentials to `.env`, and re-run Sections 2–4.
Every chart and statistic in the notebook is coded to compute automatically from whatever real
data exists rather than needing code changes — nothing here was estimated or invented in the
meantime to make the story look more complete than the data actually is.

## Data limitations and caveats

- **Audio features are not from Spotify's ground-truth pipeline.** Since that pipeline is dead for
  this project, manually-entered values come from third-party estimation tools, not Spotify's own
  (now-inaccessible) analysis — treat them as good-faith approximations, not the original signal.
- **Spotify popularity is a moving target, not a fixed dataset.** It's fetched live at request
  time and stamped with a `fetched_on` date; re-running the notebook later will very likely return
  different numbers, since Spotify's popularity score is recency-weighted by design.
- **Spotify data is never persisted to this repo.** Per Spotify's Developer Terms ("do not store
  Spotify Content indefinitely"), only a live, in-memory popularity lookup is used — nothing raw
  from the Spotify API is written to `/data/raw`. Track identity data instead comes from Wikipedia.
- **The "dynamic range" proxy is not real dynamic range.** Spotify-style `loudness` is a single
  averaged dB value per track; within-album standard deviation across tracks is used as a rough
  stand-in for loudness variability, not an actual LUFS/dynamic-range measurement.
- **The "production polish" composite is a self-defined heuristic** (z-scored energy + loudness −
  acousticness), not a validated industry metric — used only to rank tracks against each other
  within this dataset.
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
  flow, no special access needed). Powers popularity lookups only — audio-features access isn't
  available to any new app regardless of credentials.
- `GETSONGBPM_API_KEY` — optional; free registration at [getsongbpm.com/api](https://getsongbpm.com/api).
  As of this writing their endpoint is Cloudflare-blocked for programmatic requests, so this
  currently has no effect, but the code will pick it up automatically if that changes.

**Running it:**

1. `jupyter notebook notebooks/data_engineering_olivia_rodrigo.ipynb`, then run all cells top to
   bottom — each of the four sections can also be re-run independently once `/data/raw` exists.
2. To add real audio-feature data: open `/data/raw/*_raw.csv`, look up each track on a tool like
   tunebat.com or chosic.com, fill in the blank columns, save, then re-run Sections 2 through 4.
3. Outputs: `/data/raw/*.csv` (one per album, plus `chart_peaks.csv`),
   `/data/processed/tracks_clean.csv`, and every chart/statistic rendered inline in the notebook.

## Data sources

- Track identity: Wikipedia (SOUR, GUTS, and *you seem pretty sad for a girl so in love* album pages)
- Popularity: [Spotify Web API](https://developer.spotify.com/documentation/web-api) (live, not persisted)
- Tempo/key: [GetSongBPM](https://getsongbpm.com) (currently blocked — see caveats)
- Chart positions: compiled from Billboard/Wikipedia coverage
- Remaining audio features: manually compiled (see notebook Section 1.4 for methodology)

## Acknowledgments

Tempo and key data provided by [GetSongBPM](https://getsongbpm.com).
