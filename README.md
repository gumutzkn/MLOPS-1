## MLOps Project 1

Hotel reservation cancellation prediction with a simple MLOps pipeline.

### Quick start (local)

- Python 3.11 önerilir. Sanal ortam oluştur: `python -m venv venv && source venv/bin/activate` (Windows: `venv\Scripts\activate`).
- Bağımlılıkları kur: `pip install --upgrade pip && pip install -r requirements.txt`.
- Ham veri: `train.csv` repo kökünde (artifacts klasörü git dışında). Gerekirse `config/config.yaml` içindeki bucket ayarlarını düzenle.
- Eğitim/pipeline: `python pipeline/training_pipeline.py`.
- Çıktılar: `artifacts/processed/` ve model `artifacts/models/lgbm_model.pkl`.

### Docker

- İmaj derle: `docker build -t ml-project:latest .`
- Çalıştır: `docker run --rm ml-project:latest python pipeline/training_pipeline.py`
- GCP ile kullanırken container içine servis hesabı anahtarını `GOOGLE_APPLICATION_CREDENTIALS=/app/key.json` ve `GCS_BUCKET_NAME` ortam değişkenleriyle geç (Jenkinsfile örnek akış).

### Jenkins (özet)

- Credentials: `github-token`, `gcp-project-id`, `gcp-bucket-name`, `gcp-key` (file). Pipeline bu ID’leri bekler; gizli değerler Jenkins’te kalır.
- Aşamalar: checkout → docker build → container içinde eğitim → GCR push → Cloud Run deploy.

### Güvenlik notları

- `key.json` ve diğer gerçek gizli dosyalar `.gitignore` ile dışlandı; repoya eklemeyin.
- GCP proje/bucket adlarını paylaşırken gizlilik gerekiyorsa placeholder kullanın.

### Bilinen gereksinimler

- Bağımlılık sürümleri `requirements.txt`’te sabitlendi (scikit-learn/imbalanced-learn uyumlu).
- Hafif veri ön işleme + SMOTE + LightGBM rastgele arama parametreleri `config/model_params.py` ve `config/config.yaml`’da.

### Test/Doğrulama

- Temel doğrulama: `python pipeline/training_pipeline.py` komutunun hatasız bitmesi.
- Docker doğrulama: `docker run --rm ml-project:latest python pipeline/training_pipeline.py`.
