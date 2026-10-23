# Data Model & Quality Notes

This file records the modeling and interpretation decisions used in the Candy Distributor dashboard. It is a narrative of the build process, **not** an exported audit log from the Power BI `.pbix`.

## Reporting grain and scope

- Primary analysis grain: **sales line** in `Candy_Sales`.
- Time period: **2021–2024**, based on `Order Date`.
- Fact rows: **10,194**.
- Master products: **15**.
- Master factories: **5**.
- Division categories: Chocolate, Other, Sugar.
- Other source files reviewed in the preparation process included **Candy_Targets** and **uszips**; they were not treated as core dimensions of the final analytical model described here.

## Relationship design

```text
Candy_Factories[Factory] (one) ────────> Candy_Products[Factory] (many)
Candy_Products[Product ID] (one) ──────> Candy_Sales[Product ID] (many)
Dim_Date[Date] (one) ──────────────────> Candy_Sales[Order Date] (many)
```

Filter direction is single from dimensions to the fact table. Product-level division and factory attributes are drawn from `Candy_Products`.

## Data-quality handling

### Locale-sensitive numeric values

During data import, fields containing decimal points could be interpreted incorrectly under an Indonesian locale. Numeric conversions were corrected in Power Query using a suitable locale (`English (United States)`) for columns such as Sales, Cost, Gross Profit, and price/cost fields. Verify this in the original query steps if reproducing from raw CSV.

### Date integrity

`Order Date` anchors the reporting calendar. `Ship Date` values were flagged during project preparation as inconsistent for shipping-delay analysis; therefore, no claim is made about actual delivery duration or route efficiency.

### Division consistency

Some sales-table division labels were inconsistent with the product master. Analyses intentionally use `Candy_Products[Division]` as the consistent classification source.

### Geographic interpretation

The report uses distributor-defined regions (Pacific, Atlantic, Interior, Gulf), US states visualized in a shape map, and location labels in source data. The count of distinct `State/Province` categories (59 on the report page) is **not** a count of US states, and the US map excludes non-US province plotting.

### Target and order-grain limitations

The goals dataset was not used for target achievement because temporal comparability was unclear. Line count is not assumed to equal verified unique orders, so AOV and order-frequency claims were deliberately omitted.

## KPI definitions and context

| KPI | Definition | Note |
|---|---|---|
| Total Sales | `SUM(Candy_Sales[Sales])` | Revenue in source currency (USD) |
| Total Cost | `SUM(Candy_Sales[Cost])` | Recorded cost, not full operating expenditure |
| Total Gross Profit | `SUM(Candy_Sales[Gross Profit])` | Ties to Sales less Cost |
| Gross Margin % | Gross Profit / Sales | Ratio of totals, not an average of row percentages |
| Total Units Sold | `SUM(Candy_Sales[Units])` | Unit volume |
| Active Products | Distinct product IDs in sales | Filter-sensitive |
| Sales per Unit | Sales / Units | Unit-weighted average realized revenue |
| Gross Profit per Unit | Gross Profit / Units | Unit-weighted average gross profit |
| Top N Profit Share | Sum of top N product profits / total in selected scope | Filter-sensitive Pareto |
| Cumulative Gross Profit % | Running cumulative product profit / total in selected scope | Sort products by gross profit descending |

## Baselines used for QA

| Metric | Value |
|---|---:|
| Sales | $141,783.63 |
| Cost | $48,340.83 |
| Gross Profit | $93,442.80 |
| Margin | 65.91% |
| Units | 38,654 |
| Products | 15 |
| 2024 YoY sales growth | 27.43% |
| Top 5 profit share | 95.06% |

Sales by year:

| Year | Sales |
|---|---:|
| 2021 | $28,953.55 |
| 2022 | $29,002.65 |
| 2023 | $36,859.32 |
| 2024 | $46,968.11 |

Sales by region:

| Region | Sales |
|---|---:|
| Pacific | $46,301.53 |
| Atlantic | $41,197.24 |
| Interior | $32,037.60 |
| Gulf | $22,247.26 |

## Reporting caveats

- `Rb` on the screenshots means Indonesian **thousands** in numeric display formatting (for example `$48,34Rb` means about `$48.34K` USD), not a currency conversion.
- Product-level market comparisons should not be interpreted causally.
- A selected product's division benchmark must remain a division-wide comparison, not a filtered value for only the selected SKU.
- The default YoY KPI represents the latest full year versus its predecessor, not compound annual growth.
- We did not independently reopen the creator's final `.pbix` or its Power Query steps for this documentation package; reconcile any future changes against the saved model.
