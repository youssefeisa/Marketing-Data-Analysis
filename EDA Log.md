## EDA Log: Customer Personality Marketing Dataset (2,240 rows)

### Phase 1 — Understanding
- Rows / cols: 2240 rows × 28 columns (original)
- Row represents: one customer record

### Phase 2 — Cleaning
- Duplicates: 0 exact duplicates; ID confirmed unique (2240/2240)
- Dt_Customer: converted text → datetime (Fixed)
- Categorical text (Education, Marital_Status, Country): normalized via strip + title case (Fixed)
  - Side effect: "USA" → "Usa" (formatting artifact only)
- Income: 24 nulls (~1.07%) → filled with median (Fixed — right-skewed distribution)
- Year_Birth: values < 1920 (impossible) → NaN → filled with median (Fixed)
- Country rare categories (Mexico=3, India=148, etc.): left as-is, caveat on sample size [Open — resolved]
- Marital_Status cleanup [Open — resolved]: Alone/Yolo/Absurd → merged into "Single";
  Together → merged into "Married"

### Phase 3 — Univariate
- Education: 5 categories, smallest Basic=54 (2.4%) — no rare-category action needed
- Marital_Status: 4 clean categories post-merge; Country: 8 categories, kept as-is
- Skew-based mean/median rule applied:
  - Median: Income (6.80), MntWines, MntFruits, MntMeatProducts, MntFishProducts,
    MntSweetProducts, MntGoldProds, NumDealsPurchases, NumWebPurchases, NumCatalogPurchases
  - Mean: Year_Birth, Recency, NumStorePurchases, NumWebVisitsMonth
- Visual outliers flagged for Phase 5: Income, MntMeatProducts, NumCatalogPurchases,
  NumDealsPurchases

### Phase 4 — Bivariate
- Numeric x Numeric: Mnt* columns strongly correlate with each other and with
  NumCatalogPurchases/NumStorePurchases (r up to 0.72). All numeric columns negatively
  correlate with NumWebVisitsMonth (strongest: Income, r=-0.55)
- Categorical x Numeric (ANOVA): Education significant for 13/14 numeric columns.
  Marital_Status only significant for Year_Birth. Country: no significant relationships.
- Categorical x Categorical (chi-square): no significant associations
- CAVEAT: Education correlates with both Income and spending columns — numeric-numeric
  correlations involving Income/spend may be partly confounded by Education. Flagged
  for business-acumen-lens.

### Phase 5 — Outliers
- Income: extreme value 666666 (repdigit pattern, likely data-entry error) → treated
  as NaN, re-imputed with median [Fixed, per Youssef's call]
- Remaining outliers (Income ~160k range, MntMeatProducts, NumCatalogPurchases,
  NumDealsPurchases): checked against Education/Marital_Status — no segment
  concentration found, proportions match overall sample → kept as-is, reported via
  median instead of mean [Open — resolved]

### Phase 6 — Features created
- Year_Joined, Month_Joined, Tenure_Days (from Dt_Customer)
- TotalMnt (sum of Mnt* columns), TotalPurchases (sum of Num*Purchases columns)
- HasKids (flag), TotalCampaignsAccepted, RespondedAnyCampaign
- Frequency = TotalPurchases, Monetary = TotalMnt, Recency_Score = Recency (RFM-style)
- Age (2014 - Year_Birth), Age_Group (quartile bins: 18-37, 38-44, 45-55, 56-74)
- Income_Tier (quartile bins: ~1.7k-35.5k, 35.5k-51.4k, 51.4k-68.3k, 68.3k-162.4k)

### Open questions asked, and what Youssef chose
- Country rare categories → left as-is, documented caveat
- Marital_Status weird categories → merged (Alone/Yolo/Absurd → Single, Together → Married)
- Income outlier 666666 → treated as error, NaN + median re-fill
- Remaining outliers (Income, MntMeatProducts, NumCatalogPurchases, NumDealsPurchases)
  → kept as-is, report median
- Age_Group binning → quartile-based
- Income_Tier binning → quartile-based
- Feature categories to build → all (date parts, aggregation, flags, RFM, binning)