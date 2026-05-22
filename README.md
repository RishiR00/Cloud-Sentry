# CloudSentry — Insights into Cloud Activity

An unsupervised anomaly detection solution that proactively monitors access patterns and traffic anomalies in **AWS CloudTrail** logs. CloudSentry combines three complementary detection models — Time Series Analysis, DBSCAN, and Isolation Forest — and fuses their signals into a single alert verdict.

📓 **Notebook:** [cloudsentry.ipynb](cloudsentry.ipynb)

---

## Summary of Findings

CloudSentry was evaluated on a 30-day synthetic AWS CloudTrail dataset (~21,600 events, ~1.5% injected anomalies). The three unsupervised detectors were fused using a majority-vote + extreme-score rule.

Key results:

- **Isolation Forest** was the strongest single model, producing cleanly separated score distributions between benign and anomalous events (ROC-AUC > 0.95).
- **DBSCAN** spatially isolated injected anomalies in PCA-projected feature space, placing them in a distinct low-density region far from all normal clusters.
- **Time Series** detected volumetric bursts scattered throughout the 30-day window for the most-alerted user (user_07).
- **The fused CloudSentry verdict** flagged ~2% of all events, achieving higher precision than any single model alone while maintaining strong recall.
- Hyperparameter tuning via grid search further improved F1 scores across all three models, with DBSCAN showing the largest lift.

---

## Overview

CloudSentry detects suspicious cloud activity such as identity gaps, misconfigurations, and active attacks — without ever requiring labeled training data. The models learn what "normal" looks like and flag deviations from it.

The notebook follows the **CRISP-DM** methodology across six phases:

1. Business Understanding
2. Data Understanding
3. Data Preparation & EDA
4. Modeling
5. Evaluation


---

## How It Works

### Detection Models

| Model | Technique | What It Catches |
|---|---|---|
| Time Series | Rolling Z-Score (7-day window) | Sudden volumetric bursts — e.g. 30× normal call rate in one minute |
| DBSCAN | Density-based clustering | Events that don't fit any cluster of normal behavior (noise points) |
| Isolation Forest | Random tree partitioning | High-dimensional outliers — unusual combinations of IP, region, action, and timing |

### Fusion Logic

The three model outputs are combined with a two-rule OR:

- **Rule 1 — Majority Vote:** Flag if at least 2 of 3 models agree the event is anomalous
- **Rule 2 — Extreme Score:** Flag if the Isolation Forest score is in the top 0.5% of all events

This trades a small amount of recall for significantly higher precision — fewer false alarms for SOC analysts to triage.

---

## Data Cleaning & EDA

Before feature engineering, the notebook runs a dedicated Phase 2b to validate and explore the dataset:

**Data Cleaning:**
- Checks for missing values across all columns; imputes any missing `sourceIPAddress` with `'unknown'`
- Removes exact duplicate rows and reports counts before and after
- Confirms and enforces correct data types (e.g. `eventTime` as datetime)

**Exploratory Data Analysis — 4-panel subplot (`plot_eda_overview.png`):**
- Event volume per user (bar chart)
- Top 10 API actions across all events (horizontal bar chart)
- Hourly event distribution with off-hours shading (histogram)
- Event distribution by AWS region (pie chart)

**Outlier & Class Balance Check:**
- Confirms the ~1.5% injected anomaly fraction
- Computes per-user event volume statistics and flags any users whose volume is more than 2 standard deviations from the mean

---

## Features Engineered

Raw CloudTrail fields are converted into 11 numeric signals before being fed to the models:

| Feature | Description |
|---|---|
| `hour`, `dayofweek` | When the event occurred |
| `is_offhours` | Before 7am or after 7pm |
| `is_weekend` | Saturday or Sunday |
| `eventName_freq` | Global frequency of this API action |
| `sourceIPAddress_freq` | Global frequency of this IP address |
| `awsRegion_freq` | Global frequency of this region |
| `user_ip_familiarity` | How often this specific user has used this IP |
| `user_region_familiarity` | How often this specific user has worked from this region |

All features are normalized with `StandardScaler` before being passed to DBSCAN and Isolation Forest.

---

## Data

Because real CloudTrail logs are sensitive, the notebook uses a **synthetic data generator** that produces realistic CloudTrail-style events across 30 days and 12 users. It injects ~1.5% of events as anomalies with the following characteristics:

- Off-hours timestamps (1am–4am or 11pm)
- Foreign IP addresses outside the user's normal prefix
- Sensitive API actions: `DeleteUser`, `StopLogging`, `CreateAccessKey`, `Decrypt`, etc.

The anomaly labels are **immediately separated** from the data after generation and stored in `hidden_truth`. The models never see these labels — they are only used in the optional validation step at the end.

---

## Output

Each event is scored and exported to `cloudsentry_results.csv` with the following columns:

| Column | Description |
|---|---|
| `ts_anomaly` | Flagged by Time Series model (0/1) |
| `dbscan_anomaly` | Flagged by DBSCAN model (0/1) |
| `if_anomaly` | Flagged by Isolation Forest model (0/1) |
| `if_score` | Continuous Isolation Forest anomaly score (higher = more anomalous) |
| `cloudsentry_alert` | Final fused CloudSentry verdict (0/1) |

Approximately **2% of events** are flagged as alerts.

---

## Visualizations

Four plots are generated across the notebook:

- **`plot_eda_overview.png`** — 4-panel EDA overview: event volume per user, top API actions, hourly distribution, and regional breakdown
- **`plot_timeseries_baseline.png`** — API calls per minute for the most-alerted user, with red dots marking CloudSentry alerts across the full 30-day window
- **`plot_dbscan_clusters.png`** — DBSCAN clusters projected into 2D via PCA, with injected anomalies shown as red circles for reference
- **`plot_isolation_forest_scores.png`** — Histogram of anomaly scores split by ground truth, showing clear separation between benign (blue) and anomalous (red) events

---

## Evaluation Metrics

Models are compared against `hidden_truth` in Phase 5b using the following metrics:

| Metric | Description |
|---|---|
| Precision | Of flagged events, the fraction that are truly anomalous |
| Recall | Of truly anomalous events, the fraction the model caught |
| F1 | Harmonic mean of precision and recall |
| ROC-AUC | Score ranking quality for models with continuous outputs |

Phase 5c also includes **hyperparameter tuning** via grid search for each model:

- **DBSCAN:** sweeps `eps` × `min_samples`
- **Isolation Forest:** sweeps `contamination` × `n_estimators`
- **Time Series:** sweeps `window_days` × `z_threshold`

---

## Requirements

```
numpy
pandas
matplotlib
scikit-learn
```

---

## File Structure

```
cloudsentry.ipynb                 # Main notebook
cloudsentry_results.csv           # Per-event alert output (generated on run)
plot_eda_overview.png             # EDA 4-panel overview (generated on run)
plot_timeseries_baseline.png      # Time series visualization (generated on run)
plot_dbscan_clusters.png          # DBSCAN cluster visualization (generated on run)
plot_isolation_forest_scores.png  # Isolation Forest score distribution (generated on run)
README.md                         # This file
```
