# Telecom Churn Dashboard - Power BI

## 1. Overview

This project turns the BigML Telecom Churn dataset (`churn-bigml-20.csv`) into a two-page Power BI dashboard exploring customer churn drivers.

**Source data:** 667 customer rows, 20 original columns - no missing values, no duplicate rows.

**Overall churn rate:** 14.2% (95 of 667 customers churned).

## 2. Analytical question

> **What customer characteristics and behaviors are associated with churn, and which segments should retention teams prioritize?**

This breaks down into the specific sub-questions the dashboard answers:

1. What is the overall churn rate, and how does it vary by state?
2. Does having an international plan or voice mail plan correlate with churn?
3. Is there a relationship between customer service calls and churn?
4. Do higher-usage (total minutes) customers churn more, less, or the same?
5. Does account tenure relate to churn risk?

## 3. Data cleaning

Performed in Power Query Editor (`Home > Transform Data`) before loading to the model.

| Step | Action | Why |
|---|---|---|
| Type verification | Confirmed each column matched its expected type (text, whole number, decimal, boolean) on import | Wrong types break aggregations and DAX functions |
| Null / duplicate check | Verified 0 nulls and 0 duplicate rows via column quality view and `Remove Duplicates` | Dataset was already clean, but this step is always run defensively |
| Text standardization | Trimmed whitespace on `State`, `International plan`, `Voice mail plan` | Prevents silent mismatches in filters/measures (e.g. `"Yes "` ≠ `"Yes"`) |

## 4. Data preparation (feature engineering)

New columns added in Power Query, all set to their correct numeric/text type before loading:

| New column | Built from | Purpose |
|---|---|---|
| `International Plan Flag` | Conditional column on `International plan` (Yes → 1, No → 0) | Numeric version for averaging |
| `Voice Mail Plan Flag` | Conditional column on `Voice mail plan` | Numeric version for averaging |
| `Churn Flag` | Custom column: `if [Churn] = true then 1 else 0`, **explicitly set to Whole Number type** | Needed for `Churn Rate` measure — this column defaulting to text was the root cause of an `AVERAGE cannot work with values of type String` error during build, fixed by re-applying the Whole Number type as the last step for this column |
| `Tenure Group` | Conditional column binning `Account length` into 0–50 / 51–100 / 101–150 / 150+ days | Turns 232 distinct raw values into a usable slicer |
| `Total Minutes` | Sum of day + evening + night + international minutes | Single high-level usage metric |
| `Total Calls` | Sum of day + evening + night + international calls | Single high-level call-volume metric |
| `Service Calls Bucket` | Conditional column binning `Customer service calls` into 0 / 1 / 2 / 3+ calls | 3+ service calls is a known churn risk signal; bucket reads cleaner than 9 raw values on a chart |
| `Index` | Index column (`Add Column > Index Column > From 1`) | Gives the scatter chart a unique per-row identifier so it plots one dot per customer instead of aggregating |

**Note on sort order:** `Tenure Group` and `Service Calls Bucket` are text categories (e.g. "0–50 days") that sort alphabetically by default, which puts them in the wrong order on charts. A numeric sort-by column was added for each and applied via each visual's `Sort by` option.

## 5. DAX measures

```DAX
Churn Rate = AVERAGE(Churn_Cleaned[Churn Flag])

Total Customers = COUNTROWS(Churn_Cleaned)

Churned Customers = SUM(Churn_Cleaned[Churn Flag])

Avg Customer Service Calls = AVERAGE(Churn_Cleaned[Customer service calls])

Intl Plan Churn Rate =
CALCULATE(
    AVERAGE(Churn_Cleaned[Churn Flag]),
    Churn_Cleaned[International plan] = "Yes"
)
```

`Churn Rate` and `Intl Plan Churn Rate` are formatted as Percentage.

## 6. Dashboard design

### Page 1 - Overview

| Visual | Fields | Purpose |
|---|---|---|
| KPI cards (×4) | `Churn Rate`, `Total Customers`, `Churned Customers`, `Avg Customer Service Calls` | Headline metrics at a glance |
| Filled map | Location: `State`; shading via Format > Fill colors > fx, gradient on `Churn Rate` | Geographic churn variation |
| Clustered column chart | Axis: `Tenure Group`; Values: `Churn Rate` | Churn risk by customer tenure |
| Slicers (×4) | `Tenure Group`, `Service Calls Bucket`, `International plan`, `Voice mail plan` | Cross-page filtering |

### Page 2 - Behavior drivers

| Visual | Fields | Purpose |
|---|---|---|
| Clustered column chart | Axis: `Service Calls Bucket`; Values: `Churn Rate` | Confirms service calls as a churn signal |
| Clustered column chart | Axis: `International plan`; Values: `Churn Rate` | Plan-type effect on churn |
| Clustered column chart | Axis: `Voice mail plan`; Values: `Churn Rate` | Plan-type effect on churn |
| Scatter chart | X axis: `Total Minutes` (Don't summarize); Y axis: `Customer service calls`; Legend: `Churn`; Values/Details: `Index` (Don't summarize) | Per-customer view of usage × service calls, colored by churn outcome |
| Slicers (×4) | Same four as Page 1, copied for consistency | Cross-page filtering |

## 7. Key findings from the built dashboard

- **International plan customers churn far more** — roughly 35% churn rate vs. ~10% for customers without one, the single strongest visual signal in the dashboard.
- **3+ customer service calls is a strong churn predictor** — churn rate drops sharply and steadily from the 3+ calls bucket down to 0 calls.
- **Voice mail plan customers churn less** — customers without a voice mail plan churn at roughly double the rate of those with one.
- **Newer and mid-tenure customers (0–150 days) churn more than long-tenure customers (150+ days)** — retention risk is front-loaded early in the customer lifecycle.
- **No strong standalone relationship between total minutes and churn** in the scatter — churned customers (dark blue) are scattered across the full usage range rather than clustering at one extreme, suggesting usage volume alone isn't a reliable churn predictor; it likely needs to be combined with service calls or plan type to be useful.

## 8. Known issues / possible follow-ups

- The `Tenure Group` and `Service Calls Bucket` slicer tiles currently truncate to "150" instead of "150+ days" in some views — worth widening the slicer tiles or shortening the label text.
- The map's color gradient is currently using default endpoint colors (not yet switched to a single light-to-dark hue) — revisit Format > Fill colors > fx to set explicit min/max colors for clearer at-a-glance reading.
- Consider adding page navigation buttons between Page 1 and Page 2 for easier viewer navigation.
- Consider renaming any remaining auto-generated visual titles (e.g. "Sum of X by Y" style) to plain-language titles for readability.

## 9. Tools used

- **Power BI Desktop** - Power Query Editor for cleaning/prep, DAX for measures, report canvas for visuals
- **Source file:** `churn-bigml-20.csv`
