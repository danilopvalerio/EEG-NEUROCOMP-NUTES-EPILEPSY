# 🧠 Epileptic Seizure Detection — CHB-MIT (LOOCV)

Pipeline completo para detecção de crises epilépticas utilizando o dataset CHB-MIT com abordagem **subject-specific**, validação **LOOCV (Leave-One-Out Cross-Validation)** e comparação entre:

- SVM (RBF)
- XGBoost
- Random Forest

---

## 📂 Estrutura do Projeto

```
.
├── data/                  # Dataset bruto (CHB-MIT)
│   ├── chb01
│   ├── chb02
│   ├── chb03
│   ├── chb04
│   └── chb05
│
├── data_processed/        # Features extraídas e janelas processadas
│
├── results/
│   ├── global/            # Resultados agregados
│   └── subject_specific/  # Resultados por paciente
│
├── utils/                 # Funções auxiliares
│
├── pipeline_epilepsia_final.ipynb
├── XGBoost.yml
├── README.md
└── .gitignore
```

---

## 📊 Dataset

- CHB-MIT Scalp EEG Database
- Frequência de amostragem: 256 Hz
- EEG multicanal
- Classificação binária (crise vs não-crise)

---

## 🧪 Metodologia

### 🔹 Estratégia

- Treinamento subject-specific
- Janelas de 4 segundos
- LOOCV por trial
- Extração de features manuais (tempo/frequência)

### 🔹 Modelos avaliados

- SVM (kernel RBF)
- Random Forest
- XGBoost

### 🔹 Métricas

- Sensitivity
- Specificity
- F1-score
- Accuracy

---

## ⏱ Comparação de Tempo (LOOCV)

| Modelo        | Tempo Total |
| ------------- | ----------- |
| SVM (RBF)     | ~1 min      |
| XGBoost       | ~9 min      |
| Random Forest | ~71 min     |

---

## 📈 Resultados

Resultados individuais:

```
results/subject_specific/
```

Resultados agregados:

```
results/global/
```

Gráficos:

- comparacao_modelos.png
- filtro_comparacao.png
- chb04_tradeoff.png

---

## ⚙️ Ambiente

Criar ambiente com conda:

```
conda env create -f XGBoost.yml
conda activate xgboost
```

Ou usando venv:

```
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## 🚀 Como Executar

Abrir:

```
pipeline_epilepsia_final.ipynb
```

Executar as células na ordem.

---

## 🎯 Objetivo

- Comparar modelos clássicos em EEG
- Avaliar custo computacional
- Analisar viabilidade para sistemas embarcados

---

## 📌 Observação

O dataset não está versionado (ver `.gitignore`).
