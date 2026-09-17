# LendingClub Credit-Risk & Mispricing Dashboard

**Recommendation: reprice `small_business` and `major_purchase` loans in Grades A–D.** Across those two purposes, realized net loss consistently runs 3.06 percentage points above what each grade's own peers lose — on $1.04bn of exposure, that gap is worth an estimated **$32.00M** in avoidable net loss over the observed book (a **9.41 bps** uplift to portfolio-level risk-adjusted yield), and the pattern holds up under a 20%-worse-than-historical stress test ($1.38M residual impact, not zero). Grades F and G are deliberately excluded from the recommendation — for these two purposes, they're actually pricing *better* than their grade peers, so folding them in would blur a real signal with noise from two of the thinnest-volume grades in the book.

This is Project 3 of a Power BI portfolio built to support a move out of a contractual, AI-evaluation data-analyst role into a genuine Data Analyst / BI Developer position. Projects 1 (Coffee Shop Sales) and 2 (Real Estate) proved star-schema modeling, Power Query, and DAX fundamentals on small, clean datasets. This one had to prove three additional things a real BI Developer job spec actually cares about: working at real scale and making deliberate modeling trade-offs on 2.26M rows instead of "just import everything"; doing applied risk/finance analysis — grade, rate, default, loss, and yield as a genuine (simplified) credit-portfolio-management problem, not another sales KPI wall; and turning analysis into a decision — a specific segment, a specific action, a specific dollar impact — rather than stopping at description.

---

## Business questions answered

1. **What does the book look like overall?** Volume, weighted-average rate, resolved-vs-open split, grade and purpose mix, origination trend (Page 1).
2. **Is LendingClub's own grade system actually pricing risk correctly within each grade** — or are specific purpose/grade combinations quietly losing more than their grade peers despite being priced about the same (Page 2)?
3. **Is that mispricing a historical artifact, or is it getting worse in recent vintages** — Comparing only fully-matured cohorts so a young cohort that simply hasn't had time to default yet doesn't get mistaken for a "safe" one (Page 3).
4. **So what should actually change, by how much would it be worth, and how confident should anyone be in that number** — Stated caveats, a suggested next step, and a stress test (Page 4).

## Tools used

Power BI Desktop (Power Query, the DAX/Tabular model, report design) end to end. Source data: the public LendingClub accepted-loans extract via Kaggle (`accepted_2007_to_2018Q4.csv.gz`, 2,260,701 rows, 151 columns, June 2007–December 2018). A small `Ref_StateRegion.xlsx` lookup table (US Census Bureau's official 4-region split) built to merge state-level geography into a Region roll-up.

## Data source and disclosure

This is real, anonymized historical loan data, not a synthetic dataset — unlike Project 2, where the underlying transactions were generated. Loss rates, charge-off rates, and yield are calculated directly from real reported payment, recovery, and status fields in the file. The **mispricing benchmark and the repricing recommendation are my own analytical framework applied to that real data** — they are not a LendingClub-published figure, a regulatory finding, or investment advice. The dataset is a single historical lending cycle (2007–2018) from one lender's own underwriting; see Limitations below for what that does and doesn't let this analysis claim.

---

## Data model

**Grain: one row per loan (`Fact_Loan`, ~2.26M rows).** The public Kaggle file has no payment-transaction history table, so loan grain is both the natural and the only available choice — not an oversight.

**Dimension tables:**

| Table | Contents | Notes |
|---|---|---|
| `Dim_Grade` | Grade (A–G), SubGrade (A1–G5), GradeSortOrder | 35 rows; needs an explicit sort column since "A1, A2 … G5" won't sort correctly on its own |
| `Dim_Purpose` | 14 loan-purpose categories | A couple of categories (`educational`, `renewable_energy`) are thin enough once filtered that I don't read much into them alone |
| `Dim_Geography` | StateCode, StateName, Region | Region built from a sourced Census 4-region lookup table, not hand-typed rules — see below. Iowa has only 14 loans in the entire file against tens/hundreds of thousands for every other state; due to LendingClub operating restriction in that state |
| `Dim_EmpLength` | Bucketed (`<1`, `1–3`, `4–6`, `7–9`, `10+`, `Unknown/n/a`) with SortOrder | |
| `Dim_LoanStatus` | Raw status text + `RiskFlag` | The single most important dimension in the model — see table below |
| `Dim_LoanAttributes` (junk dimension) | VerificationStatus × ApplicationType × InitialListStatus | A few dozen distinct combinations pre-built in Power Query rather than three separate one-attribute tables — the deliberate "not everything needs its own dimension" call |
| `Dim_Date` | Standard contiguous daily calendar, marked as the official Date table | `issue_d` and the other `_d` fields only ever carry month precision in the source ("Dec-2015"), so Fact_Loan only ever joins to the 1st-of-month row — a stated grain decision, not a bug. Native time-intelligence functions (`DATEADD`, `SAMEPERIODLASTYEAR`, etc.) still need day-level continuity in the calendar itself, which is why it's a full daily table rather than a sparse month table |

`Term` (36/60) and `VerificationStatus` were **not** given their own dimension tables — a 2-row or 3-row lookup table is over-normalizing. `Term` stays a plain attribute on the fact table; `VerificationStatus` folds into the junk dimension above.

**`Dim_LoanStatus` RiskFlag mapping** — what makes downstream risk measures honest:

| Raw status | RiskFlag |
|---|---|
| Fully Paid (incl. "Does not meet the credit policy. Status:Fully Paid") | Resolved-Good |
| Charged Off / Default (incl. legacy credit-policy variant) | Resolved-Bad |
| Current, Issued | Open-Performing |
| In Grace Period, Late (16–30 days) | Open-Delinquent-Early |
| Late (31–120 days) | Open-Delinquent-Late |

Every risk measure in this model is computed **only over Resolved loans**. A segment made up mostly of recently-issued loans will look artificially safe if still-open loans are counted as "not defaulted" — they simply haven't had time to default yet.

**Explicitly dropped columns**: `emp_title`, `title`, `desc`, `url` (free text / near-unique, no analytical value); `zip_code` (redundant with `addr_state`, would invite a confusing dual-geography model for no benefit); `policy_code` (constant); the `hardship_*`, `settlement_*`, `sec_app_*`, and joint-application fields (`annual_inc_joint`, `dti_joint`, `verification_status_joint`, `revol_bal_joint`) — joint applications are a quantified 5.3% of the book (120,710 of 2,260,701 loans).

**Kept, worth flagging:** FICO (`fico_range_low`/`fico_range_high` at origination, `last_fico_range_low`/`last_fico_range_high` most recent) was missing from the original project brief and shouldn't have been — it's arguably the single most standard credit-risk variable there is, and it gives a second, independent sense-check against grade: if FICO doesn't track cleanly with grade for some segment, that's worth investigating on its own.

**Region build method:** rather than a 51-branch conditional column, `addr_state`'s 51 distinct values (50 states + DC, confirmed against the raw file — no territories or military codes) were merged against a small externally-sourced `Ref_StateRegion.xlsx` built from the US Census Bureau's official 4-region split (DC grouped into South, per Census placement).

**Relationships:** a standard star — Fact_Loan many-to-one to every dimension above, single direction, no bidirectional filtering. Unlike Project 2 (which needed a DAX-side relationship simulation for an unrelated market-benchmark table), every dimension here has an unambiguous 1:many join, so no relationship trick was needed.

**Final model size:** 29 columns, 2,260,668 rows (2,260,701 source rows minus 33 fully-blank stray rows, filtered at the Power Query source step on `id` not null).

<img width="1251" height="767" alt="05_model" src="https://github.com/user-attachments/assets/d94a7e1d-67a5-4c93-baa0-773f0782bfdf" />

<img width="145" height="272" alt="06_queries" src="https://github.com/user-attachments/assets/a9d226bd-ce17-49a9-ae1e-f04514676683" />

---

## Performance and scale

| | Naive baseline | Final model |
|---|---|---|
| Columns | ~145 (raw extract, default types) | 29 |
| Rows | 2,260,701 | 2,260,668 |
| .pbix file size | 360 MB | **105 MB** |

A **70.8% reduction in file size** (360 MB → 105 MB, roughly a 3.4x reduction) from column pruning and star-schema dimensionalization alone, without touching row count.

The discipline applied, in order: filter the 33 blank source rows before anything else, so every downstream count is built on the real row set rather than one quietly off by 33; prune the raw ~145-column extract down to the ~29 columns actually used, in Power Query, before load — not hidden after load — since column pruning is usually the single biggest size lever on a wide flat-file export; fix data types early (`int_rate`/`dti`/FICO to decimal, dates parsed to a real date type, `term` trimmed of its leading-space text quirk before conversion to a whole number); turn off Auto Date/Time before building anything, replacing Power BI's automatic hidden per-column date table with the single shared `Dim_Date` — invisible until you go looking for it, and a well-known performance cost at this row count; and dimensionalize the repeated categorical columns (grade, purpose, state, emp length, status, verification) into real dimension tables rather than leaving them as raw text on the fact table, which is correct star-schema practice independent of any size benefit.

Incremental refresh doesn't functionally apply here — this is a static, one-time historical Kaggle extract, not a live-refreshing source. In a live production version of this dataset, one could partition `Dim_Date` by year and configure incremental refresh, so only the current year's open loans reprocess.

## Page-by-page walkthrough

<img width="1311" height="736" alt="01_portfolio_overview" src="https://github.com/user-attachments/assets/026a473c-1b40-41de-bbfe-0785d64e19e7" />
### Page 1 — Portfolio Overview
2M total loans (2,260,668), a 13.38% dollar-weighted average interest rate, a 19.98% count-based charge-off rate among resolved loans, and — headline-visible rather than a footnote — **40.37% of the book is still open** with no known outcome yet. Grade mix (B and C dominate by count, tapering sharply through F/G), purpose mix (debt_consolidation and credit_card dominate volume), and a total-exposure-by-origination-date trend from 2007 through 2018 round out the page. This page exists to display the numbers.

<img width="1307" height="738" alt="02_segment_mispricing" src="https://github.com/user-attachments/assets/12ffe847-9259-44de-9667-f03a624be542" />
### Page 2 — Segment Mispricing
The analytical core, named specifically rather than a generic "Risk Segmentation" label so the argument is immediately known. Rather than inventing an external pricing model, **use LendingClub's own grade system as the benchmark**. Grade is supposed to already price for risk — that's its entire purpose — so "mispriced" means *within a grade* (which should be risk-homogeneous), does a given purpose have realized loss meaningfully worse than its grade peers, despite being priced at roughly the same rate as the rest of that grade.

```dax
Grade Avg Net Loss Rate =
CALCULATE([Net Loss Rate], ALLEXCEPT(Fact_Loan, Dim_Grade))

Mispricing Gap = [Net Loss Rate] - [Grade Avg Net Loss Rate]
```

A grade × purpose matrix shows every combination's gap at once (colored red for worse-than-peers, green for better); a scatter plots exposure against mispricing gap, colored by grade on a deliberate teal-to-red ordinal ramp so "big and mispriced" segments visually separate from small, noisy ones instead of blending into a single default-palette color; and a `% Book Open` indicator sits next to every number on the page, so nothing here gets read as more mature than it is. The finding that falls out of it: **`small_business` and `major_purchase` show a positive mispricing gap consistently across Grades A–D (up to +6.5pp), with meaningful dollar exposure behind it. `debt_consolidation` carries the largest absolute exposure on the book but a gap under 1pp in every grade — a volume story, not a mispricing one**, and the two are kept visually separate on this page so they don't get conflated.

<img width="1306" height="734" alt="03_vintage_cohort_monitoring" src="https://github.com/user-attachments/assets/e09d4661-0d89-4454-a8af-4cf7cffb9b92" />
### Page 3 — Vintage Cohort Monitoring
The public LendingClub file is a loan-level snapshot, not a monthly performance panel. A true delinquency-emergence curve would need a loan-month tape (was this loan 30dpd in month 6, 90dpd in month 12); this dataset only has an issue date and a final/current status. The defensible approximation built instead: `Months on Book = DATEDIFF(issue_d, last_pymnt_d, MONTH)`, used as a proxy for how long a loan seasoned before its outcome.

The page overlays a cumulative-bad-rate-by-MOB curve for every issuance year (2007 highlighted red as the worst vintage, 2009 highlighted teal as the best, the rest muted grey so the two extremes read immediately against a 12-line legend that would otherwise be unreadable), and a vintage maturity table — **filtered to mature cohorts only** — so a genuinely worsening trend isn't confused with a young cohort that simply hasn't had time to go bad yet:

| Year | Charge-off Rate | Annualized Net Loss Rate |
|---|---|---|
| 2007 | 26.20% | 5.78% |
| 2008 | 20.73% | 4.86% |
| 2009 | 13.69% | 2.99% |
| 2010 | 14.01% | 2.37% |
| 2011 | 15.18% | 2.73% |
| 2012 | 16.20% | 3.04% |
| 2013 | 15.60% | 2.67% |
| 2014 | 15.09% | 2.55% |
| 2015 | 14.89% | 2.52% |
| 2016 | 17.28% | 3.20% |
| **Total** | **15.48%** | **2.70%** |

820K loans qualify as mature. Worst mature vintage: 2007 (26.20%). Best mature vintage: 2009 (13.69%). And the headline drift figure used on the recommendation page — 2011→2016, both fully mature — shows a **2.10 percentage-point** deterioration, the vintage-side confirmation that the mispricing found on Page 2 isn't just a one-off historical artifact.

```dax
Is Vintage Mature =
DATEDIFF(Fact_Loan[issue_d], MAX(Fact_Loan[last_credit_pull_d]), MONTH) >= Fact_Loan[term]
```

A calculated column anchored to `MAX(last_credit_pull_d)` — the most recent date the dataset itself actually has data for — rather than `TODAY()`. Since this is a static, one-time historical extract, evaluating maturity against the real current calendar date (now years past when the data was captured) would make nearly every loan trivially "mature," including cohorts the extract never actually observed for long enough. Anchoring to the data's own effective as-of date instead keeps the maturity check honest to what was actually observed, not to whenever the report happens to be opened.

A bookmark-driven toggle switches the trend chart between "All Cohorts" and "Mature Only," so a reviewer can see the right-censoring problem directly rather than take the maturity filter on faith.

<img width="1307" height="732" alt="04_recommendation" src="https://github.com/user-attachments/assets/dd81dce6-98cb-482f-8821-6ebb2a7f63cd" />
### Page 4 — Recommendation
A one-sentence headline banner states the action and the segment it targets. Three cards back the recommendation: the 3.06% mispricing gap, the $1.04bn exposure it sits on, and the 2.10pp vintage drift confirming it isn't static. A quantified-impact block shows the $32.00M estimated excess loss (the number reserved in the violet accent color, since it's arguably the single most important figure on the page), alongside a 14-row supporting breakdown table (grade × purpose, gap % and dollar contribution per row, so a reviewer can see exactly how the headline number was built rather than trusting a black-box total) and the 9.41 bps portfolio-yield uplift. A caveats panel states plainly what this analysis can't claim (expanded on below). A next-step panel recommends trying a reprice on new originations for two quarters, monitoring the vintage curve, before any portfolio-wide rollout. The charge-off-stress-% slider drives a live sensitivity check: at a 20% stress, the estimated excess loss falls to **$1.38M** — a ~96% reduction from the $32.00M base case — with a dynamic text measure adjusting the result as the slider moves.

---

## Key issues encountered and fixed

- **Locale-specific date parsing.** Converting `issue_d`/`last_pymnt_d`/`last_credit_pull_d` from `Mon-YYYY` text (e.g. `Dec-2015`) to a real date failed to parse under the default UK locale in Power Query, and needed Change Type → Using Locale set explicitly to English (United States). A concrete example of a regional-settings gotcha that only shows up building against US-sourced data on a UK machine — a US-based version of this project would likely never hit it, since the system locale would already match the source format. This was one of the first issues that needed to be addressed.
- **A visual-level filter silently cleared mid-build.** The vintage maturity matrix's `Is Vintage Mature = TRUE` filter got cleared during formatting at some point, which meant the matrix was quietly displaying immature cohorts (including 2017/2018, which shouldn't have appeared at all) while a card measure using the same filter baked directly into its DAX kept showing the correct, filtered figure. The two numbers disagreeing (2.10% on the card vs. an apparent 8.11pp from the table) was the tell. Root-caused by isolating each half of the drift calculation into scratch cards and finding that "year-only, no maturity filter" exactly reproduced the table's wrong value — confirming the table, not the card, was broken. Fixed by re-adding the filter, and it's the direct reason every maturity/risk filter that matters downstream now gets baked into the DAX itself with `CALCULATE`.
- **A Table visual's native Total row doesn't sum its own displayed rows.** The supporting breakdown table's auto-generated Total showed $51.36M against a manually-verified $32.00M sum of its 14 displayed rows. Root cause: `ALLEXCEPT(Fact_Loan, Dim_Grade)` inside `Grade Avg Net Loss Rate` has nothing left to except once Grade context disappears at a Total row, so it falls back to the whole book's average loss rate instead of each row's own grade average. Fixed by hiding the native Total row rather than trying to force it to reconcile, since the correct total already exists as its own headline card.
- **What-if parameter reference resolved to a column, not a measure.** `[Charge-off Stress %]` on its own threw "no current row for this column" — the bare bracket reference for a What-if parameter resolves to its underlying table column, which has no meaning without row context. Fixed with an explicit `SELECTEDVALUE('Charge-off Stress %'[Charge-off Stress %], 0)`.
- **Line-chart legibility with too many categories.** 12 vintage-year series sharing an 8-color default theme palette meant Power BI silently reused colors across different years. Fixed by setting one flat neutral grey as the default series color and overriding only the two years that matter (worst, best) with deliberate colors and heavier strokes, plus direct on-chart labels instead of relying on a 12-item legend. This was done at the end in a final styling sweep.
- **Scatter-chart legibility, same root cause.** The Page 2 grade-by-exposure scatter had the same problem in miniature — 7 grade categories reading as near-identical default purple. Fixed with a deliberate teal-to-red ordinal color ramp (rather than default categorical colors) so color encodes credit quality itself — worse grades read redder — plus a thin dark border on each marker so overlapping points separate against the dark background instead of blurring together. This was also done at the end in a final styling sweep.

## Design system

Dark mode with an electric-violet accent, distinct from Project 1's and Project 2's palettes. Background: near-black `#0B0B12` (page), `#17161F` (card/panel surface) — not pure black, to keep visible depth between layers. Text: off-white `#EDEBF5` primary, muted violet-grey `#9C97B5` for secondary labels. The accent, `#8B5CF6`, is **reserved rather than decorative** — it appears on exactly one number per page, the single most important figure on that page (the $32.00M estimated excess loss on Page 4; nowhere else), the same discipline Project 2 used for its own accent color. A violet-tinted neutral ramp (`#4B4560` → `#6E6690`) carries bars and columns that aren't making a risk judgment. Separate from the accent, a consistent **risk semantic pair** — muted red `#E5484D` for loss/charge-off/mispricing, muted teal `#3DD68C` for good/resolved-well — is used the same way on every page, so red never means something different from one page to the next. I was a little loose with some of the styling 'rules,' in order to ensure the dashboard's legibility.

## Limitations and honest notes

- **Single lending cycle, no macro control.** This analysis reflects one historical lending cycle (2007–2018) from one lender's own underwriting, with no macroeconomic or credit-cycle control beyond the built-in sensitivity check. The sensitivity check itself surfaces a real finding worth stating plainly: because `Grade Avg Net Loss Rate` is a comparatively large absolute base rate and `Mispricing Gap` is a much thinner percentage-point differential riding on top of it, even a "mild" 20% relative stress on that base rate is enough to erode the estimated impact by roughly 96%. That's a mathematically sound consequence of building a thin spread on top of a large base rate, not a flaw in the mechanic — and it means this $32.00M figure should be read as directional support for a cautious, monitored pilot, not a number to bank on holding under a worse credit cycle.
- **Loan seasoning is approximated, not measured directly.** `Months on Book` is derived from last-payment date, not a true monthly performance panel — stated in full in the Page 3 walkthrough above.
- **Grade is a benchmark, not an independent risk standard.** "Mispriced relative to grade" means a segment lost more than its own grade's peers — it does not mean LendingClub's grade itself is a perfectly risk-calibrated starting point. If grade under-prices risk system-wide, this analysis wouldn't surface that; it only surfaces relative deviation within grade.
- **Yield and annualization are simplified.** Risk-Adjusted Yield is nominal weighted-average rate minus net loss rate — it ignores duration/time-value effects (a loss in month 3 costs more than the same loss in month 33) and treats interest as fully collected at the nominal rate even on loans that later went bad. Annualized Net Loss Rate divides cumulative loss by `Term / 12` — a straight-line simplification, not a true amortization-curve-based annualization. Both are stated simplifications, not precision claims.
- **Charge-off loss is not the full funded amount.** `Realized Net Loss = funded_amnt − principal already recovered − net post-charge-off recovery`, not the naive full-balance write-off a count-based default rate implies — the more actuarially-informed version was used throughout rather than the simpler one.

## Skills demonstrated

Large-scale star-schema modeling under real performance constraints (2.26M rows, deliberate column-pruning and dimensionalization trade-offs); applied credit-risk methodology (loss-given-default vs. naive default rate, vintage/cohort analysis with explicit censoring awareness); DAX technique (`ALLEXCEPT` grade-benchmark patterns, `DATEDIFF`-based cohort measures, context-transition debugging, What-if parameters); and translating analysis into a quantified, stress-tested business recommendation rather than stopping at description.
