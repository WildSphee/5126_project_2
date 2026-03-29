# Bird Outbreak Activity vs Poultry Commodity Movement

## Problem Statement

This project tests whether bird outbreak activity from WAHIS helps explain or predict poultry-related commodity movement from FAOSTAT.

The two source files live in the project root:

- `FAOSTAT_data_en_3-28-2026.csv`
- `WAHIS-2026-03-04.csv`

The main challenge is that the data sources are very different:

- WAHIS is event-level, duplicated, and messy
- FAOSTAT is yearly and already aggregated

So the core workflow is:

1. deduplicate outbreak events
2. aggregate outbreaks to `country + year`
3. engineer outbreak features and lags
4. align them to the FAOSTAT series
5. test whether outbreak features improve prediction beyond simple history-based baselines

## Current Notebook

The main analysis lives in:

- `csv_overview.ipynb`

The notebook currently:

1. loads and profiles both CSVs
2. compares FAOSTAT and WAHIS country naming
3. standardizes country keys with a small alias map
4. filters WAHIS to avian `HxNx` subtype rows
5. deduplicates repeated WAHIS rows to one outbreak record
6. aggregates outbreak signals to `country + year`
7. prepares a FAOSTAT target table for chicken production
8. builds lagged features and a modeling table
9. runs a naive baseline, Ridge regression, and XGBoost with a time-based split

## Important Data Note

The current FAOSTAT extract does **not** contain price. It only contains yearly production values in tons.

Because of that, the notebook currently predicts:

- **next-year percentage change in chicken production**

This is a proxy target, not a true price forecast. The pipeline is structured so a true poultry price series can be swapped in later.

## Validated Findings

These are from the currently executed notebook output:

- 163 of 201 FAOSTAT country names have a direct standardized key match to WAHIS
- the `HxNx` outbreak filter keeps 45,183 WAHIS rows
- those rows collapse to 44,629 outbreak-level records
- those records aggregate to 903 `country + year` outbreak rows
- the modeling table contains 3,483 rows

Held-out model results for years 2019-2023:

| Model | MAE | RMSE | Directional Accuracy |
| --- | ---: | ---: | ---: |
| XGBoost | 0.403734 | 9.622852 | 0.659375 |
| Ridge | 0.419130 | 9.657777 | 0.676042 |
| Naive last-change baseline | 0.750307 | 13.656757 | 0.612500 |

Current headline takeaway:

- XGBoost has the best MAE and RMSE
- Ridge has slightly better directional accuracy
- both learned models beat the naive baseline on this proxy target
- outbreak signal appears usable in the merged table, but the relationship is still weak enough that better target data and richer feature design would matter

## Running With Poetry

This repo uses Poetry with an in-project virtual environment at `.venv`.

Install dependencies:

```bash
poetry install
```

Run the notebook from the Poetry environment:

```bash
poetry run jupyter notebook
```

If you want to execute the notebook non-interactively:

```bash
poetry run python - <<'PY'
from pathlib import Path
import nbformat
from nbclient import NotebookClient

path = Path("csv_overview.ipynb")
with path.open() as f:
    nb = nbformat.read(f, as_version=4)

NotebookClient(nb, timeout=1800, kernel_name="python3").execute(cwd=str(path.parent.resolve()))

with path.open("w") as f:
    nbformat.write(nb, f)
PY
```

## Suggested Next Steps

1. Replace the FAOSTAT production proxy with a real poultry price series.
2. Expand the country alias map before deeper country-level interpretation.
3. Add richer outbreak severity features such as duration, subtype weighting, and multi-year rolling signals.
4. Compare all-country modeling against regional or country-cluster models.
