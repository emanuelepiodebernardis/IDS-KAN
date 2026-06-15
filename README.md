# KAN-IDS: Kolmogorov–Arnold Networks for Embedded Intrusion Detection

Estensione del lavoro [`iot-audit`](https://github.com/emanuelepiodebernardis/iot-audit) verso classificatori **KAN (Kolmogorov–Arnold Networks)** per intrusion detection su microcontrollori.

Il lavoro precedente ha dimostrato una pipeline ML-IDS completa per TON_IoT, dal preprocessing al deployment fisico su Arduino Mega 2560 ed ESP32-C3, con tre canali di quantizzazione (m2cgen, TFLite Micro INT8, formato INT8 custom per tree ensemble). Questo repository aggiunge un **quarto canale**: una KAN a base Chebyshev quantizzata in look-up table (LUT), riusando l'infrastruttura di [`lut-kan`](https://github.com/KuznetsovKarazin/lut-kan).

---

## Contributo

La famiglia di modelli KAN era assente dalla tassonomia dei classificatori deployabili su MCU per IDS. Questo lavoro la introduce e ne misura il trade-off accuratezza / footprint / latenza sullo stesso spazio di feature e sulle stesse board del lavoro precedente.

Il punto chiave **non** è battere i gradient-boosted tree sull'accuratezza (non li batte: su dati tabulari restano davanti). È dimostrare che una KAN classificatore può essere quantizzata in LUT e deployata su microcontrollore con una catena verificata end-to-end.

---

## Risultati — Task binario (TON_IoT, 10 feature)

Confronto sullo **spazio unificato a 10 feature** (deployabile su MCU). I cinque modelli sono quelli del lavoro precedente, ristretti a questo spazio per un confronto equo. Per riferimento, sullo spazio completo a 95 feature LightGBM raggiunge F1 = 0.9992.

| Modello             | F1        | ROC-AUC   | Note                       |
|---------------------|-----------|-----------|----------------------------|
| LightGBM            | 0.984     | 0.987     | gradient boosting          |
| Random Forest       | 0.984     | 0.987     | gradient boosting          |
| XGBoost             | 0.984     | 0.987     | gradient boosting          |
| **KAN Chebyshev**   | **0.969** | **0.969** | **questo lavoro, 10 edge** |
| Logistic Regression | 0.963     | 0.940     | lineare                    |
| Decision Tree (d=5) | 0.963     | 0.908     | albero                     |

La KAN single-layer supera i baseline lineari e ad albero, resta sotto gli ensemble. Recall ≈ 0.98 (pochi attacchi mancati, errore desiderabile in un IDS).

### Deployment binario

La KAN addestrata viene quantizzata in LUT (uint8, segment-wise) riusando `build_lut_for_edges` di `lut-kan`. La decisione binaria ricostruita dalle sole LUT coincide con quella in virgola mobile al **99.95%** sul test set.

| Metrica                              | Valore                        |
|--------------------------------------|-------------------------------|
| Edge da quantizzare                  | 10 (single-layer)             |
| Memoria LUT totale                   | ≈ 5.4 KB (L = 64)             |
| Coincidenza decisioni quant vs float | 99.95%                        |
| Target                               | Arduino Mega 2560, ESP32-C3   |

I 5.4 KB entrano negli 8 KB di SRAM dell'Arduino Mega.

---

## Risultati — Task multiclass (TON_IoT, 10 classi, 10 feature)

Estensione al task **10-class** (backdoor, ddos, dos, injection, mitm, normal, password, ransomware, scanning, xss). Tutti i modelli sono stati addestrati, quantizzati e testati fisicamente su **ESP32-C3** via Wokwi.

I risultati confermano che tutti i tree-ensemble multiclass quantizzati sono fisicamente deployabili su ESP32-C3:

| Modello                        | Flash      | Macro-F1 | Latenza media | Variabilità |
|--------------------------------|------------|----------|---------------|-------------|
| Decision Tree (max_depth=15)   | ~10 KB     | 0.9438   | ~30 µs        | 3.75×       |
| Random Forest (100 alberi, d=10) | 333 KB   | 0.9446   | 2.594 ms      | 2.05×       |
| LightGBM (20 alberi/classe)    | 202 KB     | 0.9480   | 3.763 ms      | 1.76×       |
| XGBoost                        | 369 KB     | 0.9486   | 8.24 ms       | ampia       |
| **KAN multi-layer int-only**   | **346 KB** | **0.9044** | **~685 µs** *(HW reale stimata)* | **~1.02×** |

**Note sul deployment KAN multiclass:**

- La KAN integer-only v4 usa preprocessing `QuantileTransformer` O(1) via LUT (43 KB RAM, copiata da flash a `setup()`), forward pass interamente in `int16` senza operazioni float durante l'inferenza
- La latenza su Wokwi è ~2440 µs a causa della simulazione software di `logf()`; su hardware reale ESP32-C3 con FPU hardware la stima è **~685 µs**
- La variabilità di latenza è quasi costante (~1.02×) grazie alla struttura LUT a segmenti fissi, vantaggio chiave rispetto ai tree ensemble
- Flash totale: 346 KB / 3584 KB disponibili su ESP32-C3

### Feature (top-10 Mutual Information)

`src_ip_bytes`, `dst_port`, `dst_ip_bytes`, `src_port`, `duration`, `src_bytes`, `dst_bytes`, `dst_pkts`, `src_pkts`, `dns_qtype`

### Classi multiclass

| ID | Classe    |
|----|-----------|
| 0  | backdoor  |
| 1  | ddos      |
| 2  | dos       |
| 3  | injection |
| 4  | mitm      |
| 5  | normal    |
| 6  | password  |
| 7  | ransomware|
| 8  | scanning  |
| 9  | xss       |

---

## Struttura del repository

```
IDS-KAN/
├── README.md
├── requirements.txt
├── utils.py
├── .gitignore
│
├── src/
│   ├── kan_chebyshev.py              # KAN a base Chebyshev (training BCE, NumPy)
│   ├── fixed_point_quantile.py       # QuantileTransformer fixed-point Q16.16
│   ├── quantization_export.py        # pipeline quantizzazione LR, DT, MLP, XGB, LGB INT8
│   └── embedded_model_io.py          # serializzazione/deserializzazione modelli INT8
│
├── scripts/
│   ├── compare_models.py             # confronto: 5 modelli + KAN (95 e 10 feature)
│   ├── export_lut.py                 # export LUT binario + verifica + header C
│   └── export_lut_fp.py              # export LUT con preprocessing fixed-point integrato
│
├── preprocessing/
│   └── section_310_unified_feature_engineering.py   # (da iot-audit)
│
├── mcu/
│   ├── main_kan.cpp                  # firmware KAN binary (LUT uint8, single-layer)
│   │
│   ├── kan_ml_int_v4/                # KAN multiclass integer-only v4 (ESP32-C3)
│   │   ├── main.cpp                  # preprocessing RAM + forward int16 + benchmark
│   │   ├── qt_int_v4_lut.h           # LUT preprocessing O(1): KLO, FRAC, PPF (109 KB)
│   │   ├── kan_ml_layer1_v4.h        # LUT layer1 int16, 160 edge (322 KB)
│   │   ├── kan_ml_layer2_v4.h        # LUT layer2 int16, 160 edge (341 KB)
│   │   ├── kan_ml_tanh_v4.h          # LUT tanh int16 + costanti (13 KB)
│   │   ├── test_vectors_int_v4.h     # 40 vettori di test (feature grezze float32)
│   │   └── README.md                 # documentazione deployment KAN v4
│   │
│   ├── dt_mc/                        # Decision Tree multiclass (ESP32-C3)
│   │   ├── main_dt_wokwi.cpp         # firmware DT + benchmark
│   │   ├── dt_mc_nodes.h             # nodi DT flat array (~10 KB)
│   │   └── dt_test_vectors.h         # 40 vettori di test
│   │
│   ├── rf_mc/                        # Random Forest multiclass (ESP32-C3)
│   │   ├── main_rf_wokwi.cpp         # firmware RF + benchmark
│   │   ├── rf_mc_nodes.h             # nodi RF flat array (~333 KB)
│   │   └── dt_test_vectors.h         # 40 vettori di test (condivisi)
│   │
│   └── lgb_mc/                       # LightGBM multiclass (ESP32-C3)
│       ├── main_lgb_wokwi.cpp        # firmware LGB + benchmark
│       ├── lgb_mc_nodes_flat.h       # nodi LGB flat array (~202 KB)
│       └── dt_test_vectors.h         # 40 vettori di test (condivisi)
│
├── results/
│   ├── comparison_results_example.csv
│   └── kan_ids_layer_example.h
│
└── data/
    └── README.md                     # istruzioni per scaricare TON_IoT
```

---

## Setup

Servono Python 3.10+ e il repository `lut-kan` clonato nella root:

```bash
git clone https://github.com/KuznetsovKarazin/lut-kan.git
pip install -r requirements.txt
```

Servono inoltre, nella root del repo, due file dal lavoro precedente: `utils.py` (modelli, preprocessing, metriche) e il dataset TON_IoT (vedi `data/README.md`). Il file `preprocessing/section_310_...py` è incluso.

---

## Uso

**Confronto dei modelli** (riproduce i risultati del lavoro precedente sulle 95 feature e colloca la KAN sulle 10 feature):

```bash
python scripts/compare_models.py --csv train_test_network.csv
# test rapido: aggiungi --sample 40000
```

**Export LUT binario** e verifica della catena di deployment:

```bash
python scripts/export_lut.py --csv train_test_network.csv
# genera kan_ids_layer.h e verifica la coincidenza delle decisioni
```

**Export LUT con preprocessing fixed-point** (per la pipeline KAN multiclass):

```bash
python scripts/export_lut_fp.py --csv train_test_network.csv
```

**Deployment su ESP32-C3 (Wokwi o hardware fisico):**

Per ciascun modello nella cartella `mcu/`, copia i file `.h` e il corrispondente `.cpp` nella stessa cartella del progetto PlatformIO o Arduino IDE. Per la KAN:

1. Rinomina `main.cpp` → `sketch.ino` (o usa PlatformIO)
2. Carica tutti i file della cartella `kan_ml_int_v4/` nel progetto
3. Compila e flasha su ESP32-C3
4. Per classificare un pacchetto reale, sostituisci `TEST_RAW_V4[i]` con il tuo array di 10 feature `float32`

---

## Stato e lavoro futuro

**Completato:**
- KAN single-layer (task binario), catena KAN → LUT → header C verificata
- KAN multi-layer integer-only end-to-end (task multiclass 10 classi), deployment ESP32-C3
- Tree ensemble multiclass (DT, RF, LGB, XGB) quantizzati e deployati su ESP32-C3
- Benchmark latenza e footprint su Wokwi per tutti i modelli

**Prossimi passi:**
- Flash su hardware fisico e misura di latenza/SRAM on-board
- KAN multi-layer addestrata in PyTorch/PyKAN per recuperare parte del gap con gli ensemble
- Variante a B-spline
- Supporto Arduino Mega 2560 per i modelli multiclass più leggeri (DT)

---

## Crediti e licenza

Infrastruttura di quantizzazione LUT: [`lut-kan`](https://github.com/KuznetsovKarazin/lut-kan) di O. Kuznetsov.
Pipeline IDS, preprocessing e modelli di riferimento: [`iot-audit`](https://github.com/emanuelepiodebernardis/iot-audit).
Dataset TON_IoT: Moustafa et al., UNSW Canberra (CC BY 4.0).

Licenza: MIT.
