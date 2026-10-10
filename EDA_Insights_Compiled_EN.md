# EDA Insights — Compiled & Verified (All Phases)

**Source:** Numbers extracted directly from `EDA.ipynb` outputs after a full sequential re-run (execution_count 1 through 41, no gaps).
**Note:** The previous `EDA_Log.md` contained numbers that did not match the actual notebook (it has been deleted and is fully disregarded).
**Final dataset:** 2,240 rows × 46 columns (28 original + 18 engineered), 0 nulls, 0 duplicates.

---

## Phase 1 — Understanding

- 2,240 customers, 28 original columns, unique ID per row
- The Income column name had leading/trailing whitespace: `" Income "` — a naming issue
- Income: 24 missing values (1.07%) — the only column with nulls
- Income max = **666,666** (75th percentile only 68,522) — a blatant outlier
- Year_Birth: range 1893–1996 — contains logically impossible values
- Wine (mean 304) and Meat (mean 167) are by far the two largest spending categories; all other categories sit between 26–44
- Store is the strongest purchase channel (mean 5.79), Catalog the weakest (mean 2.66)
- Legacy campaign acceptance rates (Cmp1–5): 1.3%–7.5%; Response (the most recent): **14.9%**
- Dt_Customer span: 2012-07-30 to 2014-06-29

---

## Phase 2 — Cleaning

| Decision | Details | Type |
|---|---|---|
| Column name fix | `" Income "` → `"Income"` | Fixed |
| Duplicates | 0 exact, 0 ID duplicates | Fixed |
| Dt_Customer | Converted to datetime | Fixed |
| Income nulls (24) | Imputed with median = **$51,381.50** | Fixed |
| Year_Birth (3 values <1920: 1893, 1899, 1900) | Set to NaN, then imputed with median (1970) | Fixed |
| String normalization | Education, Marital_Status, Country | Fixed |
| **Marital_Status consolidation** | Alone/Yolo/Absurd→Single, Together→Married | **Open** ⚠️ |

**Final baseline after all consolidation (from Phase 3):**
- Marital_Status: Married 64.5%, Single 21.7%, Divorced 10.4%, Widow→Other 3.4%
- Education: Graduation 50.3%, Phd 21.7%, Master 16.5%, 2N Cycle 9.1%, Basic→Other 2.4%
- Country: Spain 48.9%, Saudi Arabia 15.0%, Canada 12.0%, Australia 7.1%, India 6.6%, Germany 5.4%, Usa 4.9%→Other, Mexico 0.1%→Other
  - ⚠️ **USA (4.9%) sat right on the edge of the 5% threshold** — merging it deserves discussion given its commercial importance

---

## Phase 3 — Univariate Analysis

**Skew (10 of 14 columns right-skewed, |skew| > 1):**

| Column | Skew | Mean | Median | Outliers (IQR) |
|---|---|---|---|---|
| Income | **6.80** | 52,237.98 | 51,381.50 | 8 |
| NumDealsPurchases | 2.42 | 2.33 | 2.00 | 86 |
| MntSweetProducts | 2.14 | 27.06 | 8.00 | 248 |
| MntFruits | 2.10 | 26.30 | 8.00 | 227 |
| MntMeatProducts | 2.08 | 166.95 | 67.00 | 175 |
| MntFishProducts | 1.92 | 37.53 | 12.00 | 223 |
| MntGoldProds | 1.89 | 44.02 | 24.00 | 207 |
| NumCatalogPurchases | 1.88 | 2.66 | 2.00 | 23 |
| MntWines | 1.18 | 303.94 | 173.50 | 35 |
| NumWebPurchases | 1.38 | 4.08 | 4.00 | 4 |

**Symmetric (|skew| ≤ 1):** Year_Birth (-0.09), Recency (-0.002), NumStorePurchases (0.70), NumWebVisitsMonth (0.21, 8 outliers)

**Core insight:** Niche spending categories (Sweet/Fruits/Fish/Gold) carry proportionally far more outliers than Wine despite their small size — spending is concentrated in a small minority of customers, unlike Wine, which is more widespread and stable.

---

## Phase 4 — Bivariate Analysis

### Direct answer: what correlates with Web Purchases?

| Variable | r with NumWebPurchases |
|---|---|
| MntWines | **0.542** |
| NumStorePurchases | 0.503 |
| MntGoldProds | 0.422 |
| Income | 0.381 |
| NumCatalogPurchases | 0.378 |
| **NumWebVisitsMonth** | **-0.056 (essentially zero)** |

**Critical insight:** Website visits do not predict actual web purchases (r≈0). Any "conversion rate" metric built on visit counts is statistically unsupported.

### The broader pattern
NumWebVisitsMonth is negative against almost everything: Income (-0.550), MntMeatProducts (-0.539), NumCatalogPurchases (-0.520) — customers who visit the site more tend to have lower income and lower spending overall, not higher loyalty.

### Strongest correlation in the entire dataset
MntMeatProducts × NumCatalogPurchases = **0.724**

### ANOVA (categorical × numeric)
13 of 42 pairs are statistically significant, and **12 of the 13 involve Education** (the sole exception: Marital_Status × Year_Birth). Country and Marital_Status have virtually no effect on spending or channel usage.

> ⚠️ **Guardrail:** Education is the real confound behind any Income↔Spending relationship, not Marital_Status or Country.

### Chi-square (categorical × categorical)
Marital_Status × Country: p = **0.0496** (extremely borderline, treat with caution). Education × Country and Education × Marital_Status: not significant.

---

## Phase 5 — Outliers Deep Dive

**1,244 outliers across 11 of 14 columns** (zero outliers in: Year_Birth, Recency, NumStorePurchases).

Highest rates: MntSweetProducts (11.07%), MntFruits (10.13%), MntFishProducts (9.96%), MntGoldProds (9.24%), MntMeatProducts (7.81%)

### ⚠️ Methodological guardrail: the "concentration" check has a bias
The code's rule (>50% = concentrated) ignores the fact that Married already makes up 64.5% of the entire sample. Comparing against baseline:

**False concentration** (roughly matches population share, not a real signal): MntFruits, MntMeatProducts, MntFishProducts, MntSweetProducts, MntGoldProds, NumCatalogPurchases (all show Married at 60–65%, close to baseline)

**Genuine concentration** (clear deviation from baseline):
- Income outliers: Phd 50.0% (baseline 21.7%), Married 87.5% (baseline 64.5%)
- MntWines outliers: Phd 54.3%
- NumWebPurchases outliers (n=4 only): Single 75% (baseline 21.7%) — a strong signal, but from a very small sample

### Handling of Income = 666,666
- Set to NaN, then imputed with the median ($51,381.50)
- The remaining 7 outliers (income $157k–$162k) were deliberately kept as legitimate high earners
- Income max after treatment = **$162,397**
- Shape remained (2240, 28) — zero rows dropped

---

## Phase 6 — Feature Engineering

18 new features were added (Date Parts, Aggregations, Flags, RFM-style, Binning, RFM Segmentation) → 46 total columns.

### RFM Segmentation — Customer Value Ladder

| RFM_sum | n | Avg Income | Avg Age | Avg Spending | Avg Purchases |
|---|---|---|---|---|---|
| 3 (weakest) | 129 | $34,492 | 42.3 | $39.32 | 5.52 |
| 9 | 317 | $66,579 | 48.0 | $1,084.03 | 21.11 |
| 12 (strongest) | 51 | $77,545 | 48.4 | $1,443.00 | 26.47 |

**Core insight:** The gap between the weakest and strongest segment: Income 2.25x, **Age just 6 years (nearly flat)**, but **Spending 36.7x**. Age is practically useless as a determinant of customer value; actual spending behavior is what drives it.

### Other figures
- TotalMnt: Mean $605.80 vs Median $396.00 (large gap, confirms skew)
- HasKids: **71.5%** of customers (an overwhelming majority, not a minority)
- RespondedAnyCampaign (Cmp1–5 only, excluding Response): 20.7%
- **Critical comparison:** Response alone = 14.9%, vs. 5 legacy campaigns combined = only 20.7% — Response as a single campaign nearly matches the combined performance of 5 prior campaigns
- Income_Tier Q4_Highest max = $162,397 (confirms the outlier treatment propagated correctly into derived columns)

---

## Phase 7 — Handoff (Technical QA only, not new analysis)

- Final dataset: 2,240 rows × 46 columns, 0 nulls, 0 duplicates (exact & ID-level)
- 37 numeric columns, 3 categorical, 1 datetime
- File saved: `marketing_data_cleaned.csv` (360.93 KB)
- Phase 7 here is a technical save/confirmation step only — actual Business Interpretation happened separately via business-acumen-lens, not inside this notebook

---

## Quick map: how these numbers answer the 6 core business questions

1. **Data quality (nulls/outliers):** 38 original nulls (24 Income + 7 Year_Birth + 7 fixed categorical entries), 1,244 IQR outliers across 11 columns, one genuinely fixed extreme value (666,666)
2. **Web Purchase drivers:** MntWines (r=0.542) is the strongest predictor; Web Visits (r≈-0.056) is not meaningfully related
3. **Most successful campaign:** Response alone (14.9%) nearly matches the combined performance of 5 legacy campaigns (20.7%)
4. **Average customer profile:** Age is a weak value determinant (only a 6-year gap across RFM tiers); actual spending is the real driver (36.7x gap)
5. **Category performance:** Wine and Meat dominate and are the most stable; smaller categories (Sweet/Fruits/Fish/Gold) have genuinely heavy tails
6. **Underperforming channels:** Catalog is the weakest channel by volume (mean 2.66), but it's the most strongly correlated with Meat spending (r=0.724) and Income (r=0.587) — a "premium niche" channel, not an outright failing one

---

## Guardrails to carry forward into the Report Phase

1. Education — not Marital_Status/Country — is the real confound behind income/spending-related relationships
2. The "outlier concentration in a segment" check must be compared against the baseline population share, not a fixed 50% threshold
3. Website visits do not predict purchases — any "conversion rate" claim built on visit counts is unsupported
4. The NumWebPurchases outlier segmentation (Single 75%) is based on n=4 only — directional, not conclusive
5. Marital_Status × Country (p=0.0496) is extremely borderline — mention with strong caveats if included as a finding
