# Sales-Performance-and-Marketing-Campaign
# Vehicle Inventory Forecasting & Sales Analytics

An end-to-end project that takes a raw automotive sales dataset all the way through to a validated, production-ready demand forecasting model. It started as a data analysis exercise and grew into a full pipeline: cleaning and extending the dataset, building out business KPIs, and then spending a serious amount of time getting the forecasting right, not just picking a model that looked good, but actually stress-testing it until I trusted the number.

## Why this project

Inventory planning in automotive retail is a balancing act. Order too many units and you're sitting on capital and paying holding costs. Order too few and you're losing sales to a competitor down the street. Most of that decision-making still runs on gut feel and last year's numbers. This project was an attempt to replace that with something statistically grounded, a forecast a procurement team could actually act on, with honest uncertainty attached to it.

## The dataset

The dataset follows a star-schema structure: one fact table (`Fact_VehicleSales`) at the center, surrounded by dimension tables for vehicles, customers, dealers, regions, channels, campaigns, and sales executives. It started life as 24 months of transactional data, but 24 months isn't really enough to reliably estimate a 12-month seasonal cycle, you need several repeated cycles before a model can tell the difference between a real pattern and noise. So the dataset was extended to a full 72 months (2020–2025), with the extension built to preserve realistic behavior rather than just padding out the numbers:

- Trend and seasonality that scale together (sales grow ~6x over the period, and the seasonal swings grow right along with them)
- Per-vehicle variation, so not every model grows in lockstep, some decline, most grow, growth isn't uniform
- A genuine mix of profitable and unprofitable marketing campaigns (not everything is a win)
- Purchase behavior that actually differs by customer type, income level, and vehicle body style, instead of being uniformly random

That last point mattered more than expected. Early versions of several charts came out suspiciously flat, conversion rates nearly identical across customer segments, CLV barely different between loyal and new customers, spend not really tracking income. Digging in, the root cause was that customer and vehicle assignment in the data generation was happening via unweighted random sampling, so there was no real signal for any of these dimensions to pick up on. Fixing that (weighting purchase frequency by customer loyalty, vehicle price by income bracket, conversion likelihood by body type and customer type) is part of why the KPI outputs below actually tell a story instead of a flat line.

## What's in the analysis

Six KPI areas, each with its own set of charts and a short writeup of what actually mattered in the numbers:

1. **Vehicle sales performance** - by model, brand, and fuel type
2. **Dealer & regional comparison** - where volume concentrates, and where the efficiency gaps are
3. **Campaign ROI & conversion funnel** - which marketing spend actually converts, and which doesn't
4. **Test-drive to purchase conversion** - how intent turns (or doesn't turn) into a sale
5. **Customer segmentation & CLV** - who the highest-value customers actually are
6. **Demand forecasting & inventory planning** - the forecasting work, described in detail below

There's also an interactive Power BI dashboard that pulls a lot of this together into something a non-technical stakeholder could click through.

## The forecasting work

This is where most of the actual effort went, and it's worth walking through honestly because the process mattered as much as the final number.

**Models tested:** SARIMA (multiple specifications, plus a full grid search), Prophet, LSTM, Holt-Winters (both additive and multiplicative seasonality), and a couple of ensemble combinations.

**What went wrong along the way, and why it's worth mentioning:**

- The first SARIMA fits looked great on paper (low AIC) but turned out to be numerically degenerate, one seasonal MA coefficient came out at 3.48 × 10¹³, which is not a real number, it's an optimizer that diverged. A diagnostic check (verifying the covariance matrix and log-likelihood weren't nonsensical) caught this, and it kept catching it again every time a "best AIC" model turned out to be untrustworthy on closer inspection. Lesson: AIC tells you about in-sample fit, not whether the fit is real.
- LSTM was tested properly, fixed a data-leakage bug in the scaler, right-sized the network, added early stopping, and even then it showed real run-to-run instability (MAE ranging from 386 to 671 across five otherwise-identical runs). With only ~60 monthly data points to train on, that's expected: deep learning wants a lot more data than a 5-year monthly series can offer.
- A naive seasonal baseline (literally just "this month = same month last year") beat both SARIMA and Prophet outright. That's an uncomfortable thing to find after building more sophisticated models, but it's honest, and it's a useful sanity check that more complexity isn't automatically better.

**What won:** Holt-Winters with *multiplicative* seasonality. The additive version systematically overshot every single month in the test period, because it assumes seasonal swings stay a constant size, and this data's swings grow proportionally with the overall trend. Multiplicative seasonality doesn't make that assumption, and the difference wasn't small: MAE dropped from 504.8 to 114.4 just from that one structural choice.

**Validation, not just a leaderboard:**
- Beat the naive seasonal baseline by a wide margin (114.4 vs. 171.5 MAE)
- Train/test MAE ratio of 1.52x with nearly identical MAPE on both sides, the signature of a model that generalizes rather than memorizes
- Directly tested whether the model's recency-weighted seasonal estimate was a problem (i.e., is it just repeating last year?) by building an equal-weighted alternative from scratch and backtesting it, it performed ~5x worse, which settled the question with evidence instead of a hunch
- Cross-checked the final forecast against Prophet as an independent second opinion before presenting a range rather than a single overconfident number

**Final result:** 92% forecast accuracy (MAPE 8.0%) on a 12-month out-of-sample test period, with a 6-month forward forecast used to generate actual inventory planning recommendations (forecast units, safety stock, reorder points) at both the aggregate and individual model level.

## Tech stack

- **Python** - pandas, numpy, statsmodels, Prophet, TensorFlow/Keras, scikit-learn
- **Power BI** - interactive dashboard layer
- **Matplotlib** - all custom charting

## A note on the dataset

This is a synthetic dataset, built and iteratively refined to behave like real automotive sales data, realistic seasonality, trend, customer behavior, and marketing performance, rather than pulled from an actual company. It's meant to demonstrate the full analytical and forecasting process end to end, including the debugging and validation work that a real project would require, not to represent any real business's actual figures.

## Limitations, stated plainly

- It's synthetic data, so absolute dollar figures shouldn't be read as real-world benchmarks
- Forecasting methods that lean on seasonal decomposition (which is all of them here) are structurally limited in how far into the future they can meaningfully project, they extrapolate learned patterns, they don't predict genuinely novel shifts
- Per-model and per-campaign-type forecasts get noisier at that granularity, since individual series have far less data to work with than the aggregate

