# Healthcare Analytics: Risk-Stratified Outcomes and Forward Revenue Projection

A connected chain of statistical techniques applied to a synthetic 100K-patient EHR cohort. The pipeline moves from data screening through feature engineering, dimensionality reduction, clustering, hypothesis testing, and forward-looking simulation, and translates the analytical findings into a single actionable business recommendation.

## Headline finding

| Outcome | Low risk | High risk | Effect | Verdict |
|---|---|---|---|---|
| **30-day readmission** | 5.7% | 19.9% | **3.5x relative risk**, +14.2 pp absolute (95% CI [+10.7, +17.6] pp), z = 8.74, p < 1e-15 | Strongly reject H₀ |
| **Cost per stay** | $17,665 | $18,673 | +$1,009 (Cohen's d = 0.06), p = 0.148, bootstrap 95% CI on the gap [–$412, +$2,358] | Fail to reject H₀ |

### What the tests tell us

- **Readmission differs sharply by chronic risk profile.** The 30-day readmission rate is 5.7% in the Low-risk stratum and 19.9% in the High-risk stratum, a 14.2 percentage point gap and a 3.5x relative risk. The two-proportion z-test (z = 8.74, p < 1e-15) decisively rejects the null of equal rates, and the bootstrap 95% CI on the gap [+10.7, +17.6] pp excludes zero by a wide margin. The effect holds inside every level of `comorbidity_count` in the stratified confirmation, so it is not an artifact of how the strata were constructed.

- **Per-stay cost does not differ meaningfully by chronic risk.** Mean charges are $17,665 (Low) vs $18,673 (High), a $1,009 gap with Cohen's d of 0.06 (negligible effect size). Welch's t-test fails to reject the null (two-sided p = 0.148), and the bootstrap 95% CI on the cost gap [-$412, +$2,358] includes zero. Combined with PC3 in the PCA loading negatively on `comorbidity_count`, this points to per-stay cost being driven by acute episode features (DRG mix, ICU exposure, procedure type) rather than by the patient's underlying chronic risk profile.

- **Chronic risk and acute cost are decoupled in this cohort.** The same segmentation that strongly predicts readmission fails to predict per-stay cost. Any intervention strategy needs to separate the question of who is likely to come back (a chronic-risk question) from the question of who runs up the bill on a given stay (an acute-episode question). They are not the same patients.

### Healthcare recommendations

1. **Target the High-risk cohort with a care-transition program.** With 2,484 High-risk admissions per year at a 19.9% readmission rate, a 2 percentage point absolute reduction would prevent roughly 50 readmissions annually. Evidence-backed levers include 7-day post-discharge follow-up, medication reconciliation, transitional-care nurse visits, telehealth check-ins, and social-determinants screening. Because the High-risk readmission rate sits well above the CMS national benchmark of roughly 13 to 15%, this is also a Hospital Readmissions Reduction Program (HRRP) penalty-exposure issue, not just a quality issue.

2. **Do not stratify per-stay cost reduction by chronic risk.** The data does not support the framing that high-risk patients cost more per stay. Cost-reduction efforts should instead be episode-level: targeting high-cost DRGs, length-of-stay reduction on long stays, unplanned-ICU avoidance, and procedure-specific supply chain savings. Mixing chronic-risk segmentation into per-stay cost strategy will misallocate effort.

3. **Measure the care-transition program with a pre-registered randomized rollout.** The 2,484-patient High-risk cohort is large enough to power a randomized rollout, or a stepped-wedge design if randomization is operationally difficult. A target detectable effect of a 2 to 3 percentage point reduction in the 19.9% baseline is feasible at this sample size. Pre-register the design before launch so the result is credible to leadership and external reviewers.

4. **Phase 2: monitor the Medium-risk stratum for upstream graduation.** The Medium stratum (3,561 patients, 13% readmission rate, right at the CMS benchmark) is the natural cohort to watch for patients who will eventually move into the High stratum. Once the High-cohort program is running, a lower-cost intervention here, such as automated discharge education without dedicated transitional-care staffing, is the logical phase-2 expansion.

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
