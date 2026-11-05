# CardioGluco

A Flask portfolio demo that maps eight entered health markers to one of three Random Forest classes and displays the model's class scores—not a diagnosis.

![Python 3.11.9](https://img.shields.io/badge/python-3.11.9-3776AB?logo=python&logoColor=white)
![Flask 3.0.3](https://img.shields.io/badge/Flask-3.0.3-000000?logo=flask&logoColor=white)
![scikit-learn 1.5.0](https://img.shields.io/badge/scikit--learn-1.5.0-F7931E?logo=scikit-learn&logoColor=white)


## Overview

CardioGluco is a small Flask web application with a browser-based form and a JSON prediction endpoint. A Random Forest consumes age, gender, BMI, blood pressure, blood sugar, cholesterol, smoking, and family-history values, then returns one class and scores for all three classes. The data and labels in this repository are small and have no documented clinical provenance, so the project demonstrates an end-to-end ML interface rather than validated disease prediction. It is intended for educational and portfolio use only.

## Key features

- Interactive form for eight inputs; the browser sliders constrain numeric values to Age 18–90, BMI 15–45, BP 80–200, Sugar 70–300, and Cholesterol 100–320.
- `POST /predict` returns the predicted class, a rounded score for each class, and a static recommendation bundle keyed to the prediction.
- One Random Forest predicts exactly one of `Diabetes`, `Heart Disease`, or `No Disease`. This is not a multi-label model; a person can have both diabetes and heart disease.
- Flask serves the UI and API locally; `render.yaml` configures Gunicorn for Render.

## Data provenance and limitations

`health_data.csv` contains 15 rows and nine columns: eight input features plus `Disease`. No dataset citation, public source, or clinical collection method is documented in the repository. The CSV rows match the first 15 records embedded in `train_model.py`; that file describes its 40 hard-coded records as “your original 15 rows + more synthetic data.” The origin of those first 15 records is not explained, while the additional 25 are presented as synthetic. Treat the available data as unverified, constructed examples—not clinical records.

| Item | Observed in `health_data.csv` |
|---|---|
| Rows / columns | 15 / 9 |
| Class counts | Diabetes: 5; Heart Disease: 4; No Disease: 6 |
| Missing values | 0 in all columns |
| Age | 28–65 |
| BMI | 22.0–34.2 |
| BP | 110–150 |
| Sugar | 80–210 |
| Cholesterol | 170–250 |
| Gender | Male: 8; Female: 7 |
| Smoking | Yes: 6; No: 9 |
| Family history | Yes: 8; No: 7 |

The `Disease` values are already assigned in the CSV and in the training script. There is no recorded clinical outcome, labeling protocol, or executable threshold rule that explains how they were assigned. The model therefore learns associations between these input rows and their supplied labels; if those labels were chosen from the same feature patterns, the model reproduces those assumptions rather than discovering independent clinical outcomes. The repository does not provide enough provenance to resolve that possibility.

## Model and evaluation

`train_model.py` does **not** load `health_data.csv`: it builds a separate 40-row dataset directly in code. It label-encodes Gender, Smoking, FamilyHistory, and Disease; uses no feature scaling; and fits `RandomForestClassifier(n_estimators=100, random_state=42)`. Its 80/20 split uses `random_state=42` without stratification. The script prints a holdout accuracy and class names, then pickles the estimator and encoders as `model.pkl`; it does not save an evaluation report.

The committed `model.pkl` is loaded by `app.py` at startup. `render.yaml` runs `train_model.py` during the Render build, so deployment recreates the artifact from the 40 hard-coded rows before starting Gunicorn. `scikit-learn==1.5.0` is pinned in `requirements.txt`, which helps keep pickle producer and consumer versions aligned; scikit-learn pickles are still version-sensitive and should only be loaded from a trusted source.

The reproducible evaluation below is deliberately separate: `evaluate_model.py` fits the same Random Forest settings to `health_data.csv`, using a stratified 80/20 split with seed 42, then runs shuffled 5-fold stratified cross-validation. It does **not** evaluate the committed or deployed `model.pkl`.

| Evaluation on `health_data.csv` | Result |
|---|---:|
| Stratified holdout accuracy | 1.000 |
| Stratified holdout macro-F1 | 1.000 |
| 5-fold CV accuracy, mean ± std | 0.867 ± 0.163 |
| 5-fold CV macro-F1, mean ± std | 0.756 ± 0.204 |

The holdout contains only three examples—one per class—so its perfect scores are not meaningful evidence of generalization. Heart Disease has only four examples, fewer than the five CV folds; one validation fold cannot contain an example of that class, and the CV estimate is highly unstable. The supplied labels and unknown provenance are further limitations. These metrics describe agreement with this tiny table only, not medical or real-world performance.

Reproduce the metrics and regenerate [`docs/evaluation.md`](docs/evaluation.md):

```bash
pip install -r requirements.txt
python evaluate_model.py
```

## Application and API

| Method and route | Behavior |
|---|---|
| `GET /` | Serves the form in `templates/index.html`. |
| `POST /predict` | Accepts JSON and returns a prediction, class scores, and recommendations. |

The JSON input keys are `age`, `gender`, `bmi`, `bp`, `sugar`, `cholesterol`, `smoking`, and `familyHistory`. Gender must be `Male` or `Female`; Smoking and Family History must be `Yes` or `No`. Numeric values are converted to floats. The slider limits are client-side only: the API does not enforce those ranges or additional type/domain constraints. Exceptions are returned as HTTP 400 JSON with the exception text in `error`.

The response's `probabilities` values are `predict_proba` scores rounded to one decimal place; they are not described or validated as calibrated clinical probabilities. Recommendations come from fixed per-class content in `recommendations.py`, not from a patient-specific treatment plan. The app runs with debug mode disabled and has no authentication or application secret configured.

## Run locally

Python 3.11.9 is configured in both `runtime.txt` and `render.yaml`.

```bash
pip install -r requirements.txt
python train_model.py
python app.py
```

Open `http://127.0.0.1:5000`. `app.py` loads `model.pkl` when the module starts, so train the model first if the file is absent or you want to regenerate it.

## Deploy to Render

The checked-in Blueprint configuration defines:

- Service: `cardio-gluco` (Python web service)
- Build: `pip install -r requirements.txt && python train_model.py`
- Start: `gunicorn app:app`
- `PYTHON_VERSION=3.11.9`, matching `runtime.txt`

No verified live URL is recorded in this repository.

## Repository notes

- `model.pkl` is tracked in Git; `train_model.py` writes it, and the Render build overwrites it by retraining.
- `diabetes.jpg` and `diabetes-symptoms-and-treatment.jpg` are not referenced by the active app or `templates/index.html`; they are not used as UI screenshots.
- No automated test suite, CI workflow, or `LICENSE` file is present.
- This project is not suitable for diagnosis, treatment decisions, or other clinical use. Consult a licensed healthcare professional for health concerns.
