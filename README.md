# Healthcare Analytics: Risk-Stratified Outcomes and Forward Revenue Projection

A connected chain of statistical techniques applied to a synthetic 100K-patient EHR cohort. The pipeline moves from data screening through feature engineering, dimensionality reduction, clustering, hypothesis testing, and forward-looking simulation, and translates the analytical findings into a single actionable business recommendation.

## Headline finding

| Outcome | Low risk | High risk | Effect | Verdict |
|---|---|---|---|---|
| **30-day readmission** | 5.7% | 19.9% | **3.5x relative risk**, +14.2 pp absolute (95% CI [+10.7, +17.6] pp), z = 8.74, p < 1e-15 | Strongly reject H₀ |
| **Cost per stay** | $17,665 | $18,673 | +$1,009 (Cohen's d = 0.06), p = 0.148, bootstrap 95% CI on the gap [–$412, +$2,358] | Fail to reject H₀ |

The actionable lever is **readmission, not cost**. A care-transition program targeting the 2,484-patient High-risk stratum could plausibly prevent ~50 readmissions per year per 2-percentage-point reduction.

## Method, at a glance

1. **Data screening** — Tukey IQR outlier flagging plus physiological consistency checks (`systolic_bp > diastolic_bp`, `icu_days ≤ length_of_stay_days`). 1,251 rows flagged at source; 95 flow into the merge and are excluded.
2. **Feature engineering** — `comorbidity_count`, `pulse_pressure`, `daily_charge`, plus AHA/WHO clinical binning of age, BP, and BMI.
3. **PCA + suitability tests** — Bartlett rejects identity (χ² = 250,410, p ≈ 0) but KMO = 0.511 (miserable band). Kaiser retains 7 components for only 69% cumulative variance. *Documented diagnostic, not a working compression.*
4. **K-Means** — Silhouette peaks at k = 3, score 0.143 (problematic band). The partition mostly recovers the engineered `high_cost_flag`; not a clinical persona discovery.
5. **Pivot to a rule-based stratification** — `charlson_index ≥ 2 and age ≥ 65` for High, the inverse for Low, Medium otherwise. Produces a clean monotonic gradient on demographics, vitals, *and* on outcomes that were not used to define the strata.
6. **A/B testing** — Welch's t-test on cost (null), two-proportion z-test on readmission (reject), confounder panel, stratified confirmation on `comorbidity_count`. Both unstratified findings survive the stratification.
7. **Bootstrap + Monte Carlo** — 5,000-iteration percentile CIs on the mean cost, readmission rate, and the high-vs-low cost gap. Monte Carlo projects next-year revenue under a 10% volume-growth scenario: $134.6M – $140.4M (95% CI), cross-validated against a closed-form analytical mean (0.01% relative difference) and a parametric lognormal CI (within 0.5%).
8. **Conclusions** — Translate the analytics into a single business recommendation.

## Running it

### Local
```bash
git clone <this-repo>
cd <this-repo>
python -m venv .venv && source .venv/bin/activate  # optional
pip install -r requirements.txt
# Drop patients.csv and patient_outcome.csv into ./data/
jupyter lab healthcare_risk_stratification.ipynb
```

### Colab
1. Upload `healthcare_risk_stratification.ipynb` to Colab.
2. The first data cell auto-mounts your Drive when running on Colab and looks for the CSVs at `/content/drive/MyDrive/DSO 545/Final/`. Adjust `DATA_DIR` in that cell if your path differs.

## Repo layout

```
.
├── healthcare_risk_stratification.ipynb   ← main analysis
├── README.md
├── requirements.txt
├── .gitignore
└── data/                                   ← put the two CSVs here (or a generator script)
    ├── patients.csv
    └── patient_outcome.csv
```

## Tech stack

pandas · NumPy · scikit-learn · statsmodels · scipy · factor_analyzer · seaborn · matplotlib

## Data

Synthetic EHR cohort (~100,000 adult US patients, 2018–2024), calibrated against CDC NHANES (vitals), CDC NCHS (disease prevalence), and CMS (hospitalization and readmission) benchmarks. No real patients; safe to commit.

## Limitations

- Synthetic generator drew most features approximately independently, suppressing the correlations PCA and K-Means need to work cleanly. The diagnostic finding (KMO 0.511, silhouette 0.143) is itself a key takeaway.
- We identified confounders empirically but did not formally control for them. Regression adjustment (or matched-pair design) is the next step for any causal claim.
- The Monte Carlo projection assumes the cost distribution stays the same year over year; regime changes (new reimbursement rules, demographic or payer-mix shifts) are not modeled.

## Author

Olivia Cai — May 2026
