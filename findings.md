# Findings: Reporting Delay and Flock Loss

## Headline

The notebook suggests that longer observed reporting delays are associated with larger flock losses, but the more causal-leaning estimate is smaller and statistically uncertain. The safest conclusion is that the relationship is suggestive, not conclusive.

## What the main results say

- Primary sample: 4,988 event-outbreak rows from a Europe-only, HxNx, non-wild, poultry-premises subset.
- Median reporting lag: 182 days.
- Baseline OLS estimate: +0.803% flock loss per additional day of delay.
- Baseline OLS 95% CI: +0.665% to +0.940% per day.
- Double Machine Learning estimate: +0.277% per additional day.
- DML 95% CI: -0.095% to +0.650% per day.

## Interpretation

The baseline regression finds a strong positive association between longer delay and larger flock loss. If read literally, that would imply roughly:

- about +27% flock loss for 30 extra days of delay
- about +105% flock loss for 90 extra days of delay

But the DML result is more cautious. After flexibly adjusting for the observed confounders, the estimated effect becomes much smaller and the confidence interval includes zero. That means the notebook does not provide strong causal evidence that delay definitely increases flock loss, even though the simple association is clearly positive.

## Which result should be trusted most

For presentation, the best summary is:

- OLS shows a positive association.
- DML weakens the claim and makes it statistically uncertain.
- Matching should not be treated as the headline estimate here.

The matching robustness check produces an implausibly huge effect size, which is more consistent with poor overlap or unstable matching than with a believable causal effect.

## What “lag” means in this notebook

The treatment variable is `reporting_lag_days`, defined as:

`reported_dt - start_dt`

where:

- `start_dt` is the earliest observed `event_started_on` date within an `event_id` + `outbreak_id` group
- `reported_dt` is the earliest observed `reported_on` date within that same group

So in plain language, lag is:

"How many days passed between when the outbreak is recorded as starting and when it is recorded as reported in this WAHIS extract."

Important caveat:

- this is the earliest report date visible in this file after collapsing rows
- it may still be later than the true first-ever report in the full WAHIS history

So the lag here is best understood as an observed reporting delay in this extract, not a guaranteed ground-truth first notification delay.

## Important limitations

- The data are observational, so unobserved confounding may still explain part of the relationship.
- The lag variable may contain measurement error because the extract may not preserve the full reporting history.
- The sample contains 4,988 rows but only 339 unique `event_id`s, so many rows are not fully independent.
- The Europe-only restriction improves comparability but limits generalization.
- Flock loss is measured using `quant_total_1_deaths_killed_slaughtered_total`, which is a practical severity proxy rather than a perfect biological measure.

## Suggested wording for a report or slide

"Longer observed reporting delays are associated with larger flock losses in the restricted WAHIS sample, but more flexible causal adjustment reduces the estimate and makes it statistically uncertain, so the findings should be presented as suggestive rather than conclusive."
