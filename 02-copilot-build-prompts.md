# Copilot build prompts

Twenty-six prompts build the dashboard in six phases. Run them in order in Copilot's Agent Mode, one prompt per turn, and complete each check before sending the next prompt.

## Read first

Copilot can build almost all of this, but treat every step as a draft and check it before the next prompt.

- **Accuracy.** Agent Mode scored 57.2% on SpreadsheetBench when it launched ([Microsoft](https://www.microsoft.com/en-us/microsoft-365/blog/?p=279414)). [Certain] So each prompt spells out the exact formula, which makes Copilot implement it rather than invent one, and the Checks sheet (P23) catches what slips through.
- **File and settings.** Copilot only works on .xlsx, .xlsb or .xlsm files saved to OneDrive or SharePoint with AutoSave on, and it only edits when calculation is set to Automatic ([Copilot FAQ](https://support.microsoft.com/office/7a13758f-d61e-4a56-8440-f2c9a07802ec)). [Certain] Keep calculation on Automatic while you build.
- **Desktop Excel.** Work in desktop Excel so the Bloomberg formulas calculate. Agent Mode has been generally available on desktop since January 2026 ([Excel blog](https://techcommunity.microsoft.com/blog/excelblog/agent-mode-in-excel-is-now-generally-available-on-desktop/4457408)). [Certain]
- **Design choice.** The Model is built with dynamic-array formulas, not Power Query joins, because Copilot doesn't create Power Query queries or VBA inside the file. [Likely] That keeps every component editable by Copilot. The trade-off is that the Model reads live Bloomberg cells, so confirm the Requesting count is zero before trusting any number. Power Query and VBA appear only in P24, where Copilot writes the code and you paste it in.
- **Verify fields before P5.** Spend 30 minutes in FLDS on the fields marked U in [03-bloomberg-field-reference.md](03-bloomberg-field-reference.md). Also confirm that YAS_YLD_SPREAD measures against the GoC curve, not a single benchmark bond.
- **XBB must return ISINs.** If your XBB pull returns Bloomberg IDs instead, add ID_ISIN to it before P5.
- **Rhythm.** Use Agent Mode and send one prompt per turn. OneDrive version history is your undo, so note the time before each prompt. Save the file as .xlsm before P24.

## Where everything lives

Twelve sheets sit between the two raw tabs and the dashboard, and only one of them holds live Bloomberg formulas.

| Sheet | Holds | Built by |
| --- | --- | --- |
| README | Rules Copilot follows on every prompt | P1 |
| Settings | AsOfDate, term buckets, scenarios | P3, P21 |
| Overrides | Manual analytics for private placements and bad data | P3, then you |
| SecMaster | Static Bloomberg fields, stored as values | P4, then your paste |
| BBG | The only live Bloomberg formulas (apart from XBB) | P5, P6 |
| Mappings | Rating scale, sector map, parent overrides | P7, P8, then your review |
| Model | One row per portfolio and benchmark bond, with every derived column | P9 to P13 |
| M1 to M9 | One module per sheet | P14 to P22 |
| Checks | Pass/fail data-quality tests | P23 |
| History | Archived daily snapshots (tblHistory, tblPosHist) | P24, then your paste |
| M10 | History charts and implied trades | P25 |
| Dashboard | Linked layout, no calculations of its own | P26 |

The raw tabs, Raw_portfolio (portfolio holdings table from the accounting export) and XBB (benchmark holdings array from Bloomberg), are never edited. The swap to index data later changes one LET variable (bench) in P9.

## Phase 1 — Foundations

Three prompts set the rules, show Copilot your raw data, and create the settings tables.

### P1 · Rules sheet

```
Create a sheet README as the first tab with these rules, one per row:
1. Never edit Raw_portfolio or XBB.
2. Bloomberg formulas only on sheet BBG (XBB excepted).
3. P = portfolio, B = benchmark (XBB for now).
4. Model columns: header in row 1 (no spaces), one spill formula in row 2, named m_<Header>.
5. Grids use SUMIFS on m_ names with fixed row labels. No PivotTables, OFFSET, INDIRECT or TODAY.
6. Treat Bloomberg text returns (#N/A ...) as blank.
7. Units: weights decimal, yields %, spreads bp, durations years.
8. After each task, list assumptions and anything unmapped.
```

**Check:** every later prompt starts with "Per README", so you never have to restate these rules.

### P2 · Inventory (read-only)

```
Per README, change nothing. For table Raw_portfolio and the array on sheet XBB, list each column's header, type and a sample value. Identify the columns for ISIN, par, market value (clean or dirty), accrued, weight, asset type or cash flag, currency and as-of date. List the non-bond rows (cash, FX, futures).
```

**Check:** you now know which XBB column holds ISINs, and whether portfolio market value includes accrued interest. P5 and P9 depend on both.

### P3 · Settings and overrides

```
Per README, create sheet Settings with:
- tblSettings (Key, Value): AsOfDate = as-of date from Raw_portfolio; BenchCode = XBB; ArchiveFolder = blank. Name the AsOfDate value cell AsOfDate.
- tblMapTerm: LowerYrs 0,1,3,5,7,10,20; Bucket <1,1-3,3-5,5-7,7-10,10-20,20+; Band Out,Short,Short,Mid,Mid,Long,Long; MidYrs 0.5,2,4,6,8.5,15,25.
Create sheet Overrides with empty table tblOverrides: ID, UltParent, Sector, Maturity, Rating, Y, D, SD, S, C, Eligible, Reason.
```

**Check:** AsOfDate holds a date value, not a TODAY() formula.

## Phase 2 — Bloomberg layer

Bloomberg formulas sit on one sheet (BBG). Static fields are pulled once and stored as values in SecMaster, so only price-driven fields refresh daily.

### P4 · Security master table

```
Per README, create sheet SecMaster with empty table tblSecMaster, headers: ID, SECURITY_DES, ISSUER, ULT_PARENT_TICKER_EXCHANGE, BICS_LEVEL_1_SECTOR_NAME, BICS_LEVEL_3_INDUSTRY_NAME, CNTRY_OF_INCORPORATION, CRNCY, CPN, CPN_TYP, CPN_FREQ, MATURITY, MTY_TYP, NXT_CALL_DT, AMT_ISSUED, AMT_OUTSTANDING, RTG_SP, RTG_MOODY, RTG_DBRS, RTG_FITCH, PullDate.
```

**Check:** the headers match the Bloomberg mnemonics exactly, because P6 reads them as field names.

### P5 · Daily Bloomberg grid

```
Per README, create sheet BBG.
A1 "ID"; A2: sorted unique ISINs of bond rows in Raw_portfolio and XBB.
B2: ="/isin/"&A2#
C1:M1 headers: PX_MID, INT_ACC, YLD_YTM_MID, YAS_BOND_YLD, DUR_ADJ_MID, OAS_EFF_DUR, CNVX_MID, YAS_YLD_SPREAD, OAS_SPREAD_DUR_MID, MTG_WAL, MTG_FACTOR.
C2:M3000: =IF($A2="","",BDP($B2,C$1))
O1 "Requesting"; O2: count of cells in C2:M3000 containing "Requesting".
```

**Check:** O2 falls to zero once the data loads. If rows show Invalid Security, test one bond's identifier format (the /isin/ prefix versus the ISIN followed by Govt or Corp) and fix B2.

### P6 · Static pull for new IDs

```
Per README, on BBG from column Q:
Q1 "NewID"; Q2: IDs in A2# not in tblSecMaster[ID].
R1 onward: tblSecMaster headers SECURITY_DES to RTG_FITCH.
R2 down to row 3000: =IF($Q2="","",BDP("/isin/"&$Q2,R$1))
Extend the O2 count to cover this block.
```

**You, after P6:**

1. When O2 reads zero, copy the static block's filled rows (from Q2 across to the last static column) and paste them as values directly under tblSecMaster, so the table extends. Fill in PullDate.
2. Each day, repeat this for any new IDs.
3. Each week, delete every tblSecMaster row so that all IDs re-pull, because ratings and parents change.

## Phase 3 — Mappings

Run these once SecMaster holds data, because the sector map is built from the BICS values Bloomberg actually returns.

### P7 · Rating scale

```
Per README, create sheet Mappings with tblMapRating (Code, N):
S&P and Fitch AAA=1, AA+=2 ... D=22; Moody's Aaa=1, Aa1=2 ... C=21; DBRS AAA=1, AA (high)=2 ... D=22. NR and WR mean no rating.
Add tblRtgBucket (N, Bucket): 1 AAA; 2-4 AA; 5-7 A; 8-9 BBB; 10 BBB-; 11+ HY.
Create a LAMBDA named CLEANRTG that strips watch and outlook marks such as *-, *+, (P) and a trailing u.
```

**Check:** every rating in tblSecMaster maps after CLEANRTG. Bloomberg may show DBRS ratings in a short form (for example AAL), so add any that don't map to tblMapRating.

### P8 · Sector maps

```
Per README, on Mappings:
1. tblSectors (Sector, L1, Order): Federal, Federal SSA, Provincial, Municipal (L1 Government); Financial, Communication, Energy, Industrial, Infrastructure, Real Estate, Securitization (L1 Corporate); Off-benchmark; Unmapped.
2. tblMapSector: every distinct pair of BICS_LEVEL_1_SECTOR_NAME and BICS_LEVEL_3_INDUSTRY_NAME in tblSecMaster, with your proposed Sector and a Review flag where unsure. Use XBB's sector column, if any, as a cross-check.
3. tblMapParent (ULT_PARENT_TICKER_EXCHANGE, Sector) for overrides; seed Canada Housing Trust as Federal.
```

**You, after P8:** clear every Review flag yourself.

- Provincial agencies such as Hydro-Québec go to Provincial.
- Supranationals and foreign agencies go to Federal SSA.
- The usual judgment calls are utilities, pipelines and P3 projects, which split across Infrastructure and Energy. Confirm them against the index constituent file once you have it.
- If BICS level 3 comes back blank for bonds, map on level 2 instead (find its mnemonic in FLDS).

## Phase 4 — Model

Five prompts build one row per portfolio and benchmark bond, then every derived column the modules read. From here on the prompts use plain keyboard symbols (*, /, <=).

### P9 · Model core

```
Per README, create sheet Model. A1:D1 headers Src, ID, Par, MV.
A2: one LET formula stacking
- P: Raw_portfolio bond rows grouped by ISIN, summing Par and dirty MV (add accrued if MV is clean);
- B: XBB bond rows (ISIN, par or blank, market value or weight).
Keep the B rows in one LET variable named bench.
Name the columns m_Src, m_ID, m_Par, m_MV (INDEX of the A2 spill).
E: w = MV / total MV of the same Src. F: Sign = 1 for P, -1 for B.
```

**Check:** w sums to 1 on each side, and the P row count equals the number of unique bond ISINs in Raw_portfolio.

### P10 · Attributes and ratings

```
Per README, add Model columns looked up by ID from tblSecMaster; tblOverrides wins where filled:
Issuer, UltParent, Crncy, CpnType, CpnFreq, Maturity, NextCall, Callable (MTY_TYP contains "CALL"), AmtIssued = MAX(AMT_ISSUED, AMT_OUTSTANDING).
RtgN: apply CLEANRTG to the S&P, Moody's, DBRS and Fitch ratings and map to N. With 1-2 ratings take the worst; with 3-4 take the second-worst (LARGE(r,2)).
RatingBucket from tblRtgBucket; "NR" if no rating.
```

**Check:** spot-check five bonds against the terminal, including one with split ratings. The rule matches the index's: the lower of two ratings, the middle of three, and the middle of the three lowest of four.

### P11 · Analytics

```
Per README, add Model columns from BBG by ID; tblOverrides wins where filled:
PxDirty = PX_MID + INT_ACC.
Y = YAS_BOND_YLD if Callable, else YLD_YTM_MID.
D = OAS_EFF_DUR if (Callable or MTG_WAL > 0) and it is > 0, else DUR_ADJ_MID.
SD = OAS_SPREAD_DUR_MID if > 0, else D.
S = YAS_YLD_SPREAD. C = CNVX_MID. WAL = MTG_WAL. Factor = MTG_FACTOR.
AnalyticsSrc = "Manual" if any override was used, "Missing" if D or S is blank, else "BBG".
```

**Check:** the only rows showing Missing are private placements still waiting for overrides.

### P12 · Classification and eligibility

```
Per README, add Model columns:
EffMat = NextCall if Callable and YAS_BOND_YLD < YLD_YTM_MID; AsOfDate + WAL*365.25 if WAL > 0; else Maturity.
TermYrs = (EffMat - AsOfDate)/365.25. TermBucket and Band from tblMapTerm, approximate match on LowerYrs.
Sector = tblMapParent by UltParent, else tblMapSector by BICS pair, else "Unmapped".
IneligReason = first failing test: Crncy not CAD "Non-CAD"; CpnType not fixed "Non-fixed"; CpnFreq not 2 "Not semi-annual"; TermYrs <= 1 "<1yr"; AmtIssued < 100m "Small issue"; RtgN > 10 or blank "Below BBB/NR". A filled tblOverrides Eligible wins (reason "Override").
Eligible = IneligReason blank. FTSE_Sector = Sector if Eligible, else "Off-benchmark".
PeerGroup = FTSE_Sector, plus " " & RatingBucket for Corporate sectors.
```

**Check:** benchmark rows flagged Off-benchmark should be rare, and each one is a data problem worth a look. The 10-institutional-buyer test, and exclusions such as real return bonds and AT1 or insurer Tier 1 capital, can't be tested from these fields, so set those in tblOverrides.

### P13 · Risk and active columns

```
Per README, add Model columns:
CTD = w*D; CSD = w*SD; DTS = SD*S; wDTS = w*DTS; wY = w*Y; wDY = w*D*Y; wS = w*S; wC = w*C.
aW = Sign*w; aCTD = Sign*CTD; aCSD = Sign*CSD; aDTS = Sign*wDTS.
Blank inputs give blank outputs, not zero.
```

**Check:** the sum of m_aCTD equals portfolio duration minus benchmark duration.

## Phase 5a — Modules M1 to M5

Each prompt builds one module sheet from the Model's m_ names. [01-dashboard-spec.md](01-dashboard-spec.md) explains how to read each module.

### P14 · M1 headline strip

```
Per README, create sheet M1. Rows: Yield, DurWtdYield, Duration, Convexity, Spread, SpreadDur, DTS, DTSRatio, Breakeven, Issuers, EffIssuers. Columns: P, B, Active (P-B).
By side: Yield = sum wY; Duration = sum CTD; Convexity = sum wC; Spread = sum wS; SpreadDur = sum CSD; DTS = sum wDTS.
DurWtdYield = sum wDY / sum CTD. Breakeven = sum wS / sum CSD. DTSRatio = DTS P / DTS B, Active column only.
Issuers = unique UltParent. EffIssuers = 1 / sum of squared parent shares, corporate parents only.
Colour the Active column only.
```

**Check:** Active Duration equals the sum of m_aCTD.

### P15 · M2 duration by term

```
Per README, create sheet M2. Rows: every tblMapTerm bucket. Columns: P CTD, B CTD, Active CTD (SUMIFS on m_CTD and m_aCTD by m_TermBucket).
Add Short, Mid and Long subtotals, and a total that must equal M1 active Duration.
Add duration dispersion per side: SQRT(sum of w*(D - avg D)^2).
Column chart of Active CTD by bucket, short to long, with total active duration in the title.
```

**Check:** the total row ties to M1 to six decimals.

### P16 · M3 sectors

```
Per README, create sheet M3. Rows: tblSectors in Order. Columns: P w, B w, Active w, Active CTD, Active DTS (SUMIFS on m_w, m_aW, m_aCTD, m_aDTS by m_FTSE_Sector).
Total row must tie to M1.
Diverging data bars on the three active columns: axis at cell midpoint, min/max = +/- max absolute value.
```

**Check:** Active w totals zero, and the Unmapped row is empty. If it isn't, P8 needs another pass.

### P17 · M4 heatmap

```
Per README, create sheet M4. Grid rows: tblSectors. Columns: every term bucket.
Dropdown cell selMeasure with aW, aCTD, aDTS.
Each cell = SUMIFS of the selected m_ measure by m_FTSE_Sector and m_TermBucket. Add row and column totals.
3-colour scale on the grid only: -max abs red, 0 white, +max abs blue. 2 decimals.
```

**Check:** with aCTD selected, the row totals match M3's Active CTD and the column totals match M2.

### P18 · M5 credit quality

```
Per README, create sheet M5. Rows: AAA, AA, A, BBB, BBB-, HY, NR. Columns: P w, B w, Active w, Active DTS (SUMIFS by m_RatingBucket).
Beside it, link DTSRatio and Breakeven (P and B) from M1.
Bar chart of Active DTS by bucket.
```

**Check:** Active w totals zero, and Active DTS totals M1's active DTS.

## Phase 5b — Modules M6 to M9

Four prompts finish the modules: issuers, relative value, scenarios and the off-benchmark sleeve.

### P19 · M6 issuers

```
Per README, create sheet M6.
Top 10 overweights and top 10 underweights by UltParent, ranked by sum of aDTS (GROUPBY, SORTBY, TAKE). Columns: Parent, Sector, P w, B w, Active w, Active DTS, avg S.
Issuer active share = 0.5 * sum of ABS(sum of aW by parent).
HHI and N_eff = 1/HHI for P and B, corporate sleeve only, weights rescaled to the sleeve.
Diverging data bars on Active DTS.
```

**Check:** active share sits between 0 and 1. Judge it against your own history once M10 runs, not against a fixed threshold.

### P20 · M7 relative value

```
Per README, create sheet M7.
Dropdown selPeer from unique m_PeerGroup.
FILTER B bonds (D, S) and P bonds (ID, D, S, w) in that group, D > 0 only.
Fit S = a + b*LN(D) on the B bonds with LINEST; show a and b.
Residual = S - fit for each P bond; weighted residual = sum of w*residual / sum of w.
Scatter: B grey, P accent colour, logarithmic trendline on B.
```

**Check:** the trendline equation on the chart matches a and b.

### P21 · M8 scenarios

```
Per README, on Settings add tblScenarios (Name, dr2, dr5, dr10, dr30, dsCorp, dsProv in bp; x in %; Horizon in years):
Parallel -50; Parallel +50; Bull steep -50/-40/-25/-10; Bear steep 0/10/25/40; Bear flat 40/25/10/0; Corp +25; Prov +10; Spreads x1.25 (x = 25); Risk-off -40/-40/-30/-20 with x = 30. Horizon 0.25.
Create sheet M8, one row per scenario, in bp:
Rates = -sum of M2 Active CTD * shock at the bucket's MidYrs (linear between 2/5/10/30, flat outside).
Spread = -(sum aCSD where m_Sector is Corporate L1) * dsCorp - (sum aCSD where m_Sector = Provincial) * dsProv - (sum aDTS) * x/100.
Carry = (Yield P - Yield B) * 100 * Horizon.
Total = Rates + Spread + Carry. Sort worst last; colour Total.
```

**Check:** Parallel -50 gives a Rates figure of about M1's active Duration * 50. The spread shocks use m_Sector rather than FTSE_Sector, so off-benchmark corporates widen too. Convexity is left out until you've confirmed CNVX_MID's scaling.

### P22 · M9 off-benchmark

```
Per README, create sheet M9: P rows with Eligible = FALSE, one row per IneligReason. Columns: w, CTD, DTS (sum wDTS), w-weighted avg Y and S, share of P DTS, share with AnalyticsSrc = Manual. Total row.
```

**Check:** the total w equals the P weight in M3's Off-benchmark row.

## Phase 6 — Checks, history and dashboard

The last four prompts make the build trustworthy, start the history, and assemble the page.

### P23 · Checks

```
Per README, create sheet Checks with columns Check, Value, Status (PASS, FAIL or INFO):
sum of w = 1 for P and for B; M2, M3 and M5 totals equal M1 actives (within 1e-6); BBG Requesting count = 0; IDs missing from tblSecMaster = 0; P rows with AnalyticsSrc Missing = 0; Unmapped sectors = 0; unmapped ratings = 0; duplicate IDs per side = 0; ID count < 3000; B rows Off-benchmark (INFO); % of P MV with Manual analytics (INFO); AsOfDate vs XBB date, if XBB has one.
Name an overall PASS/FAIL cell ChecksStatus.
```

**Check:** break something on purpose (delete one mapping row) and confirm ChecksStatus turns FAIL.

### P24 · Archive code (you paste it)

```
Write code for me to paste; don't change the workbook.
1. VBA macro ArchiveSnapshot: save the Model sheet's used range as values to ArchiveFolder\model_YYYYMMDD.csv, and save M1, M2, M3 and M5 results as a long table (Date, BenchCode, Side P/B/A, Metric, Bucket, Value; Metric = M1 row label or module column name) to metrics_YYYYMMDD.csv. Use AsOfDate and BenchCode from tblSettings; overwrite the same date.
2. Power Query M that combines metrics_*.csv into tblHistory and model_*.csv into tblPosHist, both on sheet History.
```

**You, after P24:**

1. Save as .xlsm and paste the VBA into a module (Alt+F11).
2. Paste each query into Power Query's Advanced Editor and load it to History.
3. Set ArchiveFolder in tblSettings to a shared folder.
4. Confirm with compliance that storing archived Bloomberg values internally is allowed under your licence. [Guessing on the specifics]

### P25 · M10 history and implied trades

```
Per README, create sheet M10 from tblHistory and tblPosHist:
1. Line charts by Date of active Duration, DTSRatio and active Yield, with month-ends marked.
2. Implied trades for P rows, newest first: PrevPar = Par on the previous archived date (0 if new); TradePar = Par - PrevPar * (Factor / PrevFactor), using a factor ratio of 1 when blank; 0 if the bond matured or was called; include bonds that disappeared (sales). Value = TradePar * PxDirty / 100.
```

**Check:** on a day with no trades, the only rows are paydowns or maturities you can explain. Month-end markers are a reporting convention only, because the index rebalances daily.

### P26 · Dashboard

```
Per README, create sheet Dashboard with links only, no new calculations. Top to bottom: ChecksStatus and AsOfDate; M1 table; M2 chart beside M3 table; M4 heatmap beside M5 chart; M6 tables beside M7 scatter; M8 table beside M9 table; M10 charts. Move the selMeasure and selPeer dropdowns to the top and repoint their names. Hide gridlines; freeze the top rows.
```

**Check:** changing selMeasure on the Dashboard redraws M4, and no Dashboard cell contains a SUMIFS.

## Daily routine once built

1. Refresh Raw_portfolio and XBB.
2. Wait for the Requesting count to reach zero.
3. Paste any new IDs into SecMaster.
4. Clear the Checks.
5. Run ArchiveSnapshot and refresh the History queries.
