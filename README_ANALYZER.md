# BorderWatch Analyzer - v10.3

> **End-to-end intelligence pipeline for multi-layer cartel network analysis.**
> Reads 6 CSVs from the generator → produces ranked suspects, ML burner detection, and an interactive HTML dashboard.

---

## Purpose

The analyzer is a 163-cell Jupyter notebook (`Mexico_Cartel_DataSet_v10_3.ipynb`) that consumes the synthetic data from the generator and produces:

1. **Burner-phone detection** across L1-L4 layers using 1:1 contact pattern, recurring-pair confirmation, and Random Forest classification
2. **Identity attribution** linking each burner phone to its real-name owner via cell-tower co-location
3. **Network analysis** (SNA) - community detection, centrality, cell-leader identification
4. **Suspect priority ranking** with composite score (0-100) combining 9 evidence signals
5. **Interactive HTML dashboard** with embedded Folium map, Plotly charts, and Excel export
6. **Executive Excel report** with 2 sheets: Named Suspects (~145) + Anonymous Burners (~1,648)

---

## Pipeline Stages (12 Stages, 163 Cells)

| Stage | Cells | What |
|---|---|---|
| **0** Load & sanity | 1-7 | Read 6 CSVs, validate columns, load ground truth |
| **1** Layer profiling | 8-30 | Per-layer volume, partner distribution, Sacred Charts |
| **2** Burner detection | 31-50 | 1:1 contact rule, recurring-pair confirmation per layer |
| **3** Cross-layer | 51-65 | SIM-based linkage L3↔L1, chain-leaker detection |
| **4** Network (SNA) | 66-85 | Communities, centrality, hierarchy, leader nomination |
| **5** Behavioral signals | 86-100 | Night ratio, SIM-missing, blackout, follow-up pairs |
| **6** ML pipeline | 101-120 | Feature engineering, train/test split, Random Forest, SHAP |
| **7** Threshold tuning | 110-115 | F2-optimal threshold via Precision-Recall analysis |
| **8** Drone & GPS | 121-130 | L5 drone activations, border-arrival forecasting |
| **9** Young suspects | 131-145 | Sub-population detection by handler profile |
| **10** Discovery | 146-155 | Extended discovery, unified suspects, blackout list |
| **11** Mapping | 138 | Folium interactive map with 8 toggleable layers |
| **12** Dashboard | 159 | HTML report + Excel export + base64 download |

---

## Outputs

After a full run, the analyzer creates in the working directory:

| File | What |
|---|---|
| `BorderWatch_Intel_Summary_YYYYMMDD_HHMM.html` | Self-contained interactive dashboard |
| `BorderWatch_Suspects_Full_YYYYMMDD_HHMM.xlsx` | 2-sheet Excel report |
| `borderwatch_intelligence_map.html` | Standalone Folium map (8 layers) |
| `burner_per_cell.png` | Per-cell relationship breakdown (Sacred Chart #2) |
| `cartel_hierarchy_graph.png` | L0-L1 hierarchy with 5 named commanders |
| `burner_legit_matches.csv` | Burner ↔ legit attribution table |
| `rf_burner_model.pkl` | Trained Random Forest bundle |

---

## Key Metrics (Typical Run)

| Metric | Value |
|---|---|
| Total phones scored | ~3,700 |
| Burners detected (L1 + L2 + L4) | ~1,900 |
| Burner emails detected (L3) | ~500 |
| **Random Forest AUC** | **0.809** |
| F2-optimal threshold | ~0.44 (varies with `n_jobs=-1`) |
| Named suspects (Excel sheet 1) | ~145 |
| Anonymous burners (Excel sheet 2) | ~1,648 |
| 5 cell commanders identified | Eleanor, Cassandra, Damian, Magnus, Roderick |

---

## Statistical Tests

Three primary tests are run on detected burners vs. legit population:

| Test | Method | Typical Result |
|---|---|---|
| **A - Night activity** | chi² | Burner 65.16% vs. Legit 12.22% night calls. Cramér's V = 0.379 (medium) |
| **B - Call duration** | Mann-Whitney U | Burner median 107s vs. Legit 330s. Rank-biserial r ≈ −0.288 (small). Direction: **SHORTER** - coded coordination dominates |
| **C - SIM-missing** | chi² | Burner 10.13% null vs. Legit 1.24%. OR ≈ 8.96× (Large) |

All three tests produce `p < 0.001`. Test B's small effect size is methodologically honest - real-world behavioral signals are rarely "large" in observational data.

---

## Dashboard Layout

The HTML dashboard is structured top-to-bottom:

1. 🚨 **Header** - title + 6 metric cards (burners, leakers, follow-ups, SIM-swap, AUC, priority targets)
2. 📋 **Top 20 Priority Suspects** - evidence breakdown table with score, tier, signal chips
3. 📥 **Download Full Suspects Report** - Excel button (also floating FAB bottom-right)
4. 🔍 **Phone Suspicion Lookup** - typeahead search across 2,000 scored phones
5. 🗺️ **Interactive Intelligence Map** - embedded Folium with 8 active layers
6. 🎯 **Suspect Priority Ranking** - lollipop chart (sized 11×7 inline)
7. 📊 **Burner Spike per Layer** + 👥 **Burners vs General Population** (side by side)
8. 📈 **Score Distribution Histogram** + 🏙️ **Confirmed Burners by Cartel Cell**
9. 🌐 **SNA - Operational Cells** - interactive Plotly network graph
10. 👔 **Cartel Leadership Hierarchy** - interactive Plotly tree (5 commanders)
11. 👶 **Young Suspects** - interactive Plotly with adjustable threshold slider
12. 🛸 **Drone Activations & Night Movements** - 2-panel chart
13. ⚠️ **Gray-Zone Bridges** + 🚨 **Pre-Border Phone Blackout** (side by side)
14. 🎯 **Multi-Criteria Suspect Scoring** + 📈 **Forecast: Next High-Risk Cities** (side by side)
15. 🤖 **ML Hardening Before/After ROC** + 🧬 **SHAP Feature Impact**

---

## Excel Report Structure

**Sheet 1: Named Suspects** (~145 rows, 12 columns)

One row per unique person, aggregated across all their burners:

| Column | Notes |
|---|---|
| `suspect_name` | Real-name attribution from L1 cell-tower analysis |
| `attribution_score` | Confidence (0-100), higher = more certain |
| `layer` | Layers where active (e.g. "L1 - Manufacturers (GSM), L3 - Email") |
| `email` | If active in L3 |
| `burner_phone` | All burner phones for this person, `\|`-separated |
| `legit_phone` | The person's known legitimate phone |
| `activity_volume` | Total events across all their burners |
| `contacts_count` | Unique partners across all burners |
| `contacts_detail` | First 5 contacts `\|`-separated, empty if >5 |
| `central_score` | Composite suspicion score (0-100) |
| `priority_tier` | IMMEDIATE (≥75) / MONITORING (≥40) / WATCHLIST (≥20) / LOW |
| `matched_city` | Inferred operational city |

**Sheet 2: Anonymous Burners** (~1,648 rows, 9 columns)

Burner phones the L1-tower attribution couldn't link to a known identity (mostly L2/L3/L4-only burners). Same columns minus identity fields.

---

## How to Run

1. Confirm the 6 CSVs from the generator are in the working directory (especially `burner_to_owner_ground_truth.csv`).
2. Open `Mexico_Cartel_DataSet_v10_3.ipynb`.
3. Run all cells (Cell → Run All). Expected runtime: ~10-15 minutes.
4. The final cell (159) saves `BorderWatch_Intel_Summary_*.html` and `BorderWatch_Suspects_Full_*.xlsx`.
5. Open the HTML dashboard in any browser.

---

## Dependencies

```
numpy, pandas, scipy, scikit-learn, joblib, matplotlib,
plotly, folium (+plugins), networkx, shap, openpyxl, seaborn
```

All available via `pip install` or `conda`.

---

## Sacred Charts (3 protected invariants)

The notebook produces three named invariant charts that must remain visually consistent:

1. **Sacred Chart #1** - Stage 2 1:1 burner spike per layer (4-panel partner-count histogram with Gamma fit)
2. **Sacred Chart #2** - Per-cell relationship breakdown (`burner_per_cell.png`, Rule 4 compliance)
3. **Sacred Chart #3** - Bridge events distribution (uniform 8-22 range, no spike at bin=15)

These charts are referenced in publication and must not be silently restructured.

---

## Identity Attribution Notes

The attribution algorithm in Stage 9 uses **L1 cell-tower co-location**: for each detected burner phone, it finds the dominant non-burner phone at the same tower → attributes the burner to that non-burner's real name.

**Limitation**: Burners that operate only in L2/L3/L4 (no GSM voice activity) cannot be tower-attributed. These become `UNATTRIBUTED` (~74% of all confirmed burners) and appear only in the Excel's Anonymous Burners sheet.

Average attribution confidence: ~16% (algorithm is heuristic, not exact match). For investigation purposes, this is a starting hypothesis - manual verification required.
