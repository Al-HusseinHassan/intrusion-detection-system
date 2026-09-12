# UNSW-NB15 Network Intrusion Detection

Machine learning pipeline for detecting and classifying network intrusions using the [UNSW-NB15](https://research.unsw.edu.au/projects/unsw-nb15-dataset) dataset. The project covers preprocessing, exploratory data analysis (EDA), and a three-stage XGBoost modeling approach: binary detection, attack-family classification, and a targeted sub-model for the four attack categories that are hardest to tell apart.

## Overview

UNSW-NB15 contains labeled network flow records covering normal traffic and nine attack categories (Analysis, Backdoor, DoS, Exploits, Fuzzers, Generic, Reconnaissance, Shellcode, Worms). This notebook builds a complete workflow to:

1. Load and combine the official training and testing CSVs
2. Explore feature distributions, class imbalance, correlations, and attack behavior
3. Bin several near-continuous/discrete numerical features into categorical buckets
4. Train and evaluate three XGBoost models:
   - **Detection model** — Normal vs. Attack
   - **Classification model** — attack traffic only, grouped into 6 attack families
   - **DoS-family sub-model** — a focused classifier separating the four attack categories (Analysis, Backdoor, DoS, Exploits) that were hardest to distinguish in the main classification model

## Dataset

- **Source:** UNSW-NB15 (UNSW Canberra Cyber, Australian Centre for Cyber Security)
- **Format:** `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv`, combined into a single working dataframe
- **Loading:** The notebook was built to run on Google Colab and reads CSVs from Google Drive:
  ```python
  drive.mount("//content//drive")
  df = pd.concat([
      pd.read_csv("//content//drive//MyDrive//Colab Notebooks//UNSW-NB15//UNSW_NB15_training-set.csv"),
      pd.read_csv("//content//drive//MyDrive//Colab Notebooks//UNSW-NB15//UNSW_NB15_testing-set.csv"),
  ])
  ```
  Update these paths (and remove the `drive.mount` call) if running locally or on another platform.

To reproduce this project, download the UNSW-NB15 CSVs from the [official source](https://research.unsw.edu.au/projects/unsw-nb15-dataset) and place them in the folder referenced above.

## Features

The dataset includes 49 flow-level features spanning:
- **Basic connection info** — `dur`, `proto`, `service`, `state`
- **Packet/byte features** — `spkts`, `dpkts`, `sbytes`, `dbytes`, `rate`
- **TTL, window, and load features** — `sttl`, `dttl`, `swin`, `dwin`, `sload`, `dload`
- **Timing and jitter features** — `sinpkt`, `dinpkt`, `sjit`, `djit`, `tcprtt`, `synack`, `ackdat`
- **Connection-count ("ct_*") features** — e.g. `ct_srv_src`, `ct_dst_ltm`, `ct_src_dport_ltm`
- **Binary flags** — `is_ftp_login`, `is_sm_ips_ports`

Target columns:
- `label` — 0 (normal) / 1 (attack)
- `attack_cat` — one of 9 attack categories, or `Normal`

## Preprocessing

- Combined the training and testing CSVs into one dataframe
- Checked for missing values, duplicates, and constant/near-constant features
- Removed rare noise values in `is_ftp_login` (values 2 and 4, found to be extremely rare and not meaningfully tied to any attack category)
- Binned several skewed/discrete numerical features into `low`/`moderate`/`high` categorical buckets based on their empirical distributions: `sttl`, `dttl`, `swin`, `dwin`, `ct_state_ttl`
- Dropped redundant, highly correlated numerical features identified via the correlation heatmap: `spkts`, `dpkts`, `ct_dst_ltm`, `ct_dst_sport_ltm`, `ct_dst_src_ltm`, `ct_src_ltm`, `ct_srv_dst`, `ct_srv_src`
- One-hot encoded categorical features (`proto`, `service`, `state`, plus the newly binned columns)
- Scaled numerical features with `RobustScaler` (chosen given the heavy skew and outliers found during EDA)
- Label-encoded multi-class targets for the classification models
- For the classification stage, grouped the 9 raw attack categories into 6 attack families, since 4 of them (Analysis, Backdoor, DoS, Exploits) behaved too similarly to separate reliably and were merged into a single `DoS_family` label:

  | Attack family | Original categories |
  |---|---|
  | DoS_family | Analysis, Backdoor, DoS, Exploits |
  | Fuzzers | Fuzzers |
  | Generic | Generic |
  | Reconnaissance | Reconnaissance |
  | Shellcode | Shellcode |
  | Worms | Worms |

## Exploratory Data Analysis

The EDA section investigates:
- Class balance for `label` and `attack_cat`, and the significant imbalance across attack categories
- Distribution and treatment of near-constant/rare features (`trans_depth`, `response_body_len`, `is_ftp_login`, `ct_ftp_cmd`, `ct_flw_http_mthd`, `is_sm_ips_ports`)
- Univariate statistics, skewness, and outlier behavior for all numerical features
- Protocol, service, and connection-state associations with specific attack categories (e.g. UDP strongly tied to Generic attacks, HTTP to Worms)
- Differences between Normal and Attack traffic in duration, byte/packet counts, traffic rate, and load
- Per-category behavior differences (duration profiles, byte/packet volume, rate) across the nine attack types
- Correlation analysis identifying redundant feature groups (e.g. `sbytes`/`spkts`, `dbytes`/`dpkts`, the `ct_*` count features) before dropping them

## Modeling

All models use **XGBoost** (`XGBClassifier`). Categorical features are one-hot encoded and numerical features scaled with `RobustScaler` before training. Class imbalance in the multi-class models is handled with `compute_sample_weight(class_weight='balanced')`.

| Model | Task | Data | Target | Params | Accuracy |
|---|---|---|---|---|---|
| Detection | Normal vs. Attack | All traffic | `label` | `n_estimators=100, max_depth=10, lr=0.1` | 93.88% |
| Classification | Attack family | Attacks only | `attack_group` (6 families) | `n_estimators=200, max_depth=10, lr=0.1`, weighted | 92.69% |
| DoS-family sub-model | Fine-grained split | Analysis/Backdoor/DoS/Exploits only | `attack_cat` | `n_estimators=200, max_depth=6, lr=0.1`, weighted | 56.6% |

A normalized confusion matrix is plotted for the classification model to visualize exactly where the DoS-family categories were being confused with each other — which is what motivated building the dedicated sub-model.

> **Note:** Unlike CICIDS2017/NSL-KDD-style benchmarks, this dataset does not yield near-100% accuracy. The detection and family-classification models land in the low-to-mid 90s, and the DoS-family sub-model — trying to separate Analysis, Backdoor, DoS, and Exploits from each other — tops out around 57% accuracy, with Analysis and Backdoor (the rarest, most overlapping classes) performing worst (precision/recall in the 0.1–0.3 range). This reflects genuine overlap between these attack types in UNSW-NB15, not a bug in the pipeline.

## Requirements

```
numpy
pandas
matplotlib
seaborn
scikit-learn
xgboost
```

Install with:
```bash
pip install numpy pandas matplotlib seaborn scikit-learn xgboost
```

If not running on Google Colab, remove the `google.colab` import and `drive.mount()` call, and point the CSV paths to your local dataset location.

## Usage

1. Download `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv` from the UNSW-NB15 dataset page.
2. Update the file paths in the "Read Dataset" section.
3. Run the notebook top to bottom — sections are organized as: **Read Dataset → Understand Features → EDA → Detection Model → Classification Model → DoS-family Model**.

## Project Structure

```
UNSW_NB15.ipynb   # Full pipeline: preprocessing, EDA, and 3 XGBoost models
```

## Acknowledgments

- Dataset: Moustafa, N. and Slay, J., "UNSW-NB15: A Comprehensive Data Set for Network Intrusion Detection Systems", MilCIS 2015.
- UNSW Canberra Cyber, Australian Centre for Cyber Security
