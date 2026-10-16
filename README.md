# Mortgage Early-Warning Project

- **Purpose:** Identify loans at risk of serious delinquency or REO within the next 12 months.

- **Data:** Freddie Mac 2016–2025 origination samples and their available monthly performance history.

- **Pipeline:** Bronze → Silver → Gold → Modelling → Monthly scoring → Dashboard.

- **Evaluation:** Chronological data splits, comparison with logistic regression, and a review budget covering the top 5% of eligible loans each month.

- **Results:** Model comparisons using precision, recall, lift, and estimated versus observed event rates.

- **Limitations:** Remaining calibration gaps and a historical scoring snapshot based on March 2026 features.

- **Reproduction:** Data access, environment and package versions, notebook execution order, model training, and job configuration.

- **Screenshots:** Portfolio overview, risk distribution, loan review queue, model performance, calibration, and the successful scoring job.
