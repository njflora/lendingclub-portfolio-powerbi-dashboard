# DAX measures

Every measure in the model, with a short note on why it's built the way it is. They're in the order the report uses them, so you can follow the logic without opening Power BI. The same code is in `pbip/LendingClub.SemanticModel/definition/tables/_Measures.tmdl`.

---

## Foundation

**Weighted Avg Rate.** Weighted by funded amount, so a few small high-rate loans can't skew the portfolio figure. `int_rate` is stored as a whole number (13.38), hence the `/ 100`.

```dax
Weighted Avg Rate =
DIVIDE(
    SUMX(Fact_Loan, Fact_Loan[funded_amnt] * Fact_Loan[int_rate]),
    SUM(Fact_Loan[funded_amnt])
) / 100
```

**Total Loans, Total Exposure.**

```dax
Total Loans = COUNTROWS(Fact_Loan)
Total Exposure = SUM(Fact_Loan[funded_amnt])
```

**Resolved Loan Count, Pct Book Open.** 40.37% of the book is still open. Every risk measure below only uses resolved loans, so new loans don't look safe just because they haven't had time to default.

```dax
Resolved Loan Count =
CALCULATE(
    COUNTROWS(Fact_Loan),
    Dim_LoanStatus[RiskFlag] IN {"Resolved-Good", "Resolved-Bad"}
)

Pct Book Open = 1 - DIVIDE([Resolved Loan Count], COUNTROWS(Fact_Loan))
```

---

## Risk

**Charge-off Rate.** The simple, count-based headline.

```dax
Charge-off Rate =
DIVIDE(
    CALCULATE(COUNTROWS(Fact_Loan), Dim_LoanStatus[RiskFlag] = "Resolved-Bad"),
    [Resolved Loan Count]
)
```

**Net Loss Rate.** The measure the rest of the model is built on. A charged-off loan is rarely a 100% loss: principal already repaid stays repaid, and some money comes back through recoveries. So the loss is funded amount minus principal repaid minus net recoveries, as a share of funded amount on resolved loans.

```dax
Net Loss Rate =
VAR ResolvedLoans =
    CALCULATETABLE(
        Fact_Loan,
        Dim_LoanStatus[RiskFlag] IN {"Resolved-Good", "Resolved-Bad"}
    )
VAR TotalNetLoss =
    SUMX(
        ResolvedLoans,
        IF(
            RELATED(Dim_LoanStatus[RiskFlag]) = "Resolved-Bad",
            Fact_Loan[funded_amnt] - Fact_Loan[total_rec_prncp]
                - (Fact_Loan[recoveries] - Fact_Loan[collection_recovery_fee]),
            0
        )
    )
VAR TotalFunded = SUMX(ResolvedLoans, Fact_Loan[funded_amnt])
RETURN
    DIVIDE(TotalNetLoss, TotalFunded)
```

---

## Yield

**Annualized Net Loss Rate.** A 60-month loan builds up more loss than a 36-month one at the same risk, just by being around longer. Dividing by the average term in years puts them on the same footing. It's straight-line, which is a simplification.

```dax
Annualized Net Loss Rate = DIVIDE([Net Loss Rate], AVERAGE(Fact_Loan[Term]) / 12)
```

**Risk-Adjusted Yield.** Rate earned minus annualised loss. It ignores time value and assumes interest is collected in full; that's stated in the README's limitations.

```dax
Risk-Adjusted Yield = [Weighted Avg Rate] - [Annualized Net Loss Rate]
```

---

## Mispricing

The core idea: use LendingClub's own grade as the benchmark. Grade is meant to price risk already, so a purpose is "mispriced" if it loses more than the rest of its grade.

```dax
Grade Avg Net Loss Rate = CALCULATE([Net Loss Rate], REMOVEFILTERS(Dim_Purpose))

Mispricing Gap = [Net Loss Rate] - [Grade Avg Net Loss Rate]
```

`REMOVEFILTERS(Dim_Purpose)` removes only the purpose filter, so in a grade × purpose matrix each cell is compared with its whole grade. Any other filter on the page (year, state, term) still applies to both sides, so the comparison stays like for like. An earlier version used `ALLEXCEPT(Fact_Loan, Dim_Grade)`, which also stripped those other filters from the benchmark.

The gap only means something per grade. On a card with no grade in context it comes out at zero, because the benchmark becomes the whole book.

---

## Vintage cohorts (Page 3)

Two calculated columns on `Fact_Loan`. The public file has no monthly performance history, so seasoning is approximated from the last payment date.

```dax
Months on Book = DATEDIFF(Fact_Loan[issue_d], Fact_Loan[last_pymnt_d], MONTH)

Is Vintage Mature =
DATEDIFF(Fact_Loan[issue_d], MAX(Fact_Loan[last_credit_pull_d]), MONTH) >= Fact_Loan[term]
```

A loan is mature once its full term has passed, measured against the latest date in the data rather than `TODAY()`. Otherwise, years after the extract, almost everything would count as mature.

**Cumulative Bad Rate by MOB.** Drives the vintage curves: for each months-on-book value, the share of the cohort that had gone bad by then.

```dax
Cumulative Bad Rate by MOB =
VAR CurrentMOB =
    CALCULATE(MAX(Fact_Loan[Months on Book]), ALL(Dim_Date[Year]))
VAR VintageTotal =
    CALCULATE(COUNTROWS(Fact_Loan), ALL(Fact_Loan[Months on Book]))
VAR BadByThisMOB =
    CALCULATE(
        COUNTROWS(Fact_Loan),
        FILTER(ALL(Fact_Loan[Months on Book]), Fact_Loan[Months on Book] <= CurrentMOB),
        Dim_LoanStatus[RiskFlag] = "Resolved-Bad"
    )
RETURN DIVIDE(BadByThisMOB, VintageTotal)
```

**Mature Loan Count.** 820K loans.

```dax
Mature Loan Count = CALCULATE(COUNTROWS(Fact_Loan), Fact_Loan[Is Vintage Mature] = TRUE)
```

**Best and Worst Mature Vintage.** The year with the lowest or highest charge-off rate among mature cohorts (2009 at 13.69%, 2007 at 26.20%).

```dax
Worst Mature Vintage =
VAR WorstRate =
    MAXX(
        VALUES(Dim_Date[Year]),
        CALCULATE([Charge-off Rate], Fact_Loan[Is Vintage Mature] = TRUE)
    )
VAR WorstYear =
    CALCULATE(
        MIN(Dim_Date[Year]),
        FILTER(
            VALUES(Dim_Date[Year]),
            CALCULATE([Charge-off Rate], Fact_Loan[Is Vintage Mature] = TRUE) = WorstRate
        )
    )
RETURN WorstYear & "  ·  " & FORMAT(WorstRate, "0.00%")
```

`Best Mature Vintage` is the same pattern with `MINX`.

**Vintage Drift 2011→2016.** Both years are fully mature, so it's a fair comparison: +2.10 percentage points.

```dax
Vintage Drift 2011→2016 =
CALCULATE([Charge-off Rate], Dim_Date[Year] = 2016, Fact_Loan[Is Vintage Mature] = TRUE)
- CALCULATE([Charge-off Rate], Dim_Date[Year] = 2011, Fact_Loan[Is Vintage Mature] = TRUE)
```

---

## Recommendation (Page 4)

The recommendation covers `small_business` and `major_purchase` in grades A–E (the README explains why F and G are left out). The grade and purpose filters live inside the measures, so they can't be lost if someone clears a visual filter.

**Estimated Excess Loss: $31.31M.** For each grade × purpose cell, the gap times that cell's exposure, then summed. It has to be worked out cell by cell; one blended gap times total exposure comes out at roughly zero, because gaps above and below the grade average cancel out.

```dax
Estimated Excess Loss =
CALCULATE(
    SUMX(
        SUMMARIZE(Fact_Loan, Dim_Grade[grade], Dim_Purpose[purpose]),
        [Mispricing Gap] * CALCULATE(SUM(Fact_Loan[funded_amnt]))
    ),
    Dim_Purpose[purpose] IN {"small_business", "major_purchase"},
    Dim_Grade[grade] IN {"A", "B", "C", "D", "E"}
)
```

**Segment Exposure ($985.27M) and Segment Mispricing Gap (3.18%).**

```dax
Segment Exposure =
CALCULATE(
    SUM(Fact_Loan[funded_amnt]),
    Dim_Purpose[purpose] IN {"small_business", "major_purchase"},
    Dim_Grade[grade] IN {"A", "B", "C", "D", "E"}
)

Segment Mispricing Gap = DIVIDE([Estimated Excess Loss], [Segment Exposure])
```

**Excess Loss Uplift (bps): 9.21.** The estimate as a share of the whole book.

```dax
Excess Loss Uplift (bps) =
DIVIDE(
    [Estimated Excess Loss],
    CALCULATE(SUM(Fact_Loan[funded_amnt]), ALL(Fact_Loan))
) * 10000
```

**Excess Loss Contribution.** The row-level figure in the breakdown table (10 rows). It has no filters of its own, because the table supplies them.

```dax
Excess Loss Contribution = [Mispricing Gap] * CALCULATE(SUM(Fact_Loan[funded_amnt]))
```

### Benchmark sensitivity

A What-if parameter (0–50%, steps of 5) raises the grade-peer benchmark and recalculates the estimate. It answers "how much worse would the peers have to be before this gap disappears?" At 20%, the estimate falls to $4.89M.

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
    Dim_Purpose[purpose] IN {"small_business", "major_purchase"},
    Dim_Grade[grade] IN {"A", "B", "C", "D", "E"}
)
```

**Sensitivity Narrative.** Shown in a card, since a text box can't display a measure.

```dax
Sensitivity Narrative =
"If grade peers' losses were " & FORMAT(SELECTEDVALUE('Charge-off Stress %'[Charge-off Stress %], 0), "0")
& "% higher, the estimated excess loss would fall to $" & FORMAT(DIVIDE([Stressed Excess Loss], 1000000), "0.00")
& "M (base case: $" & FORMAT(DIVIDE([Estimated Excess Loss], 1000000), "0.00")
& "M). The finding is sensitive to the benchmark; treat it as directional support for a cautious pilot, not a guaranteed figure."
```
