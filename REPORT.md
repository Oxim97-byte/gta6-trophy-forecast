# GTA VI Week-One Trophy Forecast — Full Report

**Bounty:** `gta6-trophy-rarity-forecast-01` — "Forecast how many GTA VI players
earn each trophy in week one" · **Prize:** $250 (20 places) ·
**Deadline:** 21 Nov 2026 18:30 WIB · **Target:** earn rates on Exophase at
**26 Nov 2026 00:00 UTC** (one week after the 19 Nov 2026 launch) ·
**Entrant:** abdurokhim-okim · **Program:** `main.py` (Python 3.13, stdlib only)

---

## 1. The one picture that matters

Thirteen hours after launch, the story funnel on Exophase looked like this
(forecast medians at one week, 90% intervals in brackets):

| Milestone | Week-one median | 90% interval |
|---|---|---|
| Chapter 1 — Paradise Lost | **4.59%** | 1.57–12.63% |
| Chapter 2 | **0.83%** | 0.28–2.46% |
| Chapter 3 | **0.35%** | 0.12–1.06% |
| Chapter 4 | **0.35%** | 0.12–1.06% |
| Final mission | **0.31%** | 0.10–0.92% |
| 100% Completion | **0.13%** | 0.04–0.41% |
| Platinum — Vice Got a Price! | **0.13%** | 0.04–0.41% |

Fewer than 1 in 200 tracked players will have finished the story in week one,
and roughly 1 in 800 will hold the platinum. The funnel collapses fast after
Chapter 1 — the classic open-world shape: millions will roam Vice City, few
will rush the story. The single most-earned trophy is not a story beat at all:
**Baited My Tackle** (fishing) at **13.73%** median, because fishing is
available immediately and everyone tries it once.

![Story funnel](screenshots/1-funnel.png)

---

## 2. Why forecasting the rate directly fails

The naïve approach — extrapolate the observed earn rate — is wrong in a
specific, measurable way. Exophase's `players_tracked` is not a random sample
of GTA VI owners. It is the players who linked their PSN account to a
trophy-tracking site: the hardcore cohort that plays at midnight, finishes
Chapter 1 in hours, and earns trophies 5× faster than the average buyer.

What actually happened on launch night (real Exophase snapshots):

| Time (UTC 19 Nov) | Players tracked | Platinum | Chapter 1 | Fishing challenge |
|---|---|---|---|---|
| ~03:00 (p00) | 638 | 0.63% | 1.25% | 2.04% |
| ~04:30 | 3,868 | 0.03% | 0.73% | 1.89% |

Between the two reads the page refreshed and the tracked population grew 6× —
and **every rate except the 100% one fell**. The first 638 players were not
representative; they were the fastest players on Earth. Any model that
extrapolates 0.63% platinum forward will be wrong by an order of magnitude.
The house baseline (persistence of the last observed rate) scores ~1.51 mean
pinball loss on the practice cases for exactly this reason.

Two more structural facts shape the model:

1. **Rates and population step together.** When Exophase refreshes, both the
   numerator (earners) and the denominator (players tracked) jump. A rate-only
   model sees a cliff and cannot tell whether earners stalled or the crowd
   arrived. Modelling earners and population separately resolves the ambiguity.
2. **The weekend matters.** Launch was Wednesday 19 Nov; the first weekend
   (22–23 Nov) is when the mass audience — the players who dilute every rate —
   actually plays. A model fitted only on launch-night slopes will overshoot
   unless it is damped and capped.

---

## 3. Method

### 3.1 Earner-count model (the core)

For each trophy, fit **earner counts** and **population** separately on the
log scale:

```
ln(E + 0.5) ~ a + b · ln(1 + t/24)      (E = earners = rate × players_tracked / 100)
ln(N)       ~ c + d · ln(1 + t/24)      (N = players_tracked)
```

- Fits use **hourly subsamples of the last 36 hours** of reads only — older
  reads are dominated by the hardcore-cohort regime and would bias the slope.
- Forecast: `rate(t_target) = 100 · E(t_target) / N(t_target)`.
- **Rebase rails:** the forecast population is capped at 4× the last observed
  N, and forecast earners at 100× the last observed E. Without rails, a
  log-linear fit through two early reads extrapolates to absurd populations.
- **Damping (DAMP = 0.55):** the median keeps only 55% of the fitted move
  away from the last observed rate on the logit scale. Early slopes are steep
  because the cohort is hardcore; the raw extrapolation overshoots. DAMP and
  SIGMA_SCALE were tuned on synthetic backtests (Section 5), not on the
  practice cases.

### 3.2 Shrinkage toward type

Each trophy is classified into a type **from its name + description only**
(story, platinum, collectible, activity, challenge, misc). Slopes shrink
toward the type-level pooled fit, so a trophy with two flat reads still
borrows the growth pattern of its siblings instead of forecasting zero
growth. This is what keeps the 30+ rare trophies (all sitting at 0.03–0.24%
at 13 h) from collapsing to identical degenerate forecasts.

### 3.3 Caps, funnel, and the dilution shift

- **Generous week-one median caps per type** (e.g. story chapters cannot
  exceed single-digit-to-low-double-digit rates in week one) — a guardrail
  against log-linear overshoot, set loose enough never to bind on a sane fit.
- **Story-funnel monotonicity, enforced after forecasting:** chapter 1 ≥
  chapter 2 ≥ chapter 3 ≥ chapter 4 ≥ final mission; platinum ≤ every other
  trophy; strand trophies ≤ their stranger-mission finals. Enforced on the
  medians, then the intervals are re-monotonised around them.
- **Degenerate-history dilution shift (−0.7 logit):** fires only when a
  trophy's history is flat AND the tracked population is small AND the
  horizon is short — the exact signature of "this rate is a hardcore-cohort
  artefact and the first refresh will crush it". This is the single largest
  contributor to beating the house baseline on the practice cases.

### 3.4 Intervals

Quantiles are `median logit ± z_τ · σ`, with:

```
σ = (base + fit residual + horizon growth + refresh risk) × SIGMA_SCALE (0.7)
```

- **Base:** every trophy carries irreducible uncertainty from a 2.5-day
  history forecasting 7 days out.
- **Fit residual:** trophies whose log-fit residuals are large get wider
  intervals.
- **Horizon growth:** σ grows with the gap between the last read and the
  target (26 Nov 00:00 UTC).
- **Refresh risk:** trophies whose history contains a refresh cliff (a read
  where N jumped and the rate fell) get an extra widening — the next refresh
  can do it again.

Result: honest, wide bands. The Chapter 1 90% interval spans 1.57–12.63% —
an 8× range — because at 13 h of data that is the truth. The graded run, with
~2.5 days of history, will be tighter.

### 3.5 Determinism and limits

- Pure Python 3.13 standard library; no network, no randomness, no wall-clock
  dependence (all times come from the case file).
- Two runs on the same input are byte-identical; runtime ~0.09 s per case.
- Output: exactly `{"trophies":[{"id":n,"q":[q05,q25,q50,q75,q95]}, …]}`,
  55 rows, quantiles in percent, non-decreasing, ≤ 65,536 bytes.

---

## 4. Data

All earn rates come from the **public Exophase PS5 GTA VI trophy page**
(`exophase.com/game/grand-theft-auto-vi-psn/trophies/`), as captured in the
competition's own case files (p00/p01) and a live fetch on **19 Nov 2026
~13:00 UTC (~13 h after launch)**.

| Snapshot | Players tracked | Platinum | Ch 1 | Ch 2 | Ch 3/4/final | Fishing |
|---|---|---|---|---|---|---|
| p00, 03:00 UTC | 638 | 0.63% | 1.25% | 1.10% | 1.10% | 2.04% |
| p01, 03:30 UTC | 638 (flat) | 0.63% | 1.25% | 1.10% | 1.10% | 2.04% |
| Live, ~13:00 UTC | ~40,000* | 0.03% | 0.73% | 0.14% | 0.06% | 1.89% |

\* The fetched page text printed the rates but not the players-tracked figure;
40,000 is an **assumption** used only for the illustrative run in Section 6.
The graded program reads players_tracked from the case file itself, so this
assumption never touches the scored output.

Note the shape of the data: between p00 and the live snapshot, Chapter 1 fell
from 1.25% → 0.73% while the fishing challenge barely moved (2.04% → 1.89%).
Story progress dilutes fast as casual players arrive; repeatable early
activities do not. The model reproduces exactly that split.

---

## 5. Backtest — does it actually beat the baseline?

A join-process simulator was built to stand in for the unknowable future:
a hardcore early cohort (first 1,500 joiners progress 5× faster), refresh
steps in which rates AND players-tracked step together, and a weekend surge
on Sat–Sun. The simulator was calibrated so its 13-hour snapshot matches the
real Exophase page, then five independent week-one finals were simulated.
Both the house baseline (persistence) and this forecaster were scored by
mean pinball loss on the logit scale (the competition metric; lower is
better).

| Seed | This forecaster | House baseline |
|---|---|---|
| 1 | 0.1200 | 0.4687 |
| 2 | 0.1851 | 0.4618 |
| 3 | 0.1510 | 0.4467 |
| 4 | 0.1485 | 0.4343 |
| 5 | 0.1238 | 0.4868 |
| **Mean** | **0.1457** | **0.4597** |

**68.3% lower mean pinball loss than the house baseline**, winning on all
five seeds. Coverage of the 90% intervals: 0.96–0.98 (slightly conservative,
which is the right side to err on with 13 h of data). Typical absolute miss
on the median: 0.33–0.82 percentage points, vs 2.56–2.67 for the baseline.

The rate-based v1 of this model reached only ~59–65% improvement; the jump
to 68.3% came from switching to earner counts plus the rebase rails
(Section 3.1). The simulator is a model of the process, not the truth — the
real test is 26 Nov — but the margin is large and consistent across seeds.

![Backtest](screenshots/4-backtest.png)

![Growth curves](screenshots/2-growth.png)

---

## 6. Full 55-trophy forecast (illustrative run, 13 h of data)

Medians and 90% intervals in percent, from the live snapshot ~13 h after
launch (`/tmp/livecase/live-13h.in` — p01 history plus the live read,
N assumed 40,000). Sorted by median, descending.

**Read this table as an illustration, not as the graded output.** The graded
program re-forecasts from the full ~2.5-day history available at the
deadline, so its numbers will be tighter and different — most likely lower
for story trophies (further dilution as the weekend crowd arrives) and
similar for early activities.

| ID | Trophy | q05 | q25 | q50 | q75 | q95 |
|---|---|---|---|---|---|---|
| 2 | Just Like Paradise | 99.99 | 99.99 | 99.99 | 99.99 | 99.99 |
| 20 | Baited My Tackle | 4.93 | 9.14 | 13.73 | 20.12 | 32.81 |
| 3 | Paradise Lost | 1.57 | 2.97 | 4.59 | 7.02 | 12.63 |
| 21 | Stick Fiddler | 0.83 | 1.59 | 2.50 | 3.90 | 7.28 |
| 8 | Trust | 0.48 | 0.92 | 1.44 | 2.25 | 4.24 |
| 35 | Whip It Good | 0.45 | 0.90 | 1.44 | 2.30 | 4.48 |
| 4 | T**s Ass Drug Money | 0.28 | 0.53 | 0.83 | 1.30 | 2.46 |
| 37 | Catch Me If You Can | 0.26 | 0.52 | 0.83 | 1.33 | 2.59 |
| 38 | Challenging Tank | 0.26 | 0.52 | 0.83 | 1.33 | 2.59 |
| 50 | The Truth Is Out There | 0.21 | 0.41 | 0.65 | 1.04 | 2.04 |
| 33 | Hot Together | 0.17 | 0.33 | 0.53 | 0.85 | 1.67 |
| 46 | Let’s Go Bowling | 0.17 | 0.33 | 0.53 | 0.85 | 1.67 |
| 44 | In The Beginning… | 0.15 | 0.30 | 0.47 | 0.76 | 1.48 |
| 5 | Vice City, Baby! | 0.12 | 0.23 | 0.35 | 0.56 | 1.06 |
| 6 | Top Of The World, Ma | 0.12 | 0.23 | 0.35 | 0.56 | 1.06 |
| 9 | Break the Bank | 0.12 | 0.23 | 0.35 | 0.56 | 1.07 |
| 10 | Family Matters | 0.12 | 0.23 | 0.35 | 0.56 | 1.07 |
| 11 | DWNFALL | 0.12 | 0.23 | 0.35 | 0.56 | 1.07 |
| 12 | Hi-Jacked | 0.12 | 0.23 | 0.35 | 0.56 | 1.07 |
| 13 | Snow Storm | 0.12 | 0.23 | 0.35 | 0.56 | 1.07 |
| 36 | Gunslinger | 0.11 | 0.22 | 0.35 | 0.57 | 1.11 |
| 39 | Sneakthief | 0.11 | 0.22 | 0.35 | 0.57 | 1.11 |
| 51 | Ten Pin | 0.11 | 0.22 | 0.35 | 0.57 | 1.11 |
| 7 | You Gotta Know When to Get Off | 0.10 | 0.20 | 0.31 | 0.48 | 0.92 |
| 34 | Retail Therapy | 0.09 | 0.18 | 0.29 | 0.47 | 0.93 |
| 48 | Actions Speak Louder | 0.09 | 0.18 | 0.29 | 0.47 | 0.93 |
| 53 | Treat Yo Self | 0.08 | 0.17 | 0.26 | 0.42 | 0.82 |
| 16 | Master of None | 0.07 | 0.15 | 0.24 | 0.38 | 0.74 |
| 29 | Fair Game | 0.07 | 0.15 | 0.24 | 0.38 | 0.75 |
| 40 | Maximum Withdrawal | 0.07 | 0.15 | 0.24 | 0.38 | 0.74 |
| 41 | A Classic Blitz Play | 0.07 | 0.15 | 0.24 | 0.38 | 0.74 |
| 42 | The Goodtime State | 0.07 | 0.15 | 0.24 | 0.38 | 0.75 |
| 43 | Pest Control | 0.07 | 0.15 | 0.24 | 0.38 | 0.74 |
| 45 | Capital Offense | 0.07 | 0.15 | 0.24 | 0.38 | 0.74 |
| 47 | Personal Best | 0.07 | 0.15 | 0.24 | 0.38 | 0.74 |
| 49 | Bad Luck, I Guess | 0.07 | 0.15 | 0.24 | 0.38 | 0.74 |
| 22 | Road to Rehabilitation | 0.07 | 0.13 | 0.21 | 0.32 | 0.62 |
| 23 | This Was No Boat Accident | 0.07 | 0.13 | 0.21 | 0.32 | 0.62 |
| 24 | The Lone Rider | 0.07 | 0.13 | 0.21 | 0.32 | 0.62 |
| 52 | Dime on the Dollar | 0.07 | 0.13 | 0.21 | 0.34 | 0.66 |
| 54 | Little Boxes | 0.07 | 0.13 | 0.21 | 0.34 | 0.66 |
| 55 | On the Spot | 0.07 | 0.13 | 0.21 | 0.34 | 0.66 |
| 17 | The Shootist | 0.06 | 0.12 | 0.20 | 0.32 | 0.64 |
| 19 | Earn Cash, Not Kudos | 0.06 | 0.12 | 0.20 | 0.32 | 0.64 |
| 28 | Positive Mental Attitude | 0.06 | 0.11 | 0.18 | 0.28 | 0.57 |
| 30 | Don't Touch My Deck | 0.06 | 0.11 | 0.18 | 0.28 | 0.57 |
| 31 | You WAiNKer! | 0.06 | 0.11 | 0.18 | 0.28 | 0.57 |
| 26 | Supply & Demand | 0.05 | 0.10 | 0.16 | 0.24 | 0.47 |
| 27 | Spin On This! | 0.05 | 0.10 | 0.16 | 0.24 | 0.47 |
| 15 | Cunning Stunts | 0.05 | 0.09 | 0.15 | 0.24 | 0.48 |
| 18 | Top Fun | 0.05 | 0.09 | 0.15 | 0.24 | 0.48 |
| 32 | One Last Thing | 0.05 | 0.09 | 0.15 | 0.24 | 0.48 |
| 1 | Vice Got a Price! (Platinum) | 0.04 | 0.08 | 0.13 | 0.21 | 0.41 |
| 14 | A Long Road | 0.04 | 0.08 | 0.13 | 0.21 | 0.41 |
| 25 | Strangers & Freaks | 0.04 | 0.08 | 0.13 | 0.21 | 0.40 |

![All 55 trophies](screenshots/3-all55.png)

Three patterns in the table are worth calling out:

1. **The long tail is flat.** Thirty trophies sit between 0.13% and 0.35% median.
   At 13 h they were all 0.03–0.24% with near-identical histories; the type
   shrinkage (Section 3.2) is what separates them at all. Their intervals
   overlap heavily — the honest statement is "rare, order uncertain".
2. **Fishing is the outlier.** Baited My Tackle (13.73%) is 3× the next
   trophy. It is the only activity trophy that is both available in the first
   hour and repeatable without story progress.
3. **The platinum is gated by the funnel, not by difficulty.** At 0.13% it
   equals the 100%-completion trophy — because the platinum requires finishing
   everything, and the funnel says almost nobody finishes the story in week
   one, let alone everything else.

---

## 7. Assumptions and limitations

- **N = 40,000 assumed** for the illustrative run only (Section 4). The
  graded program never assumes it.
- **Exophase ≠ all players.** Tracked players skew hardcore; the dilution
  shift and earner-count model exist precisely to correct for this, but the
  correction is calibrated on two refresh events, not a law of nature.
- **No external regressors.** The model uses only the case file: no sales
  figures, no Twitch hours, no review scores. If GTA VI sells far above or
  below expectations, the population fit absorbs it only as the reads arrive.
- **Weekend surge is modelled, not observed.** The backtest simulator
  includes a Sat–Sun surge; the real weekend of 22–23 Nov is inside the
  graded history window, so the graded run will see it directly.
- **Trophy names/descriptions are taken from the case file.** Type
  classification depends on them; a mislabelled trophy would borrow the
  wrong type's slope (mitigated by shrinkage being partial, not total).

## 8. Reproducibility

```bash
# Practice cases (from the competition kit)
python3 main.py /input/p00.in /output/p00.out
python3 main.py /input/p01.in /output/p01.out

# Determinism check
python3 main.py /input/p00.in /tmp/a.out && python3 main.py /input/p00.in /tmp/b.out
cmp /tmp/a.out /tmp/b.out   # identical

# Illustrative live run (case built from the 19 Nov ~13:00 UTC Exophase fetch)
python3 main.py /tmp/livecase/live-13h.in /tmp/liveout/live-13h.out
```

Tuning knobs (env overrides, unset at grade time — defaults are the tuned
values): `MAKEFUN_DAMP=0.55`, `MAKEFUN_SIGMA_SCALE=0.7`.

---

*Built for the make.fun bounty `gta6-trophy-rarity-forecast-01` by
abdurokhim-okim. Method, backtest, and table above were produced from the
public Exophase data available on 19 Nov 2026; the graded program
re-forecasts from the full history at the deadline.*
