# Spotify Track Popularity Analysis

This project builds a dataset of tracks released from 1980 to 2023 using the Spotify Web API, then explores which audio features relate to a track's Spotify popularity score. `data.py` searches Spotify for tracks by release year and fetches each track's metadata and audio features into one CSV per year. `train.ipynb` combines the 8,773 collected tracks, runs exploratory data analysis, and fits a random-forest regressor to predict popularity.

## Features

- **Year-based crawler.** `data.py` gets an app token with the Client Credentials flow, then pages through `GET /v1/search?q=year:<year>&type=track` 50 tracks at a time. For each track it calls `GET /v1/tracks/{id}` (artists, release date, popularity) and `GET /v1/audio-features/{id}` (acousticness, danceability, energy, and so on).
- **Resumable output.** Each run appends to `spotify_api_<year>.csv` if the file already exists, so several pages of the same year end up in one file.
- **Batch runner.** `script.py` calls `data.py` for a range of years and pages.
- **Analysis notebook.** `train.ipynb` contains:
  - popularity distribution, feature histograms and a correlation heatmap
  - popularity vs. feature scatter plots, broken down by musical key and BPM
  - a `RandomForestRegressor` baseline, a `GridSearchCV` hyperparameter search, feature importances and residual analysis

## Results

`train.ipynb` uses an 80/20 train/test split (`random_state=42`) over the 8,773 collected tracks:

| Model | Test MSE | Test R² |
| --- | --- | --- |
| RandomForestRegressor, 100 trees | 43.51 | 0.488 |
| RandomForestRegressor, grid-searched (`max_depth=10`, `min_samples_leaf=4`, `min_samples_split=2`, `n_estimators=300`) | 42.39 | 0.502 |

The grid search tried 81 parameter combinations with 5-fold cross-validation. Release year dominates the feature importances: 0.51 in the baseline model and 0.70 in the tuned model. Every audio feature scores below 0.05.

## Repository structure

```
.
├── data.py                    # Spotify Web API crawler (one year, one page of 50 tracks per run)
├── script.py                  # loops data.py over years 2018-2019 and pages 0-3
├── train.ipynb                # EDA and popularity regression
├── album_template.py          # sample Spotify search response, kept for reference
├── spotify_api_1980.csv ... spotify_api_2023.csv   # collected data, one file per release year
├── spotify_api_template.csv   # header-only CSV with the output schema
├── .env.example               # template for the Spotify API credentials
└── spotify-2023.csv           # reference dataset of 953 popular 2023 tracks
```

## Collected data schema

Every `spotify_api_<year>.csv` has these columns:

| Column | Source |
| --- | --- |
| `track_name`, `track_id` | search result |
| `artist(s)_name`, `artist_count` | track info (multiple artists joined with ` & `) |
| `released_year`, `released_month`, `released_day` | album release date (month/day are empty when Spotify only gives the year or month) |
| `popularity` | track info (0-100) |
| `acousticness_%`, `danceability_%`, `energy_%`, `instrumentalness_%`, `liveness_%`, `speechiness_%` | audio features x 100 |
| `valence_%` | audio feature, stored on the original 0-1 scale |
| `duration_ms`, `key`, `mode`, `loudness`, `tempo(bpm)`, `time_signature` | audio features |

## Setup

```bash
pip install pandas numpy requests python-dotenv
```

`data.py` also imports `tensorflow` and `keras` at the top of the file, though it never uses them. Install both, or delete those two imports. The notebook needs `pandas`, `numpy`, `matplotlib`, `seaborn` and `scikit-learn`.

Create a Spotify app in the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard). Copy the template and fill in the app's credentials:

```bash
cp .env.example .env
```

`data.py` reads `CLIENT_ID` and `CLIENT_SECRET` from `.env` with `python-dotenv`. `.env` is listed in `.gitignore`, so it stays out of version control.

Spotify restricted the audio-features endpoint for newly created apps in November 2024. With new credentials, the crawler may receive errors for `/v1/audio-features`. It skips those tracks.

## Usage

Collect one page of 50 tracks released in a given year. `-y/--year` sets the year (default 2020). `-t/--turn` sets the page, and the search offset is `50 * turn` (default 0):

```bash
python data.py -y 2022 -t 0
python data.py -y 2022 -t 1
```

Each run fetches only 50 tracks to stay within the API rate limits. Run it several times with increasing `-t` to collect more tracks for a year. The committed data covers pages 0-3 (about 200 tracks) for each year.

To crawl several years in one go, edit the ranges in `script.py` and run it. It invokes `data.py` with a Windows-style path (`.\data.py`), so adjust that on macOS or Linux.

```bash
python script.py
```

To reproduce the analysis, open `train.ipynb` in the repository root and run all cells. It loads every `spotify_api_*.csv` file in the folder.

## Acknowledgements

- Track metadata and audio features come from the [Spotify Web API](https://developer.spotify.com/documentation/web-api).
- The column naming of the collected CSVs follows `spotify-2023.csv`, which matches the Kaggle dataset [Most Streamed Spotify Songs 2023](https://www.kaggle.com/datasets/nelgiriyewithana/top-spotify-songs-2023).
