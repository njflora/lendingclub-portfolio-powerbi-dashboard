# DAX Measures Reference

Every measure used across the four report pages, with the reasoning behind it. Grouped in build order (foundation → risk → pricing → mispricing → vintage → recommendation), matching the project's [README](./README.md) walkthrough, so a reviewer who won't open Power BI directly can still follow the analytical logic end to end.

---

## Foundation measures

**Weighted Avg Rate** — dollar-weighted, not a naive average of the `int_rate` column. A simple average of averages would let a handful of small, high-rate loans distort the portfolio-level figure just as much as a handful of large ones.

```dax
Weighted Avg Rate =
DIVIDE(
    SUMX(Fact_Loan, Fact_Loan[funded_amnt] * Fact_Loan[int_rate]),
    SUM(Fact_Loan[funded_amnt])
)
```

**Resolved Loan Count** / **Pct Book Open** — 40.37% of the live model is still open with no known outcome. Every risk measure below is deliberately scoped to resolved loans only, so a segment full of recently-issued loans doesn't look artificially safe just because it hasn't had time to default yet.

```dax
Resolved Loan Count =
CALCULATE(COUNTROWS(Fact_Loan), Dim_LoanStatus[RiskFlag] IN {"Resolved-Good","Resolved-Bad"})

Pct Book Open = 1 - DIVIDE([Resolved Loan Count], COUNTROWS(Fact_Loan))
```

---

## Risk measures

**Charge-off Rate** — count-based, intuitive headline number, computed only over resolved loans.

```dax
Charge-off Rate =
DIVIDE(
    CALCULATE(COUNTROWS(Fact_Loan), Dim_LoanStatus[RiskFlag] = "Resolved-Bad"),
    [Resolved Loan Count]
)
```

**Net Loss Rate** — the more precise, dollar-weighted measure, and the one the rest of the model leads with. A LendingClub charge-off is rarely a 100% loss: principal and interest already received stay received, and there's a post-charge-off recovery process. Using realized loss-given-default dollars instead of a binary default flag is the difference between an actuarially-informed loss estimate and a naive count-based default rate.

```dax
Net Loss Rate =
DIVIDE(
    CALCULATE(
        SUMX(Fact_Loan,
            IF(Dim_LoanStatus[RiskFlag] = "Resolved-Bad",
                Fact_Loan[funded_amnt] - Fact_Loan[total_rec_prncp] - (Fact_Loan[recoveries] - Fact_Loan[collection_recovery_fee]),
                0)
        ),
        Dim_LoanStatus[RiskFlag] IN {"Resolved-Good","Resolved-Bad"}
    ),
    CALCULATE(SUM(Fact_Loan[funded_amnt]), Dim_LoanStatus[RiskFlag] IN {"Resolved-Good","Resolved-Bad"})
)
```

---

## Pricing / yield measures

**Risk-Adjusted Yield** — nominal rate earned minus realized loss rate, as a net spread. Deliberately simple: it ignores duration/time-value effects and treats interest as fully collected at the nominal rate even on loans that later went bad. A known, stated simplification, not a precise actuarial yield.

```dax
Risk-Adjusted Yield = [Weighted Avg Rate] - [Net Loss Rate]
```

**Annualized Net Loss Rate** — needed to compare 36- and 60-month loans fairly, since a 60-month loan naturally accumulates more cumulative loss exposure over its life at the *same* underlying risk level, purely by being outstanding longer. A straight-line simplification; a true amortization-curve-based annualization is more correct but out of scope here.

```dax
Annualized Net Loss Rate = DIVIDE([Net Loss Rate], Fact_Loan[Term] / 12)
```

---

## The mispricing measure — the analytical core

The key design decision: use LendingClub's own grade system as the pricing benchmark, rather than inventing an external one. Grade is supposed to already price for risk, so "mispriced" means *within a grade*, does a purpose/segment lose meaningfully more than its grade peers despite being priced about the same.

```dax
Grade Avg Net Loss Rate =
CALCULATE([Net Loss Rate], ALLEXCEPT(Fact_Loan, Dim_Grade))

Mispricing Gap = [Net Loss Rate] - [Grade Avg Net Loss Rate]
```

`ALLEXCEPT(Fact_Loan, Dim_Grade)` removes every filter except grade, so in a matrix sliced by Grade × Purpose this measure always shows "what did this whole grade do" — letting each purpose's actual loss rate be compared against its own grade's average on the same visual. Note: this measure has no meaning outside grade context — dropped on a standalone card with no grade filter, it nets to (approximately) zero by construction, since positive and negative segment deviations cancel out across a grade's own average. It's designed for row/matrix context, not a KPI card.

---

## Vintage cohort measures

**The limitation stated up front:** the public LendingClub file is a loan-level snapshot, not a monthly performance panel. `Months on Book` is a defensible approximation using last-payment date as a proxy for how long a loan seasoned before its outcome — not a substitute for a true loan-month performance tape.

```dax
Months on Book (MOB) = DATEDIFF(Fact_Loan[issue_d], Fact_Loan[last_pymnt_d], MONTH)
```

**Is Vintage Mature** — the single most important guard on this page. Comparing a fully-seasoned old cohort's ultimate default rate to a barely-seasoned recent cohort's rate and concluding recent originations are "safer" is the classic vintage-analysis mistake (right-censoring) — a young cohort simply hasn't had time to go bad yet.

```dax
Is Vintage Mature =
DATEDIFF(Dim_Date[MonthStart], TODAY(), MONTH) >= SELECTEDVALUE(Fact_Loan[Term])
```

**Best/Worst Mature Vintage** — finds the min/max charge-off rate among mature cohorts only, and the year it occurred in.

```dax
Worst Mature Vintage =
VAR WorstRate =
    MAXX(VALUES(Dim_Date[Year]), CALCULATE([Charge-off Rate], Fact_Loan[Is Vintage Mature] = TRUE))
VAR WorstYear =
    CALCULATE(MIN(Dim_Date[Year]), FILTER(VALUES(Dim_Date[Year]), CALCULATE([Charge-off Rate], Fact_Loan[Is Vintage Mature] = TRUE) = WorstRate))
RETURN WorstYear & "  ·  " & FORMAT(WorstRate, "0.00%")
```

*(`Best Mature Vintage` is the same pattern with `MINX`.)*

**Mature Loan Count** — 820K loans qualify.

```dax
Mature Loan Count = CALCULATE(COUNTROWS(Fact_Loan), Fact_Loan[Is Vintage Mature] = TRUE)
```

**Vintage Drift 2011→2016** — the headline year-over-year deterioration figure carried through to the recommendation page, both endpoints restricted to mature cohorts so the comparison is apples-to-apples.

```dax
Vintage Drift 2011→2016 =
CALCULATE([Charge-off Rate], Dim_Date[Year] = 2016, Fact_Loan[Is Vintage Mature] = TRUE)
- CALCULATE([Charge-off Rate], Dim_Date[Year] = 2011, Fact_Loan[Is Vintage Mature] = TRUE)
```

---

## Recommendation-page measures

**Estimated Excess Loss** — dollars of net loss avoided had `small_business` and `major_purchase` performed at their own grade's average, summed across every grade × purpose combination within those two purposes. Confirmed **$32.00M**, cross-validated by manually summing the 14-row supporting breakdown table to the cent.

```dax
Estimated Excess Loss =
CALCULATE(
    SUMX(
        SUMMARIZE(Fact_Loan, Dim_Grade[grade], Dim_Purpose[purpose]),
        [Mispricing Gap] * CALCULATE(SUM(Fact_Loan[funded_amnt]))
    ),
    Dim_Purpose[purpose] IN {"small_business", "major_purchase"}
)
```

*(An earlier draft, `[Mispricing Gap] * [Total Exposure]` with no grade/purpose breakdown, returned ~0% — a real, instructive dead end: grade-average deviations cancel out by construction across the whole unfiltered book, so the measure has to be evaluated at grade × purpose granularity and then summed, not applied as a single blended multiplication.)*

**Segment Mispricing Gap** / **Segment Exposure** — the two proof-point numbers behind the headline: 3.06% and $1.04bn respectively.

```dax
Segment Mispricing Gap =
DIVIDE(
    [Estimated Excess Loss],
    CALCULATE(SUM(Fact_Loan[funded_amnt]), Dim_Purpose[purpose] IN {"small_business", "major_purchase"})
)

Segment Exposure =
CALCULATE(SUM(Fact_Loan[funded_amnt]), Dim_Purpose[purpose] IN {"small_business", "major_purchase"})
```

**Excess Loss Uplift (bps)** — the $32.00M reframed as a portfolio-level yield impact: 9.41 bps.

```dax
Excess Loss Uplift (bps) =
DIVIDE(
    [Estimated Excess Loss],
    CALCULATE(SUM(Fact_Loan[funded_amnt]), ALL(Fact_Loan))
) * 10000
```

**Excess Loss Contribution** — the row-level column behind the supporting breakdown table (14 rows: 7 grades × 2 purposes). Deliberately has no purpose filter baked in — it's designed to be read inside a Grade × Purpose table with a visual-level filter applied, not used standalone (dropped on its own card with no row context, it shows 0.00 for the same cancellation reason as `Grade Avg Net Loss Rate` above).

```dax
Excess Loss Contribution = [Mispricing Gap] * CALCULATE(SUM(Fact_Loan[funded_amnt]))
```

### Sensitivity check

A What-if parameter (`Charge-off Stress %`, numeric range 0–50%, step 5%) drives a live "what if loss rates rise" stress test.

```dax
Stressed Grade Avg Net Loss Rate =
[Grade Avg Net Loss Rate] * (1 + SELECTEDVALUE('Charge-off Stress %'[Charge-off Stress %], 0) / 100)

Stressed Mispricing Gap = [Net Loss Rate] - [Stressed Grade Avg Net Loss Rate]

Stressed Excess Loss =
CALCULATE(
    SUMX(
        SUMMARIZE(Fact_Loan, Dim_Grade[grade], Dim_Purpose[purpose]),
        [Stressed Mispricing Gap] * CALCULATE(SUM(Fact_Loan[funded_amnt]))
    ),
    Dim_Purpose[purpose] IN {"small_business", "major_purchase"}
)
```

At a 20% stress, `Stressed Excess Loss` falls to **$1.38M** — roughly a 96% reduction from the $32.00M base case. That's a large swing, but a mathematically sound one: `Grade Avg Net Loss Rate` is a comparatively large absolute base rate, and `Mispricing Gap` is a much thinner percentage-point differential sitting on top of it, so even a "mild" relative stress on the base rate can swamp most of the differential. See the README's Limitations section for why this was kept as-is rather than dampened with a gentler stress mechanic.

**Sensitivity Narrative** — a dynamic text measure, displayed in a Card visual (a native Text Box can't bind to a measure), narrating the stressed result as the slider moves.

```dax
Sensitivity Narrative =
"At a " & FORMAT(SELECTEDVALUE('Charge-off Stress %'[Charge-off Stress %], 0), "0") &
"% stress, estimated excess loss falls to $" & FORMAT(DIVIDE([Stressed Excess Loss], 1000000), "0.00") &
"M (base case: $32.00M) — this estimate is highly sensitive to loss-rate assumptions; treat it as directional support for a cautious pilot, not a guaranteed figure."
```
