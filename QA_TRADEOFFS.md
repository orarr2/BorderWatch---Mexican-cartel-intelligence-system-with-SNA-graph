# BorderWatch v10.3 - Design Trade-offs Q&A

> Reference for instructors and researchers presenting BorderWatch.
> Each question covers a design decision, the trade-offs considered, and why this option was chosen.

---

## Architecture & Scale

### Q1: Why 5 commanders and not 7 or 10?

**The choice**: 5 commanders, each running 2-3 of the 12 cartel cells.

| Option | Pros | Cons |
|---|---|---|
| 3 commanders | Very strong signal, easy to detect | Unrealistic - would be a single command structure |
| **5 (chosen)** | Realistic span-of-control, balanced detection | Some lieutenants must be invented |
| 10+ commanders | More distributed, more realistic chaos | Detection becomes statistical only, ML models struggle |

**Why 5**: Mirrors real Mexican cartel structure (Sinaloa, CJNG, Gulf, Northeast, Tijuana) where 4-6 plaza bosses cover most border activity. Provides clear ★ markers in SNA without trivializing detection.

### Q2: Why 12 cartel cells?

**The choice**: One cell per major US-MX border crossing: TIJ, JUA, NOG, NLD, REY, MAT, MEX, PIE, AGU, SON, PAL, CUL.

**Why**: These 12 crossings handle ~95% of land border trafficking. Adding more would dilute signal. Using fewer would lose geographic realism. The 12-cell choice aligns with DEA/CBP regional reports.

### Q3: Why ~80 burner pairs per cell?

**The choice**: Per-cell burner pair count is anchored around 80 (range 70-90 depending on commander).

**Trade-off**: More pairs = stronger statistical signal but unrealistic ops count. Fewer = realistic but sparse data, ML struggles. 80 is approximately the operational scale a real plaza boss could manage (intermediate-volume border ops).

---

## Burner Detection Logic

### Q4: Why detect burners by 1:1 contact pattern (Rule 1)?

**The choice**: A phone is flagged as a burner candidate if it contacts exactly 1 unique partner.

**Trade-off**: Strict 1:1 misses 2-contact burners but has very low false-positive rate. Loose threshold (≤3 contacts) catches more but adds many legit users (e.g. spouses who only call each other).

**Why 1:1**: Operational discipline in cartel ops dictates a burner is dedicated to one specific counterpart. The 1:1 rule is the **Sacred Chart #1 invariant**.

### Q5: Why require recurring 1:1 with ≥30 calls over ≥2 days?

**The choice**: A burner candidate becomes "confirmed" only if it makes ≥30 calls to the same partner spanning ≥2 days.

**Trade-off**: Reducing thresholds catches more burners but more noise; increasing them misses real burners.

**Why these numbers**: A real burner generates 50-300 events over weeks. 30 calls over 2 days is the minimum "real activity" threshold. One-time pairs are likely legit (e.g. customer-vendor).

### Q6: Why a "max 2 contacts" rule (Rule 4)?

**The choice**: Each burner is allowed primary + at most one secondary contact.

**Why**: Cartel tradecraft limits burner exposure. A burner used with 3+ contacts is much more likely to be detected/seized → operations security demands strict limits. The generator enforces this strictly; the analyzer occasionally finds 3-contact "violations" (~0.6%) due to slip pairs or follow-up overlap - these are detection cross-fire, not data quality issues.

---

## Call Duration Model

### Q7: Why bimodal distribution for burner call duration (70% short + 30% long)?

**The choice**: Mixture distribution - 70% calls in [30, 120] seconds + 30% calls in [1800, 4000] seconds.

| Option | Pros | Cons |
|---|---|---|
| Pure short (uniform 30-120s) | Maximally clean signal | Unrealistic - coordination needs occasional long sessions |
| **70/30 mixture (chosen)** | Realistic, models two ops regimes | More variance, smaller effect size in tests |
| Pure long (uniform 1800-4000s) | Unrealistic noise level | Reverses expected signal |
| Lognormal (legit-like) | Maximally realistic | No detectable signal |

**Why 70/30**: Counter-narcotics literature documents two operational regimes: brief coded coordination (the dominant mode) and rare extended planning. The 70/30 mixture yields a measurable but realistic signal - burner median ~107s vs legit median ~336s, with Mann-Whitney rank-biserial r ≈ -0.288 (small effect). The "small effect" is methodologically honest for observational data.

### Q8: Why does Test B show "SHORTER" direction?

**The choice**: The direction (LONGER vs SHORTER) is computed dynamically from the data - `_direction` flips based on the sign of rank-biserial r.

**Why**: Since burners have median ~107s and legits have median ~336s, burners are quantitatively shorter. The dynamic switch ensures the printed interpretation always matches the empirical data, regardless of generator parameter changes.

This is the operational truth: cartel coordination is **brief and coded**, exactly as the literature predicts.

---

## ML Model Choices

### Q9: Why Random Forest and not XGBoost or a neural network?

**The choice**: `RandomForestClassifier(n_estimators=200, max_depth=8, class_weight='balanced', random_state=SEED, n_jobs=-1)`.

**Trade-off**:
| Model | Pros | Cons |
|---|---|---|
| Logistic Regression | Highly interpretable | Insufficient capacity for non-linear interactions |
| **Random Forest (chosen)** | Good capacity, very interpretable via SHAP, robust to feature scaling | Larger memory footprint |
| XGBoost | Slightly better accuracy in some settings | Less interpretable defaults, more hyperparameters |
| Neural Network | Maximum flexibility | Overkill for tabular, opaque, slow |

**Why RF**: Best balance of interpretability and performance for this tabular, mid-sized dataset. SHAP support is excellent.

### Q10: Why only 8 features and not more?

**The chosen 8**:
- `hour`, `day_of_week`, `month` - temporal
- `device_missing`, `sim_missing` - counter-surveillance signals
- `call_zscore`, `caller_total_calls` - volume normalization
- `is_night` - derived temporal flag

**Excluded by design**:
- `is_one_to_one` - direct label leakage (this IS the burner definition)
- `caller_partners` - strongly correlated with `is_one_to_one`
- `is_burner_phone` - the actual label

**Why**: With label-leaky features, AUC easily hits 0.99 - but the model has learned the label, not the underlying signal. The 8 chosen features yield AUC ~0.81, which is the **honest classifier performance** absent label leakage. This is "Variant C" hardening.

### Q11: Why time-based 70/30 split and not random 80/20?

**The choice**: Sort all events by timestamp, use earliest 70% for train, latest 30% for test.

**Trade-off**:
| Option | Pros | Cons |
|---|---|---|
| Random 80/20 | Higher AUC, more samples for train | Temporal leakage - burner that's "born" mid-period can leak into both train and test |
| **Time-based 70/30 (chosen)** | No temporal leakage, realistic operational scenario | Lower AUC, smaller train set |
| Forward-chained k-fold | More robust evaluation | Slower, complicates threshold tuning |

**Why time-based**: In real intel work, you train on past data and predict future. Random split is a methodological shortcut that overstates model capability.

### Q12: Why F2 metric for threshold tuning?

**The choice**: F2 = `(5 × P × R) / (4 × P + R)`. F2 weights recall ~4× more than precision.

**Trade-off**:
| Metric | When to use |
|---|---|
| F1 (equal weight) | Balanced cost of FP and FN |
| **F2 (recall-weighted, chosen)** | Missing a burner is far costlier than investigating a legit phone |
| Precision@K | Limited investigator capacity |

**Why F2 for intel work**: An undetected burner = an active cartel comm channel that we miss. A wrongly-flagged legit phone = an extra investigation hour. The asymmetric cost justifies recall-weighting. The F2-optimal threshold typically lands at ~0.44 vs the default 0.50.

---

## Identity Attribution

### Q13: Why does attribution use only L1 cell-tower data?

**The choice**: Map each burner to its dominant `cell_tower_id` (from `mfg`), then find the dominant non-burner phone at that tower, then attribute the burner to that non-burner's real name.

**Trade-off**: This works only for burners that have L1 voice activity. L2/L3/L4-only burners cannot be tower-attributed → marked `UNATTRIBUTED` (~74% of all confirmed burners).

**Why this approach**: L1 has unique features (cell_tower_id) that other layers lack. Extending to L2/L3/L4 would require:
- Per-layer city aggregation (no tower in those layers)
- Cross-layer SIM matching (already done for L3, partial coverage)
- Lower-confidence heuristic attribution

This is acknowledged in the methodology as a **deliberate scope limit**. Real intel work also struggles with this - most "dark" burners stay anonymous.

### Q14: Why is attribution confidence so low (~16% mean)?

**The reason**: confidence_pct = (burner's calls at tower / non-burner's total calls). A non-burner might have 80 total calls but only 13 at the burner's tower → confidence = 16%.

**Why this is OK**: Confidence_pct measures **co-location strength**, not certainty. For investigation, even 5-15% confidence is a starting hypothesis worth examining. For executive deliverables, the Excel sorts by attribution_score descending so analysts see highest-confidence first.

---

## Dashboard & Reporting

### Q15: Why include UNATTRIBUTED burners in a separate Excel sheet?

**The choice**: Sheet 1 = Named Suspects (~145 rows). Sheet 2 = Anonymous Burners (~1,648 rows).

**Why split**: The 1,648 burners are real detected operational signals - discarding them loses information. But they have no identity → they cannot appear in the executive "Named Suspects" deliverable. Splitting into two sheets:
- Sheet 1 serves command-level decision-making (named targets, prioritized)
- Sheet 2 supports operational analysts (phone-level tracking, anonymous patterns)

### Q16: Why aggregate burners by (suspect_name, legit_phone)?

**The choice**: One row per unique (person, legit phone) pair. A person operating 14 burners → 1 row with `burner_phone = "P1 | P2 | ... | P14"`.

**Trade-off**:
| Option | Pros | Cons |
|---|---|---|
| Per-burner row (no aggregation) | Detailed phone-level view | Same person appears N times, hard to count "how many suspects" |
| **Per-person row (chosen)** | Clean executive count | Phone-level detail is in joined column |
| Per-burner per-layer | Most granular | Confusing - same person × layer × burner combinations |

**Why per-person**: Executive deliverable answers "who are our targets?", not "which phones are active?". The 145-row count matches operational expectations (small set of high-value targets).

### Q17: Why is Score Histogram + Cells Breakdown side-by-side?

**The choice**: Two charts displayed via `grid2` (CSS grid 1fr 1fr).

**Why**: The histogram answers "how is suspicion distributed?" and the cells breakdown answers "where are suspects geographically?". Both are complementary contexts. Pairing them side-by-side encourages quick visual cross-reference: "we have 145 immediate-tier suspects, concentrated in TIJ + JUA".

---

## Reproducibility & Validation

### Q18: Why does AUC vary slightly between runs (0.809 vs 0.811)?

**The reason**: `RandomForestClassifier(..., n_jobs=-1)` parallelizes tree training; the order of tree results is non-deterministic across runs.

**Why we accept this**: Fixing `n_jobs=1` slows training significantly (~5×). The variance is small (±0.003 on AUC) and below the meaningful threshold for methodology discussion. If exact reproducibility is required, set `n_jobs=1`.

### Q19: Why does the F2-optimal threshold shift ±0.02 between runs?

**Same reason as Q18**: Random Forest parallelism creates minor variance in probability predictions, which shifts the F2 maximum slightly.

**Why we accept this**: The shift is much smaller than the operational tolerance. A threshold of 0.43 vs 0.45 produces nearly identical Sheet 1 contents.

### Q20: Why the Sacred Charts protected status?

**The choice**: Three charts (Sacred #1, #2, #3) are protected from silent restructuring.

**Why**: These charts are referenced in any publication accompanying the BorderWatch methodology. Changing their structure between runs (or between code versions) would break external documentation. Visual improvements are allowed; structural changes must be coordinated.
