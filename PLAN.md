# Plan: Differentiated Leader Strategies for MK1, MK2, MK3

## Context
The base class `AdaptiveStackelbergLeader` gives each leader a working linear EWLS strategy.
But each follower has a distinct behavioural profile revealed by data analysis, meaning a
one-size-fits-all approach leaves profit on the table. This plan defines what makes each
concrete leader different and why.

---

## Data Analysis Summary (critical inputs to the plan)

| | MK1 | MK2 | MK3 |
|---|---|---|---|
| u_F range | [2.90, 3.85] | [1.50, 51.83] | [1.23, 1.58] |
| u_F mean | 3.51 | ~2.4 (ex outlier) | 1.34 |
| Linear R² | **0.16** (significant) | 0.008 (not sig.) | 0.038 (borderline) |
| Slope days 1–50 | 1.23 | -19.5 (outlier-driven) | 0.21 |
| Slope days 51–100 | 1.95 | **+0.79** | 0.46 |
| Autocorrelation | none | none | none |
| Key issue | Slope drifting upward | Regime flip at ~day 50 | Near-constant follower |

All followers are **stateless** (no autocorrelation) — they react only to the current leader
price, not their own history. This justifies the linear reaction function model.

---

## File to modify
`comp34612_project.ipynb` — cell 41 only.
The `AdaptiveStackelbergLeader` base class stays unchanged.

---

## LeaderMK1

**Follower profile:** Genuine linear reaction (R²=0.16, only significant one). Slope is
drifting upward over time (1.2 → 1.9 across 100 days), suggesting the follower becomes
more responsive as time goes on. Very stable otherwise.

**EWLS fit (all 100 days, decay=0.95):** a=1.700, b=0.471, residual std=0.164.

**Critical finding:** With a=1.700, B = 3×1.700 − 5 = **+0.10 > 0** — the profit
function is convex, so the analytic optimum is at the price cap. The threshold is
a=5/3=1.667; our estimate sits only 0.033 above it. This means `max_price` is the
dominant lever, not the fitted slope. **The member working on MK1 should experiment
with raising `max_price` above 50 in the live simulation** — if the follower keeps
responding as expected, higher prices mean higher profit.

Expected profit at cap: **~5,217/day → ~156,500 over 30 days.**

**What to change vs base:**
- `decay=0.95` (down from 0.97) — more aggressive discounting to track the drifting slope.
- `max_price=50` — starting point; **tune upward** based on live simulation results.

**Override:** Just change `decay` in `__init__`. No method overrides needed.

```python
class LeaderMK1(AdaptiveStackelbergLeader):
    def __init__(self, name, engine):
        super().__init__(name, engine, min_price=1.0, max_price=50.0, decay=0.95)
```

---

## LeaderMK2

**Follower profile:** Complete regime flip around day 50. Days 1–50: slope ≈ -19
(dominated by the day-36 outlier spike of u_F=51.83). Days 51–100: slope ≈ +0.79, normal
linear behaviour. Early data is from a completely different strategy and actively misleads
the fit. The outlier filter alone is not enough — it removes the spike but leaves 49 days
of data from the wrong regime.

**EWLS fit (days 51–100 only, decay=0.97):** a=1.071, b=0.989.
Optimal price u_L*=**29.3** (not 43 — earlier estimate used the wrong full-history fit).
Expected profit: **~1,433/day → ~42,977 over 30 days.**

**Critical finding:** The slope from days 51–100 has p=0.14 — not statistically
significant. There are only 50 training points in a narrow u_L range, so the slope
estimate carries real uncertainty. **The member working on MK2 must prioritise
exploration in the first few simulation days** — playing a range of prices (e.g. 10,
20, 40) to quickly generate data points outside the historical range and sharpen the
slope estimate before committing to the greedy optimum.

**What to change vs base:**
- In `start_simulation()`: only load the **last 50 days** of history (days 51–100)
  instead of all 100. This throws away the wrong-regime data entirely.
- Keep the outlier filter (day 36 is in days 1–50 anyway, so it gets dropped automatically).
- `decay=0.97` — fine, the 50 loaded days are already from the right regime.
- `max_price=50` — keeps headroom above the estimated optimum of 29.3.

**Override:** `start_simulation()` only.

```python
class LeaderMK2(AdaptiveStackelbergLeader):
    def __init__(self, name, engine):
        super().__init__(name, engine, min_price=1.0, max_price=50.0, decay=0.97)

    def start_simulation(self):
        # Only use the second half of history — first 50 days are a different regime
        for t in range(51, 101):
            lp, fp = self.get_price_from_date(t)
            self.u_L_hist.append(lp)
            self.u_F_hist.append(fp)
        self._fit_ewls()
```

---

## LeaderMK3

**Follower profile:** Follower price is nearly independent of the leader price (R²=0.038,
borderline significance). Prices are very tightly clustered around 1.34 ± 0.07. The
follower behaves almost like a **fixed-price agent**. The slope is very small and
unreliable given the narrow historical u_L range.

**Implication:** Instead of trusting the slope (which could be noise), model the follower
as a **slowly-varying constant**: `u_F ≈ mean(recent u_F)`. This is equivalent to fitting
only the intercept `b` and setting `a=0`. The optimal price under a constant follower:

```
S_L = 100 - 5*u_L + 3*b  =>  pi_L = (u_L-1)*(100 + 3b - 5*u_L)
u_L* = (105 + 3b) / 10
```

With b ≈ 1.346 (EWLS fit on days 71–100, decay=0.93): u_L* = **10.91**, residual
std=0.074. This is the cleanest fit of the three — the constant model captures MK3
almost perfectly. Expected profit: **~490/day → ~14,713 over 30 days.**

**What to change vs base:**
- Override `_fit_ewls()` to set `self.a = 0.0` always, fitting only the intercept `b`
  as the exponentially weighted mean of recent follower prices.
- Load only last **30 days** of history in `start_simulation()` — very recent prices are
  the best predictor of the near-constant follower.
- `decay=0.93` — shorter effective memory to track slow drift in follower's base price.
- `max_price=15.0` — keep (game rule constraint).

**Override:** `start_simulation()` and `_fit_ewls()`.

```python
class LeaderMK3(AdaptiveStackelbergLeader):
    def __init__(self, name, engine):
        super().__init__(name, engine, min_price=1.0, max_price=15.0, decay=0.93)

    def start_simulation(self):
        # Use only recent 30 days — follower near-constant, recent mean is best predictor
        for t in range(71, 101):
            lp, fp = self.get_price_from_date(t)
            self.u_L_hist.append(lp)
            self.u_F_hist.append(fp)
        self._fit_ewls()

    def _fit_ewls(self):
        # Treat follower as a constant (a=0); only track the slowly-drifting mean price
        xs, ys = self._filter_outliers(self.u_L_hist, self.u_F_hist)
        n = len(ys)
        if n < 1:
            return
        weights = [self.decay ** (n - 1 - i) for i in range(n)]
        sw  = sum(weights)
        swy = sum(w * y for w, y in zip(weights, ys))
        self.a = 0.0
        self.b = swy / sw  # exponentially weighted mean of follower prices
```

---

## Summary of differences

| | LeaderMK1 | LeaderMK2 | LeaderMK3 |
|---|---|---|---|
| History used | All 100 days | Last 50 days only | Last 30 days only |
| decay | 0.95 | 0.97 | 0.93 |
| max_price | 50 (tune up) | 50 | 15 |
| Fitted a | 1.700 | 1.071 | 0.000 (forced) |
| Fitted b | 0.471 | 0.989 | 1.346 |
| u_L* | 50 (at cap, B>0) | 29.3 | 10.91 |
| π*/day | ~5,217 | ~1,433 | ~490 |
| π* 30 days | ~156,500 | ~42,977 | ~14,713 |
| Model | Linear (slope + intercept) | Linear (slope + intercept) | Constant (intercept only, a=0) |
| Overrides | `__init__` only | `__init__` + `start_simulation` | `__init__` + `start_simulation` + `_fit_ewls` |
| Key risk | max_price is the lever — tune it | Slope p=0.14, explore early | None — very clean fit |
| Rationale | Slope drifts up, need aggressive decay | Early regime is wrong, discard it | Slope unreliable, near-constant follower |

---

## Verification
After implementing, test each leader with the functional test pattern:
```python
ldr = LeaderMK1('test', FakeEngine())
ldr.start_simulation()
print(ldr.a, ldr.b)              # check fitted model
print(ldr.new_price(101))        # check day-101 price is within [min, max]
print(ldr.expected_profit(ldr.new_price(101)))  # check profit is positive
ldr.end_simulation()
print(len(ldr.u_L_hist))         # should be 0 after reset
```
Then run the GUI simulation in Colab against each follower and compare accumulated profits.
