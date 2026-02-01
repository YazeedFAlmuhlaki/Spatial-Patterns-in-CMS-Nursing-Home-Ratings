# Spatial Patterns in CMS Nursing Home Ratings (USA)
Predicting low-rated nursing homes and validating spatial clustering with ESDA + spatially aware evaluation.

## Project Summary
This project builds a spatial data science workflow using **CMS nursing home data** across the United States (including AK/HI/territories).  
I create a clean facility-level dataset, perform **non-spatial EDA** and **ESDA** (spatial autocorrelation), then train classification models to predict whether a facility is **low-rated**. To avoid overly optimistic results, I evaluate models using both **random split** and a **spatial block split**.

**Key outputs**
- Clean merged dataset: `data/processed/final_spatial_table.csv`
- EDA + ESDA findings (Global Moran’s I + Local Moran/LISA)
- Baseline and nonlinear models evaluated under spatial holdout
- Threshold-tuned Random Forest to prioritize catching low-rated facilities

---

## Data Sources
Two CMS datasets were used:

1) **Facility Characteristics (CMS-671)**  
   Used as the main feature table.

2) **Provider Information** (subset of columns kept)
   - CMS Certification Number (CCN) / Provider Number
   - Overall Rating
   - Latitude
   - Longitude

**Geography**
- Nationwide coverage: USA including Alaska, Hawaii, and territories (as available in CMS files).

---

## Target Definition
I define the classification target `low_rated` using **Overall Rating**:

- `low_rated = 1` if Overall Rating ∈ {1, 2}
- `low_rated = 0` if Overall Rating ∈ {3, 4, 5}

Handling missing labels:
- Rows with missing Overall Rating were **dropped** (no label imputation).

Class distribution in the final dataset:
- `low_rated = 0`: ~58.16%
- `low_rated = 1`: ~41.84%

---

## Data Cleaning and Validation
### Row/column integrity
Final merged dataset:
- Rows: **14,575**
- Nulls: **0**

### Coordinate validation
Facilities must fall within broad geographic bounds:
- Latitude: 18–72
- Longitude: -180 to -66

Rows outside bounds were dropped.

### Outlier handling (flag + cap)
Some numeric columns contained extreme erroneous values that would distort EDA and modeling.  
I applied a transparent approach:
- Keep raw values
- Create an outlier flag
- Create a capped (winsorized) version using the 99.5th percentile

Examples:
- `Total Residents_capped` and `Medicaid Census_capped` were created with q99.5 caps.
- Outliers were flagged (e.g., `is_outlier_total_residents`, `is_outlier_medicaid_census`).

### Sparse feature engineering (specialized beds)
Many specialized bed columns are extremely sparse (98–99% zeros).  
For these, I created binary availability indicators (0 vs >0), e.g.:
- `has_dialysis_beds`, `has_other_specialized_rehab_beds`, etc.

---

## Exploratory Data Analysis (EDA)
### Numeric insights (examples)
After capping, comparisons by `low_rated` showed meaningful differences:
- `Total Residents_capped`: low-rated facilities tend to be larger (median higher), with overlap.
- `Medicaid Census_capped`: low-rated facilities tend to have higher Medicaid census.

### Categorical insights
- **Hospital Based**
  - Hospital-based facilities are rare (~3.5%) and show notably lower low-rated rate vs non-hospital-based.
- **Ownership Type**
  - Ownership type is strongly associated with low-rated status.
  - Large for-profit categories show higher low-rated rates than the main non-profit category.

A confounding check (Ownership × Hospital Based) suggested the hospital-based association generally persists within major ownership types (small subgroups interpreted cautiously).

---

## ESDA (Exploratory Spatial Data Analysis)
### Spatial setup
- Points created from (Longitude, Latitude)
- Projected to EPSG:3857 for distance-based analysis
- kNN spatial weights used (k=8)
- Nationwide geography produced **3 disconnected components** in weights (expected with distant geographies)

### Global Moran’s I (target clustering)
For `low_rated`:
- Moran’s I: **0.077**
- p-value (permutations): **0.001**
- z-score: **20.2837**

Interpretation: **statistically significant positive spatial autocorrelation**, indicating non-random spatial clustering.

### Moran scatterplot
A Moran scatterplot confirmed a positive relationship between the outcome and its spatial lag.

### Local Moran (LISA)
Local Moran’s I (p < 0.05) identified significant spatial structure for ~13.39% of facilities:
- HH hotspots: ~2.43%
- LL coldspots: ~3.27%
- HL outliers: ~4.22%
- LH outliers: ~3.47%

Note: Since `low_rated` is binary, LISA categories are best interpreted as **where** clusters/outliers occur, not as within-class outcome rates.

---

## Modeling
### Feature set
Predictors included:
- Numeric facility features (including capped versions where appropriate)
- Binary `has_*` service indicators
- Categorical variables (`Hospital Based`, `Ownership Type`) via one-hot encoding

Leakage prevention:
- Excluded: `Overall Rating`, identifiers (`Provider Number`), and any target-derived spatial features (e.g., LISA labels).

### Evaluation strategy
Two splits were used:
1) **Random split (stratified 80/20)**  
2) **Spatial block split (80/20 by 100 km grid blocks)**  
   - Blocks: 1,096 total; test blocks: 220
   - Block overlap: 0 (no spatial leakage)

### Baseline results (spatial split)
- Dummy (most frequent): Recall=0, F1=0
- Logistic Regression: ROC-AUC ~0.6886, PR-AUC ~0.6075, Recall ~0.43, F1 ~0.5074
- Random Forest (default 0.50 threshold): ROC-AUC ~0.6864, PR-AUC ~0.5906, Recall ~0.556, F1 ~0.5702

### Threshold tuning (spatial split)
To prioritize catching low-rated facilities, Random Forest threshold was tuned.

Chosen threshold:
- **0.30** (maximized F1 while keeping high recall)

Final Random Forest @ threshold=0.30 (spatial split):
- ROC-AUC: **0.6864**
- PR-AUC: **0.5906**
- Precision: **0.501**
- Recall: **0.8789**
- F1: **0.6382**
- Predicted positive rate: **0.752**

Interpretation: High recall with many false positives. Suitable for **screening/prioritization** use cases where missing low-rated facilities is costly.


---

## Limitations
- Nationwide point data creates disconnected components in spatial weights due to distant geographies; kNN remains useful but interpretation should acknowledge geography.
- This is an observational study. Associations do not imply causation.
- Facility-level predictors may not capture all drivers of ratings (e.g., staffing details, regional policy differences, patient case mix).

---

## Future Work
- Spatial cross-validation (blocked k-fold) instead of a single holdout.
- Model calibration and decision-curve analysis for screening use cases.
- Add contextual regional covariates (socioeconomic, healthcare access, policy environment).
- Interpretability: permutation importance / SHAP for Random Forest under spatial evaluation.

---

## License / Ethics Note
This project uses public administrative data. The goal is educational and methodological: demonstrating a spatial data science workflow for nationwide facility-level analysis.

---

## Contact
If you have questions or want to collaborate, feel free to reach out.



