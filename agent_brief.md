# SALUS-ML Literature Review Agent — Technical Brief

## Overview

The SALUS-ML Literature Review Agent is an automated n8n workflow that performs systematic literature review and adaptive locus extraction across 24 extreme-environment human populations. It replaces weeks of manual screening with an automated pipeline that searches, extracts, validates, and writes structured locus data to the ALDB Google Sheet.

---

## Workflow Architecture

```
Manual Trigger
      ↓
Code Node — Population List (24 populations as JSON)
      ↓
Loop Over Items — Batch size 1 (one population per iteration)
      ↓
Wait — 60 seconds (API rate limit management)
      ↓
AI Agent — Extraction (Gemini 2.5 Pro)
  ├── System message: expert population genomicist identity
  └── User message: per-population extraction prompt with JSON schema
      ↓
Basic LLM Chain — Critic/Reviewer (Gemini 2.5 Pro)
  └── Validates each locus against 3 inclusion criteria, assigns tier
      ↓
Code Node — JSON Parser (cleans output, generates locus IDs)
      ↓
Google Sheets — Append Row (writes to ALDB)
```

### Node Descriptions

| Node | Type | Purpose |
|---|---|---|
| Manual Trigger | Trigger | Starts the workflow on demand |
| Code in JavaScript | Code | Defines all 24 populations as structured JSON output |
| Loop Over Items | Control flow | Iterates one population at a time (batch size = 1) |
| Wait | Control flow | 60-second pause between populations to manage API rate limits |
| AI Agent | AI | Core extraction engine; searches literature and returns JSON array of loci |
| Basic LLM Chain | AI | Critic: reviews extraction against inclusion criteria, assigns tiers, flags ambiguous entries |
| Code in JavaScript1 | Code | Parses critic JSON, generates locus IDs, formats for Google Sheets |
| Append row in sheet | Integration | Writes each validated locus as a new row in ALDB |

---

## The Self-Review (Critic) Pattern

A key design feature is the two-step AI architecture. Rather than trusting the extraction agent directly, a separate critic LLM call reviews every locus before it enters the database.

**Why this matters:** A single LLM call may extract loci with incomplete evidence, hallucinate DOIs, or include entries that only partially meet inclusion criteria. The critic catches these issues before they enter the database.

**Flow:**
1. Agent extracts loci and returns a JSON array
2. Critic receives the array and checks each locus against all 3 criteria
3. Loci meeting all criteria → `status = validated`, Tier 1 or 2
4. Loci meeting 2 criteria → `status = flagged_for_review`, Tier 3
5. Loci meeting 1 criterion → excluded entirely
6. Parsed output written to Google Sheets

---

## Prompts

### Agent System Message

```
You are an expert population genomicist and scientific literature analyst working on
the SALUS-ML project — a machine learning framework for classifying adaptive resilience
loci across extreme-environment human populations.

You have deep knowledge of:
- Population genetics statistics (PBS, FST, iHS, XP-EHH, LSBL, CMS)
- The published literature on human adaptation to extreme environments
- Genomic databases (gnomAD, dbSNP, Ensembl, UCSC)
- Selection scan methodologies and their interpretation

Your job is to identify and extract only high-confidence adaptive loci that meet ALL
three of these criteria:
1. SELECTION SIGNAL — statistically significant result from at least one published
   selection test
2. ENVIRONMENTAL SPECIFICITY — derived allele is elevated in the focal population
   vs reference populations
3. FUNCTIONAL MECHANISM — the source paper proposes a biological mechanism linking
   the locus to the selective pressure

You are rigorous and conservative. You do not include loci unless you are confident
they appear in the peer-reviewed literature. You never fabricate DOIs, gene names,
or statistics. If you are uncertain about a detail, you set it to null rather than
guessing.
```

### Agent User Message (Per-Population Extraction Prompt)

```
Search the scientific literature for adaptive loci in {{ $json.population }} population
adapted to {{ $json.environment }}.

Find peer-reviewed studies reporting positive selection signals (PBS, FST, iHS,
XP-EHH, LSBL, CMS or equivalent) in this population. For each validated adaptive
locus found, extract and return a JSON array with this exact structure:

[
  {
    "gene": "gene symbol",
    "rsid": "rsID if reported, else null",
    "chromosome": "chromosome number",
    "position_hg38": "genomic position in hg38 if reported, else null",
    "population": "{{ $json.population }}",
    "environment_category": "{{ $json.environment }}",
    "selection_tests": "comma-separated tests e.g. PBS, FST, iHS",
    "selection_statistic_value": "reported value if available, else null",
    "derived_allele_freq": "DAF in focal population if reported, else null",
    "proposed_mechanism": "biological mechanism from the paper",
    "source_doi": "DOI of source paper",
    "source_title": "full title of source paper"
  }
]

Rules:
- Only include loci from peer-reviewed published studies
- Only include loci with at least one statistically significant selection test
- Only include loci where a functional mechanism is proposed
- Return ONLY the JSON array, no explanation or extra text
- If no qualifying loci found, return []
```

### Critic Prompt

```
You are a scientific reviewer for the SALUS-ML Adaptive Loci Database.

Review the following extracted locus data from the AI Agent:
{{ $json.output }}

For each locus in the array, check it against these three inclusion criteria:
1. SELECTION SIGNAL — has at least one statistically significant selection test
2. ENVIRONMENTAL SPECIFICITY — derived allele elevated in focal vs reference pops
3. FUNCTIONAL MECHANISM — a biological mechanism is proposed

For each locus assign a tier:
- Tier 1: meets all 3 criteria with strong evidence
- Tier 2: meets all 3 criteria but with moderate evidence
- Tier 3: meets 2 of 3 criteria with some uncertainty

Return ONLY a JSON array with this exact structure, no other text:
[
  {
    "gene": "...", "rsid": "...", "chromosome": "...",
    "position_hg38": "...", "population": "...",
    "environment_category": "...", "selection_tests": "...",
    "selection_statistic_value": "...", "derived_allele_freq": "...",
    "proposed_mechanism": "...", "tier": 1,
    "tier_rationale": "one sentence explaining tier assignment",
    "source_doi": "...", "source_title": "...",
    "status": "validated or flagged_for_review",
    "flag_reason": "reason if flagged, else null",
    "extracted_by": "agent", "extraction_date": "{{ $now }}"
  }
]

If a locus meets only 1 criterion, exclude it entirely.
If the input is an empty array, return [].
Return ONLY the JSON array, no explanation.
```

---

## Populations Covered

The agent processes 24 populations in 8 batches of 3 (to manage API rate limits):

| Batch | Populations | Selective pressure |
|---|---|---|
| 1 | Tibetan highlanders, Sherpa (Nepal), Andean Quechua/Aymara | High-altitude hypoxia / diet |
| 2 | Ethiopian Amhara/Oromo, Ladakhi Himalayan, Bajau sea nomads | High-altitude / diving |
| 3 | Haenyeo divers (Jeju), Turkana (Kenya), Daasanach (East Africa) | Diving / arid heat |
| 4 | Greenlandic Inuit, Indigenous Siberians, Fuegians (Yamana/Selknam) | Extreme cold |
| 5 | African Pygmies (Baka/Batwa), Sub-Saharan Africans, Amazonian indigenes | Rainforest / malaria / pathogens |
| 6 | Tsimane/Moseten, Hadza (Tanzania), San/Khoe-San | Parasites / hunter-gatherer |
| 7 | European pastoralists, East Asians, Maasai (East Africa) | Dairy / UV / pastoralism |
| 8 | Polynesian/Māori/Samoan, Mexican indigenous, Bedouin (Sinai desert) | Ocean crossing / desert heat |

---

## Known Limitations

- Agent uses training-time knowledge, not live database searches. DOIs should be spot-checked.
- Some entries may have hallucinated DOIs — always verify critical loci against PubMed.
- Very recent papers (post-training cutoff) will not be captured.
- Free tier API rate limits require 60-90 second waits between populations.
- Genomic positions should be verified against dbSNP before Phase 2 feature computation.
