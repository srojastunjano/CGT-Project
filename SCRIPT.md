# Group 7 — Presentation Script
## COMP34612 Computational Game Theory | May 2026 | 10 minutes

> **Timing note:** At a natural confident pace (~160 words/min) this script runs 10:00–10:15.
> If you run over in rehearsal, cut the last sentence of slide 11.
> Stage directions are in *[italics]* — do not read them aloud.

---

## Timing overview

| Slide | Title | Speaker | Target |
|-------|-------|---------|--------|
| 1 | Title | Person 1 | 15s |
| 2 | The Problem | Person 1 | 70s |
| 3 | Data Analysis: Three Key Findings | Person 1 | 75s |
| 4 | Stackelberg Optimal Price | Person 2 | 65s |
| 5 | RLS Algorithm | Person 2 | 75s |
| 6 | RLS Tracking Chart | Person 2 | 50s |
| 7 | Code Architecture | Person 3 | 40s |
| 8 | LeaderMK1 | Person 3 | 55s |
| 9 | LeaderMK2 | Person 3 | 55s |
| 10 | LeaderMK3 | Person 3 | 55s |
| 11 | The Journey | Person 4 | 35s |
| 12 | Final Results | Person 4 | 45s |
| 13 | Course Integration & Novel Contributions | Person 4 | 60s |
| 14 | References | Person 4 | 5s |

---

## Slide 1 — Title `[Person 1]` `15s`

Good morning. We are Group 7, and our presentation is on adaptive Stackelberg pricing using Recursive Least Squares. I will start by setting up the problem.

---

## Slide 2 — The Problem `[Person 1]` `70s`

In this game we are the Stackelberg leader. We set a price first, the follower observes it, and then responds with their own price. We have one hundred days of historical data showing how three followers — MK1, MK2, and MK3 — have behaved in the past, and then thirty live trading days, from day 101 to 130, to maximise our accumulated profit. There are also three hidden followers — MK4, MK5, and MK6 — which are parameter-shifted variants of the known ones that we never observe directly.

Our demand model is S_L equals 100 minus 5 u_L plus 3 u_F, so our daily profit is u_L minus 1 times S_L. MK3 caps our price at 15; for MK1 and MK2 there is no upper bound. The core challenge is that the follower is a black box — its strategy is unknown and time-varying. This is the classical 1934 von Stackelberg setup: the leader commits first, and the follower best-responds. Our task is to price optimally without ever knowing the follower's true strategy.

---

## Slide 3 — Data Analysis: Three Key Findings `[Person 1]` `75s`

Before writing any code, we ran a systematic analysis of the hundred-day history. Three findings came out of that, and they directly determined every design decision that follows.

Finding one: the narrow history band. All three followers only responded to leader prices in a range of 0.185 units — u_L between 1.715 and 1.9. Any OLS slope fitted in that tiny range extrapolates very badly to the live prices of 10, 20, or higher where we actually want to operate. This is the root cause of the original EWLS underperformance.

Finding two: MK2 day-36 contamination. On day 36, the MK2 follower priced at 51.8 — more than three standard deviations above every other observation. That single data point is enough to drag the full-history OLS slope to minus 19, making it completely unusable. We cannot use days 1 to 50 for MK2 at all.

Finding three: MK3's slope R-squared is 0.038 — statistically indistinguishable from zero. Fitting a slope to MK3 adds noise, not information. The correct model is a constant, and the statistical evidence is unambiguous.

---

## Slide 4 — Stackelberg Optimal Price `[Person 2]` `65s`

*[Person 2 steps forward]*

With the follower's reaction function estimated as u_F equals a times u_L plus b, we can derive the leader's optimal price analytically. This follows directly from Lectures 2 and 3.

We substitute into the demand model to get S_L equals A plus B times u_L, where A is 100 plus 3b and B is 3a minus 5. Multiplying out, our daily profit is u_L minus 1 times A plus B u_L — a quadratic in u_L. Setting the derivative to zero gives the interior optimum: u_L star equals B minus A over 2B. *[point to formula on slide]* But this only holds when B is negative — that is, when the slope a is less than five thirds. When B is non-negative, profit is increasing in price and we clamp to the maximum allowed price.

The key point is that we only need accurate estimates of a and b to price optimally every day. That is exactly what the RLS is going to deliver.

---

## Slide 5 — RLS Algorithm `[Person 2]` `75s`

Lecture 6 gives us Recursive Least Squares with a forgetting factor lambda — the principled online estimator for time-varying parameters.

The algorithm updates four quantities on every live trading day. *[point to each equation in turn]* The feature vector phi is [1, u_L] transposed — the two-parameter regression from Lecture 5. The Kalman gain K equals P phi divided by lambda plus phi-transpose P phi — it tells us how much weight to put on the new observation relative to our current belief. The covariance matrix P is then updated and divided by lambda, which keeps us open to revision as parameters change. And finally, theta — our estimates of b and a — is nudged by the gain times the prediction error.

Why not EWLS? EWLS re-fits the entire history on every trading day — that is O(n) per step, and it is fundamentally a Lecture 4 batch method. RLS is O(1): one matrix multiply per day, designed specifically for online tracking. The forgetting factor lambda is the principled mechanism for handling time-varying parameters. At lambda equals 0.88, data ten steps old carries only 0.88 to the power of 10 — twenty-eight percent — of today's weight.

---

## Slide 6 — RLS Tracking Chart `[Person 2]` `50s`

This chart gives us visual evidence for every design choice we made. *[gesture to the 3×3 grid]*

In the top row, MK1's slope drifts steadily from 0.7 to 1.9 across 100 days. That directly justifies lambda equals 0.88 — we need fast forgetting so that live observations rapidly dominate over stale history.

In the middle row, the day-36 spike in MK2 is clearly visible as a sharp corruption in both b and a. That validates our decision to discard days 1 to 50 entirely before warm-starting.

And in the bottom row, MK3's slope hovers near zero throughout all 100 days. That confirms the constant model is correct, and that an OLS warm-start is needed to identify the live intercept.

---

## Slide 7 — Code Architecture `[Person 3]` `40s`

*[Person 3 steps forward]*

I will walk through the code structure and then each leader in turn.

We built a mixin class called _RLSMixin that holds all shared RLS logic — the four Lecture 6 equations written exactly once. *[gesture to the bottom box]* It provides six methods: initialise, update, compute optimal price, decide next price, record and update, and batch warm-start. Each of the three leader classes inherits from both the base Leader class and this mixin, and only overrides its own lambda, its historical data window, and its exploration pool. There is no code duplication — every leader runs the identical Lecture 6 update step.

---

## Slide 8 — LeaderMK1: vs MK1 & MK4 `[Person 3]` `55s`

LeaderMK1 faces two problems. First, the narrow history band makes OLS slope estimates unreliable at live prices. Second, the original implementation explored at price 38, collapsing demand to near zero and earning just 148 profit per day.

Our solution: we set lambda to 0.88, so after 10 live observations old history carries only 28% weight. Starting with P equals 1000 times the identity gives a trace of 2,000, which immediately exceeds our exploration threshold of 50. The single exploration price is 19 — just one unit from the Stackelberg optimum — so even the exploration day earns around 975. Then 29 greedy days.

For MK4: if the historical price spread exceeds 0.5, the OLS warm-start runs and exploration auto-skips. Lambda of 0.88 then re-learns any shifted MK4 slope within just one or two live observations.

Result: 28,570 — a gain of 418 over the baseline.

---

## Slide 9 — LeaderMK2: vs MK2 & MK5 `[Person 3]` `55s`

LeaderMK2's problem is the day-36 spike. u_F of 51.8 drags the full-history OLS slope to minus 19. The original five-day exploration included three days at u_L equals 2, earning around 100 per day — roughly 2,850 in wasted profit.

Our solution: discard days 1 to 50 entirely. An OLS warm-start from the clean days 51 to 100 brings the trace of P below 0.5, which is our exploration threshold, so exploration never fires and all 30 days are greedy. We also clip the residual at plus or minus 5 before each Kalman update — Huber-style protection. A future spike of 48 units is capped; theta shifts by at most 5 divided by the norm of the Kalman gain. Lambda of 0.92 re-learns any MK5 drift within about 15 live steps.

Result: 31,714 — a gain of 2,548. The biggest single improvement in the project.

---

## Slide 10 — LeaderMK3: vs MK3 & MK6 `[Person 3]` `55s`

LeaderMK3 has a slope R-squared of 0.038 — statistically zero. But the live intercept b doubled from 1.34 historically to 2.73 during live play, and the old exploration at price 3 earned only 186 per day.

Our solution: OLS warm-start on days 71 to 100 gives theta of approximately [0.09, 0.70] and a covariance trace of 90.7 — above our threshold of 80. All three exploration prices — 10, 11, and 13 — fire in sequence, each earning around 530 per day near the live optimum. After those three exploration days the trace drops below 80 and we go greedy. The price formula u_L star equals 105 plus 3b all over 10 then updates every day as the RLS tracks b in real time. Any MK6 shift in b is re-learned within three to five observations and the formula adjusts automatically.

Result: 16,008 — above the static theoretical maximum of 15,970, because RLS tracks real-time b fluctuations that a fixed formula simply cannot.

---

## Slide 11 — The Journey: Iterative Improvements `[Person 4]` `35s`

*[Person 4 steps forward]*

Every row in this table represents a diagnosed failure followed by a targeted fix. *[gesture to the table]* We started at 72,810 with the EWLS baseline. Each step — switching to RLS, correcting the exploration prices, detecting the MK2 contamination, adding the warm-start — was motivated by measuring what went wrong and isolating the cause. The biggest lesson we took from this whole process: the largest single gain came from data preprocessing, not from the algorithm change itself. Final total: 76,292.

---

## Slide 12 — Final Results `[Person 4]` `45s`

Our final numbers: MK1 at 28,570, a gain of 418. MK2 at 31,714, a gain of 2,548. MK3 at 16,008, a gain of 516. Total 76,292 — a 4.8% improvement over the EWLS baseline.

Three things worth highlighting. First, MK2's gain came primarily from contamination detection and preprocessing, not from the RLS update itself. Second, MK3 exceeded the static theoretical maximum — online tracking of the intercept in real time genuinely outperforms any fixed pricing formula. And third, the lambda-based forgetting factor means our approach generalises to MK4, MK5, and MK6 automatically — shifted parameters are re-learned without any hard-coding.

---

## Slide 13 — Course Integration & Novel Contributions `[Person 4]` `60s`

On the left, every lecture from 2 to 6 is directly implemented in our code. Lectures 2 and 3 give us the Stackelberg formula, called on every greedy day. Lecture 4 is our batch OLS warm-start. Lecture 5 is the two-parameter feature vector and parameter design. And Lecture 6 is the RLS update itself, O(1) per step.

On the right are four contributions that go beyond the course material, each grounded in a published paper. Huber residual clipping from 1964 — without it the MK2 spike shifts our slope by 46 units; capped at 5, it is harmless. Information-directed exploration from Pronzato and Müller 2012 — we only explore while trace of P exceeds a threshold, not on a fixed schedule. OLS warm-start from Söderström and Stoica — initialising P from X-transpose-X inverse rather than a heuristic prior. And contamination detection from Rousseeuw and Leroy — z-score analysis to identify and discard poisoned history windows.

---

## Slide 14 — References `[Person 4]` `5s`

References are on screen — happy to take any questions.

---

## Q&A preparation

| Question | Who answers | What to say |
|----------|-------------|-------------|
| Why λ=0.88 specifically for MK1? | Person 3 | 0.88 to the power 10 is 0.28 — ten live steps is enough to flush the narrow-band history. Tuned empirically by trying values from 0.85 to 0.95 and measuring profit. |
| How does the clip threshold affect convergence? | Person 2 or 3 | Clipping bounds the innovation so theta never jumps more than clip divided by the norm of K per step. Convergence is slightly slower but the estimator is protected against outliers. |
| What if MK4/5/6 are very different from MK1/2/3? | Any | The forgetting factor means we re-learn within roughly 1 over 1 minus lambda steps. For lambda 0.88 that is about 8 steps; for lambda 0.95 about 20 steps. The approach is parameter-agnostic by design. |
| Why discard days 1–50 for MK2 rather than down-weight them? | Person 1 or 3 | Down-weighting still includes the spike in the weighted history. Discarding is cleaner and justified by the regime change at day 50 — the two halves are genuinely different distributions. |
| How did MK3 exceed the theoretical maximum? | Person 3 or 4 | The static maximum assumes a fixed follower price. Live b fluctuates slightly above its historical mean on some days. RLS tracks those fluctuations and prices above what any fixed formula would choose. |
