# Impact of Substance Abuse on Academic Development — Analysis Project (Python)

## Project overview
This repository contains the reproducible Python-based analysis for a cross-sectional survey assessing the impact of substance abuse on students' academic development. The analysis evaluates:
- Prevalence and sources of commonly used substances (alcohol, marijuana, cigarettes, khat).
- Patterns of substance use and socio-demographic determinants.
- Academic outcomes (attendance, grades, concentration, participation).
- Relationships between substance use and mental health (depression, anxiety, stress).
- Effectiveness/coverage of awareness and intervention activities.

## General objective
To assess the impact of substance abuse on academic development.

## Specific objectives
1. Identify the types of substances abused by students and the sources of these substances.  
2. Examine patterns of substance use among students and the socio-demographic factors influencing these patterns.  
3. Evaluate the impact of substance abuse on academic performance (attendance, grades, participation).  
4. Investigate the relationship between substance abuse and mental health (depression, anxiety, stress).

---

## Repository structure
- data/
  - raw/ — raw/unmodified survey exports (CSV/Excel).
  - processed/ — cleaned datasets used in analysis (Parquet / CSV).
- code/
  - 01_data_cleaning.py — data cleaning, variable creation, and quality flags.
  - 02_descriptives.py — prevalence tables, sources, and plots.
  - 03_pattern_analysis.py — pattern index, reliability checks, and ordinal models.
  - 04_academic_models.py — regressions for academic outcomes.
  - 05_mental_health.py — mental-health scoring and associations.
  - 06_intervention_analysis.py — attendance, perceived effectiveness, and adjusted comparisons.
  - utils.py — reusable helper functions (e.g., Cronbach's alpha, CI functions, plotting utilities).
- notebooks/
  - exploratory_analysis.ipynb — interactive exploration and figures.
- output/
  - tables/ — final CSV/TSV tables used in reporting.
  - figures/ — publication-quality PNG/SVG files.
  - logs/ — run logs and diagnostics.
- docs/
  - data_dictionary.md — variable names, types, coding, permitted values.
  - analysis_plan.md — condensed statistical plan (derived from README).
- README.md — this file.

---

## Data expectations and naming conventions
- One row per respondent.
- Example variable names (adapt to your dataset; update docs/data_dictionary.md accordingly):
  - id  
  - start_time, end_time (ISO timestamps)  
  - age, gender, institution, year_of_study, programme_category  
  - alcohol_freq, marijuana_freq, cigarette_freq, khat_freq  
    - Frequency categories: "never", "occasionally", "regularly" (or more granular)  
  - alcohol_source, marijuana_source, cigarette_source, khat_source  
    - Multi-response encoded as semicolon/comma-separated strings or list types  
  - missed_classes_due_to_use (binary)  
  - concentration_score, grades_impact_score (numeric/scale)  
  - depress_item1..n, anxiety_item1..n, stress_item1..n  
  - attended_awareness (binary), perceived_effectiveness (Likert)

---

## Analysis workflow (Python)

Phase 1 — Data cleaning & variable structuring
- Compute completion_time (seconds) = end_time − start_time. Flag very short completions for quality checks (e.g., < 180 seconds).
- Standardize categorical variables (strip, lower, unify synonyms).
- Convert frequency variables to ordered categoricals and create binary indicators (e.g., used_alcohol = alcohol_freq != "never").
- Expand multi-response fields into separate dummy columns using string splitting and one-hot encoding.
- Summarize and visualize missingness; use multiple imputation when appropriate.

Python snippets (examples)
```python
# pandas + datetime example
import pandas as pd
df = pd.read_csv("data/raw/survey_data.csv")
df['start_time'] = pd.to_datetime(df['start_time'])
df['end_time'] = pd.to_datetime(df['end_time'])
df['completion_sec'] = (df['end_time'] - df['start_time']).dt.total_seconds()
df['low_quality'] = df['completion_sec'] < 180

# standardize gender
df['gender'] = df['gender'].str.strip().str.lower().map({
    'm':'Male','male':'Male','man':'Male',
    'f':'Female','female':'Female','woman':'Female'
}).fillna('Other')

# frequency to ordered categorical
freq_order = ['never','occasionally','regularly']
df['alcohol_freq'] = pd.Categorical(df['alcohol_freq'], categories=freq_order, ordered=True)
df['used_alcohol'] = (df['alcohol_freq'] != 'never').astype(int)

# multi-response to dummies
dummies = df['alcohol_source'].fillna('').str.get_dummies(sep=';')
df = pd.concat([df, dummies.add_prefix('alcohol_source_')], axis=1)
```

Phase 2 — Substance type & source identification
- Compute prevalence proportions (with 95% CIs; use Wilson or Agresti–Coull).
- Produce institution-stratified proportions and proportion plots (bar charts, stacked bars).
- Cross-tabulate substance × source (treat source columns as binary flags), reporting proportions (not raw counts).
- Tests: chi-square or Fisher's exact when appropriate.

Phase 3 — Pattern analysis & socio-demographic influences
- Construct a substance-use pattern index (sum of ordinal numeric scores) and evaluate internal consistency (Cronbach's alpha).
- If alpha is low, consider factor analysis (e.g., sklearn / factor_analyzer).
- Use ordinal logistic regression for ordered outcomes (tools: statsmodels.miscmodels.ordinal_model or mord). Validate proportional odds assumption; alternatively, use multinomial logistic (statsmodels MNLogit).
- Account for clustering by institution with GEE (statsmodels) or cluster-robust SEs.

Phase 4 — Academic development impact assessment
- Binary outcomes: logistic regression (statsmodels.Logit or GLM family=Binomial). Use cluster-robust SEs or GEE for institutional clustering.
- Continuous outcomes: linear regression (statsmodels.OLS) with robust SEs if necessary.
- Report ORs for binary models and betas for continuous models, with 95% CIs.

Phase 5 — Mental health relationship analysis
- Create composite scores (sum/mean) for depression, anxiety, stress. Compute reliability (alpha) with pingouin or custom function.
- Bivariate correlations: Pearson (parametric) or Spearman (nonparametric).
- Multivariable linear regressions to assess associations adjusting for confounders.

Phase 6 — Knowledge, attitudes & interventions
- Describe attendance and perceived effectiveness.
- Compare prevalence by attendance using chi-square and adjusted logistic regression; consider propensity-score methods (scikit-learn) to control for selection bias.

---

## Recommended Python packages
- Data manipulation: pandas, numpy
- Stats & modeling: scipy, statsmodels, scikit-learn, patsy
- Ordinal models & reliability: mord (ordinal regression), factor_analyzer, pingouin (alpha), pingouin or custom alpha
- Missing data/imputation: sklearn.experimental.IterativeImputer, fancyimpute, missingpy
- Mixed/clustered models: statsmodels (GEE, MixedLM), pymer4 (via rpy2) for GLMM if needed
- Visualization: seaborn, matplotlib, plotly
- Utilities: joblib, tqdm
Install via:
```bash
pip install pandas numpy scipy statsmodels scikit-learn seaborn matplotlib pingouin mord factor_analyzer
```

---

## Reproducibility & environment
- Use a virtual environment (venv/conda). Save exact dependencies:
```bash
pip freeze > requirements.txt
# or using conda
conda env export > environment.yml
```
- Parameterize file paths and seed values at the top of each script.
- Save cleaned datasets to `data/processed/` and never overwrite `data/raw/`.
- Log session info and package versions into `output/logs/session_info.txt`.

---

## Diagnostics & robustness
- Check collinearity (VIF using statsmodels).
- Use robust or cluster-robust standard errors (statsmodels `cov_type='HC3'` or `cov_type='cluster'`).
- For ordinal models, test the proportional odds assumption; use alternative models if violated.
- Sensitivity analyses: exclude low-quality responses, different index definitions, multiple imputation (compare results).

---

## Output deliverables
- Tables: participant characteristics, substance prevalence & sources (proportions + 95% CI), pattern models, academic outcome models, mental health models.
- Figures: prevalence plots, substance × source heatmap, forest plots of model estimates, distributions for composite scores.
- Reproducible report (Jupyter Notebook or a generated PDF via nbconvert) combining code, outputs, and interpretation.


---

## Running the pipeline (examples)
```bash
python3 code/01_data_cleaning.py --input data/raw/survey_data.csv --output data/processed/cleaned_data.parquet
python3 code/02_descriptives.py --input data/processed/cleaned_data.parquet --outdir output
python3 code/03_pattern_analysis.py ...
```

---

## License & contact
- Add a LICENSE file per institutional policy (e.g., MIT, CC-BY).
- Project lead / contact: buriro-ezekia (ezekia.buriro1810@gmail.com).

