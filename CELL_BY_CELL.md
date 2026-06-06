# BorderWatch v10.3 — Cell-by-Cell Reference

> Walk-through of every cell in both notebooks.
> Use this when teaching, debugging, or extending the codebase.

---

# Part 1: Generator Notebook (`BorderWatch_DataGenerator_v10_3.ipynb`)

The generator has ~23 cells. Cell numbers below refer to the notebook index (0-based).

## Cell 0 — Sliders & Configuration

Defines all user-tunable `ipywidgets` sliders and consolidates them into a global `CONFIG` dict. All downstream cells read from `CONFIG`. Includes `_seed` for reproducibility.

**Key outputs**: `CONFIG`, `_seed`, slider widgets displayed in notebook.

## Cell 1 — Imports & Globals

Imports `numpy`, `pandas`, `random`, `datetime`, `os`. Initializes module-level `rng = np.random.default_rng(_seed)` and `_noise_rng` for independent noise streams. Sets `OUT_PATH = "./"`.

## Cell 2 — Cartel Structure & Ground Truth

**Largest cell (~1500 lines)**. Defines:

- `CARTEL_CELLS` — list of 12 cells (id, city, shell_company)
- `COMMANDER_NAMES` — 5 hardcoded commander names
- `_LIEUTENANT_NAMES` — 15 random lieutenant names
- `BURNER_PHONE_ALIAS` — dict mapping each burner phone to its commander/lieutenant alias (None = anonymous)
- `BURNER_PHONE_TO_CELL` — dict mapping burner phone → cell_id
- `BURNER_TO_LEGIT_MAP` — dict mapping each burner phone → its legit owner's phone
- `PAIR_DIRECTION`, `SECONDARY_LINK` — pair-level structure with Rule 4 enforcement
- `rand_burner_duration(n)` — function returning bimodal duration (70% short, 30% long)

**Final step**: Saves `burner_to_owner_ground_truth.csv` to disk with columns `burner_phone, person_name, person_role, cell_id, city, legit_owner_phone`.

## Cell 3 — Time Distribution Helpers

Defines `LEGIT_HOUR_P` (lognormal centered around 14:00) and `SMUG_HOUR_P` (night-skewed) — both as 24-element probability arrays. Helper function `gen_duration()` for lognormal call durations.

## Cell 4 — L1 Manufacturers (GSM Calls) Generation

Generates the L1 layer. For each cell:
- Iterates over bridge events (legit cover traffic)
- Iterates over cartel cell events, using `rand_burner_duration()` for call durations
- Applies anomalies: `is_outlier_call`, missing SIM/device (`missing_pct`)
- Concatenates → saves `manufacturers.csv`

## Cell 5 — Print L1 Summary

Diagnostic prints: total rows, burners detected, bridge counts. Sanity check before L2.

## Cell 6 — L2 SMS Generation

For each burner pair active in L1, generates SMS follow-ups within 120 seconds at rate `chain_ratio_l1l2`. SMS-only burners (no L1 trace) are also possible. Saves `sms.csv`.

## Cell 7 — Print L2 Summary

## Cell 8 — L3 Email Generation

Generates email records. Critical: links `sender_sim_id` to L1 `caller_sim_id` for the same burner phone (when not SIM-swapped). Saves `email.csv`.

## Cell 9 — Print L3 Summary

## Cell 10 — L4 Chat Generation

Generates handler-associate pairs with `handler_age_band`. Multi-level structure (each handler has 5-15 associates). Saves `chat.csv`.

## Cell 11 — Print L4 Summary

## Cell 12 — L5 Waze GPS Generation

Generates GPS pings + drone activations. Trip clusters near border crossings; drone activations capped per zone (border 0.15 / transit 0.06 / deep 0.01). Saves `waze.csv`.

## Cell 13 — Print L5 Summary

## Cells 14-22 — QA & Diagnostics

A series of cells that:
- Verify Rule 4 (max 2 contacts per burner)
- Plot bridge spread (Sacred Chart #3 source)
- Verify night/day distribution per layer
- Verify SIM-swap injection count
- Show burner-pair-per-cell breakdown (Sacred Chart #2 source — saves `burner_per_cell.png`)

---

# Part 2: Analyzer Notebook (`Mexico_Cartel_DataSet_v10_3.ipynb`)

163 cells across 12 stages. Below is a stage-level walkthrough; key cells highlighted.

## Stage 0 — Load (Cells 1-7)

- **Cell 1**: Library imports (`pandas`, `sklearn`, `networkx`, `plotly`, `folium`, etc.)
- **Cell 2-6**: Read 6 CSVs into DataFrames: `mfg`, `sms`, `eml`, `cht`, `waze`
- **Cell 7**: Load ground truth `burner_to_owner_ground_truth.csv` if exists; else `burner_owner_gt = None`

## Stage 1 — Profiling (Cells 8-30)

Per-layer descriptive stats:
- Volume, unique phones per layer
- Time distributions, weekend/night ratios
- Partner-count distributions per sender
- **Cell 28** — Sacred Chart #1 (4-panel 1:1 burner spike with Gamma fit overlay)
- **Cell 29** — Auxiliary partner distribution (log-y, L1+L2 only)

## Stage 2 — Burner Detection (Cells 31-50)

- **Cell 31** — Sacred Chart #3 (Bridge events distribution, 2-panel: full + zoomed bridge range)
- **Cell 32** — City volume analysis
- **Cell 41** — Tests A (night chi²) + C (SIM-missing chi²) with Cramér's V and OR
- **Cell 43** — Sacred Chart #2 reconstruction (3-panel per-cell: primary direction, secondary contacts, Rule 4 compliance) — saves `burner_per_cell.png`
- **Cell 45** — Per-cell pair breakdown
- **Cell 49** — `confirmed_burners` set built from recurring 1:1 detection

## Stage 3 — Cross-Layer (Cells 51-65)

- L3 ↔ L1 SIM matching → `l3_sim_to_burner_phone` dict
- `cross_names_all` — chain leakers (legit names appearing in multiple cartel layers)

## Stage 4 — Network Analysis (Cells 66-92)

- **Cell 70-77** — NetworkX graph build + community detection (Louvain)
- **Cell 78-83** — Centrality metrics (degree, betweenness, eigenvector)
- **Cell 91** — Hierarchy chart (loads GT csv → 5 commanders displayed) — saves `cartel_hierarchy_graph.png`
- **Cell 92** — 5-layer hierarchy (heuristic-based, may show 7 names)

## Stage 5 — Behavioral Signals (Cells 86-100)

- Night ratios, blackout detection
- Follow-up pair detection (L1 → L2 within 120s)
- SIM-swap detection
- Gray-zone bridges (5-50% shell ratio)

## Stage 6 — ML Pipeline (Cells 101-120)

- **Cell 102-106** — Feature engineering (8 features: `hour, day_of_week, month, device_missing, call_zscore, is_night, sim_missing, caller_total_calls`)
- **Cell 107** — Time-based 70/30 train/test split
- **Cell 108** — Random Forest training (200 trees, max_depth=8, class_weight=balanced)
- **Cell 109** — Evaluation (Confusion Matrix, ROC, Feature Importance, QQ plots)
- **Cell 110** — Mann-Whitney U test (Test B): burner vs legit call duration. Direction is dynamic via `_direction = "..."` switch by `rb_r` sign.
- **Cell 111** — Bootstrap CIs (B=1000) for Tests A/B/C
- **Cell 112** — Threshold tuning via Precision-Recall curve, F2 optimization. 4-panel viz: PR curve with F2 isolines, F2 vs threshold, two confusion matrix heatmaps (before/after). Saves F2-optimal threshold to `rf_burner_model.pkl`.
- **Cell 113-114** — Variant C ROC (before/after hardening, drop leaky features)
- **Cell 119** — SHAP beeswarm

## Stage 7 — Drone & GPS (Cells 121-130)

- Drone activation aggregation per suspect
- Border-arrival time prediction
- Forecast: next high-risk cities (Plotly bar)

## Stage 8 — Young Suspects (Cells 131-145)

Handler-profile-based detection of under-30 cluster. Probability scoring 0.0-1.0.

## Stage 9 — Identity Attribution (Cell 81 location-based logic)

For each detected burner phone, attribute to a real identity via cell-tower co-location:

1. Find burner's dominant `cell_tower_id` (from `mfg`)
2. Find non-burner phone with most calls at that tower
3. Assign that non-burner's name to the burner
4. Compute `confidence_pct = (burner's_tower_calls / non_burner's_total_calls) × 100`

Builds `df_confirmed_suspects` DataFrame (one row per confirmed burner per layer).

**Tiers**:
- A: tower + city match (highest confidence)
- B: tower only
- C: city only
- D: global fallback
- E: none → `UNATTRIBUTED`

## Stage 10 — Discovery (Cells 146-155)

- Extended discovery: candidates_df with 9-signal composite score
- Unified suspects (multi-criteria)
- Pre-border blackout list

## Stage 11 — Mapping (Cell 138)

Folium map with 8 toggleable layers (all default ON):
1. GSM Communication Heatmap
2. City Alert Markers
3. Cartel Cells & Commanders
4. Chain Leakers (★)
5. Dead-Drop Staging
6. Drone Activations
7. Northbound Trafficking Routes (10 routes)
8. Southern Border Monitoring

Saves `borderwatch_intelligence_map.html`.

## Stage 12 — Dashboard (Cell 159)

The master dashboard cell:
- Builds Excel export (`BorderWatch_Suspects_Full_*.xlsx`) with 2 sheets
- Renders all charts to base64
- Composes self-contained HTML (`BorderWatch_Intel_Summary_*.html`)
- Embeds Excel as base64 data URI for in-browser download
- Includes floating action button (FAB) for quick Excel access

**Key sections in order**:
1. Header + 6 metric cards
2. Top 20 Priority Suspects table + Excel download button
3. Phone Suspicion Lookup (typeahead search)
4. Interactive Intelligence Map (Folium iframe)
5. Suspect Priority Ranking (lollipop, 11×7 max-width 820px)
6. Burner Spike + Burners vs General (grid2)
7. Score Distribution Histogram + Cells Breakdown (grid2)
8. SNA + Hierarchy + Young Suspects (interactive Plotly)
9. Drone + Blackout + Gray-Zone + Forecast + ROC + SHAP

---

## Cell Naming Conventions

- `_FIG_*` — globals that hold Plotly figure objects for dashboard re-rendering
- `b64_*` — base64-encoded PNG images for HTML embedding
- `html_*` — pre-rendered HTML fragments (typically Plotly)
- Underscore-prefixed temp variables (`_p`, `_r`, `_n_b`) avoid polluting global namespace

---

## Reproducibility Notes

- Random Forest uses `random_state=SEED` and `n_jobs=-1`. The latter causes minor non-determinism across runs (best_threshold can shift ±0.02 between runs).
- All other random operations use seeded `numpy.random.default_rng(SEED)`.
- For exact reproducibility, set `n_jobs=1` (slower).
