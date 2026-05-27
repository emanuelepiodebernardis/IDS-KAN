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

### Latenza on-device e inferenza interamente intera

La prima versione del runtime usava aritmetica float (dequantizzazione
`ymin + scale*q` per ogni edge). Su microcontrollori senza FPU questo
e' costoso. Seguendo questa osservazione, il runtime e' stato riscritto
in tre stadi: float, intero (tabella int16 pre-scalata, accumulo int32,
decisione per confronto con soglia, niente sigmoid), e fully-integer
(input pre-quantizzati in Q16.16, zero float nel ciclo di inferenza).

Latenza media misurata in simulazione su Wokwi (40 vettori di test,
accuratezza 97.5% invariata in tutti gli stadi):

| Board | float | intero | fully-integer | speedup |
|---|---|---|---|---|
| Arduino Mega 2560 | 2851 µs | 680 µs | 357 µs | 8.0× |
| ESP32-C3 | 1665 µs | 207 µs | 38 µs | 44× |

L'eliminazione del float restituisce all'ESP32-C3 il vantaggio di clock
sull'aritmetica intera: il rapporto Mega/ESP32 passa da 1.7× (float) a
9.4× (fully-integer), coerente con la differenza tra un AVR a 16 MHz e un
RISC-V a 160 MHz. La decisione binaria non richiede sigmoid: basta il
segno del logit intero accumulato.

## Risultati (TON_IoT, task multiclass a 10 classi)

Il task multiclass distingue 10 categorie (normal + 9 tipi di attacco).
Lo studio sulle feature mostra che 10 feature grezze selezionate per mutual
information sono il punto ottimale (oltre, l'accuratezza non sale). Con un
preprocessing robusto (log1p sulle feature asimmetriche + scaling), si
confrontano tre varianti di KAN, tutte deployate su ESP32-C3 in aritmetica
intera pura e validate in simulazione su Wokwi.

| Modello | macro-F1 | edge | LUT | latenza ESP32 | accur. on-device |
|---|---|---|---|---|---|
| KAN single-layer | 0.86 | 100 | 100 KB | 118 µs | 90% |
| KAN multi-layer | 0.92 | 320 | 320 KB | 691 µs | 95% |

Confronto leale con i modelli del lavoro precedente, sulle stesse 10 feature
grezze e stesso split (ogni modello col preprocessing ottimale per la sua
famiglia — i tree sono invarianti alle trasformazioni monotone):

| Modello | macro-F1 |
|---|---|
| Random Forest | 0.968 |
| LightGBM / XGBoost | 0.965 |
| **KAN multi-layer** | **0.922** |
| KAN single-layer | 0.858 |
| Decision Tree | 0.793 |
| Logistic Regression | 0.214 |

Come nel binario, la KAN è competitiva ma non supera i gradient boosting.
Il contributo è il deployment: il multiclass multi-layer porta su MCU una
KAN a due strati (Chebyshev → tanh → Chebyshev) interamente quantizzata in
LUT, con il tanh tabulato. La quantizzazione a due strati è verificata non
degradare l'accuratezza (macro-F1 int 0.916 vs float 0.918), e i logit del
firmware C coincidono al bit con il modello Python. Il multiclass è
ESP32-only (la LUT supera gli 8 KB del Mega).

## Struttura del repository

```
kan-ids/
├── README.md
├── requirements.txt
├── utils.py                    modelli, preprocessing, metriche (da iot-audit)
├── src/
│   ├── kan_chebyshev.py            KAN Chebyshev binaria (training BCE, NumPy)
│   ├── kan_chebyshev_multiclass.py KAN Chebyshev multiclass (softmax)
│   ├── kan_bspline.py              variante a base B-spline (confronto basi)
│   ├── kan_torch.py                KAN multi-layer (PyTorch, autograd)
│   └── kan_multilayer_numpy.py     replica NumPy del forward multi-layer
├── scripts/
│   ├── compare_models.py           confronto binario: 5 modelli + KAN
│   ├── compare_models_multiclass.py confronto multiclass
│   ├── unified_comparison.py       confronto leale su base condivisa
│   ├── feature_curve.py            studio accuratezza vs numero feature
│   ├── basis_comparison.py         Chebyshev vs B-spline (acc + quantizzazione)
│   ├── preproc_x_model.py          effetto preprocessing x modello
│   ├── export_lut.py               export LUT binario (float)
│   ├── export_lut_int.py           export LUT binario integer-only
│   ├── export_lut_int_multiclass.py export LUT multiclass single-layer
│   ├── export_ml_int.py            export multi-layer (2 LUT + tanh) per firmware
│   ├── ml_stage2_layer1_tanh.py    quantizzazione multi-layer: layer1 + tanh
│   └── ml_stage3_full.py           quantizzazione multi-layer: forward completo
├── preprocessing/
│   └── section_310_unified_feature_engineering.py   (da iot-audit)
├── mcu/
│   ├── main_kan.cpp                firmware base (legge l'header C)
│   ├── main_kan_wokwi*.cpp         firmware binario (float / int / fully-int)
│   ├── main_kan_mc_wokwi.cpp       firmware multiclass single-layer
│   ├── main_kan_ml_wokwi.cpp       firmware multiclass multi-layer
│   ├── kan_ids_*.h                 header LUT (generati)
│   ├── kan_ml_*.h                  header LUT multi-layer (generati)
│   ├── test_vectors*.h             vettori di test (generati)
│   └── WOKWI_GUIDE*.md             guide alla simulazione
├── results/                    CSV dei risultati + header di esempio
└── data/
    └── README.md               istruzioni per scaricare TON_IoT
```

## Setup

Servono Python 3.10+ e il repository `lut-kan` clonato nella root:

```bash
git clone https://github.com/KuznetsovKarazin/lut-kan.git
pip install -r requirements.txt
```

Servono inoltre, nella root del repo, il dataset TON_IoT (vedi
`data/README.md` per le istruzioni di download). I file `utils.py`
(modelli, preprocessing e metriche, dal lavoro precedente) e
`preprocessing/section_310_...py` sono inclusi nel repo.

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

Multiclass (export integer single-layer e multi-layer):

```bash
# single-layer: 100 edge, header per il firmware mcu/main_kan_mc_wokwi.cpp
python scripts/export_lut_int_multiclass.py --csv train_test_network.csv

# multi-layer: genera i 2 layer LUT + tanh per mcu/main_kan_ml_wokwi.cpp
python scripts/export_ml_int.py
```

## Stato e lavoro futuro

Fatto: tre classificatori KAN deployati end-to-end su ESP32-C3 in aritmetica
intera pura, tutti verificati Python→C:
- binario single-layer (97.5%, fino a 38 µs, anche su Arduino Mega);
- multiclass single-layer (90% on-device, 118 µs);
- multiclass multi-layer (95% on-device, 691 µs, macro-F1 ~0.92).
Studi a supporto: curva accuratezza/numero-feature, confronto basi
(Chebyshev vs B-spline), effetto del preprocessing, confronto leale fra
modelli sulla stessa base.

Prossimi passi: flash su hardware fisico reale; ottimizzazione del footprint
del multi-layer (variante B-spline, che quantizza piu' densamente, o L
ridotto); replica del preprocessing non-lineare direttamente su MCU.

## Crediti e licenza

Infrastruttura di quantizzazione LUT: [`lut-kan`](https://github.com/KuznetsovKarazin/lut-kan)
di O. Kuznetsov. Pipeline IDS, preprocessing e modelli di riferimento:
[`iot-audit`](https://github.com/emanuelepiodebernardis/iot-audit).
Dataset TON_IoT: Moustafa et al., UNSW Canberra (CC BY 4.0).

Licenza: MIT.
