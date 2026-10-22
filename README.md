# Regional Socioeconomic Outcomes of Flooding in Indonesia

Spatiotemporal analysis of flooding and regional socioeconomic outcomes across
all 514 Indonesian regencies and municipalities, 2016–2025, using administrative
disaster records (BNPB), official statistics (BPS), and administrative
boundaries (BIG). The pipeline covers preprocessing, a composite Flood Severity
Index, spatial statistics, trend detection, fixed-effects panel regression, and
gradient-boosted predictive modelling, with results disseminated through an
interactive dashboard.

The study is organised around two research questions:

- **RQ1 — Pattern.** Where and when is flood severity concentrated? Answered with
  the Flood Severity Index (FSI), Global Moran's I and Getis–Ord Gi\* (spatial),
  and the Mann–Kendall test (temporal).
- **RQ2 — Consequence.** Is flood exposure associated with, or predictive of,
  regional economic growth, poverty and unemployment? Answered with two-way
  fixed-effects panel regression (inferential) and XGBoost + SHAP (predictive),
  each cross-checked for robustness.

---

## Data availability

The raw source data are **not redistributed** in this repository, in line with
the data providers' terms of use. The `data/raw/` folders are intentionally
empty. Download the source files from their official providers and place them in
the folders listed below, then run the notebooks in order.

| # | Dataset | Source | Place in |
|---|---------|--------|----------|
| 1 | BPS regency/municipality master codes (Bridging Kode), one file per province (38 files) | https://sig.bps.go.id/bridging-kode/index | `data/raw/BPS/BPS_Master_Code/` |
| 2 | Annual flood (banjir) disaster records, one file per year (2016–2025) | https://gis.bnpb.go.id/ | `data/raw/BNPB/` |
| 3 | Administrative boundary geodatabase (BIG RBI50K) | https://tanahair.indonesia.go.id/portal-web/unduh/ (login required) | `data/raw/BIG/` |
| 4 | Macro socioeconomic data — GRDP (PDRB ADHK, 17 sectors), population, unemployment (TPT) | https://www.bps.go.id/ or data request via https://pst.bps.go.id/ | `data/raw/BPS/BPS_Macro/` |
| 5 | Poverty rate (P0), one file per year (2016–2025) | https://www.bps.go.id/ or https://pst.bps.go.id/ | `data/raw/BPS/P0/` |
| 6 | Mean years of schooling (RLS), one file per year (2016–2025) | https://www.bps.go.id/ or https://pst.bps.go.id/ | `data/raw/BPS/RLS/` |

Derived, aggregated outputs produced by the pipeline (the analysis panel, the
Flood Severity Index, model results) are provided under `data/processed/` and
`outputs/`.

---

## Repository structure

```
flood-indonesia-analysis/
├── README.md
├── requirements.txt
├── notebooks/            # analysis pipeline (run in numeric order)
├── data/
│   ├── raw/              # empty — download source data here (see table above)
│   │   ├── BNPB/
│   │   ├── BIG/
│   │   └── BPS/
│   │       ├── BPS_Master_Code/
│   │       ├── BPS_Macro/
│   │       ├── P0/
│   │       └── RLS/
│   └── processed/        # pipeline outputs (provided)
└── outputs/              # figures, tables, model results
```

---

## Notebook conventions

Every analysis notebook follows the same layout so they can be read and audited
the same way:

- An **Overview** cell at the top with four parts — **Purpose**, a **Data
  requirements** table (what it reads and which earlier notebook produced it), a
  **Workflow** table, and an **Outputs** table.
- The body is divided into sequential **`STEP n`** sections. Each `STEP` has a
  markdown header (`## STEP n — …`) and a matching banner inside the code cell
  (`# STEP n — …`), so the narrative and the code stay in lockstep.
- The **flood predictors** entering every RQ2 model are the three log-transformed
  dimensions `event_count_log`, `HCI_log`, `PDI_log`; the **controls** are
  `log_population` and `rls_years` (mean years of schooling). `grdp_pc` is
  deliberately excluded as a bad control.
- The **FSI** is used only for RQ1 (spatial/temporal) and for dashboard display;
  it is **not** a regressor in the RQ2 models.
- For a group of related tests the **flood-block joint Wald test** (H₀: all three
  flood coefficients = 0) is the quantity to read, not any single control's
  significance star.

---

## Execution order

Run the notebooks in numeric order. Preprocessing (1–3) must be run first; the
RQ1 notebooks (5–7) produce the master workbook and panel that the RQ2 notebooks
(8–11) consume; the dashboard notebooks (12–16) run last.

### Preprocessing

| Notebook | Purpose | Key output |
|----------|---------|------------|
| `1_preprocessing_bnpb` | Compile and clean the annual BNPB flood records (2016–2025) and attach BPS regency codes. | `1_bencana_compiled_with_bps.xlsx` |
| `2_preprocessing_map` | Join the BIG administrative geometry to BPS codes, compute regency centroids, and enrich the flood data with geography. | `Kab_Kota_with_BPS.gpkg`, `2_regency_centroids.xlsx` |
| `3_preprocessing_consolidated` | Consolidate all BPS socioeconomic series (GRDP + 17 sectors, poverty, unemployment, population, schooling) into one regency-year panel. | `bps_full_panel_complete.xlsx` |

### RQ1 — flood severity: pattern in space and time

| Notebook | Purpose | Key output |
|----------|---------|------------|
| `4_flood_eda` | Exploratory analysis of the compiled flood records (distributions, zero-event regencies, national trends) before index construction. | EDA figures |
| `5_rq1_step1` | Build the Flood Severity Index (FSI): construct the three log dimensions (Freq, HCI, PDI), derive the k-means severity typology and the η²-based dimension weights (0.306 / 0.370 / 0.324), validate against Ward clustering + bootstrap, and test **Global Moran's I**. | FSI + cluster sheets, `fig3_*`, `table2_*`, `table4_morans_i_summary` |
| `6_rq_step2` | Local spatial analysis: compute the **Getis–Ord Gi\*** hot-/cold-spots for the FSI and its three dimensions, run a k-NN sensitivity check, and export the **master workbook** (`6_flood_master.xlsx`) — the single source of truth for the FSI, clusters and spatial results used downstream. | `6_flood_master.xlsx`, `table5_gi_top10_hotspots`, `fig4_gi_hotspot_map` |
| `7_rq1_step3` | Temporal trends: per-regency **Mann–Kendall** test on the annual log-weighted FSI, Hamed–Rao correction for the national aggregate, and **Benjamini–Hochberg (BH-FDR)** correction across the 514 regency tests. Conclusions list the regencies with a significant increasing trend. | per-regency trend results, trend maps |

### RQ2 — socioeconomic consequence: panel construction and inference

| Notebook | Purpose | Key output |
|----------|---------|------------|
| `8_rq2_panel_construction` | Build the model-ready **balanced panel** (514 × 10 = 5,140 regency-years): load the flood log/z forms from notebook 6, merge the three outcomes, the 17 sector GRDP series and the controls, diagnose each variable **before and after** transformation (winsorise/log1p), and export two sheets. | `8_panel_data.xlsx` (`transformed` + `pre_transform`), `appendix_descriptive_stats.tex` |

**Fixed-effects regression (notebook 9 family).** Two-way fixed effects (regency
+ year), reported under both clustered and Driscoll–Kraay standard errors; an
effect is **robust** only if significant under both, **suggestive** if under one.
Each file saves a `fit_9*.json` read by the comparison notebook and the dashboard.

| Notebook | Purpose |
|----------|---------|
| `9a_growth_3dim` | Headline FE model — GRDP growth on the 3 composite flood dimensions (the contemporaneous "growth is robust" result). |
| `9a_growth_3dim_lagB` | Lagged (y+1) robustness: flood at *t−1*, controls at *t* — tests whether the growth effect is immediate or delayed. |
| `9b_sectors_3dim` | FE across the 17 GRDP sectors (3 dimensions), with BH-FDR correction — the source of the robust sector result (trade, education, health). |
| `9b_sectors_3dim_lagB` | Lagged (y+1) robustness for the 17 sectors. |
| `9c_growth_7var` | Growth FE using the 7 disaggregated flood components instead of the 3 dimensions (specification robustness). |
| `9d_sectors_7var` | 17-sector FE using the 7 disaggregated components. |
| `9e_model_comparison` | Side-by-side fit comparison (3-dimension vs 7-component) for growth and sectors; exports `model_comparison_3dim_vs_7var.xlsx`. |

**Social-outcome regression (notebook 10 family).** Same FE design applied to the
two social outcomes (log scale, with a raw-scale robustness twin). Each saves a
`fit_10*.json`.

| Notebook | Purpose |
|----------|---------|
| `10a_tpt_3dim` | FE — unemployment (TPT) on the 3 dimensions (the "suggestive / fragile" result). |
| `10a_tpt_3dim_lagB` | Lagged (y+1) robustness for unemployment. |
| `10b_poverty_3dim` | FE — poverty on the 3 dimensions (the "non-significant throughout" result). |
| `10b_poverty_3dim_lagB` | Lagged (y+1) robustness for poverty. |
| `10c_tpt_7var` | Unemployment FE with the 7 disaggregated components. |
| `10d_poverty_7var` | Poverty FE with the 7 disaggregated components. |
| `10e_social_comparison` | Side-by-side fit comparison (3-dimension vs 7-component) for the two social outcomes; exports `model_comparison_social_3dim_vs_7var.xlsx`. |

**Robustness and prediction (notebook 11 family).** Two kinds of check sit here:
a *mixed-effects* re-estimation (inferential robustness on the FE result) and an
*XGBoost + SHAP* model (the predictive counterpart). The two answer different
questions — **association** (FE / mixed) vs **out-of-sample prediction** (XGBoost)
— and are read together: a variable can be statistically associated yet
predictively useless, which is a central finding of the study, not a defect.

| Notebook | Purpose |
|----------|---------|
| `11_instance_mixed_effect_growth` | Re-estimate the growth model as a linear mixed model (random regency intercept, year fixed) to confirm the flood conclusion is stable across the fixed-vs-random choice. |
| `11_instance_mixed_effect_poverty_tpt` | Same mixed-effects check for poverty and unemployment. |
| `11_xgboost_shap_growth` | XGBoost + SHAP for growth: out-of-sample performance vs a naive baseline, SHAP attribution, convergence with the FE result. |
| `11_xgboost_shap_poverty` | XGBoost + SHAP for poverty (highest full-sample R², but SHAP shows it is control-driven). |
| `11_xgboost_shap_tpt` | XGBoost + SHAP for unemployment (fails to beat the naive baseline out-of-sample). |
| `11_xgboost_shap_*_matched` | Matched-control re-run of each outcome on regencies with near-uniform population and schooling — the decisive test that the predictive skill was control-driven. |
| `11_xgboost_shap_sectors` | The full XGBoost + SHAP pipeline across all 17 sectors, compared against the FE flood block in `fit_9b.json`. |
| `11_xgboost_shap_sectors_matched` | Matched-control version of the 17-sector predictive pipeline. |

### Dashboard

| Notebook | Purpose |
|----------|---------|
| `12_build_dashboard_data_flood` | Assemble the flood / FSI / spatial layers for the dashboard. |
| `13_build_dashboard_economic` | Assemble the economic (growth, sectors) layers. |
| `14_build_dashboard_social` | Assemble the poverty / unemployment layers. |
| `16_build_dashboard_model` | Assemble the predictive-outlook (XGBoost / SHAP) layer. |

---

## Reproducing the results end to end

1. Download the six source datasets into the `data/raw/` folders (table above).
2. Run `1`–`3` to build the preprocessed panels.
3. Run `4`–`7` to build the FSI and the spatial/temporal results; notebook `6`
   writes the master workbook the RQ2 notebooks depend on.
4. Run `8` to build `8_panel_data.xlsx`.
5. Run the `9*` and `10*` FE notebooks (each writes its `fit_*.json`), then the
   `9e` / `10e` comparison notebooks.
6. Run the `11*` robustness and predictive notebooks.
7. Run `12`–`16` to refresh the dashboard data.

Within each notebook the `STEP` sections run top to bottom; nothing needs to be
run out of order.

---

## Environment

Python 3.11. Install dependencies with:

```
pip install -r requirements.txt
```

Key libraries: `geopandas`, `libpysal`, `esda` (spatial); `linearmodels`,
`statsmodels` (panel and mixed effects); `xgboost`, `shap`, `scikit-learn`
(predictive); `streamlit`, `plotly` (dashboard).

---

## Dashboard

Live dashboard: https://indonesia-flood-dashboard.streamlit.app
Tools: Python, Streamlit, Plotly, GeoPandas. Base map: © OpenStreetMap contributors.

---

## Citation

If you use this code or the derived data, please cite the associated paper
(details to be added on publication).
