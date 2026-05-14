# 🧠 Pipeline Computacional para Detecção Automatizada de Crises Epilépticas em Sinais de EEG

![Python](https://img.shields.io/badge/Python-3.9%2B-blue?style=for-the-badge&logo=python&logoColor=white)
![Machine Learning](https://img.shields.io/badge/Machine%20Learning-Scikit--Learn%20%7C%20XGBoost-orange?style=for-the-badge)
![Neuroscience](https://img.shields.io/badge/Neuroscience-EEG%20Processing-lightgrey?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Em%20Desenvolvimento-success?style=for-the-badge)

Este repositório documenta o desenvolvimento, a implementação e a avaliação estatística de um pipeline avançado de _Machine Learning_ aplicado à Neurociência Computacional. O foco central desta pesquisa é a detecção automatizada de crises epilépticas através da análise de sinais contínuos de Eletroencefalograma (EEG), utilizando dados de acesso aberto do dataset multicêntrico **SeizeIT2**.

A motivação por trás desta arquitetura é a busca pelo ponto de equilíbrio ideal (_trade-off_ clínico) entre uma alta **Sensibilidade** (capacidade algorítmica de detectar crises reais) e uma baixíssima **Taxa de Alarmes Falsos (FAR)**. Este balanço é o principal gargalo tecnológico para a viabilização de algoritmos de detecção em dispositivos vestíveis (_wearables_) de baixo consumo energético para monitoramento ambulatorial de longo prazo.

---

## 📑 Índice

1. [Fundamentação Teórica e Escopo](#1-fundamentação-teórica-e-escopo)
2. [O Conjunto de Dados (SeizeIT2)](#2-o-conjunto-de-dados-seizeit2)
3. [Arquitetura em Estágios (Notebooks)](#3-arquitetura-em-estágios-notebooks)
   - [Fase 1: Pré-processamento e Extração](#fase-1-pré-processamento-e-extração-de-features)
   - [Fase 2: Treinamento e Avaliação LOSO](#fase-2-treinamento-e-avaliação-loso)
   - [Fase 3: Análise Clínica e Visualização](#fase-3-análise-clínica-e-visualização)
4. [Métricas de Desempenho](#4-métricas-de-desempenho)
5. [Estrutura de Diretórios](#5-estrutura-de-diretórios)
6. [Guia de Instalação e Execução](#6-guia-de-instalação-e-execução)
7. [Tecnologias Utilizadas](#7-tecnologias-utilizadas)
8. [Contexto Acadêmico (PIBIC - UEPB)](#8-contexto-acadêmico)

---

## 1. Fundamentação Teórica e Escopo

A epilepsia é um distúrbio neurológico caracterizado por descargas elétricas anormais e excessivas no cérebro. O padrão-ouro para diagnóstico é o monitoramento por Vídeo-EEG em ambiente hospitalar, o que é custoso, intrusivo e limitado a curtos períodos. O desenvolvimento de soluções algorítmicas de aprendizado de máquina permite a transição desse monitoramento para o dia a dia do paciente.

Este projeto não apenas treina modelos preditivos, mas constrói um ecossistema completo de tratamento de sinais biológicos, lidando com os desafios inerentes ao EEG: extrema presença de ruídos (musculares, oculares, elétricos) e severo desbalanceamento de classes (pacientes passam a esmagadora maioria do tempo em estado interictal/sem crise).

---

## 2. O Conjunto de Dados (SeizeIT2)

O pipeline foi validado contra a coorte de pacientes do dataset **SeizeIT2** (identificador OpenNeuro `ds005873`).

- **Natureza:** Dados de Eletroencefalografia clínica contínua.
- **Formato de Arquivo:** EDF (_European Data Format_).
- **Anotações (Ground Truth):** Eventos de crise marcados minuciosamente por especialistas clínicos (identificados nos metadados como `eventType = sz*`).
- **Integração Cloud:** A etapa de aquisição dispensa o download manual terabyte a terabyte. O pipeline se conecta diretamente ao _Amazon S3_ hospedado pelo OpenNeuro usando `boto3`, vasculhando os diretórios e transferindo estritamente os arquivos relevantes para a modelagem.

---

## 3. Arquitetura em Estágios (Notebooks)

A modularidade foi priorizada para permitir que pesquisadores ajustem etapas específicas (como a frequência de corte de um filtro) sem precisar reprocessar e retreinar toda a rede estrutural. O código-fonte está dividido em 3 ambientes Jupyter.

### Fase 1: Pré-processamento e Extração de Features

**(Arquivo: `notebook1_preprocessing_features.ipynb`)**

Esta fase lida com a física e a matemática do sinal. É responsável por isolar a atividade cerebral de artefatos externos.

- **Cadeia de Filtragem Digital (Motor: `mne`):**
  1. **Filtro Passa-alta (0.5 Hz):** Projetado para remover o desvio de linha de base (_baseline wander_) associado a movimentos físicos e respiração.
  2. **Filtro Notch (50 Hz):** Filtro rejeita-faixa com Q-factor elevado para anular o ruído harmônico gerado pela rede elétrica local durante a captura dos dados.
  3. **Filtro Passa-baixa (40 Hz):** Atenua frequências superiores onde o ruído de contração muscular (EMG) costuma se sobrepor às ondas cerebrais significativas (Alfa, Beta, Delta, Teta).
- **Suavização Híbrida (WOSG):** Aplicação de uma Transformada _Wavelet_ (`PyWavelets`) seguida por uma filtragem polinomial de Savitzky-Golay, refinando a morfologia da onda.
- **Segmentação Temporal:** O fluxo contínuo de EEG não pode ser processado de uma vez. Ele é dividido usando uma técnica de janela deslizante (_sliding window_) com resoluções configuráveis de **4, 5 e 6 segundos** operando com **50% de sobreposição**.
- **Engenharia de Atributos:** Mapeamento do sinal filtrado em bancos de características estatísticas (FS-A, FS-B, FS-C), variando de momentos estatísticos básicos (média, variância, curtose) a estimativas de entropia e densidade espectral de potência (PSD).
- **Output:** Gera o artefato `pipeline_meta.json`, que atua como um ponteiro de memória persistente para a próxima fase.

### Fase 2: Treinamento e Avaliação LOSO

**(Arquivo: `notebook2_training.ipynb`)**

Esta fase é o núcleo de Inteligência Artificial do pipeline. Seu desenho metodológico impede o vazamento de dados e foca em cenários do mundo real.

- **Paradigma de Validação - LOSO (_Leave-One-Subject-Out_):** Essencial na área da saúde. Se o dataset possui 20 pacientes, o modelo é treinado usando 19 pacientes e testado em 1 paciente virgem de contato com o algoritmo. Esse processo se repete 20 vezes (uma para cada paciente isolado). Isso testa a capacidade real de generalização cerebral do modelo.
- **Tratamento de Desbalanceamento Intenso:** A aplicação de _Undersampling_ (técnicas de subamostragem em razões de 1:3 e 1:5). **Importante:** Isso é aplicado estritamente no conjunto de treinamento para evitar que o modelo sofra viés em direção à classe majoritária, mantendo o _test set_ intacto.
- **Seleção de Características (Dimensionality Reduction):** Para evitar a maldição da dimensionalidade, os modelos são submetidos a uma pré-etapa de `SelectKBest` com pontuação `mutual_info_classif`, isolando os marcadores mais discriminativos.
- **Grid de Experimentos:** O pipeline não roda apenas um modelo. Ele itera sobre **54 configurações distintas**, cruzando os 3 tamanhos de janela, os 3 pacotes de _features_, as taxas de balanceamento e testando diferentes estimadores (_XGBoost_, _Random Forest_ e _Support Vector Machines_).

### Fase 3: Análise Clínica e Visualização

**(Arquivo: `notebook3_analysis.ipynb`)**

A transição da avaliação de _Machine Learning_ puro para a avaliação Neuromédica.

- **Conversão de Janela para Evento:** O paciente não tem uma "crise de 4 segundos", ele tem um evento contínuo. Este notebook funde as predições temporais curtas de volta em eventos prolongados, avaliando se o sistema conseguiria alertar um cuidador a tempo.
- **Otimização Multiobjetivo (Frente de Pareto):** Comparações de _Trade-off_ entre detecções corretas vs geração de fadiga de alarmes. Quais configurações garantem o máximo possível de segurança para o paciente (Sensibilidade) sem inviabilizar o uso do dispositivo por alarmar o tempo todo (FAR)?
- **Comparativo V2 vs Baseline:** Análise quantitativa profunda atestando o ganho numérico trazido pelas técnicas implementadas (filmagens extras, seleção de atributos).
- **Exportação de Artefatos Gráficos:** Responsável pela geração em alta fidelidade (`dpi=300`) dos boxplots, gráficos de dispersão e fronteiras de decisão (ex: `fig_07_pareto_front.png`, `fig_06_tradeoff_sensitivity_far.png`) essenciais para relatórios acadêmicos.

---

## 4. Métricas de Desempenho

O pipeline calcula duas categorias distintas de métricas:

**Métricas em Nível de Janela (_Window-Level_):** Úteis para debugar a convergência matemática do modelo.

- _Acurácia_, _Precision_, _Recall_, _F1-Score_, _AUC-PR_ (Área sob a curva de Precisão-Recall).

**Métricas em Nível de Evento Clínico (_Event-Level_):** O real balizador da utilidade do projeto.

- **Event Sensitivity:** Porcentagem de crises epilépticas registradas no EEG que o algoritmo sinalizou com sucesso.
- **Event FAR (_False Alarm Rate per hour_):** Quantidade de alertas errôneos disparados por cada hora ininterrupta de monitoramento. Valores altos causam rejeição clínica do sistema.

---

## 5. Estrutura de Diretórios

O repositório está organizado para separar explicitamente dados brutos, artefatos gerados e códigos operacionais:

```text
├── data/
│   ├── raw_edf/                 # Arquivos originais intactos do OpenNeuro (S3)
│   ├── processed/               # Sinais após aplicação de filtros e limpeza
│   ├── features/                # Matrizes .npy / .csv de atributos extraídos (FS-A/B/C)
│   └── pipeline_meta.json       # Manifesto JSON que guia a interoperabilidade
├── notebooks/
│   ├── notebook1_preprocessing_features.ipynb
│   ├── notebook2_training.ipynb
│   └── notebook3_analysis.ipynb
├── results/
│   ├── csv/                     # Logs detalhados (results_baseline.csv, results_v2.csv)
│   └── figures/                 # Repositório de gráficos salvos pelo notebook 3
├── requirements.txt             # Árvore de dependências do Python
└── README.md                    # Documentação do projeto
```

## 6. Guia de Instalação e Execução

Este projeto foi validado em ambiente Python 3.9+.

**Passo 1: Clonar e isolar o ambiente**

Recomenda-se veementemente a não instalação dos pacotes de neurociência na sua máquina global devido a potenciais conflitos de dependência do scipy e mne.

```bash
git clone <URL_DO_REPOSITORIO>
cd <NOME_DA_PASTA>

# Criação do ambiente virtual (VENV)
python -m venv venv

# Ativação (Windows)
venv\Scripts\activate

# Ativação (Linux/macOS)
source venv/bin/activate
```

**Passo 2: Instalação de Bibliotecas Científicas**

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

**Passo 3: Ordem de Invocação**

Abra o Jupyter (`jupyter notebook`) e execute estritamente na ordem numérica:

1. Execute o **Notebook 1** na íntegra. ⚠️ O download do S3 via Boto3 e o processamento de Wavelets podem levar de dezenas de minutos a horas dependendo do seu hardware e banda de internet.
2. Verifique a criação do `data/pipeline_meta.json`.
3. Execute o **Notebook 2** (Treinamento LOSO).
4. Por fim, execute o **Notebook 3** para processar as saídas CSV e gerar os gráficos finais em `results/figures/`.

---

## 7. Tecnologias Utilizadas

A base algorítmica deste projeto se sustenta nos seguintes pilares open-source:

- **Core & Vetorização:** `numpy`, `pandas`, `scipy`
- **Neurociência & Sinais:** `mne` (MNE-Python), `PyWavelets`
- **Machine Learning:** `scikit-learn` (SVM, Random Forest, Metrics), `xgboost` (Gradient Boosting)
- **Visualização Analítica:** `matplotlib`, `seaborn`
- **Integração AWS:** `boto3`
- **Utilidades:** `tqdm` (Barras de progresso essenciais para longos processamentos), `json`, `os`, `pathlib`

---

## 8. Contexto Acadêmico

Este pipeline computacional foi integralmente arquitetado e desenvolvido por **Danilo Pedro da Silva Valério**, aluno de graduação do curso de Bacharelado em Ciência da Computação da Universidade Estadual da Paraíba (UEPB), Campus Campina Grande.

O desenvolvimento consolida os esforços de pesquisa vinculados ao **Programa Institucional de Bolsas de Iniciação Científica (PIBIC)**, sob a orientação especializada da Professora **Sabrina de Figueiredo Souto**. A pesquisa investiga diretamente os impactos, o mapeamento e a viabilidade da aplicação moderna de Inteligência Artificial e Aprendizado de Máquina no nicho complexo da Neurociência Computacional.
