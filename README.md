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
con una catena verificata end-to-end.

## Risultati (TON_IoT, task binario)

Confronto sullo **spazio unificato a 10 feature** (deployabile su MCU). I
cinque modelli sono quelli del lavoro precedente, ristretti a questo spazio
per un confronto equo. Per riferimento, sullo spazio completo a 95 feature
LightGBM raggiunge F1 = 0.9992.

| Modello | F1 | ROC-AUC | Note |
|---|---|---|---|
| LightGBM | 0.984 | 0.987 | gradient boosting |
| Random Forest | 0.984 | 0.987 | gradient boosting |
| XGBoost | 0.984 | 0.987 | gradient boosting |
| **KAN Chebyshev** | **0.969** | **0.969** | **questo lavoro, 10 edge** |
| Logistic Regression | 0.963 | 0.940 | lineare |
| Decision Tree (d=5) | 0.963 | 0.908 | albero |

La KAN single-layer supera i baseline lineari e ad albero, resta sotto gli
ensemble. Recall ≈ 0.98 (pochi attacchi mancati, errore desiderabile in un IDS).

### Deployment

La KAN addestrata viene quantizzata in LUT (uint8, segment-wise) riusando
`build_lut_for_edges` di `lut-kan`. La decisione binaria ricostruita dalle
sole LUT coincide con quella in virgola mobile al **99.95%** sul test set.

| Metrica | Valore |
|---|---|
| Edge da quantizzare | 10 (single-layer) |
| Memoria LUT totale | ≈ 5.4 KB (L = 64) |
| Coincidenza decisioni quant vs float | 99.95% |
| Target | Arduino Mega 2560, ESP32-C3 |

I 5.4 KB entrano negli 8 KB di SRAM dell'Arduino Mega.

## Struttura del repository

```
kan-ids/
├── README.md
├── requirements.txt
├── src/
│   └── kan_chebyshev.py        KAN a base Chebyshev (training BCE, NumPy)
├── scripts/
│   ├── compare_models.py       confronto: 5 modelli + KAN (95 e 10 feature)
│   └── export_lut.py           export LUT + verifica decisione + header C
├── preprocessing/
│   └── section_310_unified_feature_engineering.py   (da iot-audit)
├── mcu/
│   └── main_kan.cpp            firmware di base che legge l'header C
├── results/
│   ├── comparison_results_example.csv
│   └── kan_ids_layer_example.h
└── data/
    └── README.md               istruzioni per scaricare TON_IoT
```

## Setup

Servono Python 3.10+ e il repository `lut-kan` clonato nella root:

```bash
git clone https://github.com/KuznetsovKarazin/lut-kan.git
pip install -r requirements.txt
```

Servono inoltre, nella root del repo, due file dal lavoro precedente:
`utils.py` (modelli, preprocessing, metriche) e il dataset TON_IoT (vedi
`data/README.md`). Il file `preprocessing/section_310_...py` è incluso.

## Uso

Confronto dei modelli (riproduce i risultati del lavoro precedente sulle
95 feature e colloca la KAN sulle 10 feature):

```bash
python scripts/compare_models.py --csv train_test_network.csv
# test rapido: aggiungi --sample 40000
```

Export LUT e verifica della catena di deployment:

```bash
python scripts/export_lut.py --csv train_test_network.csv
# genera kan_ids_layer.h e verifica la coincidenza delle decisioni
```

## Stato e lavoro futuro

Attuale: KAN single-layer (gradiente esatto), task binario, catena
KAN → LUT → header C verificata in simulazione.

Prossimi passi: flash su hardware fisico e misura di latenza/SRAM on-board
(stile lavoro precedente); KAN multi-layer addestrata in PyTorch/PyKAN per
recuperare parte del gap con gli ensemble; estensione al task multiclass;
variante a B-spline.

## Crediti e licenza

Infrastruttura di quantizzazione LUT: [`lut-kan`](https://github.com/KuznetsovKarazin/lut-kan)
di O. Kuznetsov. Pipeline IDS, preprocessing e modelli di riferimento:
[`iot-audit`](https://github.com/emanuelepiodebernardis/iot-audit).
Dataset TON_IoT: Moustafa et al., UNSW Canberra (CC BY 4.0).

Licenza: MIT.
