# DANA Valencia — Data Analysis, Dashboard and Predictive Model

A data-driven decision-support tool for civil protection, built around the structural damage
assessment of buildings affected by an extreme DANA (*Depresión Aislada en Niveles Altos*) event
in the urban area of Valencia.

The project covers the full analytical pipeline: from raw inspection data to an interactive
operational dashboard with an integrated Large Language Model (LLM) assistant for natural-language
data exploration.

---

## Overview

| | |
|---|---|
| **Scenario** | Extreme DANA event affecting the urban area of Valencia, with widespread structural damage. |
| **Target users** | Civil Protection agencies, local authorities, municipalities, and emergency-management bodies. |
| **Main objective** | Provide a data-driven tool to rapidly assess building damage levels, identify intervention priorities, and allocate resources efficiently during and after the emergency. |

The system answers a single operational question quickly and reproducibly: **which buildings should
be inspected and secured first?**

---

## Pipeline

```
Raw Data (DANA)
      │
      ▼
Cleaning & Feature Engineering   ──►  Exploratory Data Analysis (EDA)
      │                                     (patterns & correlations)
      ▼
Predictive Modeling (Decision Tree + SMOTE)
      │
      ▼
Interactive Dashboard (Dash)  +  LLM Assistant
      │
      ▼
Informed Decision Making (prioritisation & safety)
```

---

## 1. Data preparation and feature engineering

The raw dataset is cleaned and reduced to an analytically meaningful structure.

**Main preprocessing steps**

- Removal of technical GIS metadata (IDs, geometry, mission fields).
- Elimination of empty or constant columns.
- Removal of duplicate and fully empty rows.
- Filtering of invalid inspections (records where `result` is missing or `NONE`).
- Type correction (binary string fields cast to integer; damage and severity fields cast to numeric).
- Median/mode imputation of residual missing values.

**Derived features**

- `antiguedad_year` — construction year extracted from the `antiguedad` timestamp.
- Semantic normalisation of `tipologia` into grouped building categories.

**Synthetic indicators**

- `danos_Total` — total damage across building zones.
- `urgente_Total` — total urgency score.

**Final dataset:** **4,578 inspected buildings** and **19 selected variables**, grouped into:

| Category | Variables |
|---|---|
| Risk classification (target) | `result` — classes `GREEN`, `YELLOW`, `RED`, `BLACK` |
| Damage & urgency indicators | `danos_*`, `urgente_*` — per building zone |
| Synthetic severity index | `IGD` (*Índice de Gravedad*) |
| Structural characteristics | `tipologia`, `antiguedad_year` |

---

## 2. Predictive model

**Goal.** In an emergency, Civil Protection must prioritise interventions even when inspections are
incomplete. Predicting the risk class allows critical situations to be anticipated, with particular
focus on the rare but most dangerous classes (`RED` and `BLACK`).

**Approach**

- **Algorithm:** Decision Tree Classifier (`max_depth=8`) — chosen for interpretability.
- **Features:** numerical predictors only, with median imputation of missing values.
- **Split:** 75% / 25% train/test, stratified to preserve class proportions.
- **Class imbalance:** addressed with **SMOTE** (`k_neighbors=16`), which oversamples the
  under-represented classes in the training set only.

**Class distribution before and after SMOTE (training set)**

| Class | Without SMOTE | With SMOTE |
|---|---:|---:|
| Green | 2,119 | 2,119 |
| Yellow | 1,135 | 2,119 |
| Red | 158 | 2,119 |
| Black | 21 | 2,119 |

**Results on the (real, non-oversampled) test set**

*Model without SMOTE*

| Class | Precision | Recall | F1 |
|---|---:|---:|---:|
| Green | 0.84 | 0.92 | 0.88 |
| Yellow | 0.78 | 0.69 | 0.73 |
| Red | 0.81 | 0.49 | 0.61 |
| Black | 0.00 | 0.00 | 0.00 |

*Model with SMOTE*

| Class | Precision | Recall | F1 |
|---|---:|---:|---:|
| Green | 0.89 | 0.74 | 0.81 |
| Yellow | 0.76 | 0.76 | 0.76 |
| Red | 0.75 | 0.45 | 0.56 |
| Black | 0.03 | **0.71** | 0.07 |

**Interpretation.** SMOTE trades a small amount of overall precision for a substantial gain in
recall on the `BLACK` class (from 0.00 to 0.71). This is a deliberate operational trade-off: in
emergency management, **failing to detect a critical building is far more costly than
misclassifying a low-risk one.**

---

## 3. Interactive dashboard

An operational dashboard built with **Plotly Dash**, designed to support fast, informed decisions in
crisis contexts. It is organised into tabs:

- **Damage heatmap** — geospatial density of damage, filterable by risk class.
- **Building detail** — selectable table with per-building damage, urgency, and location.
- **Global KPIs** — risk-class distribution and damage by building type.
- **Risk severity** — adjustable IGD / total-damage thresholds with live KPIs.
- **Urgent buildings** — map of the buildings requiring priority intervention.
- **Old vs new buildings** — comparison driven by a configurable construction-year cut-off.
- **AI assistant** — natural-language chart generation (see below).

Geocoding of street addresses is performed via **OpenStreetMap / Nominatim** (`geopy`), with results
cached in `street_coords_norm.csv` to avoid repeated lookups. Maps are centred on Valencia
(`lat 39.47, lon -0.38`) using the `carto-positron` style.

### LLM assistant

The dashboard integrates a Large Language Model to enable visual data exploration through natural
language. A user question (e.g. *"show me a scatter of IGD vs danos_Total"*) is sent to a locally
hosted model, which returns Python/Plotly code that is executed on the live dataset and rendered as
an interactive chart.

- **Serving:** local **LM Studio** endpoint (`http://localhost:1234/v1/chat/completions`).
- **Model:** `meta-llama-3.1-8b-instruct`.
- The generated code is sanitised and executed in an isolated local scope (`df`, `px`, `go` only),
  with error handling surfaced back to the user.

---

## Requirements

- Python 3.9+
- Core libraries:

```
pandas
numpy
seaborn
matplotlib
plotly
scikit-learn
imbalanced-learn
dash
geopy
requests
openpyxl
```

Install:

```bash
pip install pandas numpy seaborn matplotlib plotly scikit-learn imbalanced-learn dash geopy requests openpyxl
```

For the AI assistant, a running **LM Studio** instance serving `meta-llama-3.1-8b-instruct` is required.

---

## Usage

1. Place the raw dataset `datos_DANA.xlsx` in the project root.
2. Open `DANA_visualization.ipynb` and run the **Data Preparation** section to generate the cleaned
   dataset (`dana_valencia_normalizzato.xlsx` / `.csv` / `.pkl`).
3. Run the **Classification** section to train and evaluate the models (with and without SMOTE).
4. Run the **Dashboard and LLM** section to launch the Dash application:

```python
app.run(debug=True)
```

   
   *(The AI-assistant tab requires the LM Studio endpoint to be active.)*

---

## Notes and limitations

- The first dashboard run performs live geocoding and may take several minutes; subsequent runs read
  from the cache.
- Model performance on the `BLACK` class remains limited by the very small number of original
  samples; the SMOTE configuration prioritises recall over precision by design.
- The LLM assistant executes generated code locally; it should only be used with a trusted model and
  in a controlled environment.

---

## Author

**Manzo Vincenzo**
