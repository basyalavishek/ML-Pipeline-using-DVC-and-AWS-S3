# ML Pipeline using DVC and AWS S3

An end-to-end SMS spam classification pipeline using [DVC](https://dvc.org/) for pipeline orchestration, versioning, and experiment tracking, with **AWS S3** as the remote data/model storage backend.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Pipeline Stages](#pipeline-stages)
- [Getting Started](#getting-started)
- [Usage](#usage)
  - [Run the Pipeline](#run-the-pipeline)
  - [Run Experiments](#run-experiments)
  - [Sync with AWS S3](#sync-with-aws-s3)
- [Configuration](#configuration)
- [Results](#results)
- [Tech Stack](#tech-stack)

## Overview

This project builds a machine learning pipeline that classifies SMS messages as **spam** or **ham** (not spam). The entire workflow — from raw data ingestion to model evaluation — is automated with DVC, which tracks every data artifact, model, and metric. All large artifacts (data, models, reports) are stored in an AWS S3 bucket while the lightweight pipeline definitions and code live in Git.

## Features

- **5-stage automated ML pipeline** defined in `dvc.yaml` and orchestrated with `dvc repro`.
- **Parameterized training** via `params.yaml` — change hyperparameters without touching code.
- **Experiment tracking with DVC Live** (`dvclive`) — log metrics and params per run, compare experiments with `dvc exp show`.
- **Versioned data & models** — every pipeline output is content-addressed and tracked.
- **AWS S3 remote storage** — data, models, and metrics are synced to the cloud and versioned.
- **Reproducible builds** — dependencies managed with `uv` and pinned via `uv.lock`.

## Project Structure

```
.
├── data/                    # Pipeline artifacts (tracked by DVC, NOT git)
│   ├── raw/                 # train.csv, test.csv (data_ingestion)
│   ├── interim/             # preprocessed train/test (data_preprocessing)
│   └── processed/           # TF-IDF vectorized train/test (feature_engineering)
├── dataset/
│   └── spam.csv             # Source dataset
├── experiments/             # DVC experiment outputs
├── models/                  # Trained model artifacts (tracked by DVC)
│   └── model.pkl
├── reports/                 # Evaluation metrics (tracked by DVC)
│   └── metrics.json
├── src/                     # Pipeline stage scripts
│   ├── data_ingestion.py
│   ├── data_preprocessing.py
│   ├── feature_engineering.py
│   ├── model_building.py
│   └── model_evaluation.py
├── logs/                    # Per-stage logging output
├── dvclive/                 # DVC Live experiment tracking output
├── dvc.yaml                 # DVC pipeline definition
├── dvc.lock                 # Locked hashes of pipeline inputs/outputs
├── params.yaml              # Configurable pipeline parameters
├── pyproject.toml           # Project metadata & dependencies
├── uv.lock                  # Locked dependency versions
└── .dvc/                    # DVC config & internal cache
```

## Pipeline Stages

| Stage | Script | Input | Output |
|-------|--------|-------|--------|
| `data_ingestion` | `src/data_ingestion.py` | `dataset/spam.csv` | `data/raw/{train,test}.csv` |
| `data_preprocessing` | `src/data_preprocessing.py` | `data/raw` | `data/interim/*_processed.csv` |
| `feature_engineering` | `src/feature_engineering.py` | `data/interim` | `data/processed/*_tfidf.csv` |
| `model_building` | `src/model_building.py` | `data/processed` | `models/model.pkl` |
| `model_evaluation` | `src/model_evaluation.py` | `models/model.pkl` | `reports/metrics.json` |

**What each stage does:**

1. **Data Ingestion** — Downloads the raw dataset, drops irrelevant columns, renames `v1`/`v2` to `target`/`text`, and splits into train/test sets using `test_size` from `params.yaml`.
2. **Data Preprocessing** — Encodes the target labels, removes duplicate rows, and normalizes text (lowercasing, tokenization, stopword/punctuation removal, stemming).
3. **Feature Engineering** — Applies **TF-IDF vectorization** with `max_features` dimensions to convert text into numerical features.
4. **Model Building** — Trains a **Random Forest classifier** with `n_estimators` and `random_state` from `params.yaml`, saving it as `models/model.pkl`.
5. **Model Evaluation** — Loads the saved model, predicts on the test set, computes metrics (accuracy, precision, recall, AUC), saves them to `reports/metrics.json`, and logs them with **DVC Live**.

## Getting Started

### Prerequisites

- Python **>= 3.12**
- [uv](https://docs.astral.sh/uv/) (package manager)
- [DVC](https://dvc.org/doc/install) (installed via project dependencies)
- An **AWS account** with an S3 bucket and IAM credentials (only needed for the S3 remote)

### Installation

```bash
# 1. Install dependencies
uv sync

# 2. Activate the virtual environment
source .venv/bin/activate
```

## Usage

### Run the Pipeline

```bash
# Run the entire pipeline (only re-runs stages whose inputs changed)
dvc repro

# View the pipeline DAG
dvc dag

# Pull data/models from the remote (on a fresh clone)
dvc pull
```

### Run Experiments

This project uses **DVC Live** (`dvclive`) to log metrics and parameters for every run.

```bash
# Run a new experiment (creates a DVC experiment, not a git commit)
dvc exp run

# List all experiments and compare their metrics
dvc exp show

# Apply a previous experiment to the workspace
dvc exp apply <experiment-name>

# Remove an experiment
dvc exp remove <experiment-name>
```

**Tip:** Experiment metrics/params can also be explored visually via the [DVC VS Code extension](https://marketplace.visualstudio.com/items?itemName=Iterative.dvc).

### Sync with AWS S3

The DVC remote is configured to `s3://ml-pipeline-dvc`. To set it up on a fresh clone:

```bash
# 1. Install S3 support
uv pip install "dvc[s3]" awscli

# 2. Configure AWS credentials
aws configure

# 3. Add the remote (if not already configured)
dvc remote add -d dvcstore s3://ml-pipeline-dvc

# 4. Push pipeline artifacts to S3
dvc push

# 5. Pull artifacts from S3
dvc pull
```

Commit the code, `dvc.yaml`, and `dvc.lock` to Git as usual — the heavy artifacts stay in S3.

## Configuration

All tunable parameters live in `params.yaml`:

```yaml
data_ingestion:
  test_size: 0.20

feature_engineering:
  max_features: 35

model_building:
  n_estimators: 22
  random_state: 2
```

Edit any value, then run `dvc repro` or `dvc exp run` — DVC will detect the parameter change, re-run only the affected stages, and update all downstream artifacts.

## Results

Latest model metrics (stored in `reports/metrics.json`):

| Metric | Value |
|--------|-------|
| Accuracy | 0.9401 |
| Precision | 0.8629 |
| Recall | 0.6859 |
| AUC | 0.9165 |

## Tech Stack

- **DVC** — pipeline orchestration, data/model versioning, experiment tracking
- **DVC Live** (`dvclive`) — experiment logging and metrics visualization
- **AWS S3** — remote storage for data, models, and metrics
- **scikit-learn** — preprocessing, TF-IDF, Random Forest
- **NLTK** — text preprocessing (tokenization, stopwords, stemming)
- **uv** — Python dependency management
