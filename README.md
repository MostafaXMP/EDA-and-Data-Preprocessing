# Song Popularity Preprocessing Pipeline

Interactive data cleaning/preprocessing pipeline for a Spotify tracks dataset, built with the eventual goal of predicting `track_popularity`. This isn't a model yet — it's the "get the data actually usable" step, which honestly ended up being 90% of the work like it always does.

## What's in here

- **`Preprocessing_V2.ipynb`** — main notebook. Loads the data, runs EDA, then walks through cleaning (duplicates → missing values → outliers → encoding).
- **`helper_V2.ipynb`** — the guts. `ImputationHandler` and `EncodingHandler` classes, plus a couple of small utility functions (`string_to_bool`, `remove_outliers`). Gets pulled into the main notebook with `%run helper_V2.ipynb`, so **it has to sit in the same folder** or the run cell will blow up looking for it.
- **`songs_train.csv`** — the raw dataset. 26,266 tracks, 23 columns. Spotify audio features (danceability, energy, loudness, valence, tempo, etc.) plus playlist/genre metadata and the target column, `track_popularity`.

## Why it's structured this way

I kept going back and forth between "just write a clean script" and "make it interactive so I can actually see what's happening at each step" — went with interactive. The `Preprocessing` class walks you through EDA first (shape, dtypes, nulls, duplicate counts, boxplots per numeric column, pairplots in batches of 4, corr heatmap) and sets some flags based on what it finds (`missing_values`, `duplicates`, `outliers`, `encoding`). Then `data_cleaning()` actually acts on those flags and asks what you want to do about each issue.

Basically: look first, then decide, rather than hardcoding "always impute with median" or whatever.

## Heads up before you run it

This thing is **fully interactive** — it uses `input()` a lot (yes/no prompts for outliers, sampling, encoding strategy, imputation strategy, even manual fill values). That means:

- Run it cell by cell in Jupyter, not as a script or with "Run All" — you'll need to actually answer the prompts as they come up.
- It will NOT work in a non-interactive environment (CI, `nbconvert --execute`, etc.) without modification.
- `string_to_bool()` accepts `true/false`, `yes/no`, `y/n`, `1/0`, `t/f` (case-insensitive). Anything else raises a `ValueError`, and `_safe_input` will just keep re-asking until you give it something valid.

## Setup

```bash
pip install pandas numpy scikit-learn matplotlib seaborn beautifultable
```

`beautifultable` gets installed via `!pip install` in the first cell of the main notebook too, so if you're on a fresh env it's handled, just slow.

## Running it

1. Open `Preprocessing_V2.ipynb`.
2. Run the imports cell (this also pulls in everything from `helper_V2.ipynb`).
3. Run the cell that loads `songs_train.csv` and kicks off `exploratory_data_analysis()` — go through the prompts.
4. Run `data_cleaning()` — same deal, answer as it goes.
5. Last cell dumps the cleaned/encoded dataframe to `Data/train_processed.csv`. The `Data/` folder gets created automatically if it doesn't exist.

If you said yes to saving a sample during EDA, you'll also get `Data/train_sample_data.csv` (random 150 rows, or fewer if the dataset's smaller than that).

## Missing value / encoding options

Missing values (numerical):
`Drop Rows / Mean / Median / Mode / Manual Constant / Forward Fill / Backward Fill / Linear Interpolation / KNN Imputation / Iterative Imputer`

Missing values (categorical):
`Drop Rows / Most Frequent / Placeholder string / Forward Fill / Backward Fill`

Encoding:
`One-Hot / Label Encoding / Ordinal Encoding (auto) / Target-Guided Mean Encoding`

Note on "Ordinal Encoding (Auto)" — it's not real ordinal encoding in the sense of respecting a meaningful order (Low < Medium < High). It's just `OrdinalEncoder` assigning arbitrary integers to categories. Fine for some models, misleading if you actually care about order. Left a comment about this in the helper notebook so I don't forget.

For KNN / Iterative imputation, heads up that it imputes across **all** numerical columns for context, not just the ones that are actually missing — that's intentional (more signal for the imputer) but worth knowing since it'll touch columns that had no missing values too.

## Known issues / stuff I still need to fix

- There's a leftover debug line in `data_cleaning()`:
  ```python
  print(f"Missing values = self.df.isna.sum().sum()")
  ```
  Missing the `()` after `isna`, so this just prints the literal string instead of the actual count. Harmless but embarrassing, fixing next pass.
- In EDA, if you say there are outliers and confirm you want them "marked for removal," it just prints a message — it doesn't actually remove anything at that point. The real removal only happens later in `data_cleaning()` if the `outliers` flag is set. A little confusing on first read, works fine functionally though.
- No train/test split logic yet — this is purely preprocessing on `songs_train.csv`. Whatever the equivalent test file is will need to go through the same steps separately (or I'll refactor this into something that fits/transforms and can be reused — TODO).
- Encoders/imputers aren't saved anywhere, so nothing here is currently reusable on new/unseen data without re-running the whole interactive flow. Fine for a one-off cleaning pass, not fine for a real pipeline yet.
- `track_name`, `track_artist`, and `track_album_name` each have 4 missing values in the raw data — small enough that dropping those rows is probably the easiest call, but the notebook will ask rather than assume.

## Dataset quick facts

- 26,266 rows × 23 columns, no duplicate rows in the raw file.
- Target column: `track_popularity` (0–100 scale).
- Mix of numeric audio features and categorical/text metadata (track/artist/album names, playlist info, genre, subgenre).
- Genres include things like `edm`, `rap`, `latin`, `pop`, `r&b`, `rock` (based on `playlist_genre`).
