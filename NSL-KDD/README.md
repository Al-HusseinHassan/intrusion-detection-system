# NSL-KDD Network Intrusion Detection

Machine learning pipeline for detecting and classifying network intrusions using the [NSL-KDD](https://www.unb.ca/cic/datasets/nsl.html) dataset — an improved version of the classic KDD Cup 1999 dataset with duplicate records removed. The project covers preprocessing, exploratory data analysis (EDA), and three XGBoost models for binary detection, major-category classification, and fine-grained attack-label classification.

## Overview

NSL-KDD contains labeled network connection records covering normal traffic and attacks from four major categories: **DoS**, **Probe**, **R2L** (remote-to-local), and **U2R** (user-to-root). This notebook builds a complete workflow to:

1. Load and combine the official train/test splits
2. Explore feature behavior, class imbalance, and correlations
3. Train and evaluate three XGBoost models:
   - **Binary detection** — Normal vs. Attack
   - **Major-category classification** — attack traffic only, by DoS/Probe/R2L/U2R
   - **Fine-grained label classification** — attack traffic only, by specific attack name (e.g. `neptune`, `smurf`, `guess_passwd`)

## Dataset

- **Source:** NSL-KDD (Canadian Institute for Cybersecurity)
- **Format:** `kdd_train.csv` and `kdd_test.csv`, combined into a single working dataframe
- **Loading:** The notebook was built to run on Google Colab and reads CSVs from Google Drive:
  ```python
  df = pd.read_csv('/content/drive/MyDrive/Colab Notebooks/NSL-KDD/kdd_train.csv')
  df = pd.concat([df, pd.read_csv('/content/drive/MyDrive/Colab Notebooks/NSL-KDD/kdd_test.csv')], axis=0, ignore_index=True)
  ```
  Update this path (and remove the `drive.mount` call) if running locally or on another platform.

To reproduce this project, download the NSL-KDD CSVs from the [official source](https://www.unb.ca/cic/datasets/nsl.html) and place them in the folder referenced above.

## Features

The dataset includes 41 connection-level features spanning:
- **Basic connection features** — duration, protocol_type, service, flag, src_bytes, dst_bytes, land
- **Error/unusual packet features** — wrong_fragment, urgent, hot, num_failed_logins, etc.
- **Content features** — logged_in, num_compromised, root_shell, num_file_creations, etc.
- **Traffic (time-based and host-based) features** — count, srv_count, serror_rate, same_srv_rate, dst_host_* rates, etc.

Attack labels are grouped into four major categories:
| Category | Example attacks |
|---|---|
| DoS | back, land, neptune, pod, smurf, teardrop, apache2, udpstorm, processtable, mailbomb |
| Probe | ipsweep, nmap, portsweep, satan, mscan, saint |
| R2L | ftp_write, guess_passwd, imap, multihop, phf, warezclient, warezmaster, spy, httptunnel, xlock, xsnoop |
| U2R | buffer_overflow, loadmodule, perl, rootkit, ps, xterm |

## Preprocessing

- Combined train and test CSVs, then removed duplicate rows
- Dropped the constant column `num_outbound_cmds`
- Derived `label` (0 = normal, 1 = attack) and `major_cat` (normal/Dos/Probe/R2L/U2R) columns from the raw `labels` field
- Dropped highly correlated numerical features (|corr| > 0.9): the SYN-error family (`srv_serror_rate`, `dst_host_serror_rate`, `dst_host_srv_serror_rate`), the REJ-error family (`srv_rerror_rate`, `dst_host_rerror_rate`, `dst_host_srv_rerror_rate`), and `num_compromised`
- One-hot encoded categorical features (`protocol_type`, `service`, `flag`)
- Scaled numerical features with `RobustScaler` (chosen for its robustness to the extreme outliers found during EDA)
- Label-encoded multi-class targets for the category and fine-grained models
- For the fine-grained model, removed the single-sample `xsnoop` class since stratified splitting requires at least two samples per class

## Exploratory Data Analysis

The EDA section investigates:
- Class balance (normal vs. attack) and imbalance ratio across individual attack labels vs. grouped major categories
- Categorical feature distributions (protocol, service, flag) and their association with specific attacks
- Numerical feature statistics, skewness, and outliers (visualized with log-scaled histograms)
- Features most different between normal and attack traffic, and between major attack categories
- Feature correlation heatmaps and redundant-feature groups
- Near-perfect separators for individual attack types (e.g. `dst_host_same_src_port_rate` isolating Probe, `flag=S0` isolating DoS) and a discussion of whether these reflect real signal or dataset artifacts

## Modeling

All models use **XGBoost** (`XGBClassifier`) with:
```python
n_estimators=500, learning_rate=0.05, max_depth=6,
subsample=0.8, colsample_bytree=0.8, random_state=42
```
Class imbalance in the multi-class models is handled with `compute_sample_weight(class_weight='balanced')`.

| Model | Task | Data | Target | Test size | Accuracy |
|---|---|---|---|---|---|
| Model 1 | Binary detection | All traffic | `label` (normal/attack) | 20% | 99.79% |
| Model 2 | Major-category classification | Attacks only | `major_cat` (DoS/Probe/R2L/U2R) | 20% | 99.98% |
| Model 3 | Fine-grained classification (weighted) | Attacks only | `labels` (specific attack) | 20% | 99.65% |

> **Note:** Overall accuracy is very high, but Model 3's per-class metrics drop sharply for rare attack labels with only a handful of samples (some classes have precision/recall of 0.00–0.50), which is expected given the extreme long-tail distribution of individual attack types in NSL-KDD. See the notebook's full classification reports for per-class detail.

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

If not running on Google Colab, remove the `google.colab` import and `drive.mount()` call, and point the CSV paths to your local dataset location.

## Usage

1. Download `kdd_train.csv` and `kdd_test.csv` from the NSL-KDD dataset page.
2. Update the file paths in the "Reading Dataset" section.
3. Run the notebook top to bottom — sections are organized as: **Reading Dataset → Understanding Features → Preprocessing → EDA → Modeling** (repeated per task).

## Project Structure

```
NSL_KDD.ipynb   # Full pipeline: preprocessing, EDA, and 3 XGBoost models
```

## Acknowledgments

- Dataset: Tavallaee, M., Bagheri, E., Lu, W., and Ghorbani, A.A., "A Detailed Analysis of the KDD CUP 99 Data Set", IEEE CISDA 2009.
- Canadian Institute for Cybersecurity (CIC), University of New Brunswick
