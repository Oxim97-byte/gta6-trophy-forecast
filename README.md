# GTA VI — Week-One Trophy Forecast

**make.fun bounty:** `gta6-trophy-rarity-forecast-01` — "Forecast how many GTA VI
players earn each trophy in week one" ($250, deadline 21 Nov 2026 18:30 WIB).

A deterministic, stdlib-only Python forecaster. Given a case file (Exophase
snapshots of the PS5 GTA VI trophy list), it predicts the earn rate of all
**55 trophies** at **26 Nov 2026 00:00 UTC** — one week after launch — as five
quantiles `[q05, q25, q50, q75, q95]` per trophy.

## Run it

```bash
python3 main.py /input/<case-id>.in /output/<case-id>.out
```

- Reads one case: `{"trophies":[{"id":n,"name":…,"description":…,
  "reads":[{"t":<unix>,"rate":<percent>,"players_tracked":n}, …]}, …]}`
- Writes one JSON object: `{"trophies":[{"id":n,"q":[q05,q25,q50,q75,q95]}, …]}`
- 55 rows, quantiles in percent (0–100), non-decreasing, output ≤ 65,536 bytes.
- Runtime ~0.09 s per case, 1 CPU, fully deterministic (two runs byte-identical).

## Method in one paragraph

Earn **rates** are a trap early on: Exophase's tracked population is a
self-selected hardcore cohort, so launch-night rates (platinum 0.63% at 638
players) collapse the moment the page refreshes (0.03% at 3,868 players).
Instead of forecasting the rate, the model forecasts **earner counts** and
**population** separately, both as `ln(E + 0.5) ~ ln(1 + t/24)` OLS fits on
hourly subsamples of the last 36 h, then divides: `rate = 100 · E / N`.
Slopes shrink toward type-level pooled fits (type classified from the trophy
name + description only); week-one medians are capped per type; the story
funnel is enforced (chapter 1 ≥ 2 ≥ 3 ≥ 4 ≥ final; platinum ≤ everything);
a dilution shift fires only on flat history + small N + short horizon.
Quantiles are `median logit ± z_τ · σ`, with σ growing in fit residual,
horizon, and refresh risk. Full detail, backtests, and the complete
55-trophy table: see [`REPORT.md`](REPORT.md).

## Headline forecast (illustrative run, 13 h of real data)

From the live Exophase snapshot ~13 h after launch (19 Nov 2026 ~13:00 UTC):

| Trophy | Median | 90% interval |
|---|---|---|
| Chapter 1 — Paradise Lost | 4.59% | 1.57–12.63% |
| Chapter 2 | 0.83% | 0.28–2.46% |
| Final mission | 0.31% | 0.10–0.92% |
| Platinum — Vice Got a Price! | 0.13% | 0.04–0.41% |
| Baited My Tackle (fishing) | 13.73% | 4.93–32.81% |
| Just Like Paradise | 99.99% | 99.99–99.99% |

**Caveat:** these medians come from only ~13 h of history. The graded program
re-forecasts from the full ~2.5-day history available at the deadline, so its
numbers will be tighter and different. The table above is an honest
illustration, not the graded output.

## Validation

- Practice cases p00/p01: valid output (55 rows, monotone, in range),
  deterministic, ~0.09 s.
- Synthetic backtest (join-process simulator: hardcore early cohort, refresh
  steps, weekend surge, calibrated to the real Exophase page): **mean pinball
  loss 0.1457 vs house baseline 0.4597 — 68.3% lower**, over 5 seeds
  (mine 0.1200–0.1851, baseline 0.4343–0.4868); coverage 0.96–0.98.
- Charts (4K PNG, `screenshots/`): story funnel, growth curves, all 55
  trophies, backtest comparison.

## Files

| File | What it is |
|---|---|
| `main.py` | The forecaster (the graded program) |
| `REPORT.md` | Full method, assumptions, backtest, 55-trophy table |
| `README.md` | This file |
| `LICENSE` | MIT |
| `screenshots/` | 4K charts (local visual evidence, not part of the program) |

No dependencies beyond Python 3.13 standard library. No network access at
grade time; all calibration constants are baked into `main.py`.
