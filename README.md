# Barbados Health Facility Registry - Data Engineering Pipeline

##  Video Presentation
https://www.loom.com/share/b13e350623fa44ac8b5b0402c7833569  


---

##  Overview
This project contains a data engineering pipeline designed to ingest, profile, and clean the **National Health Facility Registry of Barbados**.

The source dataset was aggregated from disparate sources (OCR scans, licensing forms), resulting in significant data quality issues. This pipeline transforms the raw input into a **tidy, deduplicated dataset** optimized for downstream **Knowledge Graph** construction and **Retrieval-Augmented Generation (RAG)** applications.

---

##  How to Run
To ensure reproducibility, this project uses **relative file paths**.

### 1. Prerequisites
- Python 3.8+
- The raw file `health_registry.csv` must be in the root directory

### 2. Install Dependencies
```bash
pip install pandas numpy rapidfuzz matplotlib networkx
```

### 3. Execution
Open and run the Jupyter Notebook:
```bash
jupyter notebook health_registry_eda_clean.ipynb
```

The script will generate `cleaned_health_registry.csv` in the same directory.

---

##  Data Profiling & Quality Report
Summary of findings from the Exploratory Data Analysis (EDA) phase.

Before cleaning, the dataset exhibited high volatility consistent with merged records from multiple systems.

| Dimension | Findings |
|---------|----------|
| Completeness | **Critical:** `facility_id` was missing in over 3,500 rows (visualized in the notebook), rendering it unusable as a primary key. GPS coordinates were frequently empty. |
| Consistency | Parish names suffered from severe OCR errors, including reversed text (e.g., `.tS semaJ`) and casing inconsistencies. |
| Validity | Facility names contained non-text artifacts such as emojis (♥), ghost tokens (`NewLine`), and dangling punctuation. |
| Uniqueness | ~20% of rows were duplicates due to spelling variations (e.g., `Abbott Clinic` vs `Abbott & Sons`). |

---

##  Cleaning Methodology

### 1. Nuclear Text Normalization
Standard cleaning was insufficient for the corruption level present. Custom regex-driven functions were implemented to:

- **Fix Reversed Text**  
  Detected strings ending in `.tS` and reversed them programmatically.
- **Correct OCR Hallucinations**  
  Mapped known OCR errors (e.g., `Leahcim` → `St. Michael`) and removed ghost parish tokens embedded within facility names.
- **Standardize Facility Types**  
  Normalized variations (`ctr`, `center`, `poly`) into a controlled vocabulary (`Clinic`, `Polyclinic`, etc.).

### 2. Geospatial Parsing
The `gps_location` column contained mixed formats (WKT Points and decimal latitude/longitude).

**Logic**
- Applied bounding box validation for Barbados (Lat ≈ 13.1, Lon ≈ -59.5)
- Automatically detected and corrected swapped latitude/longitude coordinates

**Outcome**  
Invalid or reversed coordinates were corrected where possible and flagged otherwise.

### 3. Entity Resolution (Deduplication)
**Strategy:** Blocked fuzzy matching using RapidFuzz

- **Blocking:** Comparisons restricted to facilities within the same parish (improves performance and accuracy)
- **Metric:** Levenshtein distance
- **Threshold:** 85% similarity

**Result:** Duplicate entities were merged while preserving legitimately distinct facilities with similar names.

---

##  Downstream Architecture Strategy

### Knowledge Graph Strategy
**Response to Prompt:** *“Entities and Relationships to Extract”*

To transform the flat CSV into a Knowledge Graph (e.g., Neo4j), the following schema was modeled to enable hierarchical querying.

#### Entities (Nodes)
- `(:Facility)` — properties: `facility_id`, `name`, `capacity`, `coordinates`, `remarks`
- `(:Parish)` — extracted from `clean_parish` for spatial filtering
- `(:ServiceType)` — extracted from `clean_type` to group facilities by function

#### Relationships (Edges)
- `(:Facility)-[:LOCATED_IN]->(:Parish)`  
  *Use case:* Find all facilities in **St. Michael**
- `(:Facility)-[:PROVIDES_SERVICE]->(:ServiceType)`  
  *Use case:* List all **Polyclinics** regardless of location

> See the notebook for a visualization of this schema.

---

##  RAG (Retrieval-Augmented Generation)
To integrate this dataset with an LLM, a **hybrid retrieval strategy** is recommended.

**Orchestration**
- LangChain or LlamaIndex

**Retrieval Methods**
- **Structured Filters:** Use `clean_parish` and `clean_type` columns for deterministic filtering (prevents hallucinations)
- **Vector Search:** Semantic search over the `remarks` column to extract qualitative insights (e.g., facilities needing upgrades)

**Grounding**
- Structured graph queries ensure the LLM does not invent non-existent facilities.

---

##  Repository Structure
```text
/
│── health_registry_eda_clean.ipynb   # Main ETL, Profiling & Schema Visualization
│── health_registry.csv               # Raw Input Data
│── cleaned_health_registry.csv       # Final Cleaned Output
└── README.md                         # Documentation
```
