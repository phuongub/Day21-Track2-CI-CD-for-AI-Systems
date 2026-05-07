# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an educational MLOps lab (VinUni AIInAction — Day 21: CI/CD for AI Systems). It teaches students how to build an end-to-end MLOps pipeline for a Wine Quality classification model. The codebase is intentionally filled with `# TODO` placeholders that students complete across three steps.

**Goal:** Train a RandomForest classifier on the UCI Wine Quality dataset, track experiments with MLflow, version data with DVC, and deploy via GitHub Actions CI/CD to a cloud VM running FastAPI.

## Architecture

```
[Local Machine]  --git push-->  [GitHub]  --Actions-->  [Runner: Test -> Train -> Eval -> Deploy]
                                                                    |                    |
                                                              dvc pull/push             REST API
                                                                    |                    v
                                                            [Cloud Storage]          [Cloud VM]
                                                              data/                  mlops-serve
                                                              models/latest/         POST /predict
```

- **Training (`src/train.py`):** Reads `params.yaml` and CSV data, trains `RandomForestClassifier`, logs metrics (`accuracy`, `f1_score`) to MLflow, writes `outputs/metrics.json`, and saves `models/model.pkl`.
- **Serving (`src/serve.py`):** FastAPI app running on a cloud VM. On startup, downloads `models/latest/model.pkl` from object storage. Exposes `GET /health` and `POST /predict` (expects 12 float features).
- **CI/CD (`.github/workflows/mlops.yml`):** Four sequential jobs triggered on push to `main` when `data/**.dvc`, `src/**.py`, or `params.yaml` change:
  1. **test** — runs `pytest tests/ -v` on synthetic data (no cloud needed).
  2. **train** — authenticates to cloud, `dvc pull`s data, runs `python src/train.py`, uploads model to object storage, saves metrics artifact.
  3. **eval** — reads `accuracy` from train job output; fails pipeline if `accuracy < 0.70`.
  4. **deploy** — SSHs into VM, restarts `mlops-serve` systemd service, and health-checks `/health`.
- **Data Versioning (DVC):** Large CSVs live in cloud storage; `.dvc` pointer files are committed to git. Students run `dvc push` before `git push` so CI runners can pull the data.

## Common Commands

### Environment Setup
```bash
python -m venv .venv
# Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### Data Generation (run once)
```bash
python generate_data.py
```

### Local Training & Experiment Tracking
```bash
# Run training (reads params.yaml)
python src/train.py

# Launch MLflow UI to compare experiments
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

### Tests
```bash
# Run all unit tests
pytest tests/ -v

# Run a single test
pytest tests/test_train.py::test_train_returns_float -v
```

### DVC (Data Version Control)
```bash
dvc init
dvc remote add -d myremote gs://<BUCKET>/dvc   # or s3://, azure://
dvc remote modify myremote credentialpath sa-key.json
dvc add data/train_phase1.csv
dvc add data/eval.csv
dvc add data/train_phase2.csv
dvc push
```

### Simulating New Data (Step 3)
```bash
python add_new_data.py   # Appends train_phase2.csv into train_phase1.csv
dvc add data/train_phase1.csv
git add data/train_phase1.csv.dvc
git commit -m "data: add new samples"
dvc push
git push origin main
```

## Key Files & Their Roles

| File | Purpose |
|---|---|
| `src/train.py` | Training script with MLflow logging. Must produce `outputs/metrics.json` and `models/model.pkl`. |
| `src/serve.py` | FastAPI inference server. Reads `GCS_BUCKET` env var. |
| `tests/test_train.py` | Unit tests using temporary synthetic data. Validates train output, metrics file, and model file. |
| `.github/workflows/mlops.yml` | CI/CD pipeline definition. Four jobs: test → train → eval → deploy. |
| `params.yaml` | Hyperparameters for `RandomForestClassifier` (`n_estimators`, `max_depth`, `min_samples_split`). |
| `generate_data.py` | Downloads UCI Wine Quality dataset and splits into `train_phase1.csv`, `eval.csv`, `train_phase2.csv`. |
| `add_new_data.py` | Merges `train_phase2.csv` into `train_phase1.csv` to simulate data updates. |
| `requirements.txt` | Pins versions: `mlflow==2.13.0`, `scikit-learn==1.4.2`, `dvc[gs]==3.50.1`, `fastapi==0.111.0`, etc. |

## Important Notes

- **Cloud Provider:** The lab uses GCP as the default example (GCS, GCE), but AWS and Azure are supported with equivalent substitutions. If modifying cloud SDK calls, preserve the abstraction pattern (e.g., `google-cloud-storage` vs `boto3`).
- **Eval Gate:** The accuracy threshold is hardcoded at `0.70` in `src/train.py` (`EVAL_THRESHOLD`) and enforced in the GitHub Actions `eval` job. The deploy job only runs if this gate passes.
- **Systemd Service:** On the VM, `mlops-serve.service` runs `src/serve.py`. It expects `GOOGLE_APPLICATION_CREDENTIALS` and `GCS_BUCKET` environment variables. The service must be restarted after each model upload for the new artifact to be downloaded.
- **Test Isolation:** `tests/test_train.py` creates small random datasets in a `tmp_path` so tests can run in CI without cloud credentials or DVC.
- **Git Ignore:** Data CSVs, `mlflow.db`, `models/`, `outputs/`, and `sa-key.json` are ignored. Only `.dvc` pointer files should be committed for data.
