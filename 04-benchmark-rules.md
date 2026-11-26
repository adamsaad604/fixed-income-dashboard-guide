# Benchmark rules that affect the build

The FTSE Canada Universe Bond Index ground rules decide eligibility, ratings, term buckets and sectors. The workbook mirrors them so that active numbers reconcile once index data replaces the ETF proxy. Source: *FTSE Canada Universe Bond Index Series Ground Rules*, v4.7, July 2025.

## Eligibility [Certain]

A bond must meet every test:

- denominated in CAD
- effective term to maturity over one year
- fixed-rate coupon, paid semi-annually
- at least $100 million issued, counting the original issue plus reopenings
- index rating of BBB or better
- at least 10 institutional buyers at issue

Private issues that pass these tests are eligible. The test is size and buyer count, not public versus private.

## Exclusions [Certain]

- floating-rate notes and convertibles
- residential and commercial MBS (including NHA MBS) and other monthly-pay or prepayable securities
- real return bonds
- issues targeted at retail investors
- AT1 capital and life-insurer Tier 1 capital

ABS qualify only when structured as semi-annual-pay bullets. The Securitization appendix still lists CMBS even though the exclusions remove them; treat the exclusion as binding and confirm with FTSE. [Certain on the text; Likely on the reading]

## Index rating [Certain]

The index uses DBRS, S&P, Moody's and Fitch:

- two ratings: take the lower
- three ratings: take the middle
- four ratings: take the middle of the three lowest

The rating buckets are AAA/AA, A and BBB, with no notches. The index ignores unsolicited and private ratings, which a workbook formula can't screen out. The Bloomberg composite combines the agencies differently, so it won't match.

## Term [Certain]

- Option bonds are bucketed by effective term: the option exercise date or the final maturity.
- Fixed-to-variable bonds use the reset date as their maturity.
- The term buckets are Short (1–5 years), Mid (5–10) and Long (10+).

## Sectors [Certain]

- Federal includes Non-Agency, Agency, and Sovereign, Supranational and Foreign Agency (the last renamed on 1 January 2025).
- Maple bonds issued and settled on or after 1 January 2025 are included. Classify their issuer by country of incorporation, not country of risk.
- Map Canada Housing Trust to Federal Agency, and provincial crown corporations and agencies to Provincial.

## Rebalancing and pricing [Certain]

- The index is rebalanced daily. New issues enter on or after their issue date, and bonds leave on the day their remaining effective term reaches one year.
- Prices are the CanDeal DNA composite mid at 4:00 p.m. Toronto time. Bloomberg prices differ, so run analytics for both sides through Bloomberg and use index data only for weights and classification. [Likely]
- Archive the benchmark every day. Month-end markers on history charts are a reporting convention only.

## ETF proxy versus the index

- XBB is a fund, not the index. It holds cash, its weights are based on net asset value, and its holdings count differs from the index's constituent count. [Certain]
- Strip out cash and derivatives and rescale bond weights to 100% before any comparison.
- The issuer's website shows holdings and characteristics as of the prior business day. [Likely]
- Treat active risk against XBB as approximate until index data replaces it.

## Swapping to index data

1. Keep the benchmark rows in one LET variable (`bench`) in the Model, as prompt P9 sets up.
2. Build the index loader against a sample constituent file as soon as one is available. Use the index's own sector and rating columns as the authoritative classification.
3. Run both benchmarks in parallel for about 20 business days. Archive both, and compare headline duration, sector weights and the issuer list. Expect the gaps to come from ETF cash, sampling and lag.
4. Switch over. Keep Bloomberg analytics for both sides, and store the index's own duration and yield as audit columns only.
5. Keep `BenchCode` on every archived row so the switch date stays visible. Don't splice ETF-based and index-based active history together without that marker.
