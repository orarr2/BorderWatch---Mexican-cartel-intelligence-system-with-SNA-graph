# BorderWatch v10.3 — Data Logic & Behavior

> Comprehensive technical reference for how the BorderWatch data simulation works:
> distributions, layer relationships, anomaly injection, and pipeline flow.

---

## The 5 Communication Layers

### L1 — GSM (Voice Calls) | `manufacturers.csv`

- **Volume**: ~500K rows
- **Real-world analog**: Cell tower voice call records (CDR)
- **Key columns**: `call_time`, `caller_phone`, `receiver_phone`, `caller_sim_id`, `caller_device_id`, `cell_tower_id`, `call_duration_sec`, `is_outlier_call`, `is_night`, `city_location`

**Behavior**:
- ~95% legitimate business calls (lognormal duration, ~336s median)
- ~5% cartel burner calls — **bimodal** distribution:
  - 70% short coded coordination (30-120 sec)
  - 30% long planning sessions (1800-4000 sec)
  - Resulting median: ~107s, IQR [68s, 1841s]
- Hour distribution: `LEGIT_HOUR_P` for legit, `SMUG_HOUR_P` for cartel (75% night-skewed)
- Cell-tower IDs cluster by city zone (border / transit / deep)

### L2 — SMS (Text Messages) | `sms.csv`

- **Volume**: ~85K rows
- **Real-world analog**: SMS messaging logs from telecom providers
- **Key columns**: `sms_time`, `sender_phone`, `receiver_phone`, `sender_sim_id`, `sender_device_id`, `sms_body_class`

**Behavior**:
- 85% of cartel burner pairs in L1 produce SMS follow-ups within 120 seconds
- `sms_body_class` is a discrete category (greeting / coordination / status / question)
- Time-locality with L1 enables `follow_up_pairs` detection

### L3 — Email | `email.csv`

- **Volume**: ~50K rows
- **Real-world analog**: Email server logs (sender domain, recipient, timing)
- **Key columns**: `sent_time`, `sender_email`, `receiver_email`, `sender_sim_id`, `sender_device_id`

**Behavior**:
- 45% of L2-active phones also send email
- `sender_sim_id` matches L1 `caller_sim_id` → SIM-based cross-layer attribution
- ~10% chain-leaker scenario: legit-looking sender connects to cartel-aligned receivers

### L4 — Chat (Encrypted Messaging) | `chat.csv`

- **Volume**: ~70K rows
- **Real-world analog**: Encrypted-app metadata (Signal/Telegram/Wickr endpoints)
- **Key columns**: `message_time`, `handler_phone`, `associate_phone`, `handler_age_band`, `message_type`

**Behavior**:
- Handler-associate pairs are a 2-level structure: each handler manages ~5-15 associates
- `handler_age_band` enables young-suspect detection (under-30 cluster)
- Encryption is metadata-only — content is not modeled

### L5 — Waze GPS | `waze.csv`

- **Volume**: ~30K rows
- **Real-world analog**: Mobile phone GPS pings + drone activation logs
- **Key columns**: `event_time`, `full_name`, `latitude`, `longitude`, `place_name`, `city`, `is_night_movement`, `drone_id`

**Behavior**:
- Suspect movement traces lead toward border crossings (northbound bias)
- Drone activations cluster spatially+temporally near actual crossings
- Drone activation rate: 0.15 (border zone), 0.06 (transit), 0.01 (deep inland)
- Co-location enables dead-drop staging detection

---

## Cartel Structure (Ground Truth)

The generator hardcodes the following hierarchy and saves it to `burner_to_owner_ground_truth.csv`:

- **12 cartel cells** — one per major US-MX border crossing:
  TIJ, JUA, NOG, NLD, REY, MAT, MEX, PIE, AGU, SON, PAL, CUL
- **5 commanders** — fixed names, each running 2-3 cells:
  - Eleanor Whitfield Drake (TIJ, MAT, PAL — 3 cells, 510 pairs)
  - Cassandra Merriweather Sloan (JUA, MEX, CUL — 3 cells, 474 pairs)
  - Damian Hollister Vane (NLD, AGU — 2 cells, 330 pairs)
  - Magnus Dunbar Ellison (NOG, PIE — 2 cells, 324 pairs)
  - Roderick Talbot Ainsley (REY, SON — 2 cells, 288 pairs)
- **~15 lieutenants** (3 per commander, names regenerate each run)
- **~960 burner pairs** total (~80 per cell)
- **~1,118 anonymous burner phones** with no name in ground truth

The ground truth CSV has columns: `burner_phone`, `person_name`, `person_role`, `cell_id`, `city`, `legit_owner_phone`.

---

## Burner Pair Generation

Each cartel cell has ~80 burner pairs. For each pair:

1. **Pick anchor city** matching the cell (e.g. CELL-TIJ → Tijuana cell-tower clusters)
2. **Assign two phone numbers** (random 10-digit Mexican format `+52 XXXXXXXXXX`)
3. **Assign a single SIM ID + device ID per phone** (with `swap_count=140` exceptions)
4. **Pick anchor commander** for the pair (one of the 5)
5. **Generate event timeline** over the campaign period (~3 months) with anchored intensity profile
6. **Decide primary direction**:
   - 65% unidirectional (A → B only)
   - 35% bidirectional (some A→B + some B→A)
7. **Add ~20% chance of a 2nd contact** for each phone, with Rule 4 cap (max 2 contacts per burner)

Burner pair sizes range 8-22 events (uniform distribution — Sacred Chart #3 invariant).

---

## Legitimate Population

The non-cartel population is generated to provide cover-traffic:

- **~12,000 legit phones** with realistic name pools (`LEGIT_NAMES_POOL`)
- Each phone makes 30-150 calls per period, distributed across times of day
- Hour distribution: lognormal centered around 14:00 (`LEGIT_HOUR_P`)
- Duration: lognormal with mean log=6.5, sigma=0.9 (median ~336s)
- Bidirectionality: 2.5% reciprocity rate (rare)
- City distribution: weighted toward populous areas (Tijuana, Ciudad Juarez, etc.)

---

## Bridge Events (Sacred #3)

A **bridge event** is a phone with 8-22 unique partners — distinguishing it from:
- Burners (exactly 1 partner)
- Legit individuals (typically 2-7 partners)
- Power users (24+ partners)

Bridges are typically operational intermediaries (couriers, lookouts, fixers) and form the medium-volume backbone of cartel ops. The distribution is **uniform across bins 8-22** by design — Sacred Chart #3 verifies this property has no spike at any specific value (e.g. bin=15).

---

## SIM-Swap Mechanism

Mid-period, 140 phones swap their SIM ID:

- Original SIM `S1` is associated with phone `P1` for first half of period
- A new SIM `S2` replaces `S1` for second half (phone unchanged)
- Detection signature: same `caller_phone` paired with two different `caller_sim_id` values across time
- Used to test cross-layer attribution: SIM-based linkage L3↔L1 fails on swapped phones

---

## Cross-Layer Linkage Rules

| Edge | Rule | Detection Use |
|---|---|---|
| L1 → L2 | 85% of cartel pairs in L1 produce SMS follow-up within 120s | `follow_up_pairs` metric |
| L2 → L3 | 45% of cartel SMS senders send email within campaign | `chain_leaker` detection |
| L3 → L1 | Email's `sender_sim_id` matches L1 `caller_sim_id` for same phone | Identity attribution |
| L4 ↔ L1 | Chat handlers' phone numbers often appear in L1 voice records | Operational cluster identification |
| L5 ↔ all | GPS pings near border crossings indicate active trip | Trip detection + drone trigger |

---

## Anomaly Injection (5 types)

The generator injects 5 deliberate anomalies to test analyzer detection:

1. **1:1 burner pattern** — burner phones have exactly 1 partner (Sacred #1)
2. **Chain leakers** — legit-named phones appear in multiple cartel layers (~5% of legits)
3. **SIM-swap** — 140 phones change SIM mid-period
4. **Pre-border blackout** — some cartel phones go silent in the hours before a confirmed border crossing
5. **Night-shifted activity** — cartel phones cluster in 22:00-06:00 (75% vs. 12% baseline)

Each anomaly produces a measurable statistical signal that the analyzer's tests A/B/C and ML model can exploit.

---

## Reproducibility

- All random number generation uses `rng = np.random.default_rng(_seed)`
- Same seed → identical output (down to single byte)
- Slider changes don't require seed change (slider values feed into generation deterministically)

---

## Expected Output Sizes

```
manufacturers.csv                 ~80 MB
sms.csv                            ~12 MB
email.csv                          ~7 MB
chat.csv                           ~9 MB
waze.csv                           ~4 MB
burner_to_owner_ground_truth.csv  ~200 KB
```

Total: ~112 MB. Run time: 3-6 minutes on a typical laptop.
