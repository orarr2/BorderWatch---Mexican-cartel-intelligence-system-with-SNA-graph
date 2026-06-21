# BorderWatch v10.3 - Mexican Cartel Dataset Story with Intelligence Analysis

> An end-to-end, fully-synthetic **counter-narcotics intelligence pipeline**: a multi-layer
> data generator that simulates a Mexico–USA border trafficking network, paired with an
> analyzer that detects burner phones, attributes identities, maps the network, and produces
> an interactive intelligence dashboard.

The project is built for **counter-narcotics training, ML benchmarking, social-network-analysis
(SNA) research, and investigator scenario training** - all on synthetic data with full ground truth.

---

## What's in here

### Notebooks
| File | Role |
|---|---|
| [`BorderWatch_DataGenerator_v10_3.ipynb`](BorderWatch_DataGenerator_v10_3.ipynb) | **Generator** - produces the 5-layer synthetic dataset + ground truth |
| [`Mexico_Cartel_DataSet_v10_3.ipynb`](Mexico_Cartel_DataSet_v10_3.ipynb) | **Analyzer** - 163-cell pipeline: detection → attribution → SNA → ML → dashboard |
| [`Mexico_Cartel_DataSet_v10_3_Exercises.ipynb`](Mexico_Cartel_DataSet_v10_3_Exercises.ipynb) | 20 hands-on exercises that rebuild the analyzer core |
| [`BorderWatch_LinearAlgebra_Exercises.ipynb`](BorderWatch_LinearAlgebra_Exercises.ipynb) | Linear-algebra practice tied to the dataset |

### Data (Git LFS)
The 5 communication layers, stored as parquet via **Git LFS** (see [Working with the data](#working-with-the-data)):

| File | Layer | Real-world analog |
|---|---|---|
| `manufacturers_calls.parquet` | L1 - GSM voice calls | Cell-tower CDRs |
| `supplier_logistics_sms.parquet` | L2 - SMS | Telecom SMS logs |
| `supplier_procurement_email.parquet` | L3 - Email | Email server logs |
| `procurement_sales_chat.parquet` | L4 - Chat | Encrypted-app metadata |
| `waze_smuggler_movements.parquet` | L5 - GPS | GPS pings + drone activations |

### Models & reports
| File | What |
|---|---|
| `rf_burner_model.pkl` | Trained Random Forest burner classifier (+ F2-optimal threshold) |
| `borderwatch_intelligence_map.html` | Standalone Folium map (8 toggleable layers) |
| `BorderWatch_Intel_Summary_*.html` | Self-contained interactive dashboard |
| `BorderWatch_Suspects_Full_*.xlsx` | 2-sheet executive report (Named + Anonymous suspects) |

### Documentation
| File | Covers |
|---|---|
| [`README_GENERATOR.md`](README_GENERATOR.md) | Generator: layers, sliders, distributions, output sizes |
| [`README_ANALYZER.md`](README_ANALYZER.md) | Analyzer: 12 pipeline stages, outputs, metrics |
| [`DATA_LOGIC.md`](DATA_LOGIC.md) | How the simulation works - distributions, linkage, anomalies |
| [`CELL_BY_CELL.md`](CELL_BY_CELL.md) | Walk-through of every cell in both notebooks |
| [`QA_TRADEOFFS.md`](QA_TRADEOFFS.md) | Design decisions and trade-offs (20-question Q&A) |

---

## How it works (at a glance)

```
  ┌──────────────────────────┐         ┌───────────────────────────┐
  │  DataGenerator notebook  │         │     Analyzer notebook     │
  │                          │  CSVs   │                           │
  │  12 cartel cells         │ ──────► │  Burner detection (1:1)   │
  │  5 commanders            │         │  Cross-layer linkage      │
  │  ~960 burner pairs       │         │  Network analysis (SNA)   │
  │  5 anomaly types         │         │  Random Forest + SHAP     │
  │  + ground truth          │         │  Identity attribution     │
  └──────────────────────────┘         │  Dashboard + Excel        │
                                        └───────────────────────────┘
```

1. **Generate** - the generator hardcodes a cartel hierarchy (12 cells across 5 commanders,
   ~960 burner pairs) and emits 5 communication layers plus
   `burner_to_owner_ground_truth.csv`, injecting 5 deliberate anomalies (1:1 burners,
   chain leakers, SIM-swaps, pre-border blackouts, night-shifted activity).
2. **Analyze** - the analyzer reads the layers and runs a 12-stage pipeline: profiling →
   burner detection → cross-layer linkage → SNA → behavioral signals → ML → attribution →
   discovery → mapping → dashboard.
3. **Deliver** - an interactive HTML dashboard, an interactive map, and a prioritized
   Excel suspect list.

### Headline metrics (typical run)
- Random Forest **AUC ≈ 0.81** (honest, no label leakage - "Variant C" hardening)
- **Test A (night activity):** burner 65% vs legit 12% night calls, Cramér's V ≈ 0.38
- **Test B (call duration):** burner median ~107s vs legit ~330s - direction **SHORTER**
- **Test C (SIM-missing):** OR ≈ 9× (burners far more likely to have null SIM)
- ~145 named suspects + ~1,648 anonymous burners in the Excel report

---

## Quick start

### Requirements
```bash
pip install numpy pandas scipy scikit-learn joblib matplotlib \
            plotly folium networkx shap openpyxl seaborn pyarrow
```
Git LFS is required to fetch the parquet data:
```bash
git lfs install
```

### Run the pipeline
1. **Generate the data** - open `BorderWatch_DataGenerator_v10_3.ipynb`, optionally adjust the
   sliders in the first cell, and **Run All**. This writes the 5 layer CSVs + ground truth
   (~3–6 min). See [`README_GENERATOR.md`](README_GENERATOR.md).
2. **Analyze** - open `Mexico_Cartel_DataSet_v10_3.ipynb` and **Run All** (~10–15 min). The
   final cell writes `BorderWatch_Intel_Summary_*.html` and `BorderWatch_Suspects_Full_*.xlsx`.
   See [`README_ANALYZER.md`](README_ANALYZER.md).
3. **Explore** - open the generated HTML dashboard in any browser.

> **Note on data formats:** this repo ships the layers as **parquet** (compact, LFS-tracked).
> The notebooks reference the generated **CSV** filenames (`manufacturers.csv`, `sms.csv`,
> `email.csv`, `chat.csv`, `waze.csv`). Either regenerate the CSVs with the generator, or load
> the parquet files and save them under the CSV names the analyzer expects.

---

## Working with the data

The parquet layers are stored with **Git LFS** because several exceed 50 MB
(`procurement_sales_chat.parquet` is ~84 MB). After cloning:

```bash
git lfs install
git lfs pull          # downloads the actual parquet content
```

Without `git lfs pull`, the `*.parquet` files will appear as small (~130-byte) pointer stubs.

> **Heads-up on quotas:** GitHub's free tier includes ~1 GB of LFS storage and ~1 GB/month
> bandwidth. The dataset is ~260 MB, so repeated full clones draw down the monthly bandwidth.

---

## The "Sacred Charts" (protected invariants)

Three charts are referenced in the accompanying methodology and must remain structurally stable
(visual polish is fine; structural changes must be coordinated):

1. **Sacred Chart #1** - the 1:1 burner spike per layer (partner-count histogram)
2. **Sacred Chart #2** - per-cell relationship breakdown / Rule 4 compliance
3. **Sacred Chart #3** - bridge-event distribution (uniform across 8–22, no spike at any bin)

---

## Disclaimer

All data in this project is **100% synthetic**. Names, phones, emails, locations, and the cartel
hierarchy are randomly generated for research and training. Any resemblance to real people or
organizations is coincidental.
