# Flutterwave Risk Analytics Assessment

## Overview
This repository contains the end-to-end solution for the Flutterwave Risk Analytics technical assessment. It includes a robust data engineering architecture (SQL) designed for PostgreSQL/Redshift environments, alongside a machine learning pipeline that handles severe class imbalances to predict fraudulent transactions.

## Repository Contents
* `flutterwaveassessmentcolab.ipynb`: Jupyter Notebook containing the DuckDB-powered SQL architecture, data preprocessing, feature engineering, and a RandomForest classification model.
* `flutterwave_synthetic_txs.csv`: raw transaction dataset (ensure this remains in the root directory for relative path execution).
* `README.md`: execution instructions, caveat for Part 1, and the Part 3 Business Strategy proposal.

## Setup & Execution Instructions
To run the analysis locally and reproduce the model outputs:

1. **Environment Setup:** Ensure you have Python 3.12+ installed. You can run this seamlessly in VSCode with the Jupyter extension or via a standalone Jupyter server. Or you could use Google Colab.
2. **Dependencies:** Install the required Python packages using pip:
   ```bash
   pip install pandas numpy scikit-learn duckdb
3. **Execution:** Clone this repository to your local machine.
- Open `flutterwaveriskanalyticsassessment.ipynb`
- Ensure `flutterwave_synthetic_txs.csv` is located in the same directory (i.e., folder) as the notebook.
- Select **"Run All Cells"** to execute the data extraction, and train the machine learning model.

### ⚠️ SQL Execution & Performance Caveat — for Part 1

**Local Evaluation vs. Production Scale:**
The SQL query provided in the  `flutterwaveassessmentcolab.ipynb` is executed locally using **DuckDB**. This was done intentionally to ensure the notebook is entirely reproducible on any local machine without requiring a live database connection or passing authentication credentials.

However, the query is strictly architected for a **PostgreSQL or Redshift** production environment as suggested in the assessment case document. Because it utilizes a time-bounded `self-join` to bypass the `COUNT(DISTINCT)` window function limitation native to Postgres and Redshift, running this against millions of rows requires specific infrastructure optimizations:
* **For PostgreSQL:** A Composite B-Tree Index on `(user_id, timestamp)` is required to prevent a Cartesian explosion and allow the engine to perform an aggressive index seek.
    e.g.,
    ```sql
        CREATE INDEX idx_usrtimestamp
        ON '/flutterwave_synthetic_txs.csv' (user_id, timestamp);
    ```
* **For Redshift:** The `user_id` must be set as the Distribution Key (`DISTKEY`) to eliminate network shuffling during the join, and `timestamp` as the Sort Key (`SORTKEY`) for rapid interval range filtering.

## Part 3: Risk Analytics Business Strategy

### The Business Challenge
The model successfully handles the 1.9% fraud imbalance, yielding a Recall of 0.68 and a Precision of 0.66 for the positive fraud class. While it correctly identifies the majority of fraudulent behavior, a strict binary classification threshold (0.50) presents a conflict:
- **Product Risk:** 69 legitimate transactions were falsely flagged as fraud (False Positives). Blocking these creates friction for high-value users.
- **Compliance Risk:** 64 fraudulent transactions slipped through (appearing as False Negatives). Approving these opens the network to chargebacks and regulatory fines.

### The Proposed Solution: Multi-Tiered Risk Routing
To balance the Product team's requirement for a frictionless user experience with the Compliance team's mandate to minimize network fines, we must move away from a binary "Approve/Block" system.

By utilizing the model's continuous probability output (predict_proba), we can route transactions through a Three-Tiered Risk Action System without needing to retrain the underlying algorithm.

1. **The "Frictionless" Tier (Probability < 0.40) * Business Unit Satisfied:** Product Team

- Action: Auto-Approve.
- Reasoning: The vast majority of our transaction volume falls into this low-risk bucket. By lowering the "safe" threshold slightly below 0.50, we ensure that zero legitimate, high-value users face unnecessary friction or false declines.

2. **The "Step-Up Authentication" Tier (Probability 0.40 to 0.75) * Business Unit Satisfied:** Product & Compliance Compromise

- Action: Trigger Step-Up Authentication (e.g., 2FA, OTP via SMS) or route to a Manual Review Queue.
- Reasoning: This is the grey area where our False Positives reside. Instead of blocking them outright or letting them through unquestioned, we introduce minor friction. A legitimate user will easily input an OTP to complete their purchase, whereas a bad actor using stolen card details will fail the challenge.

3. **The "Hard Decline" Tier (Probability > 0.75)**

- **Business Unit Satisfied:** Compliance & Risk Team

- **Action:** Auto-Block transaction immediately.

- **Reasoning:** Only transactions with an overwhelming mathematical probability of fraud reach this tier. We aggressively protect the Flutterwave network from chargeback fines without catching legitimate users in the crossfire.

### Future Recommendations & Next Steps
If this pipeline were moving into a live production environment, I recommend the following technical and architectural enhancements:

- **Velocity and Graph Feature Engineering:** The current model relies heavily on temporal features (hour of day, day of week). In production, we should calculate real-time velocity metrics (e.g., "number of attempts by this device in the last 10 minutes") and IP-to-Card country mismatch flags.

- **Streaming Infrastructure:** To run the rolling 1-hour and 24-hour SQL window constraints at enterprise scale, the batch architecture should be migrated to a real-time streaming engine like Apache Kafka or AWS Kinesis paired with a low-latency database.

- **Hyperparameter Tuning:** The current RandomForestClassifier utilizes base parameters (with class weighting). Implementing a GridSearch or randomized search across tree depth and estimator counts will yield tighter Precision/Recall thresholds.

- **Data Quality & Deduplication:** During exploratory data analysis, I identified 2,500 duplicate `transaction_id` records. While 722 of these were exact row duplicates, the remaining 1,778 possessed slight timestamp offsets (1-2 seconds), indicating network retry artifacts. I implemented a targeted cleaning step dropping duplicates based on `transaction_id` to prevent data leakage and skewed customer velocity metrics.

*I hope this summary is well detailed for you, and satisfactory. I'm open to walking you through this entire summary and code during the technical interview, hopefully adequate time is provided for us to discuss this.*