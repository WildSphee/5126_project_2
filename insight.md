# Flock Size Modeling Insights

## Problem Statement

This analysis asks whether the event-level WAHIS + FAOSTAT feature table can predict flock size loss, using `last_flock_lost_total` from `output/wahis_hxnx_event_level_with_faostat.csv` as the target.

The modeling problem is difficult because the target is extremely skewed:

- mean: `353,108.961`
- median: `670`
- 99th percentile: `5,863,864.130`
- max: `203,648,009`

This means a small number of very large loss events dominate the raw scale. It also means evaluation depends heavily on whether we score predictions on the original scale or on a logged target.

## Approach

The notebook in [notebooks/flock_size_ml_notebook.ipynb](/home/azureuser/5123_project_2/notebooks/flock_size_ml_notebook.ipynb) uses a time-aware split:

- train: events with `event_anchor_year < 2024`
- test: events with `event_anchor_year >= 2024`

Dataset sizes:

- train rows: `1,419`
- test rows: `679`

The workflow:

1. Build month and quarter features from event date columns.
2. Remove obvious leakage columns, especially `last_quant_total_1_deaths_killed_slaughtered_total`.
3. Train three models:
   - `Baseline Median`
   - `Glassbox Ridge`
   - `Blackbox RandomForest`
4. Compare models on holdout data.

Important leakage note:

- the removed deaths column has correlation `0.999996` with the target
- keeping it would make the task unrealistically easy and would invalidate the model comparison

The notebook already trains with `log1p(y)` internally through `TransformedTargetRegressor`, but its saved metrics are reported after converting predictions back to the original flock-size scale.

## Results

### Holdout Results on Original Flock-Size Scale

| Model | MAE | RMSE | R2 |
| --- | ---: | ---: | ---: |
| Glassbox Ridge | 440,362.838 | 6,654,162.944 | 0.283 |
| Blackbox RandomForest | 430,135.046 | 7,591,765.234 | 0.067 |
| Baseline Median | 491,748.753 | 7,875,833.028 | -0.004 |

Headline result on the original scale:

- best `R2`: `Glassbox Ridge` with `0.283`
- best `RMSE`: `Glassbox Ridge`
- best `MAE`: `Blackbox RandomForest`

### Holdout Results on Logged Target Scale

I reran the same split and feature setup using `log1p(last_flock_lost_total)` as the direct prediction target and scored `R2` on that logged scale.

| Model | MAE_log | RMSE_log | R2_log |
| --- | ---: | ---: | ---: |
| Blackbox RandomForest | 1.377320 | 1.898234 | 0.805729 |
| Glassbox Ridge | 2.225980 | 3.149505 | 0.465197 |
| Baseline Median | 3.846667 | 4.366190 | -0.027814 |

Headline result on the log scale:

- best `R2_log`: `Blackbox RandomForest` with `0.805729`

## Interpretation

The main takeaway is that the "best model" depends on the scale that matters for the decision.

On the original flock-size scale, `Glassbox Ridge` is the strongest overall model because it has the best `R2` and the best `RMSE`. That means it explains more variance in raw flock losses and handles large events better overall than the Random Forest in this holdout setup.

On the logged scale, `Blackbox RandomForest` is much stronger, with `R2_log = 0.805729`. This means it is much better at capturing relative differences and order-of-magnitude behavior once the extreme skew is compressed.

These two results are not contradictory. They reflect two different questions:

- original scale: "How well do we predict actual flock losses in raw units?"
- log scale: "How well do we predict proportional or multiplicative differences across events?"

Because the target distribution is so skewed, a model can perform very well on the log scale while still struggling with the biggest raw-loss events after back-transformation.

## Key Insights

- The flock-loss target is highly right-skewed, so raw-scale metrics are dominated by a small number of extreme outbreaks.
- Leakage control is essential here. One removed feature is almost identical to the target.
- If interpretability and raw-scale fit matter most, `Glassbox Ridge` is the best current choice.
- If relative accuracy across small and medium events matters more, the logged-target `Blackbox RandomForest` is the best current choice.
- The gap between `R2 = 0.283` on the raw scale and `R2_log = 0.805729` on the log scale shows that the table contains useful predictive signal, but much of it is easier to learn in multiplicative rather than absolute terms.

## Bottom Line

For a reportable raw flock-size prediction result, use:

- **Best model:** `Glassbox Ridge`
- **Holdout R2:** `0.283`

If you want a stronger result for modeling the target after stabilizing its skew with a log transform, use:

- **Best log-target model:** `Blackbox RandomForest`
- **Holdout R2 on log1p target:** `0.805729`
