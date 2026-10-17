# Mortgage Early-Warning Pipeline

**Databricks · PySpark · Delta Lake · Spark ML · SQL dashboards**

A mortgage monitoring project that ranks loans for review based on the risk of serious delinquency or real estate owned (REO) within the next 12 months. It combines data validation, payment-history features, model evaluation, monthly scoring, and a portfolio dashboard.

**At a 5% monthly review budget, gradient boosted trees captured 55.42% of event observations in the held-out test period, compared with 51.75% for logistic regression.**

[View dashboard](https://dbc-b3cfbe68-f849.cloud.databricks.com/dashboardsv3/01f1c8280d7f17a29c530050b60c3287/published?o=7474653100071387) · [Model results](Final%20Reports/model_comparison.csv) · [Calibration results](Final%20Reports/calibration_results.csv) · [Successful pipeline run](Final%20Reports/successful_pipeline_run.png)

The dashboard may require Databricks access. Screenshots and aggregate results are available directly in this repository.

## Test results

The evaluation uses prediction months from **January 2024 through March 2025**, with the top 5% of eligible loans selected for review independently each month and within each population.

| Population | Model | Precision at 5% | Recall at 5% | Lift |
| --- | --- | ---: | ---: | ---: |
| All eligible loans | **GBT, 50 trees** | **8.44%** | **55.42%** | **11.08×** |
| All eligible loans | Logistic regression | 7.88% | 51.75% | 10.35× |
| Current at prediction time | **GBT, 50 trees** | **3.27%** | **33.39%** | **6.68×** |
| Current at prediction time | Logistic regression | 2.66% | 27.18% | 5.44× |

For all eligible loans, the underlying event rate was **0.76%**; the selected GBT review group had an **8.44%** event rate. Among loans that were current at prediction time, GBT improved recall by **6.21 percentage points** over logistic regression at the same review budget.

Metrics are aggregated from monthly counts across **3,306,965 loan-month observations**, including **25,184 positive observations**. A loan can appear in multiple months with overlapping outcome windows. These are observation-level results, not counts of unique defaults or losses prevented. Lift compares the selected group's event rate with its population's event rate.

## Data and modeling

The project uses Freddie Mac's **2016–2025 origination samples**, retaining their available monthly performance history through **March 2026**. The counts and model results below describe the data snapshot used for the saved **v1 experiment**.

| Dataset | Size |
| --- | ---: |
| Origination records | 500,000 loans |
| Monthly performance records | 20,079,736 |
| Eligible monthly feature records | 16,407,101 |

- **Data quality:** Validate source-file coverage, record structure, missing keys, duplicate loan and loan-month keys, numeric and date conversions, and performance-to-origination matches. Required checks stop processing when they fail. Preserve raw lines and source metadata in Bronze for traceability.
- **Features:** Combine origination characteristics with calendar-based three- and six-month payment histories, loan age, balance ratios, rate changes, and estimated LTV. Payment-history windows use reporting months through the prediction month.
- **Target:** An observed 90+ day delinquency or REO event in the following 12 months. Verified payoffs without an earlier event can receive a negative label; unresolved outcomes remain missing and are excluded from supervised modeling.
- **Preprocessing:** Fit imputation and categorical encoding on training data. Apply the fitted transformations to validation, test, and scoring data.
- **Model selection:** Compare logistic regression, 50- and 100-tree Spark GBT models, and a native XGBoost challenger. Select the 50-tree GBT using validation results before final test evaluation.

| Split | Prediction months |
| --- | --- |
| Training | January 2017–September 2020 |
| Validation | January–September 2022 |
| Final test | January 2024–March 2025 |

The training and validation cutoffs require the full 12-month horizon plus a three-month reporting buffer, which is a project assumption. This separates model selection and calibration from the final test period.

Platt calibration is fitted on validation data. On the test set, it reduced the GBT Brier score from **0.00773 to 0.00655**. Risk-band calibration gaps remain, so ranking performance and probability calibration are assessed separately.

## Monthly pipeline

The five-task Databricks job processes uploaded source files, scores the latest eligible month with the saved model and calibrator, and refreshes the published dashboard.

| Task | Responsibility |
| --- | --- |
| `01_ingest_raw_files` | Load and validate the complete source files into Bronze |
| `02_clean_validate_data` | Normalize values, convert types, and refresh Silver |
| `03_build_features_labels` | Refresh Gold features and outcome labels |
| `04_score_monthly_risk` | Save risk scores, ranks, review flags, and run metadata |
| `05_refresh_dashboard` | Refresh the published dashboard after scoring succeeds |

Each downstream task requires its predecessor to succeed. The exported job is configured to run on the **first day of each month at 6 AM, America/Toronto**, with a maximum of one concurrent run.

This is a **full-refresh batch workflow**: source downloads and uploads are manual, and the complete source history must remain available. Notebooks 01–03 rebuild the Bronze, Silver, and Gold Delta tables on each run. Gold validation checks key uniqueness, label values, outcome dates, and row-count consistency without requiring fixed historical counts.

Model training is a separate workflow. Monthly scoring loads the saved v1 model and calibrator; it does not retrain them. Scoring reruns replace the matching month's model/calibration output rather than accumulating duplicate scores.

The run below demonstrates successful execution of all five tasks. It was launched manually; the monthly schedule is documented in the [job configuration](Jobs%20%26%20Piplines/mortgage_monthly_risk_pipeline.json).

![Successful five-task mortgage risk pipeline run](Final%20Reports/successful_pipeline_run.png)

## Dashboard

**Portfolio Overview** shows eligible loans, review volume, estimated risk, state concentrations, risk bands, and a ranked loan review queue. **Model Performance** compares GBT with logistic regression and shows estimated versus observed event rates.

![Mortgage portfolio overview dashboard](Final%20Reports/portfolio_overview.png)

[Model performance](Final%20Reports/model_performance.png) · [Risk distribution](Final%20Reports/risk_distribution.png) · [Loan review queue](Final%20Reports/loan_review_queue.png) · [Risk calibration](Final%20Reports/risk_calibration.png)

The latest scoring snapshot uses **March 2026 features**. The scoring timestamp records when the job ran; it does not indicate that the underlying loan data is current to that date.

## Reproduce the project

<details>
<summary>Data access, execution order, and workspace configuration</summary>

### 1. Obtain the source files

Register for access to the [Freddie Mac Single-Family Loan-Level Dataset](https://www.freddiemac.com/research/datasets/sf-loanlevel-dataset) and download the sample origination and performance files for 2016–2025. Use the applicable source-data terms.

Run [0_Project_Setup](Notebooks/0_Project_Setup.ipynb), then upload the 20 extracted text files to:

```text
/Volumes/workspace/default/freddie_mac_raw/DB Project/
```

The expected filenames are `sample_orig_2016.txt` through `sample_orig_2025.txt` and `sample_perf_2016.txt` through `sample_perf_2025.txt`. Year subfolders are supported. Keep exactly one copy of each expected file, including its full available history.

### 2. Build data and model artifacts

Import the notebooks into a Databricks workspace with Unity Catalog volumes and run them in this order:

| Notebooks | Purpose |
| --- | --- |
| [0_Project_Setup](Notebooks/0_Project_Setup.ipynb), then 01–03 | Validate files and build Bronze, Silver, Gold features, and labels |
| 04–06 | Create temporal splits, fit preprocessing, and compare logistic regression with GBT |
| 07, optional | Run the XGBoost challenger experiment |
| 08 | Fit and save probability calibration on validation data |
| 09–11 | Evaluate the held-out test period, compare models, and assess calibration |
| 12 | Score the selected month using the saved model and calibration artifacts |

For a first reproduction, start without pre-existing project split tables, model files, or prediction tables. The model-development notebooks **04–11** represent the saved v1 experiment and can reuse existing artifacts on later runs. To develop a new model with different training data, feature definitions, or settings, use a consistent new set of artifact names and regenerate the dependent models, calibration, and evaluation results. Automated experiment tracking is future work.

Project tables use `workspace.mortgage_risk`. Model artifacts use:

```text
/Volumes/workspace/mortgage_risk/model_artifacts/
```

The exported notebook metadata records **Databricks serverless environment version 6**. The optional XGBoost notebook pins **`xgboost==3.3.0`**. Exact Python, Spark, and supporting package versions have not yet been recorded in this repository.

### 3. Configure the dashboard and job

Import the [dashboard definition](Dashboard/Mortgage%20Early-Warning%20Dashboard.lvdash.json), configure its warehouse, and publish it. Adapt the [job JSON](Jobs%20%26%20Piplines/mortgage_monthly_risk_pipeline.json) to your workspace:

| Setting | Configuration required |
| --- | --- |
| `notebook_task.notebook_path` | Paths of your imported notebooks |
| `dashboard_task.warehouse_id` | Your SQL warehouse ID |
| `dashboard_task.dashboard_id` | Your imported dashboard ID |
| Table and volume paths | Your catalog, schema, and volume locations |
| `schedule` | Your desired cadence and timezone |

Leave the scoring parameter `prediction_month` blank to use the latest eligible Gold month, or supply `YYYY-MM-01` to replay a specific month. Test the complete job manually before enabling its schedule.

</details>

## Limitations and next steps

- The dataset is a Freddie Mac sample; results may not generalize to other lenders, portfolios, or economic periods.
- Temporal evaluation can include the same loan in different periods. The task is ongoing portfolio monitoring, not a benchmark on entirely unseen borrowers.
- Calibration varies across risk bands. Estimated probabilities require further validation before operational use.
- The full-refresh job reprocesses historical data. Incremental ingestion, stronger source-batch completeness checks, and drift monitoring are future work.
- Aggregate CSVs support the reported comparison. The calibration CSV was transcribed from displayed results and retains three-decimal percentage precision; a direct table export would improve reproducibility.
