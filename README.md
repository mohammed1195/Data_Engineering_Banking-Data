# Loan Score Calculation

## Purpose

This notebook builds a rules-based score from payment history, delinquency and public-record history, and financial/loan attributes. It combines the points into a numeric loan_score and assigns an A–F grade.

This is a custom score in the notebook; the saved code does not calculate or validate a FICO score.

## Scoring flow

~~~mermaid
flowchart LR
    P["Payment history<br/>2 inputs · declared 20%"] --> S["Weighted sum"]
    D["Default history<br/>4 inputs · declared 45%"] --> S
    F["Financial health<br/>4 inputs · declared 35%"] --> S
    S --> L["Final loan_score"]
    L --> G["A–F grade"]
~~~

### Inputs and point scale

- **Payment history:** last payment compared with the monthly instalment; total payment received compared with funded amount.
- **Default history:** delinquencies over two years, public records, public bankruptcies, and recent enquiries.
- **Financial health:** loan status, home ownership, funded amount relative to total high credit limit, and existing grade/sub-grade.

The configured rating points are 0 (unacceptable), 100 (very bad), 250 (bad), 500 (good), 650 (very good), and 800 (excellent). Grade/sub-grade points are also adjusted using multipliers in the notebook.

### Weights as implemented

The notebook multiplies each sum of component points by its declared percentage:

~~~text
payment component   = 0.20 × sum of 2 payment inputs
default component   = 0.45 × sum of 4 default-history inputs
financial component = 0.35 × sum of 4 financial-health inputs
loan_score = payment component + default component + financial component
~~~

![Declared category weights compared with their normalized contribution when all input points are equal](./loan_score_weight_effect.svg)

Because the groups contain different numbers of inputs, these formulas do not produce effective category weights of 20%, 45%, and 35%. If each input is at the same point value, the maximum contribution shares are approximately **11.1% payment**, **50.0% default history**, and **38.9% financial health**. These are formula-derived shares under equal input points, not observed contributions across the saved records.

If finance intended the stated percentages to apply to the three categories, normalize each category first (for example, take the average of its input points or divide its sum by the category's maximum), then apply 20/45/35. Any such change requires re-evaluating the score range and grade thresholds.

### Grade bands in the notebook

- **A:** score greater than 2,500
- **B:** greater than 2,000 through 2,500
- **C:** greater than 1,500 through 2,000
- **D:** greater than 1,000 through 1,500
- **E:** greater than 750 through 1,000
- **F:** 750 or lower

## Saved output snapshot

- The notebook's saved final-table count is **482,842 rows**.
- A displayed example of a C-grade row has component scores of 200, 1,192.5, and 402.5, summing to 1,795.
- The notebook displays sample rows but does not include a complete grade distribution, default rate by grade, or out-of-time validation results. The saved output therefore does not show whether the score ranks future credit risk reliably.

## Important analyst review before interpreting the grades

### 1. Check the delinquency points constant

For delinq_2yrs greater than 5 or a null value, the code assigns unacceptable_grade_pts, configured as **750**. Other rules use unacceptable_rated_pts, configured as **0**, for the unacceptable point level. This looks like a possible grade-threshold-versus-rating-points mix-up and can materially increase a score for high or missing delinquency. Confirm the intended business rule with finance before using the results.

The public-record and bankruptcy rules assign 100 points when their values are null or above five, while the enquiry rule assigns 0 in those cases. Confirm that these different missing/high-value treatments are intentional.

### 2. Define missing and unknown values

The home_pts and grade_pts CASE expressions have no final ELSE branch. An unrecognized value can therefore produce a null component, which can make the total score null and leave it without an A–F grade. Define explicit handling and report missing/unrecognized inputs separately.

### 3. Confirm scoring time and avoid leakage

The financial-health rules include the existing loan grade/sub-grade and loan status, including outcomes such as fully paid, late, or charged off. If this score is meant to assess risk at application time, those fields may contain information that was not available then or may already encode a prior risk assessment. Specify the scoring date and only use fields available at that point.

### 4. Confirm the row grain and joins

The payment calculation joins repayments to loans by loan_id; later steps combine data by member_id. Validate that each input is unique at its intended join key and determine whether the final table is one row per loan or per member. The displayed final count is a row count, not a confirmed count of unique borrowers.

### 5. Treat grade thresholds as provisional

The saved examples do not provide performance by grade. Before using the score to guide credit decisions, compare grades with later repayment outcomes, check stability across time and relevant customer groups, and document the reason codes behind each score. Federal Reserve model-risk guidance discusses validation, outcome analysis, and ongoing monitoring; CFPB guidance explains that adverse-action reasons must be specific and reflect the factors actually considered.

- [Federal Reserve, Revised Guidance on Model Risk Management (2026)](https://www.federalreserve.gov/supervisionreg/srletters/SR2602.htm)
- [CFPB Circular 2022-03, adverse-action reasons for credit decisions](https://www.consumerfinance.gov/compliance/circulars/circular-2022-03-adverse-action-notification-requirements-in-connection-with-credit-decisions-based-on-complex-algorithms/)

## Scope and reproducibility

This README documents the code and saved outputs embedded in Loan-Score_Calculation.ipynb; the Spark/YARN notebook was not rerun. The final count and sample rows describe that saved run only. The notebook depends on the corresponding Hive tables and Spark configuration values being present.
