# Adaptive Cyber Immune & Deception Platform — AI Component

This repository contains the AI/ML component of the **Adaptive Cyber Immune & Deception Platform**, a graduation project combining offensive (Red Team) and defensive (Blue Team) cybersecurity strategies into a single adaptive, learning-based defense system. It implements and evaluates supervised machine learning pipelines for detecting and classifying network attacks across three benchmark datasets.

## General Idea

The platform is modeled loosely on a biological immune system: rather than only blocking traffic that matches a known bad signature, it aims to recognize both familiar and unfamiliar attack behavior, adapt its defenses over time, and actively mislead attackers instead of just shutting them out.

At a high level, the platform works like this:
1. A **Red Team** simulates real attacks against the system — phishing, brute force, SQL injection, and similar techniques — to generate realistic attack scenarios.
2. A **Memory & Analysis** module stores and analyzes this attack data as "cyber immune memory," made up of known attack signatures and behavior patterns, so the system can recognize repeated or similar attacks faster in the future.
3. **Behavior monitoring** continuously watches network and system activity for anomalies that don't match any known signature — the mechanism for catching novel or zero-day attacks.
4. A **Blue Team** layer analyzes incoming data and alerts, and decides how the system should respond, acting as the adaptive decision-making core.
5. Based on that analysis, the system triggers a **dynamic response**: alerting and blocking known threats, updating its memory, or redirecting attackers into honeypots and deception environments to waste their time and gather intelligence.

The AI component in this repository is responsible for the recognition side of that loop — training models that can look at network traffic and determine whether it's malicious and, if so, what kind of attack it is. That's the specific piece implemented and evaluated here.

## Datasets & Notebooks

| Notebook | Dataset | Attack categories |
|---|---|---|
| [`CICIDS2017.ipynb`](./CICIDS2017.ipynb) | [CICIDS2017](https://www.unb.ca/cic/datasets/ids-2017.html) | DoS, DDoS, Brute Force, Web Attack, Port Scan, Bot, Infiltration, Heartbleed |
| [`NSL_KDD.ipynb`](./NSL_KDD.ipynb) | [NSL-KDD](https://www.unb.ca/cic/datasets/nsl.html) | DoS, Probe, R2L, U2R |
| [`UNSW_NB15.ipynb`](./UNSW_NB15.ipynb) | [UNSW-NB15](https://research.unsw.edu.au/projects/unsw-nb15-dataset) | Analysis, Backdoor, DoS, Exploits, Fuzzers, Generic, Reconnaissance, Shellcode, Worms |

Each notebook follows the same overall structure — load data, clean and preprocess, run exploratory data analysis, then train and evaluate XGBoost classifiers — with preprocessing and feature choices tailored to each dataset.

## Pipeline

Across all three notebooks:
1. **Load & merge** the dataset's provided train/test (or multi-file) CSVs into one dataframe
2. **Clean** — normalize column names/labels, remove duplicates, handle missing/infinite/negative values, drop constant or near-constant columns
3. **Derive labels** — binary (normal/attack) and multi-class category columns from the raw label field
4. **EDA** — class balance, feature distributions, outlier analysis, correlation analysis, and behavioral differences between normal and attack traffic
5. **Feature engineering** — drop redundant/highly correlated features, encode categorical features (one-hot), scale numerical features (`StandardScaler` or `RobustScaler` depending on the dataset), address class imbalance with `compute_sample_weight`
6. **Modeling** — train `XGBClassifier` models for detection and classification tasks, evaluate with accuracy and full classification reports

## Models & Results

| Dataset | Task | Accuracy |
|---|---|---|
| CICIDS2017 | Binary detection | 99.92% |
| CICIDS2017 | Attack classification (major category) | 99.98% |
| CICIDS2017 | Sub-attack classification | 99.77% |
| NSL-KDD | Binary detection | 99.79% |
| NSL-KDD | Major-category classification | 99.98% |
| NSL-KDD | Fine-grained attack classification | 99.65% |
| UNSW-NB15 | Binary detection | 93.88% |
| UNSW-NB15 | Attack-family classification | 92.69% |
| UNSW-NB15 | DoS-family sub-classification | 56.6% |

CICIDS2017 and NSL-KDD both reach near-perfect accuracy on binary detection and category classification, consistent with published benchmarks for these datasets — though per-class performance on rare attack types (e.g. Heartbleed, individual NSL-KDD attack labels) is noticeably weaker than the headline accuracy suggests. UNSW-NB15 is harder: detection and family-level classification land in the low-to-mid 90s, and a dedicated sub-model built to separate the four most-confused categories (Analysis, Backdoor, DoS, Exploits) only reaches ~57% accuracy, reflecting genuine overlap between these attack types in the dataset's feature space.

## Tech Stack

- **Language:** Python
- **Data & ML:** Pandas, NumPy, Scikit-learn, XGBoost, imbalanced-learn
- **Visualization:** Matplotlib, Seaborn

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

Each notebook was originally built to run on Google Colab and loads its dataset from Google Drive. To run locally, remove the `google.colab`/`drive.mount()` calls and update the file paths to point to your local copies of each dataset.

## Repository Structure

```
CICIDS2017.ipynb   # Detection & classification pipeline on CICIDS2017
NSL_KDD.ipynb       # Detection & classification pipeline on NSL-KDD
UNSW_NB15.ipynb      # Detection & classification pipeline on UNSW-NB15
```

## Acknowledgments

- Sharafaldin, I., Lashkari, A.H., and Ghorbani, A.A., "Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization", ICISSP 2018. (CICIDS2017)
- Tavallaee, M., Bagheri, E., Lu, W., and Ghorbani, A.A., "A Detailed Analysis of the KDD CUP 99 Data Set", IEEE CISDA 2009. (NSL-KDD)
- Moustafa, N. and Slay, J., "UNSW-NB15: A Comprehensive Data Set for Network Intrusion Detection Systems", MilCIS 2015. (UNSW-NB15)
