# DAX Reference - Selected Measures

These are **representative measures from the dashboard build discussion**, not an automatic extraction from the saved `.pbix`. Reconcile names and DAX with the final model before reusing verbatim.

## Calendar

```dax
Dim_Date =
ADDCOLUMNS(
    CALENDAR(
        MIN('Candy_Sales'[Order Date]),
        MAX('Candy_Sales'[Order Date])
    ),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month Name", FORMAT([Date], "MMMM"),
    "Month Short", FORMAT([Date], "MMM"),
    "Quarter", "Q" & QUARTER([Date]),
    "Year Month", FORMAT([Date], "yyyy-MM"),
    "Year Month Sort", YEAR([Date]) * 100 + MONTH([Date])
)
```

`Month Start` is a calculated **column**, not a measure:

```dax
Month Start = DATE(YEAR('Dim_Date'[Date]), MONTH('Dim_Date'[Date]), 1)
```

## Basic profitability

```dax
Total Sales = SUM('Candy_Sales'[Sales])

Total Cost = SUM('Candy_Sales'[Cost])

Total Gross Profit = SUM('Candy_Sales'[Gross Profit])

Gross Margin % = DIVIDE([Total Gross Profit], [Total Sales], 0)

Total Units Sold = SUM('Candy_Sales'[Units])

Sales per Unit = DIVIDE([Total Sales], [Total Units Sold], 0)

Gross Profit per Unit = DIVIDE([Total Gross Profit], [Total Units Sold])

Cost Share % = DIVIDE([Total Cost], [Total Sales], 0)
```

Format ratios as Percentages and monetary measures as currency. A **100% stacked bar equivalent** can be built with `[Cost Share %]` and `[Gross Margin %]` in a regular stacked chart.

## Product share

```dax
Profit Contribution % =
DIVIDE(
    [Total Gross Profit],
    CALCULATE(
        [Total Gross Profit],
        ALLSELECTED('Candy_Products'[Product Name])
    )
)
```

## Top five contribution

```dax
Top 5 Profit Share % =
VAR Products =
    FILTER(
        ADDCOLUMNS(
            ALLSELECTED('Candy_Products'[Product Name]),
            "@Profit", CALCULATE([Total Gross Profit])
        ),
        [@Profit] > 0
    )
VAR TopProfit =
    SUMX(
        TOPN(
            5,
            Products,
            [@Profit], DESC,
            'Candy_Products'[Product Name], ASC
        ),
        [@Profit]
    )
RETURN
    DIVIDE(TopProfit, SUMX(Products, [@Profit]))
```

The same pattern with `TOPN(1, ...)` and `TOPN(3, ...)` yields the Top 1 and Top 3 KPI cards.

## Cumulative profit share / Pareto

```dax
Cumulative Gross Profit % =
VAR CurrentProfit = [Total Gross Profit]
VAR Products =
    FILTER(
        ADDCOLUMNS(
            ALLSELECTED('Candy_Products'[Product Name]),
            "@Profit", CALCULATE([Total Gross Profit])
        ),
        [@Profit] > 0
    )
VAR CumulativeProfit =
    SUMX(
        FILTER(Products, [@Profit] >= CurrentProfit),
        [@Profit]
    )
VAR TotalProfit = SUMX(Products, [@Profit])
RETURN
    IF(
        ISINSCOPE('Candy_Products'[Product Name])
            && NOT ISBLANK(CurrentProfit),
        DIVIDE(CumulativeProfit, TotalProfit)
    )
```

**Important configuration:** Put **`Candy_Products[Product Name]`** (not a similarly named sales fact column) on the combo chart's X-axis, sort by Gross Profit descending, and put this measure on **Line y-axis**. This field-scope alignment resolved the missing cumulative line during construction. Equal profit values receive a common cumulative threshold under this version of the measure.

## Division benchmark for a single SKU

```dax
Division Gross Margin Benchmark % =
VAR SelectedDivision =
    SELECTEDVALUE('Candy_Products'[Division])
VAR OneProductSelected =
    HASONEVALUE('Candy_Products'[Product ID])
RETURN
    IF(
        NOT OneProductSelected || ISBLANK(SelectedDivision),
        BLANK(),
        CALCULATE(
            [Gross Margin %],
            REMOVEFILTERS(
                'Candy_Products'[Product Name],
                'Candy_Products'[Product ID]
            ),
            KEEPFILTERS(
                'Candy_Products'[Division] = SelectedDivision
            )
        )
    )
```

This measure intentionally returns blank without a single selected product. Compare **product Gross Margin %** vs **Division Gross Margin Benchmark %** by year, while preserving other slicer context.

## Notes on reproducibility

- The measures use `ALLSELECTED` for report-filter-aware product contribution. Cross-highlighting and extra product filters can affect the denominator, so test interactions before interpreting shares.
- The division benchmark assumes a product selector from `Candy_Products` and can require adaptation if filtering directly on `Candy_Sales[Product Name]`.
- Dataset quality decisions should be reviewed before extending the report to delivery-time or revenue-target KPIs.
