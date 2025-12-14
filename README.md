## MLOps Project 1 — Hotel Reservation Cancellation Prediction

End-to-end MLOps-style project that trains a **LightGBM** model to predict hotel reservation cancellations, tracks experiments with **MLflow**, stores artifacts locally and in **Google Cloud Storage (GCS)**, and serves predictions via a **Flask** web app. Includes **Docker** support and an example **Jenkins → GCR → Cloud Run** deployment pipeline.

---

### What’s included

- **Training pipeline**: ingestion → preprocessing → training + evaluation → artifact persistence
- **Preprocessing**: dedup, label encoding, skewness handling, SMOTE balancing, feature selection
- **Modeling**: LightGBM + `RandomizedSearchCV`
- **Experiment tracking**: MLflow params + metrics + model artifact
- **Artifact management**:
  - Local: `artifacts/` (raw/processed/model)
  - Remote: uploads model to `gs://<bucket>/models/lgbm_model.pkl`
- **Inference UI**: Flask + HTML form (`templates/index.html`)
- **CI/CD**: Jenkins pipeline that builds image, trains inside container, pushes to GCR, deploys to Cloud Run

---

### High-level architecture

```text
            ┌──────────────────────────┐
            │   GCS (dataset + model)  │
            │  - Hotel_Reservations... │
            │  - models/lgbm_model.pkl │
            └─────────────┬────────────┘
                          │
                          │ (training downloads data / uploads model)
                          ▼
┌───────────────────────────────────────────────────────────────┐
│ training_pipeline.py                                          │
│  1) DataIngestion  → artifacts/raw/{raw,train,test}.csv        │
│  2) DataProcessor  → artifacts/processed/{processed_*.csv}     │
│  3) ModelTraining  → artifacts/models/lgbm_model.pkl + MLflow  │
└───────────────────────────────────────────────────────────────┘
                          │
                          │ (serving downloads model if missing)
                          ▼
┌───────────────────────────────────────────────────────────────┐
│ Flask app (application.py)                                    │
│  - GET/POST / → HTML form → prediction                         │
└───────────────────────────────────────────────────────────────┘
```

---

### Repository structure

```text
.
├─ application.py                  # Flask serving app
├─ pipeline/
│  └─ training_pipeline.py         # Orchestrates ingestion → processing → training
├─ src/
│  ├─ data_ingestion.py            # Downloads CSV from GCS and splits train/test
│  ├─ data_preprocessing.py        # Preprocess + SMOTE + feature selection
│  └─ model_training.py            # LightGBM + MLflow + upload model to GCS
├─ config/
│  ├─ config.yaml                  # Ingestion + preprocessing configuration
│  ├─ model_params.py              # Hyperparameter search space
│  └─ paths_config.py              # Artifact paths
├─ templates/index.html            # Inference UI
├─ static/style.css
├─ Dockerfile
├─ Jenkinsfile
└─ requirements.txt
```

---

### Prerequisites

- **Python**: 3.11 recommended
- **GCP** (for the default ingestion + model download/upload flow):
  - A **GCS bucket** containing `Hotel_Reservations.csv`
  - A **service account** key with permissions to read the dataset and upload the trained model

---

### Configuration

- **YAML config**: `config/config.yaml`

  - `data_ingestion.bucket_name`: default bucket (currently set to `PLACEHOLDER`)
  - `data_ingestion.bucket_file_name`: CSV object name (default: `Hotel_Reservations.csv`)
  - `data_processing.*`: column lists, skewness threshold, number of selected features

- **Environment variables** (recommended over hardcoding):
  - `GCS_BUCKET_NAME` (required unless you replace the placeholder in config)
  - `GOOGLE_APPLICATION_CREDENTIALS` (path to service account JSON)
  - `MLFLOW_TRACKING_URI` (optional; if not set, MLflow logs locally)
  - `PORT` (serving port; defaults to `8080` in `application.py`)

---

### Quick start (local)

#### 1) Create venv + install

PowerShell (Windows):

```powershell
python -m venv venv
venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

macOS/Linux:

```bash
python -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

> Note: the Docker image installs the project in editable mode (`pip install -e .`). Locally, installing from `requirements.txt` is enough for running scripts.

#### 2) Set GCP env vars

PowerShell (Windows):

```powershell
$env:GCS_BUCKET_NAME = "your-bucket-name"
$env:GOOGLE_APPLICATION_CREDENTIALS = "C:\path\to\key.json"
# Optional
$env:MLFLOW_TRACKING_URI = "http://127.0.0.1:5000"   # or your remote tracking server
```

macOS/Linux:

```bash
export GCS_BUCKET_NAME="your-bucket-name"
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
# Optional
export MLFLOW_TRACKING_URI="http://127.0.0.1:5000"
```

#### 3) Run the training pipeline

```bash
python pipeline/training_pipeline.py
```

**Outputs**

- `artifacts/raw/raw.csv`, `artifacts/raw/train.csv`, `artifacts/raw/test.csv`
- `artifacts/processed/processed_train.csv`, `artifacts/processed/processed_test.csv`
- `artifacts/models/lgbm_model.pkl`
- Model uploaded to GCS: `gs://$GCS_BUCKET_NAME/models/lgbm_model.pkl`

---

### MLflow tracking (optional but recommended)

By default, MLflow logs locally to `./mlruns` (git-ignored). You can view runs with:

```bash
mlflow ui --host 127.0.0.1 --port 5000
```

If you have a remote MLflow server, set `MLFLOW_TRACKING_URI` before running training to log there.

---

### Run the Flask app (local)

```bash
python application.py
```

Then open `http://127.0.0.1:8080/`.

**Important**: the UI currently submits **already-encoded numeric** values for categorical fields (e.g., `market_segment_type`, `room_type_reserved`). This matches the current app design.

#### Notes on feature consistency

The training pipeline performs **feature selection** (`data_processing.no_of_features`) and the Flask app sends **a fixed set of 10 features in a fixed order**. In a production-grade setup, you should persist and reuse the exact preprocessing + selected feature list for inference to avoid training/serving skew.

---

### Docker

#### Build

```bash
docker build -t ml-project:latest .
```

> Note: the `Dockerfile` contains `EXPOSE 5000`, but the app actually binds to `PORT` (default `8080`). Use `-e PORT=...` and map the same port with `-p`.

#### Run the Flask app (container)

macOS/Linux:

```bash
docker run --rm -p 8080:8080 \
  -e PORT=8080 \
  -e GCS_BUCKET_NAME="$GCS_BUCKET_NAME" \
  -e GOOGLE_APPLICATION_CREDENTIALS=/app/key.json \
  -v "$GOOGLE_APPLICATION_CREDENTIALS":/app/key.json:ro \
  ml-project:latest
```

PowerShell (Windows):

```powershell
docker run --rm -p 8080:8080 `
  -e PORT=8080 `
  -e GCS_BUCKET_NAME="$env:GCS_BUCKET_NAME" `
  -e GOOGLE_APPLICATION_CREDENTIALS=/app/key.json `
  -v "$env:GOOGLE_APPLICATION_CREDENTIALS":/app/key.json:ro `
  ml-project:latest
```

#### Run training inside the container

```bash
docker run --rm \
  -e GCS_BUCKET_NAME="$GCS_BUCKET_NAME" \
  -e GOOGLE_APPLICATION_CREDENTIALS=/app/key.json \
  -v "$GOOGLE_APPLICATION_CREDENTIALS":/app/key.json:ro \
  ml-project:latest \
  python pipeline/training_pipeline.py
```

---

### Jenkins → GCR → Cloud Run (overview)

The included `Jenkinsfile` implements:

- checkout → docker build
- train inside the container (injects a base64 service account key into `/app/key.json`)
- push image to **GCR**
- deploy to **Cloud Run**

Expected Jenkins credentials IDs:

- `github-token`
- `gcp-project-id`
- `gcp-bucket-name`
- `gcp-key` (file)

> Note: `custom_jenkins/Dockerfile` is a convenience image to run Jenkins with Docker installed.

---

### Troubleshooting

- **“Bucket name not found / PLACEHOLDER”**

  - Set `GCS_BUCKET_NAME` or replace `data_ingestion.bucket_name` in `config/config.yaml`.

- **Google auth errors (DefaultCredentialsError)**

  - Ensure `GOOGLE_APPLICATION_CREDENTIALS` points to a valid service account JSON.

- **Cloud Run port issues**

  - The app binds to `PORT` (default `8080`). Make sure your Cloud Run service is configured to use that same port.

- **LightGBM native dependency errors in Docker**
  - The Dockerfile installs `libgomp1` which LightGBM typically needs.

---

### Security notes

- Do **not** commit service account keys. This repo ignores `key.json` and `*.json` via `.gitignore`.
- Avoid sharing real GCP project/bucket names publicly; use placeholders when needed.
