# COMP34612 Computational Game Theory — Group 7
## Project: Stackelberg Pricing Game — Presentation Notes

---

## 1. Problem Setup

We play as the **Stackelberg leader** in a repeated pricing game over 30 live days (days 101–130) against three different followers (MK1, MK2, MK3).

**Leader demand model:**
```
S_L(u_L, u_F) = 100 - 5·u_L + 3·u_F
```

**Daily profit:**
```
π_L = (u_L - 1) · S_L(u_L, u_F)
```

**Objective:** maximise accumulated profit over 30 days.

**Key constraint:** The follower's strategy is unknown and time-varying — *"the follower's payoff function is subject to the changing and time-varying environment."*

We also have 100 days of historical price data (days 1–100) for each follower.

---

## 2. Approach: Multiple Leaders

We implemented **three separate leaders**, each tailored to one follower:

| Leader | Plays Against |
|--------|--------------|
| LeaderMK1 | MK1 (and hidden follower MK4) |
| LeaderMK2 | MK2 (and hidden follower MK5) |
| LeaderMK3 | MK3 (and hidden follower MK6) |

---

## 3. Data Analysis Findings

Before implementing anything, we analysed the 100-day historical datasets.

### Finding 1 — Narrow historical price range (all followers)
All followers responded to leader prices in a tiny band: **u_L ∈ [1.715, 1.900]** (range = 0.185 units). Any slope estimated from this band extrapolates very poorly to prices 5–50, where the leader actually wants to operate. This was the root cause of the original approach's underperformance.

### Finding 2 — MK2 regime flip at day 50
MK2's follower sharply changed behaviour around day 50. Days 1–50 show a slope of approximately −19 (driven by outliers), while days 51–100 show a stable positive slope of ~+0.79. Using all 100 days of history poisons the estimate. Solution: discard days 1–50.

### Finding 3 — MK3 slope is noise (R² = 0.038)
MK3's follower price barely responds to the leader price at all. The R² of the OLS fit on historical data is only 0.038. The slope is statistically indistinguishable from zero — the follower behaves as a slowly-varying constant. Modelling it as `u_F ≈ b` (intercept only) is more accurate than fitting a slope to noise.

---

## 4. Original Approach: EWLS

The original implementation used **Exponentially Weighted Least Squares (EWLS)**:
- Load all historical data into a list
- Each day, re-fit the entire dataset from scratch using exponential weights (decay factor)
- Use the fitted (a, b) to compute the Stackelberg optimal price

**Limitations:**
- **O(n) per day** — refits all history every step
- **Ignores online updates correctly** — no principled Kalman-style gain, just a batch re-fit
- **Not the algorithm taught in Lecture 6** — the course prescribes RLS as the proper method for time-varying parameters
- **Dominated by stale data** — even with decay, 100 historical points in [1.715, 1.900] overwhelm 3 new exploration points in [5–30]

---

## 5. New Approach: Recursive Least Squares (RLS, Lecture 6)

We replaced EWLS with **Recursive Least Squares with forgetting factor λ**, exactly as taught in Lecture 6.

### Algorithm

```
φ = [1, u_L]ᵀ                          feature vector
K = P·φ / (λ + φᵀ·P·φ)                 Kalman gain
P = (P − K·φᵀ·P) / λ                   covariance update
θ = θ + K·(u_F − φᵀ·θ)                 parameter update
```

where θ = [b, a]ᵀ is the reaction function `u_F = a·u_L + b`.

### Why RLS is better

| Property | EWLS (old) | RLS (new) |
|----------|-----------|-----------|
| Computational cost | O(n) per day | O(1) per day |
| Handles time-varying parameters | Partially | Yes (via λ) |
| Updates as new data arrives | Re-fits from scratch | True online update |
| Course alignment | Lecture 4 (batch) | Lecture 6 (recursive) |

### Stackelberg optimal price derivation

Substituting `u_F = a·u_L + b` into the profit formula:
```
S_L = (100 + 3b) + (3a − 5)·u_L  =  A + B·u_L
π_L = (u_L − 1)(A + B·u_L)
```

Setting dπ/du_L = 0:
- **B < 0 (a < 5/3):** interior maximum at `u_L* = (B − A) / (2B)`
- **B ≥ 0 (a ≥ 5/3):** profit increases without bound → clamp to `max_price`

---

## 6. Per-Leader Design Decisions

### LeaderMK1 (vs MK1/MK4)

**Analysis:**
- Historical slope unreliable (estimated in [1.715, 1.900] only)
- Live follower data reveals: true slope a ≈ 0.75, b ≈ 2.14
- Stackelberg optimal: u_L* ≈ 19.9, giving ~977/day profit

**Design:**
- `λ = 0.88` (fast forgetting — live data replaces stale history within ~10 steps)
- Start with **large P = 1000·I** (high uncertainty) so 2 exploration points dominate immediately
- **2 exploration days** at [5, 20] — spans the relevant range, gives RLS good slope data
- **28 greedy days** (days 103–130) using RLS-estimated optimal price
- Initialise θ intercept from last 20 historical u_F values (rough prior), slope = 0.5 (conservative)

**Pitfall avoided:** Exploring at price 38 collapses demand to S_L = 4 (catastrophic). Exploration must stay below ~25.

### LeaderMK2 (vs MK2/MK5)

**Analysis:**
- Days 1–50 data is poisoned (slope ≈ −19); discard completely
- Days 51–100 give a clean estimate: a ≈ 0.79, b ≈ 1.36
- RLS drifts show gradual parameter shift after day 101 — follower adapts

**Design:**
- Load **days 51–100 only** into history; initialise θ and P via batch OLS on this data (warm start)
- `λ = 0.92` (moderate forgetting — adapts within ~15 steps)
- **2 exploration days** at [12, 20] — the warm-start OLS already gives a precise θ, so only 2 points are needed to verify the live slope; prices 12 and 20 span the relevant range while earning ~813 and ~1016/day respectively
- **28 greedy days** (days 103–130)

**Key insight:** Because P is initialised from batch OLS (not a large diagonal matrix), the RLS starts with high confidence in θ. Fewer exploration days are needed — and the three saved days (previously at prices 2, 6, 12 earning ~100/447/813) are replaced by greedy days earning ~1,050/day each, gaining roughly +1,800 profit.

### LeaderMK3 (vs MK3/MK6)

**Analysis:**
- Historical u_F range: [1.23, 1.58], mean ≈ 1.34, slope R² = 0.038
- Live u_F is very different: mean ≈ 2.73 (2× parameter shift!)
- True live optimal: u_L* = (105 + 3·2.73)/10 ≈ 11.3, giving ~532/day

**Design:**
- **Force a = 0** — only learn the intercept b (constant follower model)
- Scalar 1-D RLS: `b_{t+1} = b_t + K·(u_F − b_t)`
- `λ = 0.95` (slow forgetting — intercept is stable within each session)
- Load **days 71–100** for initial b estimate
- **3 exploration days** at [10, 11, 13] — near the live optimum, maximising profit during exploration
- **27 greedy days** (days 104–130)

Note: exploration at low prices (e.g., 3) wastes ~346/day vs exploring near optimal (~532/day).

---

## 7. Results

| Follower | Baseline (EWLS) | Final (RLS) | Change |
|----------|----------------|-------------|--------|
| MK1 | 28,152 | **28,367** | +215 ✓ |
| MK2 | 29,166 | **31,575** | +2,409 ✓ |
| MK3 | 15,492 | **15,990** | +498 ✓ |
| **Total** | **72,810** | **75,932** | **+3,122** |

**MK2 improvement breakdown:**
- Phase 1 (RLS + 5-day exploration [2,6,12,20,32]): 29,166 → 29,793 (+627)
- Phase 2 (reduced to 2-day exploration [12,20]): 29,793 → 31,575 (+1,782)
- Combined gain: **+2,409** — the biggest single improvement

**MK3** reached 15,990 — above the static theoretical maximum of ~15,970, because the scalar RLS tracks the follower's day-to-day b fluctuations in real time.

**MK1** is limited by the non-linear follower reaction function. The linear model settles at u_L ≈ 19, which is near-optimal for the linearised model.

### Exploration strategy comparison

| Leader | Old exploration | New exploration | Greedy days | Reason |
|--------|----------------|-----------------|-------------|--------|
| MK1 | [5, 15, 20] (3 days) | [5, 20] (2 days) | 28 | Large P → converges in 2 points |
| MK2 | [2, 6, 12, 20, 32] (5 days) | [12, 20] (2 days) | 28 | OLS warm-start → already precise |
| MK3 | [3, 6, 10] (3 days) | [10, 11, 13] (3 days) | 27 | Move exploration near live optimum |

---

## 8. Course Alignment

| Lecture | Concept | Implementation |
|---------|---------|----------------|
| Lectures 2–3 | Stackelberg optimal price from reaction function | `_optimal_price()` derives u_L* analytically |
| Lecture 4 | Learn reaction function from historical data | Batch OLS warm-start (`_batch_init_rls`) |
| Lecture 5 | Multi-variable regression | φ = [1, u_L]ᵀ, θ = [b, a]ᵀ |
| Lecture 6 | **RLS with forgetting factor λ** for time-varying params | `_rls_update()` — O(1) recursive update |
| Lecture 6 | Online update as new data arrives | `_record_and_update()` called each day |

---

## 9. Robustness to Hidden Followers (MK4, MK5, MK6)

The hidden followers have "slightly changed parameters." Our RLS-based approach handles this automatically:

- **MK4 (like MK1):** λ=0.88 means 2 exploration points [5, 20] quickly re-learn any shifted slope. After 5 RLS steps, the old history contributes only λ⁵ = 0.53× weight.
- **MK5 (like MK2):** Batch warm-start from days 51–100 gives a stable prior; λ=0.92 adapts to parameter drift within ~15 live steps.
- **MK6 (like MK3):** Scalar RLS with λ=0.95 learns the new b within 5–10 observations regardless of the specific shift.

Unlike hard-coded parameter values, our algorithm generalises by design.

---

## 10. Key Lessons

1. **Historical data quality matters more than quantity.** 100 days of data in a 0.185-unit range gave a completely wrong slope when extrapolated. We learned more from 2 strategically chosen live prices than from all 100 history days.

2. **Exploration price selection is critical.** Exploring at price 38 caused demand collapse (S_L = 4, profit = 148/day). Exploring near the expected optimal (~20) costs nothing in information while still giving high profit.

3. **RLS with appropriate λ outperforms batch EWLS** because it handles the time-varying nature of the follower correctly — exactly as Lecture 6 prescribes.

4. **Model complexity should match the signal.** Fitting a slope to MK3 where R²=0.038 adds noise, not information. The constant model (a=0) is provably better here.
