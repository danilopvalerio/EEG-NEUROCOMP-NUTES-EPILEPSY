# Epileptic Seizure Detection — CHB-MIT (LOOCV)

Complete pipeline for epileptic seizure detection using the CHB-MIT dataset with a **subject-specific** approach, **LOOCV (Leave-One-Out Cross-Validation)** validation, and comparison among:

- SVM (RBF)
- XGBoost
- Random Forest

---

## 📂 Project Structure

```
NOTE: Remember to create the data and data_processed folders
.
├── data/                  # Raw dataset (CHB-MIT)
│   ├── chb01
│   ├── chb02
│   ├── chb03
│   ├── chb04
│   └── chb05
│
├── data_processed/        # Extracted features and processed windows
│
├── results/
│   ├── global/            # Aggregated results
│   └── subject_specific/  # Patient-specific results
│
├── utils/                 # Helper functions
│
├── pipeline_epilepsia_final.ipynb
├── XGBoost.yml
├── README.md
└── .gitignore
```

---

## 📊 Dataset

- CHB-MIT Scalp EEG Database
- Sampling frequency: 256 Hz
- Multichannel EEG
- Binary classification (seizure vs non-seizure)

---

## 🧪 Methodology

### 🔹 Strategy

- Subject-specific training
- 4-second windows
- LOOCV per trial
- Manual feature extraction (time/frequency)

### 🔹 Evaluated Models

- SVM (RBF kernel)
- Random Forest
- XGBoost

### 🔹 Metrics

- Sensitivity
- Specificity
- F1-score
- Accuracy

---

## ⏱ Runtime Comparison (LOOCV)

| Model         | Total Time |
| ------------- | ---------- |
| SVM (RBF)     | ~1 min     |
| XGBoost       | ~9 min     |
| Random Forest | ~71 min    |

---

## 📈 Results

Individual results:

```
results/subject_specific/
```

Aggregated results:

```
results/global/
```

Plots:

- comparacao_modelos.png
- filtro_comparacao.png
- chb04_tradeoff.png

---

## ⚙️ Environment

Create the environment with conda:

```
conda env create -f XGBoost.yml
conda activate xgboost
```

Or using venv:

```
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## 🚀 How to Run

Open:

```
pipeline_epilepsia_final.ipynb
```

Run the cells in order.

---

## 🎯 Goal

- Compare classical EEG models
- Evaluate computational cost
- Analyze feasibility for embedded systems

---

## 📌 Note

The dataset is not versioned (see `.gitignore`).
