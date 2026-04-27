# HeisenBank — Quantum Fraud Detection

## Panoramica

HeisenBank è un progetto di **Quantum Machine Learning (QML)** per la rilevazione di transazioni fraudolente, sviluppato nell'ambito del corso di Tecnologie Quantistiche per la Sicurezza (TQS).

Il sistema utilizza un **Variational Quantum Classifier (VQC)**, un classificatore ibrido quantistico-classico che sfrutta circuiti quantistici parametrici per classificare le transazioni come *legittime* o *fraudolente*.

---

## Architettura del Pipeline QML

Il pipeline è articolato in 4 fasi principali:

### 1. Preprocessing e Feature Selection

- **Dataset**: `Base.csv` — dataset di transazioni bancarie con etichetta binaria `fraud_bool`
- **Bilanciamento**: undersampling della classe maggioritaria (transazioni legittime) per ottenere un dataset bilanciato 50/50
- **Encoding categorico**: `LabelEncoder` sulle variabili categoriche (`payment_type`, `employment_status`, `housing_status`, `source`, `device_os`)
- **Feature selection**: Random Forest Classifier con 100 estimator per calcolare la *feature importance*, selezionando le **top 4 feature** più rilevanti:
  - `housing_status`
  - `current_address_months_count`
  - `device_os`
  - `credit_risk_score`
- **Scaling**: `MinMaxScaler` nell'intervallo `[0, π]`, necessario per l'encoding dei dati nei gate di rotazione del circuito quantistico

### 2. Circuito Quantistico

Il circuito VQC è composto da due blocchi:

#### Feature Map — `ZZFeatureMap`
Mappa i dati classici nello spazio quantistico (Hilbert space) tramite:
- Gate **Hadamard** per la superposizione iniziale
- Gate **Phase** per l'encoding delle singole feature
- Gate **CNOT + Phase** per catturare le correlazioni tra coppie di feature adiacenti (entanglement lineare)
- **4 qubit**, 1 ripetizione

#### Ansatz — `EfficientSU2`
Ansatz variazionale predefinito dalla libreria Qiskit:
- **Rotazioni `Ry(θ)` e `Rz(θ)`** parametriche su ogni qubit per ogni layer
- **Entanglement lineare** via gate `CX`: ogni qubit è connesso al successivo (0→1→2→3)
- **1 ripetizione** (`reps=1`): 2 layer di rotazioni separati da 1 layer di entanglement
- **16 parametri** totali ottimizzabili (4 qubit × 2 gate × 2 layer)

Questa architettura offre maggiore espressività rispetto ai circuiti predefiniti come `EfficientSU2`, mantenendo lo stesso numero di parametri ma distribuendoli su più layer con una connettività più ricca.

### 3. Addestramento

- **Ottimizzatore**: `SPSA` (Simultaneous Perturbation Stochastic Approximation) — ottimizzatore stocastico gradient-free, robusto al rumore quantistico
  - 400 iterazioni
  - Callback per il plot in tempo reale della loss function
- **Sampler**: `StatevectorSampler` — simulazione esatta del circuito (senza rumore hardware)
- **Campioni di training**: 1500 (sottoinsieme del dataset bilanciato)
- **Split**: 80% training / 20% test con stratificazione

### 4. Valutazione

La valutazione del modello include:
- **Curva ROC** e calcolo dell'**AUC** (Area Under Curve)
- **Soglia ottimale** determinata tramite l'indice di Youden (massimizzazione di `TPR - FPR`)
- **Matrice di confusione** con soglia ottimizzata
- **Classification report** con precision, recall e F1-score per entrambe le classi

---

## Stack Tecnologico

| Componente | Tecnologia |
|---|---|
| Framework QML | [Qiskit](https://qiskit.org/) + [Qiskit Machine Learning](https://qiskit-community.github.io/qiskit-machine-learning/) |
| Simulatore | `StatevectorSampler` (Qiskit Primitives) |
| Ottimizzatore | `SPSA` (Qiskit Algorithms) |
| ML Classico | scikit-learn |
| Linguaggio | Python 3.12 |

## Struttura del Progetto

```
HeisenBank/
├── dataset/
│   └── Base.csv              # Dataset delle transazioni
├── qml.ipynb                 # Pipeline QML principale
├── data_exploration.ipynb    # Esplorazione e analisi dei dati
└── README.md
```

---

## Come Eseguire

1. **Installare le dipendenze**:
   ```bash
   pip install qiskit qiskit-machine-learning qiskit-algorithms scikit-learn pandas matplotlib
   ```

2. **Aprire il notebook**:
   ```bash
   jupyter notebook qml.ipynb
   ```

3. **Eseguire le celle in ordine** (1 → 4):
   - Le celle 1–2 preparano i dati e la feature map
   - La cella 3 costruisce l'ansatz custom e inizializza il VQC
   - La cella 4 avvia il training (tempo stimato: ~30–40 minuti su 1500 campioni)

> **Nota**: il training è computazionalmente intensivo. Su hardware con 16GB di RAM si consiglia di non superare i 1500 campioni di training.
