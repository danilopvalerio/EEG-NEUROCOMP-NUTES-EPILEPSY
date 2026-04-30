# Pipeline de Detecção de Crises em EEG Wearable (SeizeIT2)

## 1. Aquisição e Organização dos Dados

**Origem:** Dataset SeizeIT2, disponível no OpenNeuro, no formato BIDS com arquivos EDF e anotações TSV.

**Características do sinal:**

- EEG wearable behind-the-ear
- Taxa de amostragem: 250 Hz
- Poucos canais, mais ruidoso que EEG clínico
- Sinal não estacionário com alto ruído de movimento, EMG e contato
- Crises com duração média de aproximadamente 58 segundos

**Processamento inicial:**

- Selecionar apenas canais EEG
- Descartar canais ECG, EMG, ACC e GYR
- Alinhar o sinal com as anotações TSV
- Definir as classes: seizure e non-seizure

**Saída:** Sinal EEG bruto rotulado

---

## 2. Pré-processamento

Objetivo: reduzir ruído e preservar os componentes fisiológicos relevantes para a detecção de crises.

### 2.1 Filtro Passa-Alta

- Corte em 0,5 Hz
- Remove o drift de baixa frequência
- Etapa crítica para EEG wearable, mais suscetível a drift do que o EEG clínico

### 2.2 Filtro Notch

- Remove a interferência elétrica da rede em 50 Hz

### 2.3 Filtro Passa-Baixa

- Corte entre 40 e 45 Hz
- Remove ruído muscular (EMG) e componentes de alta frequência
- Preserva a faixa fisiológica de 0,5 a 50 Hz

### 2.4 Filtro WOSG (Wavelet-domain Optimized Savitzky-Golay)

- Refinamento adicional voltado para artefatos de movimento e oculares
- Mais eficaz que filtros convencionais em sinais não estacionários
- Dash et al. (2025) demonstraram melhora em accuracy, sensitivity e specificity ao comparar features extraídas do sinal filtrado pelo WOSG versus o sinal bruto

### 2.5 Normalização por Z-score

- Método: Z-score calculado por canal, dentro de cada registro (arquivo EDF), antes da segmentação
- O SeizeIT2 possui alta variabilidade inter-sujeito devido a diferenças de impedância de eletrodo e posicionamento do dispositivo behind-the-ear. Normalizar por canal elimina essas diferenças de amplitude entre pacientes sem apagar a diferença de amplitude entre janelas de crise e não-crise, que é uma informação clinicamente relevante e deve ser preservada para os modelos

**Saída:** Sinal EEG filtrado e normalizado

---

## 3. Segmentação

Divisão do sinal em janelas temporais para análise local.

### Parâmetros

- Tamanho da janela: 4 segundos (1000 amostras) ou 5 segundos (1250 amostras)
- Sobreposição de 50% entre janelas consecutivas

### Funcionamento da sobreposição

Cada nova janela começa na metade da anterior. Para janelas de 4 segundos:

- Janela 1: 0 a 4 s
- Janela 2: 2 a 6 s
- Janela 3: 4 a 8 s

Crises epilépticas são eventos locais no tempo. A sobreposição evita perda de informação nas bordas das janelas e aumenta a sensibilidade para detectar o início e o fim de crises.

### Janela de 4 s versus 5 s

A janela de 4 segundos captura eventos com maior sensibilidade, mas com mais ruído. A janela de 5 segundos é mais estável, porém pode diluir crises curtas. Dash et al. (2025) obtiveram maior accuracy, sensitivity e specificity com janelas de 5 segundos. Este experimento será replicado no SeizeIT2 comparando as duas configurações.

### Rotulagem

- Janela marcada como seizure (1) se houver qualquer interseção com um evento de crise nas anotações TSV
- Caso contrário, marcada como non-seizure (0)

---

## 4. Balanceamento de Classes

O SeizeIT2 apresenta forte desbalanceamento: crises de aproximadamente 58 segundos geram em torno de 12 a 14 janelas de seizure por evento, enquanto o restante do registro gera centenas de janelas non-seizure.

**Estratégia: undersampling aleatório da classe majoritária**

Selecionar aleatoriamente, dentre as janelas non-seizure, um número igual ao de janelas seizure disponíveis para cada paciente. Esta estratégia é equivalente à adotada por Dash et al. (2025) para o CHB-MIT, onde foram utilizados exatamente 2.250 frames de cada classe.

Justificativas para essa escolha:

- Simples e auditável, sem introdução de dados sintéticos
- Elimina o viés do modelo para a classe majoritária
- Preserva a distribuição real das features de crise

O undersampling é aplicado após a segmentação e antes da extração de features, dentro de cada fold do LOSO, para evitar contaminação entre treino e teste.

---

## 5. Extração de Características

Cada janela é transformada em um vetor numérico que será usado como entrada para os modelos.

### 5.1 GWST (Gaussian Window Stockwell Transform)

A GWST converte o sinal em uma representação tempo-frequência com resolução adaptativa. Suas principais vantagens sobre FFT, STFT e CWT são a preservação de fase e a adaptação da resolução conforme a frequência, o que a torna mais adequada para EEG não estacionário. Dash et al. (2025) obtiveram 90,74% de accuracy com GWST contra 70,37% com STFT, utilizando o mesmo classificador.

A faixa utilizada é de 0,5 a 50 Hz, resultando em 225 bins de frequência.

### 5.2 L1-norm

Calculada por bin de frequência na matriz tempo-frequência. Mede a energia total em cada componente de frequência. Durante crises, a energia nas baixas frequências tende a aumentar.

### 5.3 Entropia de Shannon

Calculada por bin de frequência. Mede a randomicidade e não-linearidade do sinal. Pode aumentar em atividade caótica ou diminuir em atividade rítmica durante uma crise. A combinação de L1-norm e entropia de Shannon forma o vetor principal de features com 450 dimensões (225 por feature), conforme Dash et al. (2025).

### 5.4 Features adicionais

Além das features do artigo de referência, o pipeline inclui:

- Variância por canal: mede a intensidade das flutuações; crises tendem a aumentar a variabilidade
- Energia por banda espectral (delta 0,5 a 4 Hz, theta 4 a 8 Hz, alpha 8 a 13 Hz, beta 13 a 30 Hz): potência espectral calculada por banda fisiológica
- Média por canal: offset do sinal, com baixa discriminabilidade isolada, mas uso complementar

**Saída:** Vetor de features por janela

---

## 6. Modelagem

**Entrada:** Vetores de features por janela

**Modelos utilizados:**

SVM (Support Vector Machine)

- Eficiente para separação de classes em espaço de alta dimensão
- Bom desempenho quando as classes são bem definidas

Random Forest

- Robusto a ruído
- Melhor resultado reportado em Dash et al. (2025) com hold-out: 90,74% de accuracy

XGBoost

- Geralmente superior em dados tabulares
- Ensemble de árvores com regularização, reduz overfitting

Os hiperparâmetros são selecionados via grid search com validação no conjunto de validação do LOSO.

---

## 7. Validação

**Estratégia: Leave-One-Subject-Out (LOSO)**

Em cada iteração, as janelas de um paciente são reservadas para teste e as dos demais são usadas para treino. O processo se repete para todos os pacientes.

Essa estratégia evita o vazamento de informação entre pacientes e garante que o modelo seja avaliado em sujeitos completamente novos. É obrigatória no SeizeIT2 dado o alto risco de overfitting por paciente em um dataset de 125 sujeitos com variabilidade inter-sujeito elevada. É equivalente ao LOOSICV utilizado por Dash et al. (2025) para o CHB-MIT.

---

## 8. Métricas de Avaliação

Para cada modelo, serão reportadas as seguintes métricas:

Sensitivity (Recall)

- Mede a proporção de crises corretamente identificadas
- Métrica prioritária: não detectar uma crise representa risco direto ao paciente

F1-score

- Média harmônica entre precisão e recall
- Adequada para datasets desbalanceados

Accuracy

- Proporção geral de acertos
- Menos informativa em cenários desbalanceados

Specificity

- Proporção de janelas non-seizure corretamente classificadas

---

## 9. Verificação de Overfitting

**Método:** Comparar as métricas de treino e validação ao longo dos folds do LOSO.

Sinais de overfitting:

- Treino com accuracy muito alta
- Validação com accuracy estagnada ou em queda

Análises complementares:

- Learning curve: performance em função do número de sujeitos no treino
- Avaliação do impacto do undersampling na generalização

O SeizeIT2 apresenta alto risco de overfitting por paciente devido ao número reduzido de sujeitos e à alta variabilidade inter-sujeito. O LOSO mitiga esse risco ao forçar avaliação em sujeitos nunca vistos durante o treino.

---

## 10. Experimentos

**Experimento 1: Comparação de tamanho de janela**
Comparar accuracy, F1-score e sensitivity para janelas de 4 e 5 segundos. O gráfico de métricas por tamanho de janela replica a Figura 9 de Dash et al. (2025).

**Experimento 2: Comparação de modelos**
Gráficos de barras com accuracy, F1-score e sensitivity por modelo (SVM, Random Forest, XGBoost).

**Experimento 3: Impacto do balanceamento**
Comparar resultados com e sem undersampling para avaliar se o desbalanceamento original infla artificialmente a accuracy.

**Experimento 4: Curvas de aprendizado**
Performance em função do número de sujeitos no treino, para verificar se mais dados melhorariam os modelos.

---

## 11. Saídas e Visualizações

- Gráfico de métricas por tamanho de janela (4 s vs. 5 s)
- Curvas de treino e validação por fold do LOSO
- Learning curves
- Gráficos de barras com accuracy, F1-score e sensitivity por modelo
- Matrizes de confusão por modelo

---

## 12. Fluxo Geral

OpenNeuro (dados brutos)

- Download via boto3
- Seleção de canais EEG, descarte de ECG, EMG, ACC e GYR
- Alinhamento com rótulos TSV
- Pré-processamento: Passa-alta, Notch, Passa-baixa, WOSG
- Normalização: z-score por canal, por registro
- Segmentação em janelas de 4 ou 5 segundos com sobreposição de 50%
- Rotulagem das janelas (seizure / non-seizure)
- Balanceamento: undersampling da classe majoritária
- Extração de features: GWST, L1-norm, Shannon entropy, variância, PSD por banda
- Modelagem: SVM, Random Forest, XGBoost
- Validação: LOSO
- Métricas: sensitivity, F1-score, accuracy
- Análise de overfitting e experimentos comparativos

---

## Referência

Dash, S., Dash, D.K., Tripathy, R.K., Pachori, R.B. (2025). Time–frequency domain machine learning for detection of epilepsy using wearable EEG sensor signals recorded during physical activities. _Biomedical Signal Processing and Control_, 100, 107041.
