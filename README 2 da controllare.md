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

## Risultati — task **binario** (normal vs. attacco)

Confronto sulle **top-10 feature** selezionate per mutual information,
stesso split 80/20 seed=42. Ogni modello usa il preprocessing ottimale
per la sua famiglia (i tree sono invarianti alle trasformazioni monotone).

| Modello | F1 (binary) | Accuracy | Note |
|---|---|---|---|
| LightGBM | **1.0000** | 1.0000 | gradient boosting |
| XGBoost | **1.0000** | 1.0000 | gradient boosting |
| Random Forest | **1.0000** | 0.9999 | ensemble |
| Decision Tree | **1.0000** | 0.9999 | albero (depth=10) |
| **KAN Chebyshev** | **0.9690** | 0.9690 | **questo lavoro, 10 edge LUT** |
| Logistic Regression | 0.9630 | 0.9323 | lineare |

La KAN supera la regressione logistica e si avvicina ai tree ensemble,
ma la natura del task binario (fortemente sbilanciato) lascia poco spazio
alla differenziazione. La KAN è l'unico modello deployabile su MCU
con latenza competitiva (38 µs su ESP32-C3).

### Latenza on-device — binario (Wokwi, 40 vettori, acc. 97.5%)

| Board | float | intero | fully-integer | speedup |
|---|---|---|---|---|
| Arduino Mega 2560 | 2851 µs | 680 µs | 357 µs | 8.0× |
| ESP32-C3 | 1665 µs | 207 µs | **38 µs** | 44× |

---

## Risultati — task **multiclass** (10 classi)

Le 10 classi: `backdoor`, `ddos`, `dos`, `injection`, `mitm`, `normal`,
`password`, `ransomware`, `scanning`, `xss`. Stesse top-10 feature,
stesso split 80/20 seed=42, 60 000 sample totali.

### Confronto accuratezza

| Modello | Macro-F1 | Weighted-F1 | Accuracy | MITM-F1 |
|---|---|---|---|---|
| Random Forest | **0.9504** | 0.9859 | 0.9858 | 0.6269 |
| LightGBM | **0.9504** | 0.9867 | 0.9868 | 0.6167 |
| XGBoost | 0.9486 | 0.9858 | 0.9857 | 0.6094 |
| Decision Tree | 0.9317 | 0.9754 | 0.9750 | 0.5373 |
| **KAN multi-layer** | **0.9118** | 0.9663 | 0.9623 | — |
| Logistic Regression | 0.7907 | 0.8613 | 0.8658 | 0.1739 |

> `mitm` è strutturalmente sbilanciato (1 043 sample su 211 043 = 0.5%).
> Tutti i modelli ne soffrono — il problema è nel dataset, non nei modelli.

### Confronto ingombro memoria — deployment su MCU

Questa è la dimensione **effettiva del modello** da caricare sul dispositivo,
non la RAM usata in Python per addestrarlo.

| Modello | Macro-F1 | Flash / RAM | Deployabile su ESP32-C3? |
|---|---|---|---|
| Random Forest | 0.9504 | ~24 MB (pickle) | ✗ supera 4 MB flash |
| LightGBM | 0.9504 | ~6.3 MB (testo) | ✗ supera 4 MB flash |
| XGBoost | 0.9486 | ~4.4 MB (JSON) | ✗ supera 4 MB flash |
| Decision Tree | 0.9317 | ~58 KB (405 nodi) | ⚠ teoricamente sì, ma accesso random |
| **KAN multi-layer (e2e)** | **0.9118** | **~410 KB** | ✓ **verificato su Wokwi** |
| **KAN multi-layer (fwd)** | **0.9118** | **~324 KB** | ✓ **verificato su Wokwi** |
| **KAN single-layer** | 0.858 | **~100 KB** | ✓ verificato su Wokwi |
| Logistic Regression | 0.7907 | < 1 KB | ✓ ma F1 insufficiente |

**I modelli basati su alberi (RF, LightGBM, XGBoost) non sono deployabili
su ESP32-C3** perché il loro footprint supera i 4 MB di flash disponibili.
La KAN multi-layer è il punto di compromesso ottimale: Macro-F1=0.91,
324–410 KB, accesso O(1) alla LUT flat in flash.

### Deployment KAN multiclass su ESP32-C3

| Variante | Macro-F1 | Flash | Latenza ESP32 | Acc. on-device |
|---|---|---|---|---|
| KAN single-layer | 0.858 | 100 KB | 118 µs | 90% (36/40) |
| KAN multi-layer — forward only | 0.9118 | 324 KB | 691 µs | 95% (38/40) |
| KAN multi-layer — **end-to-end** (preprocessing on-chip) | 0.9118 | 410 KB | 6149 µs | 95% (38/40) |

### Valutazione su dataset completo (211 043 sample)

Il modello multi-layer valutato su **tutti i 211 043 sample** del CSV originale
(addestrato su 48 000, preprocessing fittato solo su training, `random_state=42`):

| Metrica | 12k test set | **211k full dataset** |
|---|---|---|
| Accuracy | 0.9623 | **0.9644** (203 533/211 043) |
| Macro-F1 | 0.9118 | **0.9177** |
| Weighted-F1 | — | **0.9663** |

**Risultati per classe (211k):**

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

Macro-F1 escludendo mitm: **0.9652**. Confusioni principali: xss↔injection (~6%).

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
│   ├── compare_models.py            confronto binario
│   ├── compare_models_multiclass.py confronto multiclass
│   ├── unified_comparison.py        confronto leale su stessa base feature
│   ├── feature_curve.py             studio accuratezza vs numero feature
│   ├── basis_comparison.py          Chebyshev vs B-spline
│   ├── preproc_x_model.py           effetto preprocessing x modello
│   ├── export_lut.py / export_lut_int.py          export LUT binario
│   ├── export_lut_int_multiclass.py               export LUT multiclass single-layer
│   ├── export_ml_int.py / ml_stage*.py            export multi-layer
│   └── passo5_eval.py               pipeline end-to-end riproducibile
├── preprocessing/
│   └── section_310_unified_feature_engineering.py
├── mcu/
│   ├── main_kan_wokwi*.cpp          firmware binario (float/int/fully-int)
│   ├── main_kan_mc_wokwi.cpp        firmware multiclass single-layer
│   ├── main_kan_ml_wokwi.cpp        firmware multiclass multi-layer (fwd only)
│   ├── kan_ids_*.h / kan_ml_*.h     header LUT (generati)
│   ├── test_vectors*.h              vettori di test (generati)
│   └── WOKWI_GUIDE*.md             guide Wokwi
├── mcu_e2e/
│   ├── main_kan_e2e_wokwi.cpp       firmware ESP32-C3 end-to-end
│   ├── main_harness_12k.cpp         harness host C++ (12k sample)
│   ├── kan_ml_prep.h                knot QT (10×1000 double) + PREP_REFS
│   ├── test_e2e_12k.bin             12k sample binari
│   ├── test_vectors_e2e.h           40 sanity vector in feature grezze
│   └── WOKWI_E2E_GUIDE.md          guida Wokwi end-to-end
├── results/
│   └── full_dataset_eval.md         valutazione su 211k sample
└── data/
    └── README.md                    istruzioni download TON_IoT
```

---

## Setup

```bash
git clone https://github.com/KuznetsovKarazin/lut-kan.git
pip install -r requirements.txt
```

Dataset TON_IoT nella root del repo (vedi `data/README.md`).

## Uso

```bash
# Confronto modelli binario
python scripts/compare_models.py --csv train_test_network.csv

# Confronto multiclass (con ingombro memoria)
python scripts/unified_comparison.py --csv train_test_network.csv --task multiclass

# Export LUT multi-layer
python scripts/export_ml_int.py

# Pipeline end-to-end (preprocessing on-chip)
python scripts/passo5_eval.py
cd mcu_e2e && g++ -O2 -o harness_12k main_harness_12k.cpp -lm -I.. -I../mcu
./harness_12k test_e2e_12k.bin
# Atteso: Macro-F1: 0.9118  Accuracy: 0.9623  (12k sample)
```

---

## Stato e lavoro futuro

**Completato:**
- KAN binario single-layer: F1=0.969, **38 µs** su ESP32-C3
- KAN multiclass single-layer: Macro-F1=0.858, 100 KB flash, 118 µs
- KAN multiclass multi-layer — forward only: Macro-F1=0.9118, 324 KB, 691 µs
- KAN multiclass multi-layer — **end-to-end**: preprocessing on-chip verificato
  - Macro-F1=0.9118 (12k sample), 0.9177 (211k full dataset)
  - Accuracy=0.9644 su 211k sample
  - Firmware ESP32-C3 Wokwi: **95.0%** (38/40), latenza media **6149 µs**
- Confronto ingombro memoria su tutti i modelli: RF/LGB/XGB non deployabili
  su ESP32-C3 (> 4 MB flash), KAN è l'unico punto di compromesso valido

**Note tecniche:**
- Bug risolto: 1 ULP tra `log1p` glibc e knot numpy nella binary search QT.
  Fix: `INTERP_EPS = 1e-14` nella comparazione.
- Latenza e2e (6149 µs) dominata dal preprocessing in `double`; il solo
  forward vale 691 µs. Ottimizzazione futura: preprocessing in fixed-point.

**Prossimi passi:** flash su hardware fisico; preprocessing in fixed-point
per ridurre latenza e2e; variante B-spline per footprint LUT ridotto.

---

## Crediti e licenza

LUT quantizzazione: [`lut-kan`](https://github.com/KuznetsovKarazin/lut-kan) (O. Kuznetsov).
Pipeline IDS: [`iot-audit`](https://github.com/emanuelepiodebernardis/iot-audit).
Dataset: TON_IoT, Moustafa et al., UNSW Canberra (CC BY 4.0). Licenza: MIT.
