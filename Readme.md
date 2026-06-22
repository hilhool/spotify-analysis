# What makes a track a hit? Analysis of the most-streamed Spotify songs of 2023

Final project for the course "Data Analysis in Python".

We take the most-streamed Spotify tracks of 2023 and try to understand whether hits have anything
in common: does tempo or key matter, does landing in playlists help, in which month it pays off to
release a song, and whether major-key tracks really are streamed more eagerly than minor-key ones.

## What we investigated

We tested four hypotheses about what correlates with a track's success, plus an advanced time-series
part:

1. **H1 — Key.** Major-key tracks gather more streams on average than minor-key ones.
2. **H2 — Distribution.** The number of playlists a track lands in is linked to streams more strongly
   than any audio feature.
3. **H3 — Seasonality.** Releases are seasonal: tracks released early in the year and before summer
   gather more streams.
4. **H4 — Collaborations.** The more co-authors a track has (`artist_count`), the higher its chance of
   charting.
5. **Advanced — Time series.** Decomposition of the monthly release series into trend + seasonality +
   residual (`statsmodels.seasonal_decompose`).

## Data

The basis is the [Most Streamed Spotify Songs 2023](https://www.kaggle.com/datasets/nelgiriyewithana/top-spotify-songs-2023)
dataset from Kaggle: 953 tracks, 24 columns — stream counts, chart and playlist positions on four
platforms (Spotify, Apple Music, Deezer, Shazam) and the audio characteristics of a track: bpm, key,
danceability, energy, etc.

The dataset is raw: some numeric columns are stored as strings (`streams`, `in_deezer_playlists`,
`in_shazam_charts`), `key` has 95 missing values, `in_shazam_charts` another 50, and in one row the
stream count is replaced by a chunk of text describing the BPM. After cleaning (deduplication by
"track + artist", removal of the one row with a corrupt stream value, type casting and missing-value
handling) **948 tracks** remain (953 → 949 after dedup → 948 after dropping the broken row).

The dataset has no genre, so during cleaning we enrich it through the external
[iTunes Search API](https://itunes.apple.com/search) (no key): using the track name and the first
artist we pull `primaryGenreName`. Requests are cached in `data/itunes_cache.json` so re-runs are
reproducible and offline.

## Key findings (with numbers)

Cleaned sample: **948 tracks**. Streams are heavily right-skewed (mean ≈ 514M, median ≈ 288M,
max ≈ 3.70B), so for hypothesis testing we rely on rank/non-parametric methods.

| # | Hypothesis | Test | Key numbers | Verdict |
|---|---|---|---|---|
| H1 | Major streamed more than minor | Welch t-test + Mann-Whitney | Major median 301.5M (n=546) vs Minor 272.7M (n=402); t=1.344, **p=0.179**; Mann-Whitney **p=0.103** | **Not supported** |
| H2 | Playlists > audio features | Spearman correlation with streams | `total_playlists` **ρ=0.834**; best audio feature `speechiness_%` ρ=−0.108; most audio features \|ρ\|<0.06 | **Supported** |
| H3 | Release seasonality | Kruskal-Wallis across months (2022–2023) | **H=37.20, p<0.001**; release peak in **May (104 tracks, 2022–2023)** but its median streams (186M) are only average; highest medians in sparse Aug (285M, n=19) and Oct (275M, n=36) | **Partially supported** |
| H4 | More co-authors → charts | Spearman + Mann-Whitney | `artist_count` vs `chart_platforms` **ρ=0.024, p=0.457**; solo mean 2.43 (n=582) vs collab 2.55 (n=366), **Mann-Whitney p=0.101** | **Not supported** |

**Spearman correlation of `streams` with each predictor:**

| Predictor | ρ with streams |
|---|---|
| total_playlists | **0.834** |
| speechiness_% | −0.108 |
| danceability_% | −0.082 |
| liveness_% | −0.063 |
| acousticness_% | −0.059 |
| valence_% | −0.039 |
| energy_% | −0.033 |
| instrumentalness_% | −0.004 |
| bpm | 0.003 |

**Advanced part — time-series decomposition (release counts, 2021-01 to 2023-07):** the seasonal
component peaks in **May** (December also positive; summer and early spring negative), confirming H3's
finding that labels concentrate releases in May. The trend rises but is read cautiously — it largely
reflects survivorship bias of a top-2023 slice rather than pure market growth. The residual is small
and structureless, so trend + seasonality describe the series well.

## Conclusions

**What makes a track a hit in this data is, first and foremost, distribution — landing in playlists —
not how the track sounds or formal attributes.** Concretely:

- The number of playlists is by far the strongest correlate of streams (ρ=0.834), an order of
  magnitude above any audio feature. (Note: this is correlation, not causation — `total_playlists` and
  `streams` are partly endogenous, both reflecting a track's distribution and popularity.)
- Key (major vs minor), tempo, danceability and the other audio features have near-zero correlation
  with streams.
- Release timing drives *how many* tracks come out (a clear May peak), but not *how many streams* they
  get — releasing "in season" does not by itself guarantee popularity.
- The number of co-authors has essentially no link with charting.

## Project structure

```
data/
  spotify-2023.csv         - source dataset from Kaggle
  spotify-2023-clean.csv   - cleaned version (produced by raw.ipynb)
  itunes_cache.json        - cached iTunes Search API genre lookups
  raw.ipynb                - loading and cleaning the data
notebooks/
  analysis.ipynb           - analysis and charts
.gitignore
Readme.md
requirements.txt
```

## How to run

1. Install dependencies: `pip install -r requirements.txt`
2. Run the notebooks in order: first `data/raw.ipynb`, then `notebooks/analysis.ipynb`.

Both notebooks run top to bottom without errors. `raw.ipynb` reuses `data/itunes_cache.json`, so it
does not hit the network on a re-run.

## Team

| Member | Stage |
|---|---|
| Timur Kozlov | Data collection and cleaning |
| Timur Kozlov & Artemiy Alekseev | Analysis and hypothesis testing |
| Artemiy Alekseev | Visualisation and write-up of conclusions |
