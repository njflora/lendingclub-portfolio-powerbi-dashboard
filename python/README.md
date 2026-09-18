# Python Companion Analysis

`LendingClub_Python_Companion.ipynb` reproduces the core findings from the main [Power BI dashboard](../README.md) — the same mispricing-gap methodology, the same vintage-maturity-aware cohort analysis and the same Estimated Excess Loss recommendation figure. This is achieved via pandas/NumPy/matplotlib, working directly from the raw Kaggle CSV rather than the cleaned Power BI model.

This isn't a rebuild of the dashboard. It exists to demonstrate the analytical judgment behind the project and that it isn't confined to just one BI tool. Doing the same analysis a second time also surfaced two things worth knowing about the original build, which can be read about further in the main README's limitations section.

## Running it

1. Download `accepted_2007_to_2018Q4.csv.gz` from the [Kaggle LendingClub dataset](https://www.kaggle.com/datasets/wordsforthewise/lending-club) (not included in this repo — the raw file is ~1.6GB decompressed) and place it in `python/data/`.
2. `pip install -r requirements.txt`
3. Open `LendingClub_Python_Companion.ipynb` and run all cells top to bottom (the full pipeline runs in well under a minute).

The notebook as committed already has every cell's real output baked in, so it can be read directly on GitHub without running anything.

## Screenshots

1. Mispricing Scatter
<img width="1350" height="900" alt="mispricing_scatter" src="https://github.com/user-attachments/assets/07123994-2284-4d7d-8755-3127edd1a2fd" />
2. Vintage Cohort Curves
<img width="1350" height="825" alt="vintage_cohort_curves" src="https://github.com/user-attachments/assets/dc7e51af-e33b-4496-8b0d-898e210b4a8c" />

