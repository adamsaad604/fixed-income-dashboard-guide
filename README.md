# Fixed income dashboard build guide

Build notes for an Excel dashboard that compares a bond portfolio with a Canadian universe bond benchmark, using Bloomberg data and Copilot in Excel.

## Files

| File | What it covers |
| --- | --- |
| [01-dashboard-spec.md](01-dashboard-spec.md) | What each module measures, the formulas, how to display it, and how to read it |
| [02-copilot-build-prompts.md](02-copilot-build-prompts.md) | 26 prompts that build the workbook one component at a time |
| [03-bloomberg-field-reference.md](03-bloomberg-field-reference.md) | Bloomberg fields, identifier formats and data-limit guidance |
| [04-benchmark-rules.md](04-benchmark-rules.md) | Index rules that affect the build, how an ETF proxy differs, and the swap plan |

## How to use

1. Read the "Read first" section of 02 before opening Excel.
2. Check the fields marked U in 03 on the terminal (FLDS) before prompt P5.
3. Run the prompts in order, and complete each check before sending the next prompt.
4. Use 01 to interpret the finished dashboard.

## Conventions

- Confidence tags: [Certain] means documented evidence, [Likely] means strong inference, and [Guessing] means filling a gap.
- P is the portfolio and B the benchmark. Weights are decimals, yields are in %, spreads in bp, and durations in years.
- The benchmark is an ETF proxy (XBB) until index constituent data is available; see 04.
