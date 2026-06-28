# ALDB Schema Documentation

The Adaptive Loci Database (ALDB) is the core data artifact of the SALUS-ML project. Each row represents one validated adaptive locus extracted from the peer-reviewed literature.

## Column Definitions

### Identifiers

| Column | Type | Example | Notes |
|---|---|---|---|
| `locus_id` | String | ALDB-1782011074905-0 | Auto-generated: ALDB-{timestamp}-{index} |
| `gene` | String | EPAS1 | HGNC-approved gene symbol |
| `rsid` | String / null | rs13419896 | dbSNP rsID; null if not reported in source paper |
| `chromosome` | Integer | 2 | Chromosome number (1–22, X, Y) |
| `position_hg38` | Integer / null | 46328637 | GRCh38/hg38 genomic position; null if not reported |

### Population & Environment

| Column | Type | Example | Notes |
|---|---|---|---|
| `population` | String | Tibetan highlanders | Focal population name |
| `environment_category` | String | High-altitude hypoxia | Selective pressure category |

### Selection Evidence

| Column | Type | Example | Notes |
|---|---|---|---|
| `selection_tests` | String | PBS, FST, XP-EHH | Comma-separated list of tests reported |
| `selection_statistic_value` | Float / null | 3.4 | Primary statistic value; null if not reported |
| `derived_allele_freq` | Float / null | 0.87 | DAF in focal population (0–1); null if not reported |

### Biological Evidence

| Column | Type | Example | Notes |
|---|---|---|---|
| `proposed_mechanism` | String | Regulates HIF pathway... | Mechanism from source paper |

### Quality Control

| Column | Type | Values | Notes |
|---|---|---|---|
| `tier` | Integer | 1, 2, 3 | Quality tier assigned by critic LLM |
| `tier_rationale` | String | Meets all criteria... | Critic's one-sentence justification |
| `status` | String | validated / flagged_for_review | validated = all 3 criteria met; flagged = needs manual review |
| `flag_reason` | String / null | Missing rsID... | Populated when status = flagged_for_review |

### Source

| Column | Type | Example | Notes |
|---|---|---|---|
| `source_doi` | String | 10.1126/science.1190371 | DOI of primary source paper |
| `source_title` | String | Sequencing of 50... | Full title of source paper |

### Metadata

| Column | Type | Example | Notes |
|---|---|---|---|
| `extracted_by` | String | agent / manual | How the locus was added |
| `extraction_date` | Timestamp | 2026-06-20T21:04:17Z | ISO 8601 timestamp |

---

## Inclusion Criteria

A locus must satisfy all three criteria to receive `status = validated`:

**Criterion 1 — Selection Signal**
A statistically significant result from at least one published positive selection test: PBS, FST, iHS, XP-EHH, LSBL, CMS, or equivalent population genetic statistic.

**Criterion 2 — Environmental Specificity**
The derived allele is elevated in the focal population relative to reference populations, consistent with local adaptation.

**Criterion 3 — Functional Mechanism**
The source paper proposes a biological mechanism linking the locus to the selective pressure (e.g., EPAS1 reducing erythropoiesis at high altitude).

---

## Tier Definitions

| Tier | Criteria | ML Weight | Description |
|---|---|---|---|
| 1 | All 3 met, strong evidence | 1.0 | High-confidence locus with replicated signal and well-characterized mechanism |
| 2 | All 3 met, moderate evidence | 0.75 | Valid locus but with less statistical power or less direct functional evidence |
| 3 | 2 of 3 criteria met | 0.5 | Flagged for manual review; included with reduced weight pending verification |

Loci meeting only 1 criterion are excluded entirely.

---

## Approved Selection Tests

The following tests are recognized by SALUS-ML. Any other test (e.g. BayesScan) should be reviewed before inclusion:

- **PBS** — Population Branch Statistic
- **FST** — Fixation Index (Wright's)
- **iHS** — Integrated Haplotype Score
- **XP-EHH** — Cross-Population Extended Haplotype Homozygosity
- **LSBL** — Locus-Specific Branch Length
- **CMS** — Composite of Multiple Signals
- **Rsb** — Cross-population Rsb statistic

---

## Known Limitations

1. The agent uses training-time knowledge rather than live database searches. DOIs should be spot-checked, particularly for recently published papers.
2. Genomic positions (position_hg38) should be verified against dbSNP or Ensembl before Phase 2 feature computation.
3. Three populations have uncertain inclusion status and require additional manual review: Ladakhi Himalayan, Hadza (Tanzania), San/Khoe-San.
4. The Bedouin (Sinai desert) population has limited published literature and may require manual curation.
