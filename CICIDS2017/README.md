# CICIDS2017 Network Intrusion Detection

Machine learning pipeline for detecting and classifying network intrusions using the [CICIDS2017](https://www.unb.ca/cic/datasets/ids-2017.html) dataset. The project covers full preprocessing, exploratory data analysis (EDA), and four XGBoost-based models for binary detection, attack-category classification, and fine-grained sub-attack classification.

## Overview

CICIDS2017 contains labeled network flow data covering benign traffic and multiple real-world attack types (DoS, DDoS, brute force, web attacks, port scans, botnets, infiltration, and Heartbleed). This notebook builds a complete workflow to:

1. Clean and preprocess the raw flow-based CSV files
2. Explore class distributions, traffic behavior, and feature correlations
3. Train and evaluate models for:
   - **Binary detection** — Benign vs. Attack
   - **Multi-class attack classification** — attack traffic only, by major category
   - **Multi-class detection & classification** — full traffic (benign + attacks), by major category
   - **Sub-attack classification** — attack traffic only, by specific attack variant

## Dataset

- **Source:** CICIDS2017 (Canadian Institute for Cybersecurity)
- **Format:** Multiple CSV files, one per capture day/scenario
- **Loading:** The notebook was built to run on Google Colab and reads CSVs from Google Drive:
  ```python
  path = "/content/drive/MyDrive/Colab Notebooks/CICIDS2017"
  ```
  Update this path (and remove the `drive.mount` call) if running locally or on another platform.

To reproduce this project, download the CICIDS2017 CSVs from the [official source](https://www.unb.ca/cic/datasets/ids-2017.html) and place them in the folder referenced above.

## Preprocessing

- Normalized column names (stripped whitespace, lowercased, replaced spaces with underscores)
- Cleaned inconsistent label strings and encoding artifacts
- Removed rows with invalid negative values in `flow_duration`, `flow_iat_min`, `fwd_header_length`, and `bwd_header_length`
- Dropped columns that were entirely zero (`*_avg_bytes/bulk`, `*_avg_packets/bulk`, `*_avg_bulk_rate`)
- Derived three label columns:
  - `major_cat` — attack family (benign, dos, ddos, brute-force, web-attack, port-scan, bot, infiltration, heartbleed)
  - `sub_attack` — specific attack variant (e.g. `hulk`, `goldeneye`, `xss`, `sql-injection`)
  - `binary_cat` — 0 (benign) / 1 (attack)
- Removed rows with missing values, duplicate rows, and resolved infinite values in `flow_bytes/s` / `flow_packets/s` caused by zero-duration flows
- Dropped highly redundant numerical features (pairwise correlation ≥ 0.95) and constant features (`active_std`, `idle_std`) identified during EDA

## Exploratory Data Analysis

The EDA section investigates:
- Class balance (benign vs. attack, and across major/sub-attack categories)
- Attack behavior in flow duration, packet counts, and byte volume vs. benign traffic
- Destination port distributions for benign vs. attack traffic
- Forward/backward packet and byte ratios
- Timing behavior across attack categories (e.g. DoS vs. DDoS)
- Feature correlation with the binary target and inter-feature redundancy
- Per-category distinctive features vs. benign traffic (visualized as a scaled heatmap)

## Modeling

All models use **XGBoost** (`XGBClassifier`) with features standardized via `StandardScaler`. Categorical targets are encoded with `LabelEncoder`. Class imbalance is addressed using `compute_sample_weight(class_weight='balanced')` for the multi-class models.

| Model | Task | Data | Target | Accuracy |
|---|---|---|---|---|
| Model 1 | Binary detection | All traffic | `binary_cat` (benign/attack) | 99.92% |
| Model 2 | Attack classification | Attacks only | `major_cat` | 99.98% |
| Model 3 | Attack classification (weighted) | Attacks only | `major_cat` | 99.98% |
| Model 4 | Detection & classification (weighted) | All traffic | `major_cat` | 99.89% |
| Model 5 | Sub-attack classification (weighted) | Attacks only | `sub_attack` | 99.77% |

Train/test split is 70/30 (`random_state=42`), stratified by category for the multi-class tasks.

> **Note:** Reported accuracy is very high, consistent with published CICIDS2017 benchmarks, but per-class precision/recall drops noticeably for rare classes (e.g. Heartbleed, Infiltration, and some sub-attacks with very few samples) — see the classification reports in the notebook for details.

## Requirements

```
numpy
pandas
matplotlib
seaborn
scikit-learn
imbalanced-learn
xgboost
```

Install with:
```bash
pip install numpy pandas matplotlib seaborn scikit-learn imbalanced-learn xgboost
```

If not running on Google Colab, remove the `google.colab` import and `drive.mount()` call, and point `path` to your local dataset directory.

## Usage

1. Download the CICIDS2017 CSV files.
2. Update the `path` variable to point to your dataset location.
3. Run the notebook top to bottom — sections are organized as: **Reading Dataset → Preprocessing → EDA → Modeling** (repeated per task).

## Project Structure

```
CICIDS2017.ipynb   # Full pipeline: preprocessing, EDA, and 5 XGBoost models
```

## Acknowledgments

- Dataset: Sharafaldin, I., Lashkari, A.H., and Ghorbani, A.A., "Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization", ICISSP 2018.
- Canadian Institute for Cybersecurity (CIC), University of New Brunswick
