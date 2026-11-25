# Project Plan

This document is the source of truth for formulas, numerical fixtures, and completion criteria. The tutor uses it during each step. Independent exercises are described by topic here; their solutions are not stored in the repository.

Last updated: 2026-11-24.

## 1. Purpose

Build a market-making simulation from scratch, first in Python and then in C++. Each phase adds one idea: quote mechanics, accounting, inventory control, random prices and order arrivals, the Avellaneda–Stoikov closed form, evaluation, and a performance-minded port.

The measure of progress is whether you can explain the formula, type the implementation, and reproduce a fixed numerical result.

## 2. Decisions

These choices keep the course small enough to implement and check by hand.

- Python 3.11 or newer for Phases 1–6. C++20 for Phase 7.
- Phases 2 and 3 use the Python standard library. NumPy starts in Phase 4, through `numpy.random.Generator` with PCG64.
- Tests are `unittest` in Python and a small assert program in C++.
- Timing uses `std::chrono::steady_clock`. There is no target latency to beat.
- Core comparisons use absolute tolerance `1e-9`. Values such as `0.05` are not exact in binary floating point.
- The Avellaneda–Stoikov target is the symmetric closed form in section 7. A numerical Hamilton–Jacobi–Bellman solver is out of the core path.
- Fill size in every core fixture is 1.
- The simulator owns the random generators. A strategy is a pure function from state to quotes.
- NumPy PCG64 and `std::mt19937` do not produce the same stream. Cross-language tests use injected random numbers and the closed-form fixtures in this file.
- Formula tests compare unrounded prices. Tick size is part of the Phase 1 vocabulary. A tick-rounding function is not required.

## 3. Market vocabulary

A two-sided quote has a bid and an ask. The market maker is willing to buy at the bid and sell at the ask.

```text
mid = (bid + ask) / 2
spread = ask - bid
spread_in_ticks = spread / tick
```

For bid `99.95`, ask `100.05`, and tick `0.01`:

```text
mid = 100
spread = 0.10
spread_in_ticks = 10
```

A resting bid is a passive buy. If a sell order hits that bid, the market maker buys at the bid: inventory increases and cash decreases. A resting ask is a passive sell. If a buy order lifts that ask, the market maker sells at the ask: inventory decreases and cash increases.

A positive inventory is a long position. A negative inventory is a short position.

## 4. Canonical ledger

Starting state: cash `0`, inventory `0`, realized PnL `0`, mid `100`.

Round trip, used from the first accounting explanation:

| Step | Action | Cash | Inventory | Realized | Unrealized at mid 100 | Equity |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| 0 | start | 0 | 0 | 0 | 0 | 0 |
| 1 | buy 1 at 99.95 | -99.95 | 1 | 0 | 0.05 | 0.05 |
| 2 | sell 1 at 100.05 | 0.10 | 0 | 0.10 | 0 | 0.10 |

The captured spread is `0.10`.

Four-event ledger, used as the M1.3 and M2.4 fixture. Average cost is defined in section 5.

| Step | Action | Mid | Cash | Inventory | Average cost | Realized | Unrealized | Equity |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | start | 100 | 0 | 0 |  | 0 | 0 | 0 |
| 1 | buy 1 at 99.95 | 100 | -99.95 | 1 | 99.95 | 0 | 0.05 | 0.05 |
| 2 | buy 1 at 99.95 | 100 | -199.90 | 2 | 99.95 | 0 | 0.10 | 0.10 |
| 3 | sell 1 at 100.05 | 100 | -99.85 | 1 | 99.95 | 0.10 | 0.05 | 0.15 |
| 4 | buy 1 at 99.75 | 99.80 | -199.60 | 2 | 99.85 | 0.10 | -0.10 | 0.00 |

After step 4, `realized + unrealized = 0.00` and `cash + inventory * mid = 0.00`.

Through-zero fixture, required in M2.4:

| Step | Action | Mid | Cash | Inventory | Average cost | Realized | Unrealized | Equity |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | start | 100 | 0 | 0 |  | 0 | 0 | 0 |
| 1 | buy 1 at 100 | 100 | -100 | 1 | 100 | 0 | 0 | 0 |
| 2 | sell 2 at 102 | 101 | 104 | -1 | 102 | 2 | 1 | 3 |

The sell closes the long unit and opens one short unit at `102`.

## 5. Accounting rules

Maintain cash, inventory, average cost of the open position, and realized PnL.

On a buy of `n` units at price `p` with fee `f` per unit:

- If inventory is negative, close `min(n, -inventory)` units first. Realized PnL increases by `(average_cost - p) * closed_quantity`.
- Any remaining quantity opens or increases a long position and updates average cost.
- Cash decreases by `n * p + n * f`.
- Realized PnL decreases by `n * f` immediately.

On a sell of `n` units at price `p` with fee `f` per unit:

- If inventory is positive, close `min(n, inventory)` units first. Realized PnL increases by `(p - average_cost) * closed_quantity`.
- Any remaining quantity opens or increases a short position and updates average cost.
- Cash increases by `n * p - n * f`.
- Realized PnL decreases by `n * f` immediately.

Average cost is the average trade price of the remaining open position. Fees stay out of average cost. A flat position has no average cost.

```text
unrealized = inventory * (mid - average_cost)    when inventory is not 0
unrealized = 0                                   when inventory is 0
equity = cash + inventory * mid
```

With zero fees and a zero starting portfolio, `equity = realized + unrealized`. Fees are already inside realized PnL, so the same identity holds after fees.

Fee fixture, required in M6.3. Fee is `0.01` per unit. Buy 1 at `99.95`, then sell 1 at `100.05`, mid `100`:

```text
cash = 0.08
inventory = 0
realized = 0.08
unrealized = 0
equity = 0.08
```

The zero-fee round trip in section 4 is the same trades with fee `0` and equity `0.10`.

## 6. Execution rules

### 6.1 Deterministic cross

Used in Phases 2 and 3. One step moves the reference price from `p0` to `p1`. The quoted bid is `b` and the quoted ask is `a`, with `b < a`. A missing side cannot fill.

- If `p1 < p0` and `p1 <= b`, buy 1 at `b`.
- If `p1 > p0` and `p1 >= a`, sell 1 at `a`.
- Otherwise there is no fill.
- A jump across the whole spread fills the side in the direction of the move, once.

Fixture, with `b = 99.95` and `a = 100.05`:

| `p0` | `p1` | Fill |
| ---: | ---: | --- |
| 100 | 99.90 | buy 1 at 99.95 |
| 100 | 100.10 | sell 1 at 100.05 |
| 100 | 100.02 | no fill |
| 100.10 | 99.90 | buy 1 at 99.95 |

### 6.2 Intensity fill

Used from Phase 4 onward for stochastic execution. During a step of length `dt`, each quoted side draws its own uniform `U` from the arrival generator.

```text
probability = min(1, intensity * dt)
```

A quoted side fills one unit when `U < probability`. The fill price is the quoted price of that side.

Distance from the current mid determines intensity:

```text
bid_distance = mid - bid
ask_distance = ask - mid
intensity(distance) = A * exp(-k * distance)
```

A negative distance is allowed. The exponential is then larger than `A`, and the probability cap is `1`.

Injected fixture, independent of the optimal quote:

```text
A = 1
k = 1
distance = ln(2)
intensity = 0.5
dt = 0.1  -> probability = 0.05
dt = 3    -> probability = 1
```

Interarrival fixture for the inverse-transform explanation:

```text
U = 1 - exp(-2)
lambda = 2
interarrival = -ln(1 - U) / lambda = 1
```

### 6.3 Shared randomness

A comparison run builds one mid path from the path generator and one sequence of uniform draws from the arrival generator. Every strategy sees that same path and those same uniforms. Strategies differ only through the quotes they choose.

Phase 6 path settings, taken from the Avellaneda–Stoikov simulation section:

```text
s0 = 100
sigma = 2
T = 1
dt = 0.005
steps = 200
path_seed = 7
arrival_seed = 11
```

Two runs with these seeds produce the same metrics to tolerance `1e-12` on the same NumPy version. Record that NumPy version in `python/requirements.txt` when you create it.

## 7. Avellaneda–Stoikov closed form

Source: Avellaneda and Stoikov (2008), equations (29) and (30) in the journal text. The mid has no drift:

```text
dS = sigma * dW
```

The implementation step is the Euler increment

```text
dS = sigma * sqrt(dt) * Z
```

where `Z` is a standard normal draw.

Symbols:

| Symbol | Meaning |
| --- | --- |
| `s` | current mid |
| `q` | current inventory, positive when long |
| `gamma` | risk aversion, positive |
| `sigma` | arithmetic Brownian volatility |
| `T - t` | time remaining |
| `k` | decay of fill intensity with distance |
| `A` | intensity scale |
| `r` | reservation price, the center of the quote |
| `delta` | half-spread around the reservation price |

In this project `delta` is the half-spread around `r`. In the paper, the sum `delta_ask + delta_bid` in equation (30) is the full quoted width `2 * delta`.

```text
r = s - q * gamma * sigma^2 * (T - t)
delta = 0.5 * gamma * sigma^2 * (T - t) + (1 / gamma) * ln(1 + gamma / k)
bid = r - delta
ask = r + delta
```

Reading the formula:

- A long inventory lowers the reservation price, so both quotes move down and the ask becomes more attractive to a buyer.
- A short inventory raises the reservation price.
- The width depends on time remaining, volatility, risk aversion, and `k`. Under this exponential intensity, the width does not depend on `q`.
- At `q = 0`, the reservation price equals the mid, so the quote is symmetric around the mid.

The paper's simulation parameters are the canonical parameters:

```text
s = 100
sigma = 2
gamma = 0.1
k = 1.5
A = 140
T = 1
t = 0
```

Results at `t = 0`, within tolerance `1e-9`:

| Quantity | Value |
| --- | ---: |
| `delta` | 0.845385211376 |
| full spread `2 * delta` | 1.690770422751 |
| time-average full spread over `[0, T]` | 1.490770422751 |

The time average uses the fact that `T - t` averages to `T / 2`. The paper's Table 1 reports an average spread of `1.49` for `gamma = 0.1`. Matching `1.490770422751` before any simulation is the formula check.

Quotes at `t = 0`:

| `q` | Reservation | Bid | Ask | Full spread |
| ---: | ---: | ---: | ---: | ---: |
| 0 | 100 | 99.154614788624 | 100.845385211376 | 1.690770422751 |
| 2 | 99.2 | 98.354614788624 | 100.045385211376 | 1.690770422751 |

Distances from the mid at `q = 2`, `t = 0`:

```text
ask_distance = 0.045385211376
bid_distance = 1.645385211376
```

Intensity uses these mid distances, so the nearer ask is more likely to fill than the farther bid.

Paper-parameter intensity check at `q = 0`, `t = 0`:

```text
intensity per side = 39.392073093645
dt = 0.005
probability per side = 0.196960365468
```

Assumptions to be able to state before writing the strategy:

- The mid is an arithmetic Brownian motion without drift.
- The agent maximizes exponential utility of terminal wealth. That choice makes the reservation price independent of current cash.
- Inventory is risky because it is still held as the mid moves, and the horizon ends at `T`.
- Buy and sell arrivals are random with intensity `A exp(-k * distance)`.
- The quoted size is one unit. Queue position, cancellations, and a visible order book are outside the model.
- The original optimal spread has no fee. Fees are added later as an accounting stress test.

## 8. Phase 3 quote rule

The pre-Avellaneda inventory rule is linear:

```text
reservation = mid - eta * inventory
bid = reservation - half_spread
ask = reservation + half_spread
```

Teaching fixture:

```text
mid = 100
half_spread = 0.05
eta = 0.02
q = 2
reservation = 99.96
bid = 99.91
ask = 100.01
```

At `q = 0` this rule matches the fixed quote `99.95 / 100.05`. A long inventory lowers the bid relative to that fixed quote. The full spread stays `0.10`.

Position limit `L`:

- Quote a bid only when `inventory < L`.
- Quote an ask only when `inventory > -L`.
- At `q = 2` and `L = 2`, the bid is absent and the ask remains `100.01`.
- At `q = -2` and `L = 2`, the reservation is `100.04`, the ask is absent, and the bid is `99.99`.

Apply the limit after computing the skewed prices. A missing side cannot fill.

## 9. Metrics

All metric fixtures are hand-built series. A strategy backtest must call the same functions.

### 9.1 Sharpe ratio

Let `x1, ..., xN` be PnL increments and `N >= 2`. Use the sample standard deviation, with divisor `N - 1`.

```text
sharpe = mean(x) / sample_std(x) * sqrt(N)
```

This number describes the whole sample. It is not an annualized Sharpe ratio.

Fixture:

```text
increments = [1, 3, -1, 1]
mean = 1
sample_std = sqrt(8 / 3)
sharpe = sqrt(3 / 2) = 1.224744871392
```

If the sample standard deviation is `0`, or if `N < 2`, raise `ValueError`.

### 9.2 Maximum drawdown

Walk the equity series from left to right. Track the running peak. Drawdown at a point is `peak - equity`. Maximum drawdown is the largest of those values, in equity units.

```text
equity = [100, 110, 105, 120, 90]
maximum_drawdown = 30
```

A one-point series has maximum drawdown `0`. An empty series raises `ValueError`.

### 9.3 Inventory exposure

```text
inventory = [0, 1, 2, 1]
mean_absolute_inventory = 1
max_absolute_inventory = 2
```

### 9.4 Fill rate

```text
fill_rate = fills / arrivals
```

Fixture: `4 / 10 = 0.4`. Zero arrivals raises `ValueError`.

## 10. Strategy comparison

Phase 6 runs three quote rules on the shared path and shared uniforms from section 6.3.

1. Fixed spread. Center at the mid. Half-spread equals half of the time-average full spread `1.490770422751423`, which is `0.745385211375712`.
2. Inventory skew. Same half-spread, `eta = 0.40`, position limit `5`.
3. Avellaneda–Stoikov. Section 7, with the paper parameters and `gamma = 0.1` on the baseline run.

The Phase 2 half-spread of `0.05` stays the accounting example. It is too narrow for a volatility of `2`, so the comparison uses the wider matched half-spread above.

Gamma sensitivity uses the same seeds and `gamma` in `{0.05, 0.1, 0.2}`. For each value, record terminal equity, mean absolute inventory, and fill rate. A `k` sweep uses the same procedure and is optional.

The fee fixture in section 5 is required. The baseline horse race uses fee `0`.

## 11. Brownian motion checks

Injected standard normals, tolerance `1e-12`:

```text
sigma = 2
dt = 0.01
Z = 1
dS = 0.2
```

```text
sigma = 2
dt = 0.25
Z = [-1, 0, 1, 2]
dS = [-1, 0, 1, 2]
start = 100
path = [100, 99, 99, 100, 102]
```

One-step variance target is `sigma^2 * dt`. Horizon variance target of `S_T - S_0` is `sigma^2 * T`.

Monte Carlo check, separate from the injected fixture: 20,000 one-step increments from `numpy.random.Generator(PCG64(7))`. The sample variance of the increments is within 5 percent of `sigma^2 * dt`. Shrinking `dt` shrinks the one-step variance in proportion to `dt`. The horizon variance stays near `sigma^2 * T`.

Same seed and same algorithm produce the same path on a second run.

## 12. Future files

Create a file only when the milestone names it.

| File | First milestone |
| --- | --- |
| `python/quotes.py` | M2.1 |
| `python/test_quotes.py` | M2.1 |
| `python/accounting.py` | M2.4 |
| `python/test_accounting.py` | M2.4 |
| `python/strategies.py` | M3.1 |
| `python/test_strategies.py` | M3.1 |
| `python/simulation.py` | M4.1 |
| `python/test_simulation.py` | M4.1 |
| `python/requirements.txt` | M4.1 |
| `python/backtest.py` | M6.1 |
| `python/test_backtest.py` | M6.2 |
| `cpp/CMakeLists.txt` | M7.1 |
| `cpp/src/` and `cpp/tests/` | M7.1 |

Run Python tests from the repository root:

```text
python -m unittest discover -s python -v
```

## 13. Milestones

A milestone is complete when its check passes on your machine, you can explain the idea without reading the formula sheet, and you confirm it. Review milestones also include the exercise topic and the interview questions. The tutor does not reveal exercise solutions in advance.

### Phase 1. Market-making fundamentals

Prerequisite: you run `python --version` and see Python 3.11 or newer. No project file is created in this phase.

#### M1.1 Bid, ask, mid, spread, and tick

Compute mid, spread, and spread in ticks for a quoted market. Use the `99.95 / 100.05` quote in section 3.

Done when you can define bid, ask, mid, spread, and tick, and can compute all three numbers from a new quote.

#### M1.2 Passive execution, cash, and inventory

Apply one buy hit and one sell lift to a blank portfolio. Use the sign convention in section 3 and the first two filled rows of the round trip in section 4.

Done when a buy increases inventory and decreases cash, a sell does the opposite, and both prices are the quoted prices.

#### M1.3 Realized PnL, unrealized PnL, and equity

Build the four-event ledger in section 4 by hand, including the mark at `99.80`.

Done when equity equals `cash + inventory * mid` on every row, and realized plus unrealized equals equity.

Exercise topic: a new bid, ask, and mark of your own, with two buys and one sell, checked against the same identity.

Interview questions:

- Why is the mid different from both the bid and the ask?
- Who is the passive side when a sell order hits a resting bid?
- When does a gain become realized PnL?

### Phase 2. Fixed-spread market maker

#### M2.1 Quote helpers

Create `python/quotes.py` and `python/test_quotes.py`. Implement mid and spread.

Done when the section 3 fixture passes within `1e-9`.

#### M2.2 Fixed-spread quotes

Quote `reference - half_spread` and `reference + half_spread`. Teaching half-spread: `0.05`. Reference `100` produces bid `99.95` and ask `100.05`.

Done when that quote passes and a crossed quote is rejected by a test you wrote.

#### M2.3 Deterministic cross fill

Implement section 6.1, including the four-row fixture.

Done when each row produces the listed fill and a flat price produces no fill.

#### M2.4 Average-cost accounting

Create `python/accounting.py` and `python/test_accounting.py`. Implement section 5 with fee `0`. Pass the round trip, the four-event ledger, and the through-zero fixture.

Done when every row matches cash, inventory, average cost, realized PnL, unrealized PnL, and equity within `1e-9`.

#### M2.5 Fixed-spread market maker review

Review M2.1–M2.4 on one short deterministic price path that you choose and compute by hand before running it.

Exercise topic: extend the quote helper with spread in ticks, and reject a nonpositive tick.

Interview questions:

- What is the difference between a maker fill and a taker fill?
- Why can a fixed-spread market maker accumulate inventory?
- Which identity should every accounting test check after a fill?

### Phase 3. Inventory-based market making

#### M3.1 Linear inventory skew

Add the section 8 rule to `python/strategies.py`. At `q = 0`, it matches the fixed quote. At `q = 2`, the fixture quote is `99.91 / 100.01`.

Done when both quotes pass and the full spread remains `0.10`.

#### M3.2 Position limits

Apply limit `L = 2` from section 8.

Done when `q = 2` suppresses the bid, `q = -2` suppresses the ask, and a suppressed side cannot fill.

#### M3.3 Shared-path comparison

Run the fixed rule and the skewed rule on one hand-written price path. Use the cross rule in section 6.1.

Done when the two equity paths are equal at `q = 0`, the bids differ once inventory is long, and both paths still satisfy the equity identity.

#### M3.4 Inventory-based market maker review

Exercise topic: change `eta` and predict, before running code, whether a long inventory makes the bid lower or higher.

Interview questions:

- Why does a positive inventory move both quotes down?
- What can still go wrong if quotes are skewed and there is no position limit?
- How can a test show that a position limit is binding?

### Phase 4. Stochastic simulation

#### M4.1 Seeded random walk

Create `python/simulation.py`, `python/test_simulation.py`, and `python/requirements.txt`. Use `numpy.random.Generator(numpy.random.PCG64(seed))`.

Done when two runs with the same seed return identical values, and two different seeds are allowed to differ.

#### M4.2 Arithmetic Brownian motion

Implement `dS = sigma * sqrt(dt) * Z` and the injected paths in section 11.

Done when the injected paths match within `1e-12`, a second seeded run matches the first, and the 20,000-step variance check is within 5 percent of `sigma^2 * dt`.

#### M4.3 Poisson arrivals

Implement the inverse-transform interarrival and the capped probability `min(1, intensity * dt)`. Pass the fixtures in section 6.2.

Done when the injected uniform produces interarrival `1`, and the two probabilities are `0.05` and `1`.

#### M4.4 Intensity-based execution

Fill a resting quote with section 6.2. Use distance from the mid and an injected pair of uniforms so the test does not depend on a random draw.

Done when a uniform below the probability fills at the quoted price, a uniform above it does not fill, and a missing side does not fill.

#### M4.5 Stochastic simulation review

Exercise topic: given `sigma` and `dt`, predict the one-step variance before writing the assertion.

Interview questions:

- Why is the Brownian increment scaled by `sqrt(dt)`?
- What does order-arrival intensity mean for a resting quote?
- Why must the price path and the arrival draws be fixed before two strategies are compared?

### Phase 5. Avellaneda–Stoikov

#### M5.1 Model assumptions

Write the assumption list from section 7 in your own notes before adding the strategy. No new formula code in this milestone.

Done when you can explain reservation price, exponential utility, and intensity without looking at the sheet.

#### M5.2 Reservation price

Implement `r` for the paper parameters.

Done when `q = 0` gives `100`, `q = 2` gives `99.2`, and a positive inventory is below the mid.

#### M5.3 Optimal spread and quotes

Implement `delta`, bid, and ask. Pass the quote table in section 7, including equal full spreads at `q = 0` and `q = 2`.

Done when the time-average full spread is `1.490770422751` within `1e-9`.

#### M5.4 Fill probability

Connect section 7 quotes to section 6.2. Pass the `q = 0` probability `0.196960365468` and the `q = 2` mid distances.

Done when the ask distance is smaller than the bid distance for a long inventory, and both probabilities are capped at `1`.

#### M5.5 Single-path simulator

Simulate one path with `path_seed = 7`, `arrival_seed = 11`, and the section 6.3 clock. The strategy function only returns quotes.

Done when a second run reproduces cash, inventory, and equity to `1e-12`, and equity matches the accounting identity on every step.

#### M5.6 Avellaneda–Stoikov review

Exercise topic: change only `q` and predict the new reservation price and the unchanged full spread before running the function.

Interview questions:

- What does the reservation price represent?
- What happens to the reservation price when inventory is positive and `gamma` increases?
- Which two model assumptions are false in a real limit-order book?

### Phase 6. Backtesting and strategy evaluation

#### M6.1 Common backtest loop

Create `python/backtest.py`. Run the three section 10 strategies on one shared path and one shared uniform sequence.

Done when the mids and uniforms are identical across strategies, and the quotes differ on a step where inventories differ.

#### M6.2 Performance metrics

Create `python/test_backtest.py`. Implement section 9.

Done when Sharpe, maximum drawdown, inventory exposure, and fill rate match the hand-built fixtures, and the error cases raise `ValueError`.

#### M6.3 Fees and gamma sensitivity

Pass the fee fixture in section 5. Run `gamma` in `{0.05, 0.1, 0.2}` on seeds `7` and `11`. Record terminal equity, mean absolute inventory, and fill rate.

Done when the fee ledger matches `0.08` and the three gamma rows are reproducible on a second run.

#### M6.4 Written limitations

Write a short note in the learning log, after confirmation, covering fees, the one-unit Poisson fill model, continuous prices versus a tick grid, missing queue position, and the instability of Sharpe on a 200-step path.

Done when the note states what the horse race can and cannot support.

#### M6.5 Strategy evaluation review

Exercise topic: construct a two-point equity path with a higher terminal value and a worse maximum drawdown than another path.

Interview questions:

- How can a higher terminal PnL come with a worse drawdown?
- Why is Sharpe on a short simulated path a weak ranking rule?
- How does a per-unit fee change the cash result of a round trip that captured the spread?

### Phase 7. C++ implementation

Port the validated behavior. Keep the Python fixtures as the specification.

#### M7.1 C++ accounting and fixed spread

Create the C++20 project and port mid, spread, fixed quotes, and section 5 accounting. Use the standard library containers you actually need.

Done when the section 3 quote, the four-event ledger, and the through-zero fixture match the Python results within `1e-9`.

#### M7.2 C++ RNG and Avellaneda–Stoikov quotes

Use `<random>` for a seeded engine in the C++ simulator. Port reservation price, half-spread, and quotes. Cross-check the quote table with injected inputs, not by matching NumPy's random stream.

Done when the section 7 quote table matches within `1e-9`.

#### M7.3 Shared deterministic tests

Port the Sharpe, drawdown, inventory, and fill-rate fixtures.

Done when section 9 matches, including the `ValueError` equivalents you choose and document.

#### M7.4 Quote timing

Time a large number of Avellaneda–Stoikov quote evaluations with `std::chrono::steady_clock`. Report elapsed time per call. Keep the timed region on the quote calculation.

Done when you can say what the measurement includes and the quote fixture still passes.

#### M7.5 C++ and interview review

Exercise topic: port the through-zero average-cost update without looking at the Python implementation, then compare the numeric result.

Interview questions:

- Why is a seeded generator part of correctness as well as speed?
- What would you measure before trying to make the quote function faster?
- Which value in the C++ port owns the price path?

## 14. Project completion

The lab is complete when all 33 milestones are `confirmed` in [LEARNING_LOG.md](LEARNING_LOG.md), the Python and C++ deterministic fixtures agree, and you can derive the reservation price and half-spread from the symbols in section 7 without notes.

## 15. Reading

Primary:

- Avellaneda, M., and Stoikov, S. (2008). High-frequency trading in a limit order book. *Quantitative Finance*, 8(3), 217–224. <https://doi.org/10.1080/14697680701381228>

Use equations (29) and (30), and the simulation parameters in section 3.3 of that paper. A readable PDF is hosted with the NYU Avellaneda memorial page: <https://math.nyu.edu/inmemoriam/avellaneda/HighFrequencyTrading.pdf>. The DOI is the citation of record.

Supporting, in the order they become useful:

- Harris, L. (2003). *Trading and Exchanges: Market Microstructure for Practitioners*. Oxford University Press. Phase 1 vocabulary and market structure.
- Ho, T., and Stoll, H. R. (1981). Optimal dealer pricing under transactions and return uncertainty. *Journal of Financial Economics*, 9(1), 47–73. <https://doi.org/10.1016/0304-405X(81)90020-9>
- Glosten, L. R., and Milgrom, P. R. (1985). Bid, ask and transaction prices in a specialist market with heterogeneously informed traders. *Journal of Financial Economics*, 14(1), 71–100. <https://doi.org/10.1016/0304-405X(85)90044-3>
- Guéant, O., Lehalle, C.-A., and Fernandez-Tapia, J. (2013). Dealing with the inventory risk: a solution to the market making problem. *Mathematics and Financial Economics*, 7(4), 477–507. <https://doi.org/10.1007/s11579-012-0087-0>. Preprint: <https://arxiv.org/abs/1105.3115>
- Cartea, Á., Jaimungal, S., and Penalva, J. (2015). *Algorithmic and High-Frequency Trading*. Cambridge University Press. The limit-order-book and market-making chapters.

Optional later reading. It is not required to finish the lab:

- Cont, R., Stoikov, S., and Talreja, R. (2010). A stochastic model for order book dynamics. *Operations Research*, 58(3), 549–563. <https://doi.org/10.1287/opre.1090.0780>

Implementation references:

- Python standard library and `unittest`: <https://docs.python.org/3/>
- NumPy Generator: <https://numpy.org/doc/stable/reference/random/generator.html>
- C++ `<random>`: <https://en.cppreference.com/w/cpp/numeric/random>
- C++ `<chrono>`: <https://en.cppreference.com/w/cpp/chrono>

## 16. Out of scope

The course is a single-process simulator of one inventory and one mid price. The following work is outside it:

- live orders, exchange APIs, and broker connectivity
- a web interface, database, cloud service, or microservice split
- reconstruction of a full limit-order book, queue position, and cancellations
- numerical solution of the Hamilton–Jacobi–Bellman equation
- machine-learning execution, options pricing, and portfolio optimization
- a requirement to match NumPy and C++ random streams bit for bit
