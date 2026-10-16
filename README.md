# Mortgage Early-Warning Project
- View the Dashboard:https://dbc-b3cfbe68-f849.cloud.databricks.com/dashboardsv3/01f1c8280d7f17a29c530050b60c3287/published?o=7474653100071387
  
- **Purpose:** Identify loans at risk of serious delinquency or REO within the next 12 months.

- **Data:** Freddie Mac 2016–2025 origination samples and their available monthly performance history.

- **Pipeline:** Bronze → Silver → Gold → Modelling → Monthly scoring → Dashboard.

- **Evaluation:** Chronological data splits, comparison with logistic regression, and a review budget covering the top 5% of eligible loans each month.

- **Results:** Model comparisons using precision, recall, lift, and estimated versus observed event rates.

- **Limitations:** Remaining calibration gaps and a historical scoring snapshot based on March 2026 features.

- **Reproduction:** Data access, environment and package versions, notebook execution order, model training, and job configuration.

- **Screenshots:** Pipeline structure, Portfolio overview, risk distribution, loan review queue, model performance, calibration, and the successful scoring job.
