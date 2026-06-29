# Credit Risk Scorecard: German Credit Dataset

An end-to-end credit risk model that predicts probability of default for loan applicants and outputs a scorecard with Low / Medium / High risk bands.

---

## What this actually does

A bank needs to answer one question before giving someone a loan: will this person pay it back? This project builds the tool that answers that question. 
It looks at an applicant's checking account status, credit history, loan duration, savings, and employment, then outputs a score between 0 and 1000 along with a risk classification.

The model was trained on 1,000 real loan applicants from a German bank. 300 of them defaulted. The goal was to build something that separates those two groups as cleanly as possible, 
validate it using the same metrics banks use under Basel III, and deliver the output in formats a business team can actually use (Excel + Power BI, not just a Jupyter notebook).

---

## Results

| Metric | Logistic Regression | XGBoost |
|--------|-------------------|---------|
| AUC | **0.802** | 0.797 |
| Gini | **0.604** | 0.594 |
| KS Statistic | **0.486** | 0.462 |

Logistic regression won on every metric. On a clean, structured dataset of 1,000 rows there isn't enough complexity for XGBoost to outperform a well-tuned logistic regression. 
This also makes logistic regression the better production choice as it's more interpretable and easier for regulators to audit, which matters in a Basel III context.

Overfitting check: training AUC was 0.886 vs test AUC of 0.802, a gap of 0.084. This is expected when SMOTE is applied to the training set since it slightly inflates training performance. 
The test set contains only real applicants and that's the number that matters.

### Risk band breakdown (test set, 200 applicants)

| Band | Applicants | Default Rate | Avg Score |
|------|-----------|--------------|-----------|
| Low Risk (score >= 700) | 98 | 11.2% | 873 |
| Medium Risk (score 500-699) | 35 | 31.4% | 606 |
| High Risk (score < 500) | 67 | 56.7% | 274 |

Default rate climbs cleanly from 11% to 31% to 57% across the bands. That monotonic separation is what regulators look for when validating a scorecard, proving the model is actually ranking 
risk in the right direction, not just fitting noise.

---

## What each metric means

**PD (Probability of Default)** - the core output. If someone scores 0.72 PD, the model thinks there's a 72% chance they won't repay. This maps directly to Basel III's PD concept 
                                  under the Internal Ratings-Based approach.

**Gini coefficient** - measures how well the model separates good borrowers from bad ones. 0 = useless, 1 = perfect. Below 0.40 is weak, 0.40-0.60 is acceptable for retail credit, 
                       0.60+ is strong. At 0.604, this model clears the strong threshold.

**KS statistic** - the maximum gap between the cumulative distribution of good and bad borrowers across score buckets. At 0.486, the model separates the two groups by 48.6 percentage 
                   points at its optimal threshold. Industry rule of thumb is above 0.40 for a good scorecard.

---

## Key findings from EDA

Before any modelling, a few things stood out in the data:

- Checking account status was the single strongest protective factor (correlation -0.35 with default). Applicants with no checking account defaulted at 49% vs 11% for well-funded accounts,
  a 38 percentage point spread.

- Credit history was the strongest categorical predictor. Applicants with poor or no credit history defaulted at over 60%. Clean repayment history brought that down to 17%.

- Loan duration was the strongest risk factor (+0.22 correlation). Short loans under 12 months had a 21% default rate. Loans over 24 months had a 44% default rate. The longer someone has to repay,
  the more it can go wrong.

---

## Stack

```
Python 3.10
pandas, numpy
scikit-learn (logistic regression, train/test split, StandardScaler)
xgboost
imbalanced-learn (SMOTE)
matplotlib, seaborn
openpyxl (Excel export)
Power BI Desktop (dashboard)
```

---

## Project structure

```
credit-risk-scorecard/
    credit-risk-scorecard.ipynb   # full modelling pipeline
    credit_risk_powerbi.csv       # export feeding Excel and Power BI
    assets/
        default_rate_by_feature.png
        default_rate_by_duration.png
        correlation_heatmap.png
        feature_correlation_with_default.png
        model_comparison_roc.png
        scorecard_output.png
    credit_risk_scorecard.pbix   # power bi dashboard
    credit_risk_scorecard.xlsx   # excel outputs
```

---

## How to run it

```bash
pip install pandas numpy scikit-learn xgboost imbalanced-learn matplotlib seaborn openpyxl
```

Open `credit-risk-scorecard.ipynb` and run cells top to bottom. The dataset loads directly from UCI so no manual download needed.

---

## Why logistic regression over XGBoost

XGBoost is powerful when there are non-linear interactions across thousands of rows with messy, high-dimensional features. This dataset has 1,000 clean, pre-encoded rows. 
There isn't enough complexity for XGBoost to find patterns that logistic regression misses.
More importantly, logistic regression is interpretable. A bank's risk team can open the model coefficients and explain to a regulator exactly why applicant X was classified as High Risk. 
XGBoost doesn't give you that out of the box. 

---

## Business outputs

The project delivers beyond the notebook:

- The Power BI dashboard has an interactive slicer so you can filter by risk band and watch default rates and applicant counts update in real time.

- The CSV export is structured so every useful dimension is already a column; risk band, PD score, credit score, correct prediction flag, model name, Gini, KS.
  It loads straight into Power BI or Excel without any transformation needed.
