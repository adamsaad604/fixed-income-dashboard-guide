# Portfolio vs benchmark dashboard: specification

Ten modules and a data-checks strip compare a bond portfolio (P) with its benchmark (B). For each module this covers what it measures, the formulas, the best way to display it, and how to read it against a portfolio view. Confidence tags: [Certain] means documented evidence, [Likely] strong inference, and [Guessing] filling a gap.

## Read this first

Three things decide whether this dashboard can be trusted: one source for analytics on both sides, a daily archive, and knowing which views the ETF proxy distorts.

- **One analytics source.** Pull duration, spread, yield, rating, maturity and ultimate parent for both the portfolio and the benchmark holdings from Bloomberg by ISIN. Every active number is a difference, so mixing analytics from different systems turns part of the difference into a gap between models. [Likely] The index provider prices from a different source. Once index data arrives, use it for weights and classification only (see 04). [Likely]
- **Archive every day.** With point-in-time holdings and no transaction history, each refresh overwrites yesterday. Save each day's summary metrics, and ideally the full holdings, to a history archive. Without it, Module 10 doesn't exist and you can't separate trades from drift. [Certain]
- **What the ETF proxy distorts.** XBB is a fund, not the index: it holds cash, its weights are based on net asset value, and its holdings count differs from the index's. [Certain] Aggregates by sector, term and rating, and the headline numbers, are usable now. [Likely] Security-level active weights and active share are not, because a bond you hold that the ETF skips shows up as a full overweight. Work at issuer and bucket level until index data is live. [Likely]
- **Normalize both sides.** Strip out cash, futures and other non-bond lines, rescale bond weights to 100%, and report cash on its own line. Otherwise cash differences leak into every active weight. [Certain]
- **Use consistent definitions.** Use effective (option-adjusted) duration for callables and amortizers, and modified duration otherwise. Use one spread measure on both sides; the Canadian convention is the spread to the GoC curve. [Likely] Use yield to worst for callables, and bucket terms by effective maturity (Module 2).
- **Cache static fields.** Rating, maturity, sector and parent rarely change, so store them as values and re-pull them weekly or when a new ISIN appears. Refresh only the price-driven fields daily. [Likely]

## Notation and core formulas

Every number on the dashboard is one of four things: a weight, a weighted average, a contribution, or an active difference between portfolio and benchmark.

| Symbol | Meaning |
| --- | --- |
| i, j, k | holding, issuer, bucket (sector, term, rating) |
| P, B | portfolio, benchmark |
| MV | dirty market value (clean price plus accrued, times par) |
| D | duration: effective for callables and amortizers, modified otherwise (years) |
| SD | spread duration (years) |
| s | spread to the GoC curve (bp) |
| y | yield: YTW for callables, YTM otherwise |
| C | convexity |
| h | horizon (years) |

**Weights and active weights**

```math
w_i = \frac{MV_i}{\sum_j MV_j} \qquad a_i = w_i^P - w_i^B \qquad A_k = \sum_{i \in k} w_i^P - \sum_{i \in k} w_i^B
```

**Portfolio-level analytics** are market-value-weighted averages of any measure X:

```math
\bar{X} = \sum_i w_i X_i
```

**Contributions** to duration, spread duration and DTS:

```math
\mathrm{CTD}_i = w_i D_i \qquad \mathrm{CSD}_i = w_i \, SD_i \qquad \mathrm{CDTS}_i = w_i \, SD_i \, s_i
```

**Active contribution** of a bucket k:

```math
\Delta\mathrm{CTD}_k = \sum_{i \in k} w_i^P D_i - \sum_{i \in k} w_i^B D_i
```

The active contributions across buckets add up to active duration (D^P − D^B), and the same identity holds for spread duration and DTS. Every bucket table should therefore tie to its headline active number, which gives you a built-in reconciliation check. [Certain]

**Why these measures.** To first order, the return relative to the benchmark breaks down into carry, rates and spread:

```math
R^P - R^B \approx (\bar{y}^P - \bar{y}^B)\,h \;-\; \sum_t \Delta\mathrm{CTD}_t \, \Delta r_t \;-\; \sum_k \Delta\mathrm{CSD}_k \, \Delta s_k \;+\; \tfrac{1}{2}(\bar{C}^P - \bar{C}^B)(\Delta r)^2
```

Here Δr_t is the GoC yield change at term t, and Δs_k is the spread change in bucket k. Each module isolates one term of this equation. Keep weights in decimals so that CTD is in years. With spreads in bp, DTS comes out in year·bp; only ratios and differences matter.

## Dashboard layout

Read the page from top to bottom: whether to trust the data, how much risk you run, where it sits, what it would do in a market move, and how it got there.

| Zone | Left | Right |
| --- | --- | --- |
| Trust the data | Data checks strip (full width) | |
| How much risk | M1 Headline risk strip (full width) | |
| Where it sits | M2 Duration by term | M3 Sector, three lenses |
| Where it sits | M4 Sector × term heatmap | M5 Credit quality |
| Where it sits | M6 Top issuer overweights and underweights | M7 Relative value scatter |
| What it does | M8 Scenario table | M9 Off-benchmark sleeve |
| How it moved | M10 History and implied trades (full width) | |

M1 to M4 cover most positioning decisions, so build those first. M7, M9 and M10 can wait until the daily archive has a few weeks of history.

## Module 1 — Headline risk strip

This block answers how far you are from the benchmark, and on which kind of risk: carry, rates, credit or concentration.

**Display: a compact table, not KPI tiles.** Use three columns (P, B, Active), with conditional colour on the Active column only. Tiles show one number each and hide the benchmark comparison, which is the whole point. Group the rows by risk type.

| Group | Metric | Formula |
| --- | --- | --- |
| Carry | Yield | Σ w · y |
| Carry | Duration-weighted yield | see below |
| Rates | Duration | Σ w · D |
| Rates | Convexity | Σ w · C |
| Credit | Spread (bp) | Σ w · s |
| Credit | Spread duration | Σ w · SD |
| Credit | DTS | Σ w · SD · s |
| Credit | DTS ratio | DTS^P / DTS^B |
| Credit | Breakeven spread (bp) | Σ w · s / Σ w · SD (Module 5) |
| Concentration | Issuers / effective issuers | count; 1 / HHI (Module 6) |

```math
\bar{y}_D = \frac{\sum_i w_i D_i y_i}{\sum_i w_i D_i} \qquad \mathrm{DTS\ ratio} = \frac{\sum_i w_i^P SD_i s_i}{\sum_i w_i^B SD_i s_i}
```

Show both yields. The market-value-weighted yield is the reporting convention, but the duration-weighted yield is closer to the portfolio's true internal rate of return when the curve is steep. [Likely]

**Reading it**

- **Active duration is your level view on rates.** For example, +0.5 years of active duration earns roughly +0.25% relative if GoC yields fall 50 bp in parallel, and loses the same if they rise.
- **Active yield with the DTS ratio near 1.0** means you pick up yield without adding spread beta, through security selection, curve position or off-benchmark holdings. That's the cleanest source of excess return.
- **Active yield with the DTS ratio above about 1.2** means you're being paid for taking spread risk. Intended or not, that's a bet on spreads tightening.
- **Positive active yield with negative active duration** is the classic trade of credit for duration. It wins if rates rise or stay flat and spreads hold.
- **Negative active convexity at similar duration** points to callables, MBS, or a bullet book against a barbell benchmark. You're short volatility.
- **Average rating** is for reference only. A linear average of rating notches understates the tail risk of a BBB tilt, so judge credit risk by DTS.

## Module 2 — Duration and curve positioning

Total active duration is your level view; where it sits along the curve is your curve view, and that usually matters more day to day.

**Buckets.** Use 1–3, 3–5, 5–7, 7–10, 10–20 and 20+ years for the chart, plus a bucket under one year so the totals tie. Roll these up to the index's Short (1–5), Mid (5–10) and Long (10+) for the headline table. Bucket by effective maturity: the call date for callables that trade to call, the reset date for fixed-to-floating bonds, WAL for amortizers, and final maturity otherwise. The index buckets option bonds by effective term, so final maturity won't reconcile. [Certain]

```math
\Delta\mathrm{CTD}_t = \sum_{i \in t} \left(w_i^P - w_i^B\right) D_i \qquad \sum_t \Delta\mathrm{CTD}_t = \bar{D}^P - \bar{D}^B
```

**Barbell versus bullet.** Compare how widely durations spread around the mean on each side:

```math
\sigma_D = \sqrt{\sum_i w_i \left(D_i - \bar{D}\right)^2}
```

If the portfolio's σ_D is higher than the benchmark's at a similar duration, you're running a barbell: more convexity, and usually less yield on an upward-sloping curve. [Likely]

**Upgrade when ready: key rate durations.** Grouping duration contribution by maturity is only a proxy, because it puts all of a 30-year bond's duration in the long bucket. Active key rate durations (KRDs) at 2, 5, 10 and 30 years from Bloomberg, for both sides, are the accurate version. KRDs sum to approximately effective duration. [Certain]

**Display: columns about zero, with maturity on the x-axis from short to long.** Keeping the curve reading left to right is why vertical columns beat horizontal bars here. Put the active duration in the title. Add a small companion chart showing portfolio and benchmark CTD side by side per bucket, so the absolute profile is visible.

**Reading it**

- **All buckets positive** is a parallel long: a level view that yields fall, for example because the BoC cuts more than is priced.
- **Short negative, long positive, net near zero** is a flattener. It pays if long yields fall relative to short ones.
- **Short or belly positive, long negative** is a steepener. It pays if BoC easing pulls the front end down or term premium rebuilds at the long end.
- **Belly overweight against both wings** is a butterfly: a view that the belly is cheap on the curve.
- **Check where long-end duration comes from.** A provincial overweight usually loads the 10–20 and 20+ buckets, because provincial supply is long-dated. [Likely] That turns a sector decision into a curve bet. Cross-check in Module 4.
- **Benchmark drift.** The index rebalances daily: new issues enter on or after their issue date, and bonds leave when their remaining effective term reaches one year. [Certain] New long issues extend benchmark duration, so your active duration can drift lower without any trades on your side. Module 10 separates that drift from your own trading.

## Module 3 — Sector allocation through three lenses

Show each sector's active position three ways (weight, duration contribution and DTS). The three routinely disagree, and the disagreement is the insight.

**Rows, in a fixed order:** Federal, Federal SSA (sovereign, supranational and foreign agency, part of Federal since January 2025) [Certain], Provincial, Municipal, then Corporate split into Financial, Communication, Energy, Industrial, Infrastructure, Real Estate and Securitization, then Off-benchmark. A fixed row order keeps the visual stable from day to day.

```math
A_k = W_k^P - W_k^B \qquad \Delta\mathrm{CTD}_k = \sum_{i \in k} a_i D_i \qquad \Delta\mathrm{DTS}_k = \sum_{i \in k} a_i \, SD_i \, s_i
```

**Display: one table with three diverging data-bar columns.** The columns are P weight, B weight, then active weight, active CTD and active DTS, each with bars centred on zero. Three aligned in-cell bar columns beat three separate charts, because the rows can't drift out of alignment.

**Reading it: the disagreements**

- **Overweight corporates by weight, but neutral or underweight by DTS.** You hold short, high-quality corporates. That's long carry, not long spread beta: a defensive credit stance that looks like an overweight.
- **Underweight a sector by weight but overweight by CTD.** You hold fewer but longer bonds in that sector, which is a curve bet hiding in the sector table.
- **A provincial overweight with large positive CTD.** You're long provincial spread and long-end duration together. The view is that provincial spreads tighten or the long end rallies. The risk is long-dated provincial supply widening spreads. [Likely]
- **A Federal underweight funding a corporate overweight.** This is the classic risk-on trade: long spread and short liquidity. In a stress it hurts twice, because you miss the flight-to-quality rally while spreads widen (see Module 8's risk-off row).
- **A Financials overweight.** This is a Canadian bank spread view. Check the issuer module for concentration in a few names.
- **A large Off-benchmark row.** This is pure active risk with no benchmark offset; see Module 9.

## Module 4 — Sector × term heatmap

This is the most information-dense view on the page. It shows active duration contribution by sector (rows) and term (columns), so you can see exactly where each duration bet sits.

```math
\Delta\mathrm{CTD}_{k,t} = \sum_{i \in k \cap t} \left(w_i^P - w_i^B\right) D_i
```

Row totals equal Module 3's sector CTD, column totals equal Module 2's term CTD, and the grand total equals active duration. The grid checks itself. [Certain]

**Display: a heatmap table.** Use a diverging colour scale centred on zero, with symmetric limits so that +0.10 and −0.10 years look equally strong. Print values in the cells to two decimals of a year. Keep the totals row and column outside the colour scale. Add one dropdown to switch the measure between active weight, active CTD and active DTS on the same grid.

**Reading it**

- **One hot cell** (say Provincial × 10–20 years) is a concentrated, deliberate bet. It should map to a written thesis.
- **A whole row in the same colour** is a sector view. **A whole column in the same colour** is a curve view.
- **A diagonal pattern**, such as short corporates overweight and long Federal underweight, trades credit for duration. It earns spread and shortens duration; the net view is spreads stable and rates higher.
- **A random checkerboard of small cells** is usually legacy positions or noise. Those are candidates for clean-up trades that cut tracking error without changing the view.
- **A pale grid everywhere** means the book is close to the index. Then ask whether the active risk budget is being used at all.

## Module 5 — Credit quality

Credit risk is measured in DTS, not weight. This module shows how much spread risk you run against the benchmark, and how much cushion the spread gives you.

**Buckets:** AAA, AA, A, BBB+/BBB, BBB− on its own, then below investment grade or not rated (off-benchmark).

**Rating rule.** Use the index's rule on both sides. From DBRS, S&P, Moody's and Fitch, take the lower of two ratings, the middle of three, or the middle of the three lowest of four. [Certain] The Bloomberg composite blends the agencies differently, so it won't reconcile. The index itself buckets only AAA/AA, A and BBB; the separate BBB− bucket here is for your own view of the downgrade cliff.

**DTS works because spread volatility scales with spread level** (Dor et al., *DTS: Duration Times Spread*, 2007). With proportional spread moves, the excess return is:

```math
R_i \approx -SD_i \, \Delta s_i = -\left(SD_i \, s_i\right) \frac{\Delta s_i}{s_i} = -\mathrm{DTS}_i \cdot x
```

**Breakeven spread widening** is how many bp spreads can widen over a year before the loss wipes out a year of spread carry (ignoring roll-down):

```math
\mathrm{BE}_i = \frac{s_i}{SD_i} \qquad \overline{\mathrm{BE}} = \frac{\sum_i w_i s_i}{\sum_i w_i \, SD_i}
```

**Display:** diverging bars by rating bucket, with active DTS as the main bar and active weight as a lighter companion bar. Beside the chart, show two numbers for P against B: the DTS ratio and the breakeven spread. Avoid 100% stacked bars of the two rating mixes, because small differences disappear in them.

**Reading it**

- **A DTS ratio above 1.0** means you're long spread beta. At 1.2, a 10% proportional widening costs about 1.2× the benchmark's loss. That's a view that spreads tighten, or at least that carry outweighs the volatility.
- **A BBB overweight by weight with a DTS ratio below 1.0** means you hold short-dated BBBs: carry without much beta. That's a reasonable late-cycle stance.
- **Moving up in quality** (AAA/AA overweight, BBB underweight, DTS ratio well below 1.0) is defensive: a view that spreads widen or recession risk is rising. The cost shows as negative active yield in Module 1.
- **A breakeven spread above the benchmark's** means more carry cushion per unit of spread risk, which supports holding through modest widening.
- **The BBB− weight is your downgrade cliff.** A fall to high yield takes a bond out of an investment-grade index, so you'd likely be selling into a crowd of forced sellers. [Likely]

## Module 6 — Issuer concentration

The top overweights and underweights by issuer show where idiosyncratic risk sits; the concentration metrics show how much of it there is.

**Issuer key:** Bloomberg's ultimate parent ticker on both sides, not the accounting system's names, so that every entity of a bank or province rolls up together. [Likely]

```math
A_j = \sum_{i \in j} a_i \qquad \Delta\mathrm{DTS}_j = \sum_{i \in j} a_i \, SD_i \, s_i
```

```math
\mathrm{HHI} = \sum_j W_j^2 \qquad N_{\mathrm{eff}} = \frac{1}{\mathrm{HHI}} \qquad \mathrm{AS} = \tfrac{1}{2}\sum_j \left| W_j^P - W_j^B \right|
```

Compute HHI and the effective number of issuers (N_eff) on the corporate sleeve, rescaled to 100%; otherwise the Government of Canada, as a single issuer, swamps the measure. While the benchmark is an ETF, compute active share (AS) at issuer level, because security-level active share overstates it. [Likely]

**Display: two ranked tables side by side.** Show the top 10 overweights and the top 10 underweights, each ranked by active DTS rather than active weight. The columns are issuer, sector, P weight, B weight, active weight, active DTS (with diverging data bars) and average spread. Put N_eff and AS for P against B as two numbers above the tables. Rank by DTS because a 1% overweight in a 30-year BBB carries far more risk than a 1% overweight in a 2-year AA.

**Reading it**

- **Each top overweight should map to a credit view,** such as an issuer model or a one-page thesis. An overweight with no thesis is a legacy or liquidity position, so decide whether to keep it.
- **Underweights are positions too.** A large underweight in a big index issuer, such as one of the Big 6 banks or a large province, is effectively a short. It pays if that issuer's spreads widen and costs if they tighten.
- **N_eff below the benchmark's** means a more concentrated book and bigger single-name event risk.
- **Active share trending up over time** means the book is drifting away from the benchmark. Judge it against your own history rather than a fixed threshold.

## Module 7 — Relative value scatter

A scatter of spread against duration for every bond shows whether your holdings sit cheap or rich to their sector curve.

**Fit a curve per peer group** (Provincial, Corporate A, Corporate BBB, or by subsector) through the benchmark holdings. A log fit is the simple standard for credit curves [Likely]:

```math
\hat{s}_i = \hat{\alpha}_g + \hat{\beta}_g \ln D_i \qquad \varepsilon_i = s_i - \hat{s}_i
```

**Roll the residuals up.** The portfolio-weighted residual is the average spread you earn above (or give up below) fair value on the curve; the active-weighted version is what your positioning adds:

```math
\bar{\varepsilon}^P = \sum_i w_i^P \varepsilon_i \qquad \bar{\varepsilon}^{\mathrm{act}} = \sum_i a_i \, \varepsilon_i
```

**Display: a scatter chart,** with duration (years) on the x-axis and spread (bp) on the y-axis. Show benchmark bonds as small grey dots, portfolio holdings as accent-coloured dots sized by portfolio weight, and the fitted curve as a line. Show one peer group at a time, picked from a dropdown; all sectors on one chart is unreadable. Excel's logarithmic trendline gives you the fit directly.

**Reading it**

- **Portfolio dots above the curve** mean you own cheap bonds. Ask why before calling it alpha: poor liquidity (small or off-the-run issues), structure (callables), or an issuer the market is worried about. Cheap isn't automatically good.
- **Dots below the curve** mean you're paying up, usually for liquidity in benchmark issues, or because a holding has richened since you bought it. The latter are candidates to switch into cheaper bonds of similar duration.
- **A large residual concentrated in one issuer** means the market is pricing a name-specific risk. Cross-check it against your credit model.
- **Scope:** this is the quick in-Excel read. Spline fits and PCA belong in a dedicated tool, not this workbook.

## Module 8 — Scenario table

This module turns the risk numbers into estimated relative return under the moves you expect or fear. It's where a mismatch between your view and your positioning becomes obvious.

**Rates leg.** Interpolate each scenario's shocks at 2, 5, 10 and 30 years to the midpoint of each term bucket:

```math
\Delta R_{\mathrm{rates}} \approx -\sum_t \Delta\mathrm{CTD}_t \, \Delta r_t + \tfrac{1}{2}\left(\bar{C}^P - \bar{C}^B\right)\left(\overline{\Delta r}\right)^2
```

**Spread leg,** absolute or proportional (DTS):

```math
\Delta R_{\mathrm{spread}} \approx -\sum_k \Delta\mathrm{CSD}_k \, \Delta s_k \quad \text{or} \quad -\left(\mathrm{DTS}^P - \mathrm{DTS}^B\right) x
```

**Carry leg** over the horizon:

```math
\Delta R_{\mathrm{carry}} \approx \left(\bar{y}^P - \bar{y}^B\right) h
```

The Copilot build leaves out the convexity term until CNVX_MID's scaling is confirmed (see 03).

**Starting scenarios** (illustrative; set your own):

| Scenario | 2y | 5y | 10y | 30y | Spreads |
| --- | --- | --- | --- | --- | --- |
| Parallel rally | −50 | −50 | −50 | −50 | unchanged |
| Parallel sell-off | +50 | +50 | +50 | +50 | unchanged |
| Bull steepener (BoC cutting) | −50 | −40 | −25 | −10 | unchanged |
| Bear steepener (term premium) | 0 | +10 | +25 | +40 | unchanged |
| Bear flattener (hawkish BoC) | +40 | +25 | +10 | 0 | unchanged |
| Corporate widening | 0 | 0 | 0 | 0 | corporates +25 bp |
| Provincial widening | 0 | 0 | 0 | 0 | provincials +10 bp |
| Proportional widening | 0 | 0 | 0 | 0 | all spreads ×1.25 |
| Risk-off combined | −40 | −40 | −30 | −20 | all spreads ×1.30 |

**Display: a table,** with scenarios as rows and columns for rates, spread, carry (3 months) and total active return (bp). Put conditional colour on the total only, and sort the rows by total so the worst outcome sits at the bottom. A table works better than a chart here, because you're comparing a handful of named scenarios, each built from several components.

**Reading it**

- **The worst row should be the outcome you think least likely.** If the scenario that hurts most is your base case, the positioning contradicts the view.
- **Compare the rally and sell-off rows.** Unequal outcomes mean convexity is working for or against you.
- **Check the risk-off row.** A Federal underweight plus a corporate overweight loses on both legs: spreads widen while you miss the GoC rally.
- **Carry breakeven:** total loss in the adverse scenario ÷ (active yield / 12) = the months of carry needed to recover. If that's longer than your conviction horizon, the position is too large.

## Module 9 — Off-benchmark sleeve

Holdings outside the index are pure active risk with no benchmark offset, so give them their own line. They include private placements that fail the size or buyer tests, high-yield or unrated bonds, monthly-pay MBS and non-CAD bonds.

**Flag eligibility by rule, not by ETF membership.** The ETF doesn't hold every eligible bond, so "not in XBB" overstates what's off-benchmark. Use the index rules in 04: CAD-denominated, a fixed semi-annual coupon, an effective term over one year, at least $100 million issued, an index rating of BBB or better, and at least 10 institutional buyers at issue. Private issues that pass those tests are eligible. [Certain]

```math
W_{\mathrm{off}} = \sum_{i \notin \mathcal{U}} w_i^P \qquad \mathrm{Share}_{\mathrm{DTS}} = \frac{\sum_{i \notin \mathcal{U}} w_i^P SD_i s_i}{\mathrm{DTS}^P}
```

Here U is the set of index-eligible bonds.

**Private placements need manual analytics.** Bloomberg won't have clean data for most of them. Price them off the matching public curve (same rating and duration) plus an illiquidity premium, and compute duration from the cash flows. Mark their analytics source as "manual" on the dashboard. Some may be index-eligible, so check the buyer test by hand.

**Display: a small table,** with rows for private placements, high yield or not rated, MBS and other monthly-pay bonds, non-CAD bonds and cash. The columns are weight, CTD, DTS, yield, spread, share of portfolio DTS, and analytics source (Bloomberg or manual).

**Reading it**

- **Yield pickup against DTS:** are you being paid for the illiquidity? Compare each group's spread with the public curve at the same rating and duration.
- **A rising off-benchmark weight** means tracking error is rising quietly, even if no single trade looked large.
- **Liquidity drift:** in a redemption or a stress, you sell the liquid holdings first, so the off-benchmark weight rises without anyone deciding it should. Watch it in Module 10's history.

## Module 10 — Positioning history and implied trades

Daily point-in-time snapshots become a history only if you save them. Once saved, they separate your trades from market drift and benchmark changes, which you can't do any other way without transaction data.

**History table:** one row per date × metric × value. Long format means new metrics don't need new columns. Store the Module 1 strip for P, B and Active, along with sector active weights, term ΔCTD and the DTS ratio.

**Implied trades from par changes,** if the accounting export carries par or face amount:

```math
\Delta\mathrm{Par}_{i,t} = \mathrm{Par}_{i,t} - \mathrm{Par}_{i,t-1}
```

A non-zero change is a buy or a sell, except for maturities, calls, sinking funds and paydowns on amortizers. Tag those using the maturity date and pool factor before calling anything a trade.

**Split each weight change into trading and drift,** where P is the dirty price per unit of par:

```math
\Delta w_i = \underbrace{\frac{\Delta\mathrm{Par}_i \cdot P_{i,t}}{MV_t}}_{\text{trading}} + \underbrace{\left(\Delta w_i - \frac{\Delta\mathrm{Par}_i \cdot P_{i,t}}{MV_t}\right)}_{\text{drift}}
```

**Split active duration changes by side:**

```math
\Delta\left(\bar{D}^P - \bar{D}^B\right) = \underbrace{\Delta\bar{D}^P}_{\text{trades, aging, yield moves}} - \underbrace{\Delta\bar{D}^B}_{\text{rebalancing, aging, yield moves}}
```

**Display:**

- Line charts over time: active duration (years), the DTS ratio with a reference line at 1.0, and active yield. Month-end markers are a reporting convention only, because the index rebalances daily.
- An implied-trade table, newest first: date, ISIN, issuer, change in par, estimated trade value, sector and term bucket.

**Reading it**

- **Active duration drifting lower with no trades** is usually the benchmark extending as new long issues enter. Decide whether that's intended or needs a top-up trade.
- **The DTS ratio rising while corporate weight is flat** means spreads widened. Because DTS scales with spread level, your risk rose without a trade. That's how DTS is meant to behave, not an error.
- **Jumps on trade dates should move the dashboard in the direction you intended.** If a "steepener" trade didn't change Module 2's shape, check the sizing.

## View-consistency check

Write the view down first, then check that the dashboard shows its signature and nothing else large. The biggest numbers on the page should be the ones your view chose.

| View | Expected signature | Where to look | Unintended bet to rule out |
| --- | --- | --- | --- |
| Yields fall (BoC cuts more than priced) | Active duration > 0; positive ΔCTD in Mid and Long | M1, M2; M8 parallel rally row positive | Duration coming from a provincial overweight rather than a choice |
| Bull steepener | Positive ΔCTD at 1–7y, negative at 10y+, net near zero | M2, M4; M8 bull steepener row | Net duration not actually neutral, so a level bet rides along |
| Bear flattener | Negative ΔCTD at the front end, positive at the long end | M2; M8 bear flattener row | Long-end exposure concentrated in one sector |
| Spreads tighten / risk-on | DTS ratio > 1; corporates overweight in DTS, funded from Federal | M1, M3, M5; M8 spread rows | Liquidity: a Federal underweight is hard to reverse in stress |
| Defensive / late cycle | DTS ratio < 1; up in quality; Federal and Provincial overweight | M5; M8 risk-off row | Giving up so much carry that active yield is deeply negative |
| Provincials cheap to corporates | Provincial overweight in DTS, funded from corporates of similar duration | M3, M4, M7 | Long-end duration riding along with the provincial overweight |
| Single-issuer view | Issuer among the top overweights or underweights by DTS | M6, M7 | The same parent showing up through several entities |
| Carry, range-bound rates | Positive active yield, active duration near zero, DTS ratio near 1 | M1; M8 carry column | Yield pickup coming from illiquid off-benchmark holdings (M9) |

**The three-lens rule.** Before accepting any position as intended, confirm it in weight, duration contribution and DTS. A position that shows up in only one lens is usually a side effect of another decision.

## Data integrity checks

Put these on the dashboard itself, in a strip at the top. A silent data failure looks exactly like a position.

- **As-of dates.** Show the date of the accounting export, the benchmark holdings date and the Bloomberg pull timestamp. Flag any mismatch.
- **Weights sum.** Portfolio and benchmark weights should each sum to 100% after stripping out cash.
- **Identity ties.** The ΔCTD and ΔDTS for every bucket table (term, sector, rating, and the heatmap's rows and columns) should sum to the headline active number. A break means a holding is unmapped or double-counted.
- **Missing analytics.** Count holdings on each side with a blank duration, spread or rating, and list them. Private placements belong in the manual-override table, not here.
- **Unmapped holdings.** Count holdings whose sector, parent ticker or rating doesn't match the mapping tables.
- **Duplicate ISINs.** Accounting exports often split holdings into separate lots, so aggregate them before weighting.
- **Holdings count.** Compare today's benchmark line count with yesterday's to catch a truncated file.
- **Unexplained moves.** Flag any day where an active metric moves more than a threshold (say 0.05 years of duration) with no implied trades. It's either a market move worth knowing about or a data error.
