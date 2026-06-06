# BorderWatch Data Generator — v10.3

> **Mexico–USA Border Trafficking Network Simulator**
> Multi-layer synthetic data generator for counter-narcotics analysis and ML benchmarking.

---

## Purpose

The generator produces a realistic, fully-synthetic five-layer dataset that simulates how a Mexican drug-trafficking cartel communicates across border-region telecom infrastructure. Output is consumed by `Mexico_Cartel_DataSet_v10_3.ipynb` (the analyzer) and yields ground-truth for evaluating burner-phone detection, attribution, and network analysis techniques.

Built for: counter-narcotics intelligence training, ML model validation, social network analysis (SNA) benchmarking, and scenario-based investigator training.

---

## What Gets Generated

| File | Layer | Volume | Description |
|---|---|---|---|
| `manufacturers.csv` | L1 — GSM voice calls | ~500K rows | Cell-tower-routed voice calls with `caller_phone`, `receiver_phone`, `cell_tower_id`, `call_duration_sec`, `is_outlier_call` |
| `sms.csv` | L2 — SMS | ~85K rows | Text messages with `sender_phone`, `receiver_phone`, `sms_body_class` |
| `email.csv` | L3 — Email | ~50K rows | Email logs with `sender_email`, `receiver_email`, `sender_sim_id` (cross-layer key) |
| `chat.csv` | L4 — Chat | ~70K rows | Encrypted-app handler/associate flows |
| `waze.csv` | L5 — GPS | ~30K rows | Geolocation pings + drone activations |
| `burner_to_owner_ground_truth.csv` | Ground truth | ~1,926 rows | Maps each burner phone to its true owner: 5 commanders, 15 lieutenants, ~1,118 anonymous |

---

## Cartel Structure

- **12 cartel cells** — one per major border crossing: TIJ, JUA, NOG, NLD, REY, MAT, MEX, PIE, AGU, SON, PAL, CUL
- **5 commanders** spanning the cells (3+3+2+2+2 cell distribution)
- **15 lieutenants** (3 per commander, randomly named per run)
- **~960 burner pairs** total (~80 per cell)
- **~1,118 anonymous burner phones** with no name attribution

---

## Configuration (Sliders)

All sliders are defined as `ipywidgets` in the first generator cell. Adjust before running:

| Slider | Default | Effect |
|---|---|---|
| `_PAIRS_MULTIPLIER` | 1.0 | Scales total burner pair count |
| `chain_ratio_l1l2` | 0.85 | % of L1 calls with L2 follow-up |
| `chain_ratio_l2l3` | 0.45 | % of L2 SMS with L3 email |
| `gamma_shape` | 3.0 | Partner-count gamma distribution shape |
| `legit_bidir_pct` | 0.025 | Legit reciprocity rate |
| `night_cartel_pct` | 0.75 | Cartel night concentration |
| `bridge_pool` | 1500 | Bridge receiver pool size |
| `chat_assoc_pool` | 1000 | L4 associate pool size |
| `drone_pct` | 0.75 | Drone activation rate (capped per zone) |
| `swap_count` | 140 | Number of SIM-swap phones |
| `secondary_pct` | 0.20 | % of burners with a 2nd contact |
| `missing_pct` | 0.12 | SIM/device null rate |
| `followup_pct` | 0.15 | Follow-up rate (burner → legit) |

---

## Burner Call Duration Model

Cartel calls follow a **bimodal distribution** (mixture of two operational regimes):

- **70%** of cartel calls are **short coded coordination** — uniform [30, 120] seconds
- **30%** of cartel calls are **long planning sessions** — uniform [1800, 4000] seconds

This mixture produces:
- Burner call median ≈ 107 seconds
- Burner call IQR ≈ [68 sec, 1841 sec]

Legitimate calls follow a lognormal distribution with median ~336 seconds. The key observable signal is that cartel calls are **shorter** than legitimate ones in the median (coded coordination dominates) — matching real-world counter-narcotics literature.

---

## Cross-Layer Linkage Rules

1. **L1 → L2**: A cartel burner pair that made N voice calls produces ~`chain_ratio_l1l2 × N` SMS follow-ups between the same pair within 120 seconds.
2. **L2 → L3**: ~45% of SMS senders also send emails. Email shares the `sender_sim_id` with L1 `caller_sim_id` for the same phone — enables SIM-based cross-layer attribution.
3. **L3 → L4**: Most L4 handlers have parallel email activity.
4. **L5 — drones**: Drone activations are time-stamped clusters near border-crossing trips. Activation rate is capped per zone (border 0.15, transit 0.06, deep 0.01).
5. **SIM swaps**: 140 phones randomly switch SIMs mid-period, creating attribution challenges.

---

## Output Verification

After a successful run, the generator prints:

```
  Ground truth saved -> ./burner_to_owner_ground_truth.csv  (1,926 rows)
    Named (commander/lieutenant): ~808 (~42%)
    Anonymous (no person):        ~1,118 (~58%)
```

The 4 communication CSVs + `waze.csv` are saved to the working directory. The analyzer notebook reads these automatically.

---

## How to Run

1. Open `BorderWatch_DataGenerator_v10_3.ipynb` in Jupyter.
2. (Optional) Adjust sliders in the first cell — defaults produce a balanced dataset.
3. Run all cells (Cell → Run All).
4. Verify 6 CSVs are created in working directory.
5. Open `Mexico_Cartel_DataSet_v10_3.ipynb` to analyze.

Total run time: ~3–6 minutes on a standard laptop (depends on `_PAIRS_MULTIPLIER`).

---

## Reproducibility

The generator uses `_seed` (set in cell 1) for all random number generation. Same seed → identical CSVs. To produce varied datasets for evaluation, change `_seed` and re-run.

---

## Key Files Output

```
./manufacturers.csv                   ~500K rows
./sms.csv                             ~85K rows
./email.csv                           ~50K rows
./chat.csv                            ~70K rows
./waze.csv                            ~30K rows
./burner_to_owner_ground_truth.csv    ~1,926 rows
```
