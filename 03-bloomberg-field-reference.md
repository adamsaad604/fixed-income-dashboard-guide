# Bloomberg field reference

Every field the build uses, whether it has been confirmed against a public source, and how to stay inside data limits. Check each mnemonic in FLDS before building, starting with the ones marked U.

Status: **V** means confirmed in a public source; **U** means unverified, so check it in FLDS. Even V fields deserve a look in FLDS for the exact definition (bid, mid or ask, pricing source, workout date).

## Identifiers

- The prompts call BDP with the `/isin/` prefix plus the ISIN. [Likely; U] If a bond returns Invalid Security, test the ISIN followed by `Govt` or `Corp` on one bond, then fix the formula.
- If the benchmark pull returns Bloomberg IDs instead of ISINs, add `ID_ISIN` (V) to it.
- Use one pricing source consistently on both sides. Check which source your terminal defaults to for these bonds. [Likely]

## Static fields (pulled once, stored as values)

| Data | Mnemonic | Status | Notes |
| --- | --- | --- | --- |
| Description | `SECURITY_DES` | V | |
| Issuer | `ISSUER` | U | |
| Ultimate parent | `ULT_PARENT_TICKER_EXCHANGE` | U | Keep manual overrides for Canada Housing Trust, crown corporations and bank holding companies |
| BICS sector | `BICS_LEVEL_1_SECTOR_NAME` | V | |
| BICS industry | `BICS_LEVEL_3_INDUSTRY_NAME` | V | Bonds may return only two BICS levels; use level 2 if level 3 is blank |
| Country of incorporation | `CNTRY_OF_INCORPORATION` | U | The index uses incorporation, not country of risk, for Maple bonds |
| Currency | `CRNCY` | U | |
| Coupon | `CPN` | V | |
| Coupon type | `CPN_TYP` | V | |
| Coupon frequency | `CPN_FREQ` | U | Monthly pay means index-ineligible |
| Maturity | `MATURITY` | V | |
| Maturity type | `MTY_TYP` | V | Callable and sinkable flag; the public source is weak |
| Next call date | `NXT_CALL_DT` | U | |
| Amount issued | `AMT_ISSUED` | U | The index size test counts the original issue plus reopenings |
| Amount outstanding | `AMT_OUTSTANDING` | V | Refresh weekly |
| S&P rating | `RTG_SP` | V | Refresh weekly |
| Moody's rating | `RTG_MOODY` | V | Refresh weekly |
| DBRS rating | `RTG_DBRS` | U | Refresh weekly; check Bloomberg's DBRS format |
| Fitch rating | `RTG_FITCH` | U | Refresh weekly |
| Bloomberg composite rating | `BB_COMPOSITE` | V | Not the index's rule; reference only |

## Daily fields (live on the BBG sheet)

| Data | Mnemonic | Status | Notes |
| --- | --- | --- | --- |
| Clean price (mid) | `PX_MID` | V | |
| Accrued interest | `INT_ACC` | U | Dirty price = PX_MID + INT_ACC |
| Yield to maturity | `YLD_YTM_MID` | V | |
| Yield to worst | `YAS_BOND_YLD` | V field, U behaviour | Follows the YAS workout setting; confirm it returns yield to worst for callables |
| Modified duration | `DUR_ADJ_MID` | V | |
| Effective (OAS) duration | `OAS_EFF_DUR` | U | Highest-priority FLDS check |
| Convexity | `CNVX_MID` | U | Check the scaling; conventions differ by a factor of 100 |
| Spread to GoC | `YAS_YLD_SPREAD` | V | Depends on the YAS benchmark setting; confirm it's the interpolated GoC curve, not one bond. If it isn't, compute yield minus the interpolated GoC yield at effective maturity yourself |
| Spread duration | `OAS_SPREAD_DUR_MID` | U | Fallback: modified duration for fixed-rate bullets |
| Weighted average life | `MTG_WAL` | U | Amortizers only |
| Pool factor | `MTG_FACTOR` | V | MBS and ABS only |

## Optional fields

| Data | Mnemonic | Status |
| --- | --- | --- |
| DV01 | `RISK_MID` | V |
| I-spread | `YAS_ISPREAD` | U |
| Z-spread | `Z_SPRD_MID` | U |
| OAS | `YAS_OAS_SPRD` | V |
| Asset-swap spread | `YAS_ASW_SPREAD` | V |
| Key rate durations | `KEY_RATE_DUR_2YR` and similar | U |

## BQL notes (if you move pulls to BQL)

- A query is built from `let()`, `get()`, `for()`, `with()` and `preferences()` clauses; `get` and `for` are required. [Certain]
- Fund holdings: `for(holdings('XBB CN Equity'))`. The `holdings()` form is documented; the ticker string is unverified.
- Weights through `id().weights` are documented only for PORT portfolios. Test them in BQLX before relying on them for an ETF.
- Index members, once entitled: `members('<index ticker>')`, or the BDS fields `INDX_MEMBERS` and `INDX_MWEIGHT` (V).
- Split universes into chunks of about 250 IDs per query, because large responses can fail. BQL also doesn't carry every fixed income field that BDP has. [Certain]
- Assume BQL counts against the same data limits as BDP. [Likely]

## Data limits

- **Daily:** about 500,000 hits a day, where one security-field pair is one hit, according to university terminal guides. [Certain on the guides' text]
- **Monthly:** a limit on unique securities, set by a proprietary model and not published. University guidance ranges from 2,500 to 7,000 unique identifiers a month. [Certain on the guides' text; Guessing for your terminal] Ask your Bloomberg representative for your terminal's actual limits.
- **Benchmark size:** XBB alone listed about 1,900 holdings in late October 2026, so the benchmark universe uses much of that guidance range. [Certain]
- **Repeat pulls:** pulling the same security again in the same month doesn't add to the unique count. A stable daily universe is therefore cheap; churn and history backfills are not. [Certain on the rule as the guides state it]
- **Habits:**
  - Never backfill the benchmark with BDH; build history forward from the daily archive.
  - Never copy a sheet that holds live formulas, because the copy re-pulls everything.
  - Close duplicate copies of the workbook.
- **Stop signals:** `#N/A Limit` or `#N/A Review` means the limit is hit. The monthly limit resets on the first of the month.

## Index data on Bloomberg

- The FTSE Canada Universe Bond Index ticker is listed as `CBOBUTR` by an ETF issuer. Verify it on the terminal. [Likely]
- Constituent-level data needs a FTSE Russell data licence that is enabled on Bloomberg. [Likely]
