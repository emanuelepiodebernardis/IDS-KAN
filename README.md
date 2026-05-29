# KAN-IDS: Kolmogorov–Arnold Networks for Embedded Intrusion Detection

Estensione del lavoro [`iot-audit`](https://github.com/emanuelepiodebernardis/iot-audit)
verso classificatori **KAN (Kolmogorov–Arnold Networks)** per intrusion
detection su microcontrollori.

Il lavoro precedente ha dimostrato una pipeline ML-IDS completa per TON_IoT,
dal preprocessing al deployment fisico su Arduino Mega 2560 ed ESP32-C3,
con tre canali di quantizzazione (m2cgen, TFLite Micro INT8, formato INT8
custom per tree ensemble). Questo repository aggiunge un **quarto canale**:
una KAN a base Chebyshev quantizzata in look-up table (LUT), riusando
l'infrastruttura di [`lut-kan`](https://github.com/KuznetsovKarazin/lut-kan).

## Contributo

La famiglia di modelli KAN era assente dalla tassonomia dei classificatori
deployabili su MCU per IDS. Questo lavoro la introduce e ne misura il
trade-off accuratezza / footprint / latenza sullo stesso spazio di feature
e sulle stesse board del lavoro precedente.

Il punto chiave **non** è battere i gradient-boosted tree sull'accuratezza
(non li batte: su dati tabulari restano davanti). È dimostrare che una KAN
classificatore può essere quantizzata in LUT e deployata su microcontrollore
con una catena verificata end-to-end, cosa che `lut-kan` da solo non copre
perché lavora su regressione, non classificazione.

---

## Risultati (TON_IoT, task binario)

Confronto sullo **spazio unificato a 10 feature** (deployabile su MCU). I
cinque modelli sono quelli del lavoro precedente, ristretti a questo spazio
per un confronto equo. Per riferimento, sullo spazio completo a 95 feature
LightGBM raggiunge F1 = 0.9992.

| Modello | F1 | ROC-AUC | Note |
|---|---|---|---|
| LightGBM | 0.984 | 0.987 | gradient boosting |
| Random Forest | 0.984 | 0.987 | ensemble |
| XGBoost | 0.984 | 0.987 | gradient boosting |
| **KAN Chebyshev** | **0.969** | **0.969** | **questo lavoro, 10 edge** |
| Logistic Regression | 0.963 | 0.940 | lineare |
| Decision Tree (d=5) | 0.963 | 0.908 | albero |

La KAN single-layer supera i baseline lineari e ad albero, resta sotto gli
ensemble. Recall ≈ 0.98 (pochi attacchi mancati, errore desiderabile in un IDS).

### Deployment binario

| Metrica | Valore |
|---|---|
| Edge da quantizzare | 10 (single-layer) |
| Memoria LUT totale | ≈ 5.4 KB (L = 64) |
| Coincidenza decisioni quant vs float | 99.95% |
| Target | Arduino Mega 2560, ESP32-C3 |

I 5.4 KB entrano negli 8 KB di SRAM dell'Arduino Mega.

### Latenza on-device — task binario

Latenza media misurata in simulazione su Wokwi (40 vettori, accuratezza 97.5%):

| Board | float | intero | fully-integer | speedup |
|---|---|---|---|---|
| Arduino Mega 2560 | 2851 µs | 680 µs | 357 µs | 8.0× |
| ESP32-C3 | 1665 µs | 207 µs | 38 µs | 44× |

---

## Risultati (TON_IoT, task multiclass — 10 classi)

Il task multiclass distingue 10 categorie: `normal` + 9 tipi di attacco
(`backdoor`, `ddos`, `dos`, `injection`, `mitm`, `password`, `ransomware`,
`scanning`, `xss`). Le 10 feature selezionate per mutual information sono il
punto ottimale (oltre, l'accuratezza non sale).

### Confronto modelli multiclass

| Modello | Macro-F1 |
|---|---|
| Random Forest | 0.968 |
| LightGBM / XGBoost | 0.965 |
| **KAN multi-layer** | **0.922** |
| KAN single-layer | 0.858 |
| Decision Tree | 0.793 |
| Logistic Regression | 0.214 |

### Deployment multiclass su ESP32-C3

| Variante | Macro-F1 | Edge | LUT | Latenza ESP32 | Acc. on-device |
|---|---|---|---|---|---|
| KAN single-layer | 0.86 | 100 | 100 KB | 118 µs | 90% (36/40) |
| KAN multi-layer — forward only | 0.92 | 320 | 320 KB | 691 µs | 95% (38/40) |
| KAN multi-layer — **end-to-end** (preprocessing on-chip) | 0.92 | 320 | 320+148 KB | 6149 µs | 95% (38/40) |

### Valutazione su dataset completo (211 043 sample)

Il modello multi-layer è stato valutato su **tutti i 211 043 sample** del
dataset TON_IoT originale (il modello è addestrato su 48 000 sample,
preprocessing fittato solo sul training, pipeline riproducibile con
`random_state=42`).

| Metrica | 12k test set | **211k full dataset** |
|---|---|---|
| Accuracy | 0.9623 | **0.9644** (203 533/211 043) |
| Macro-F1 | 0.9118 | **0.9177** |
| Weighted-F1 | — | **0.9663** |

**Risultati per classe (211k sample):**

| Classe | Support | Precision | Recall | F1 |
|---|---|---|---|---|
| backdoor | 20 000 | 1.00 | 1.00 | **1.00** |
| ransomware | 20 000 | 1.00 | 1.00 | **1.00** |
| normal | 50 000 | 0.99 | 0.98 | **0.99** |
| password | 20 000 | 1.00 | 0.98 | **0.99** |
| scanning | 20 000 | 0.98 | 0.99 | **0.99** |
| dos | 20 000 | 0.99 | 0.97 | 0.98 |
| ddos | 20 000 | 0.95 | 0.93 | 0.94 |
| xss | 20 000 | 0.91 | 0.91 | 0.91 |
| injection | 20 000 | 0.90 | 0.89 | 0.90 |
| **mitm** | **1 043** | 0.34 | 0.88 | **0.49** |

> `mitm` è fortemente sbilanciato (1 043 sample = 0.5% del dataset):
> recall accettabile (0.88) ma precision bassa (0.34) per scarsità di esempi
> nel training. Macro-F1 escludendo mitm: **0.9652**.

**Principali confusioni:** xss ↔ injection (~6%), ddos → mitm (1.7%),
dos → mitm (1.2%). Classi intrisecamente simili a livello di traffico di rete.

---

## Struttura del repository

```
kan-ids/
├── README.md
├── requirements.txt
├── utils.py                     modelli, preprocessing, metriche (da iot-audit)
├── src/
│   ├── kan_chebyshev.py             KAN Chebyshev binaria (training BCE, NumPy)
│   ├── kan_chebyshev_multiclass.py  KAN Chebyshev multiclass (softmax)
│   ├── kan_bspline.py               variante a base B-spline (confronto basi)
│   ├── kan_torch.py                 KAN multi-layer (PyTorch, autograd)
│   └── kan_multilayer_numpy.py      replica NumPy del forward multi-layer
├── scripts/
│   ├── compare_models.py            confronto binario: 5 modelli + KAN
│   ├── compare_models_multiclass.py confronto multiclass
│   ├── unified_comparison.py        confronto leale su base condivisa
│   ├── feature_curve.py             studio accuratezza vs numero feature
│   ├── basis_comparison.py          Chebyshev vs B-spline (acc + quantizzazione)
│   ├── preproc_x_model.py           effetto preprocessing x modello
│   ├── export_lut.py                export LUT binario (float)
│   ├── export_lut_int.py            export LUT binario integer-only
│   ├── export_lut_int_multiclass.py export LUT multiclass single-layer
│   ├── export_ml_int.py             export multi-layer (2 LUT + tanh) per firmware
│   ├── ml_stage2_layer1_tanh.py     quantizzazione multi-layer: layer1 + tanh
│   ├── ml_stage3_full.py            quantizzazione multi-layer: forward completo
│   └── passo5_eval.py               pipeline end-to-end riproducibile (Passo 5)
├── preprocessing/
│   └── section_310_unified_feature_engineering.py   (da iot-audit)
├── mcu/
│   ├── main_kan.cpp                 firmware base (legge l'header C)
│   ├── main_kan_wokwi*.cpp          firmware binario (float / int / fully-int)
│   ├── main_kan_mc_wokwi.cpp        firmware multiclass single-layer
│   ├── main_kan_ml_wokwi.cpp        firmware multiclass multi-layer (forward only)
│   ├── kan_ids_*.h                  header LUT binario (generati)
│   ├── kan_ml_*.h                   header LUT multi-layer (generati)
│   ├── test_vectors*.h              vettori di test (generati)
│   └── WOKWI_GUIDE*.md              guide alla simulazione Wokwi
├── mcu_e2e/
│   ├── main_kan_e2e_wokwi.cpp       firmware ESP32-C3 end-to-end (preprocessing on-chip)
│   ├── main_harness_12k.cpp         harness host C++ (valutazione su 12k sample)
│   ├── kan_ml_prep.h                knot QT (10×1000 double) + PREP_REFS
│   ├── test_e2e_12k.bin             12k sample binari (feature grezze + label)
│   ├── test_vectors_e2e.h           40 sanity vector in feature grezze
│   └── WOKWI_E2E_GUIDE.md           guida Wokwi end-to-end
├── results/
│   └── full_dataset_eval.md         valutazione su 211k sample (report completo)
└── data/
    └── README.md                    istruzioni per scaricare TON_IoT
```

---

## Setup

Servono Python 3.10+ e il repository `lut-kan` clonato nella root:

```bash
git clone https://github.com/KuznetsovKarazin/lut-kan.git
pip install -r requirements.txt
```

Serve inoltre il dataset TON_IoT nella root del repo
(vedi `data/README.md` per le istruzioni di download).

---

## Uso

**Confronto modelli binario:**
```bash
python scripts/compare_models.py --csv train_test_network.csv
# test rapido: aggiungi --sample 40000
```

**Export LUT e verifica deployment:**
```bash
python scripts/export_lut.py --csv train_test_network.csv
```

**Multiclass — export integer:**
```bash
# single-layer
python scripts/export_lut_int_multiclass.py --csv train_test_network.csv

# multi-layer (genera kan_ml_layer1.h, kan_ml_layer2.h, kan_ml_tanh.h)
python scripts/export_ml_int.py
```

**Pipeline end-to-end con preprocessing on-chip:**
```bash
python scripts/passo5_eval.py
cd mcu_e2e && g++ -O2 -o harness_12k main_harness_12k.cpp -lm -I.. -I../mcu
./harness_12k test_e2e_12k.bin
# Atteso: Macro-F1: 0.9118  Accuracy: 0.9623  (12k sample)
```

---

## Stato e lavoro futuro

**Completato:**
- KAN binario single-layer: F1=0.969, 38 µs su ESP32-C3, verificato Python→C
- KAN multiclass single-layer: Macro-F1=0.86, 118 µs su ESP32-C3
- KAN multiclass multi-layer — forward only: Macro-F1=0.92, 691 µs su ESP32-C3
- KAN multiclass multi-layer — **end-to-end con preprocessing on-chip**:
  - Macro-F1=0.9118 su 12k sample (harness host C++)
  - Macro-F1=0.9177 su 211k sample (full dataset, riproducibile con seed=42)
  - Accuracy=0.9644 su 211k sample
  - Firmware ESP32-C3 verificato su Wokwi: 95.0% (38/40), latenza media 6149 µs

**Note tecniche rilevanti:**
- Bug risolto: differenza di 1 ULP tra `log1p` di glibc e knot generati da numpy
  nella binary search del QuantileTransformer bidirezionale. Fix: tolleranza
  `INTERP_EPS = 1e-14` nel confronto della binary search.
- La latenza e2e (6149 µs) è dominata dal preprocessing in `double` (QT
  bidirezionale + Acklam norm.ppf); il solo forward vale 691 µs.

**Prossimi passi:** flash su hardware fisico reale; ottimizzazione del
preprocessing in fixed-point per ridurre la latenza e2e; variante B-spline
per ridurre il footprint LUT.

---

## Crediti e licenza

Infrastruttura di quantizzazione LUT: [`lut-kan`](https://github.com/KuznetsovKarazin/lut-kan)
di O. Kuznetsov. Pipeline IDS, preprocessing e modelli di riferimento:
[`iot-audit`](https://github.com/emanuelepiodebernardis/iot-audit).
Dataset TON_IoT: Moustafa et al., UNSW Canberra (CC BY 4.0).

Licenza: MIT.
