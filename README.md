# Practical Lab 1 — Streaming Data for Predictive Maintenance (Linear Regression + Alerts)

This repository contains the code and notebook for **Practical Lab 1: Streaming Data for Predictive Maintenance with Linear Regression‑Based Alerts**.  
It extends the Data Stream Visualization workshop by training simple regressions (**Time → Axis #1–#8**) on *TRAIN* data pulled from Neon (PostgreSQL) and evaluating synthetic *TEST* datasets (**regen** and **wilk**) to produce **WARNING/ALERT/ERROR** overlays and event logs.

---

## ✨ Summary of Results

**TRAIN pull**: `Pulled 39,672 rows from Neon table 'readings_fact_csv'`  
**Axes used (TRAIN fit)**: 8 → `['Axis #1', 'Axis #2', 'Axis #3', 'Axis #4', 'Axis #5', 'Axis #6', 'Axis #7', 'Axis #8']`  
**Cadence**: `⏱️ TRAIN: 1.891s | TEST: 1.891s` (matched)

**Candidate TEST scores (lower is better):**
```
- metadata_wilk_clean.csv: score=0.1657, axes=8, dt=1.8910s, mean=0.170, std=0.255, cad=0.000, cov=0.000, ok=True
- metadata_regen_clean.csv: score=0.2216, axes=8, dt=1.8910s, mean=0.231, std=0.336, cad=0.000, cov=0.000, ok=True
```

**Selection**: `✅ Selected TEST CSV: ../data/metadata_wilk_clean.csv`  
**Total score**: `regen: 0.222 | wilk: 0.166 → Overall winner: wilk`

Side‑by‑side comparison (derived with TRAIN thresholds) also showed:
- Identical axes used for both (`8` axes).
- Identical `dt_test ≈ 1.8910 s`.
- **0 ALERT** and **0 ERROR** events for both candidates (at the chosen thresholds).

> Because both candidates have the **same cadence, same axes, and produced the same event counts (none)** under the same thresholds, the tiebreak becomes the **composite score**. The **wilk** metadata has **lower residual mean and std** and thus a **lower score (0.166 vs 0.222)**, so it is preferred.

---

## 🧠 Why **wilk** is better than **regen**

- **Lower overall score**: `0.1657` (**wilk**) vs `0.2216` (**regen**) with the same evaluation settings.  
- **Tighter residuals**: **wilk** shows a lower **mean (0.170)** and **std (0.255)** versus **regen** (**mean 0.231**, **std 0.336**), indicating predictions are, on average, closer to observed values and less dispersed.  
- **Matched cadence and axes**: Both have `dt=1.891s` and use the same 8 axes; event counts at our selected thresholds were the same (**no false alerts**), so the lower score fairly determines the winner.

In short, **wilk** preserves the temporal characteristics **and** yields **more “train‑like” behavior** (smaller residuals), which is what we want for robust overlay checks and alert logic.

---

## 🔎 Thresholds: Choosing **MinC**, **MaxC**, and **T**

We discover thresholds from residual analysis (rather than using fixed values). The method:

1. **Fit** Time → Axis regressions for all 8 axes on TRAIN.
2. **Compute residuals** on a held‑out window.
3. **Aggregate** residuals across axes and estimate center & spread (robust to outliers).
4. **Choose persistent thresholds**:
   - **MinC** = mean(residual) + **2 · std(residual)** → \~95th‑percentile exceedance (reduces noise/false alerts).  
   - **MaxC** = mean(residual) + **3 · std(residual)** → \~99.7th‑percentile exceedance (rare, stronger signal).  
   - **T** = persistence window in seconds. We require **≥ 3 consecutive samples** to avoid flicker.

With the candidate summary above (**wilk**): `mean≈0.170`, `std≈0.255`, `dt≈1.891 s`

- **MinC** ≈ `0.170 + 2·0.255 = 0.680`
- **MaxC** ≈ `0.170 + 3·0.255 = 0.935`
- **T** ≈ `3 · dt = 3 · 1.891 s = 5.673 s` → rounded/used as **~6 s**

> These conservative thresholds produced **no false Alerts/Errors** on either candidate metadata, which is consistent with typical steady‑state operation. They’re defendable using the familiar 2σ/3σ heuristic and explicit persistence (**T**) to avoid single‑tick spikes.

---

## 📦 Repository Structure (suggested)

```
.
├─ data/
│  ├─ RMBR4-2_export_test.csv        # TRAIN (from Neon dump for reproducibility)
│  ├─ metadata_regen_clean.csv       # TEST candidate (regen)
│  └─ metadata_wilk_clean.csv        # TEST candidate (wilk) ← selected
├─ notebooks/
│  └─ Practical_Lab1.ipynb           # Main analysis
├─ logs/
│  └─ events.csv                     # Alert/Error events (if any)
├─ reports/
│  └─ summary.pdf                    # Optional exported plots/summary
├─ requirements.txt
└─ README.md
```

> **Note:** The notebook also supports writing overlays/summary figures to PDF via `matplotlib.backends.backend_pdf.PdfPages` (see `reports/summary.pdf`).

---

## 🚀 Getting Started

### 1) Setup
```bash
# Python 3.10+ recommended
python -m venv .venv
source .venv/bin/activate     # (Windows) .venv\Scripts\activate

pip install -r requirements.txt
```

Create a `.env` file:
```
DATABASE_URL=postgresql+psycopg2://USER:PASSWORD@HOST/DBNAME
TRAINING_TABLE=readings_fact_csv
```

### 2) Run the notebook
Open **`notebooks/Practical_Lab1.ipynb`** and run all cells. It will:

- Pull TRAIN from Neon (`readings_fact_csv`).
- Fit regressions (Time → Axis #1–#8).
- Evaluate **regen** and **wilk** TEST metadata and compute a composite score.
- Choose the winner (**wilk**) and render overlays (with thresholds).
- Optionally export a PDF summary and write **`logs/events.csv`**.

---

## 📊 What the overlays show

- **Blue line/points**: actual TEST values per axis  
- **Black line**: regression prediction (from TRAIN)  
- **Shaded spans**: Alert/Error windows when deviations persist above **MinC/MaxC** for ≥ **T** seconds  
- **Legend**:  
  - **green dotted** = ±1σ band  
  - **orange dotted** = ±2σ band  
  - **red dotted** = ±3σ band  
  - (Quartiles/median bands may also be shown for distribution checks)

---

## 🗣️ Prompts used to create the two metadata versions

<details>
<summary><strong>Prompt 1 — General metadata generator (distribution‑matching)</strong></summary>

Please generate a new dataset called metadata.csv that:

A. Structure & size  
1. Has the same columns and same number of rows as the training data.  
2. Preserves column types (numeric stays numeric; categorical stays categorical).

B. Distribution similarity (core requirement)  
3. For each numeric column, produce new values with similar mean and standard deviation to the training data (not identical values). Target tolerance: mean within ±5%, std within ±10% of training.  
4. Keep values realistic and in-range (e.g., no negative where impossible; clip to observed min/max ± small margin).  
5. For categorical/string columns, keep the same categories and approximate the same frequencies (reshuffle—not copy rows).

C. Time column (optional, only if present)  
6. If there is a Time or timestamp column, create a sequence that continues from the last timestamp in training with the same cadence (sampling interval).
   • If continuing into a new year (e.g., 2023+) is not feasible, it’s acceptable to stay in the same year; the key is a realistic sequence that supports comparison.

D. Anti-leakage / non-copy  
7. Do not copy any row values directly from the training data. No exact duplicates of full rows.

E. Outputs  
8. Return metadata.csv as a valid CSV (same header row).  
9. Before the CSV, print a short validation report comparing per-column mean and std of training vs. metadata (two small tables) to confirm similarity for overlay checks.
</details>

<details>
<summary><strong>Prompt 2 — Shapiro–Wilk‑aware “story” metadata + Lab brief</strong></summary>

Based on Shapiro-Wilks test generate a meta data for the training data ‘RMBR4-2_export_test’’, attaching the training csv file

Context  
In the Data Stream Visualization Workshop, you learned how to stream, store, and visualize industrial current data.
This lab extends that work: you will now implement regression-based anomaly detection to generate alerts and errors when currents deviate significantly from their expected values.
This task simulates a Predictive Maintenance scenario, where early alerts and errors can flag potential failures before they occur.

Learning Objectives  
By the end of this assignment, you will be able to:  
• Extend a streaming pipeline with machine learning models.  
• Apply linear regression to detect unusual consumption trends.  
• Analyze regression residuals and discover meaningful thresholds for anomaly detection.  
• Implement an alerts/errors module in a streaming context.  
• Deliver a professional GitHub repository with reproducible code, data, and documentation.

Deliverables  
Your GitHub repository must contain:  
1. README.md (professional, setup, rules, screenshots/plots)  
2. requirements.txt (pinned dependencies)  
3. Data folder (e.g., RMBR4-2_export_test.csv for TRAIN)  
4. Codebase/Notebook that: connects to Neon, ingests/query streaming data, runs regressions (Time → Axes #1–#8), implements alerts/errors based on discovered thresholds.

🔍 Discovering Thresholds (process recapped):  
1) Fit per‑axis linear regression; record slope/intercept; plot.  
2) Analyze residuals (scatter/line/box); look for outliers.  
3) Define thresholds (MinC, MaxC, T).  
4) Implement alert/error rules:  
   - Alert: ≥ MinC above regression for ≥ T seconds continuously  
   - Error: ≥ MaxC above regression for ≥ T seconds continuously  
5) Produce synthetic TEST data from TRAIN metadata (normalize/standardize if needed).  
6) Visualize alerts/errors; annotate durations.  
7) Log results to CSV/DB.

Submission & Rubric: Push to GitHub, submit a PDF with repo URL; rubric covers setup, DB integration, simulation, regressions + residuals, threshold discovery & justification, alerts/errors implementation, and visualization.
</details>

---

## ✅ Rubric Alignment: **Visualization (0.5 pt)**

- Visual Display : In Cell 18 & 19

---

## 📌 Reproducing the wilk vs regen decision

1. Ensure both `data/metadata_regen_clean.csv` and `data/metadata_wilk_clean.csv` exist.  
2. Run the “candidate selection” cell: it computes the composite score for each.  
3. Confirm outputs match the summary above; the notebook will set the **selected TEST** to **wilk**.

---

## 📝 Notes

- If your run produces different residual stats, recompute MinC/MaxC/T using the same formulas above.
- If you intentionally stress the data to create events, lower **MinC/MaxC** or reduce **T** (but justify the change).

---

## License

Educational use for Conestoga College practical lab work.
