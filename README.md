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

**1. Data ingestion.** Six independent sources feed the pipeline, each wrapped behind its own
small class/function so any one of them can be swapped out later without touching the rest of
the code:

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
- **The 2026 album's audio features are computed locally via `librosa`** from personally-owned
  audio files (12 of its 14 tracks) — every free lookup source was exhausted first (Spotify dead,
  ReccoBeats' catalog predates the album, GetSongBPM/Tunebat/Chosic all confirmed Cloudflare-
  blocked on their actual data pages). See "OR3 methodology" below for what this changes.
- **Whatever none of the above covers** — 2 OR3 tracks with no audio file yet — is a manual-entry
  template, the last-resort fallback once every automated option is exhausted.
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

**OR3 (the 2026 album), computed via `librosa` from 12 of 14 tracks — a different methodology,
see the caveat below:**

- **Tempo sits between SOUR and GUTS, not a continuation of GUTS' slowdown**: mean 134.2 BPM
  across the 12 known OR3 tracks, vs. 141.5 (SOUR) and 128.6 (GUTS). Whatever this album is doing,
  it's not simply extending the tempo trend from the first two.
- **Widest tempo range of any album**: from 83.4 BPM ("Cigarette Smoke," the 5:40 closer) to
  161.5 BPM ("Maggots for Brains") — a wider spread than either SOUR or GUTS individually show.
- **"Cigarette Smoke" is the outlier on every axis**: slowest tempo (83.4 BPM), lowest RMS energy
  (0.140) and lowest zero-crossing rate of the batch, consistent with being the album's slow,
  dynamically expressive closer.
- **Key detection validated independently before running the full batch**: for "Drop Dead," this
  pipeline computed 129.20 BPM and G# Major — against an external claimed reference of 130 BPM
  and G#/A♭ Major (the same pitch class), a near-exact match arrived at without any tuning to fit
  it. Full detail in the notebook's Section 1.7.

**What still isn't available, and why:** 2 of the 2026 album's tracks ("Less," "Never Do") have no
audio file yet and stay pending manual entry, and Spotify popularity is 0/42 across the board —
simply pending `SPOTIFY_CLIENT_ID`/`SECRET` being added to `.env`. Not an oversight in either
case: no free, complete, programmatic source for 2026-album audio features has ever existed
(confirmed live at every turn — Spotify's endpoint is dead for any new app, ReccoBeats' catalog
necessarily predates this album, GetSongBPM/Tunebat/Chosic are all Cloudflare-blocked), which is
exactly why librosa became the fallback for the tracks that do have audio available.

**To complete the picture** (the last 2 OR3 tracks, real popularity correlations, and the
counterintuitive polish-vs-popularity check, which needs both audio features and popularity
together): add their audio files or manual entries, add Spotify credentials to `.env`, and re-run
Sections 2–4. Nothing above was estimated or invented to look more complete than the data actually
is — every number here came out of the notebook exactly as printed.

## OR3 methodology: why it isn't directly comparable to SOUR/GUTS

The 2026 album's audio features come from `librosa`, an open-source signal-processing toolkit,
run locally against personally-owned audio files — not from any API, and not the same kind of
measurement as SOUR/GUTS' Spotify/ReccoBeats-sourced features:

- **Spotify/ReccoBeats' features are outputs of a proprietary ML model** — `energy`, `valence`,
  `danceability`, etc. are trained scores normalized to 0–1 across Spotify's whole catalog.
- **librosa's outputs are raw DSP measurements** — RMS energy, spectral centroid/bandwidth/
  rolloff, zero-crossing rate, a dynamic-range proxy, and a beat-strength proxy — with no trained
  model behind them, and critically, **no equivalent to valence or danceability at all**, since
  those require a perceptual model, not just signal statistics.

Because of that, OR3's librosa output lives in its own `librosa_`-prefixed columns in
`tracks_clean.csv` rather than overwriting `energy`/`valence`/`danceability`/etc. — putting a raw
DSP number in the same column as an ML-trained score would silently conflate two different kinds
of measurement. The **one exception is `tempo`**: BPM is an objectively measurable physical
quantity regardless of who detects it, so OR3's librosa-detected tempo does populate the shared
`tempo` column — but every chart/table using it says so explicitly, since it's still a different
detection algorithm than whatever ReccoBeats uses upstream of Spotify.

**Practical effect on the charts in Sections 3–4**: OR3 correctly shows as having no data for
`energy`/`valence`/`acousticness`/etc. — not because entry is incomplete, but because those
specific ML-trained quantities were never computed for it and there's no honest way to synthesize
them from DSP measurements alone.

**Validated once before running the full batch**, per standard practice: this pipeline's
independently-computed tempo/key for "Drop Dead" (129.20 BPM, G# Major) landed almost exactly on
an external reference value (130 BPM, G#/A♭ Major) without any adjustment made to force the match
— reported as a genuine cross-check, not proof of anything beyond that one track.

**A note on the source audio**: the MP3 files used for extraction are not included in this
repository (`.gitignore` excludes them) — they're personally-owned copies of recently-released,
actively-commercial music, used only for local feature extraction, never redistributed.

## Data limitations and caveats

- **OR3's audio features are not comparable in kind to SOUR/GUTS' features** — see the dedicated
  section above. This is the single most important caveat for reading the findings above correctly.
- **SOUR/GUTS audio features come from ReccoBeats, not Spotify's own now-inaccessible pipeline.**
  Its response includes an ISRC and a live popularity figure matching Spotify's own, so this data
  clearly derives from Spotify in some way; ReccoBeats' public docs don't state whether it's
  independently computed or a cached mirror, or spell out redistribution terms. Persisting it to
  `/data/raw` was a deliberate decision made with that provenance uncertainty acknowledged.
- **The 2 remaining OR3 tracks without an audio file** ("Less," "Never Do") will need either an
  audio file (for librosa) or third-party manual lookup (tunebat.com, chosic.com) once available.
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

**For OR3's librosa extraction**: place personally-owned audio files for the 2026 album in
`OR-3/` at the project root, named `olivia-rodrigo-<track-slug>.mp3` (see Section 1.7's
`OR3_FILENAME_TO_TITLE` map in the notebook for exact expected filenames). This folder is
gitignored and not part of the repository — it has to be supplied locally.

**Running it:**

1. `jupyter notebook notebooks/data_engineering_olivia_rodrigo.ipynb`, then run all cells top to
   bottom — each of the four sections can also be re-run independently once `/data/raw` exists.
2. To add the 2 remaining OR3 tracks: either add their audio files to `OR-3/` and re-run Section
   1.7, or look them up manually on tunebat.com/chosic.com and fill in their `_raw.csv` row.
3. Outputs: `/data/raw/*.csv` (one per album, plus `chart_peaks.csv`),
   `/data/processed/tracks_clean.csv` and `or3_audio_features_computed.csv`, and every
   chart/statistic rendered inline in the notebook.

## Data sources

- Track identity: Wikipedia (SOUR, GUTS, and *you seem pretty sad for a girl so in love* album pages)
- Popularity: [Spotify Web API](https://developer.spotify.com/documentation/web-api) (live, not persisted)
- Audio features (SOUR + GUTS): [ReccoBeats](https://reccobeats.com) (free, no key; see caveats)
- Tempo/key (fallback): [GetSongBPM](https://getsongbpm.com) (currently blocked — see caveats)
- Spotify track IDs (fallback, no credentials needed): [kworb.net](https://kworb.net)
- Chart positions: compiled from Billboard/Wikipedia coverage
- OR3 (2026 album) audio features: computed locally via [librosa](https://librosa.org) from
  personally-owned audio (see "OR3 methodology" above and notebook Section 1.7)
- Remaining OR3 tracks without an audio file: manually compiled (see notebook Section 1.5)

## Acknowledgments

Tempo and key data provided by [GetSongBPM](https://getsongbpm.com).
