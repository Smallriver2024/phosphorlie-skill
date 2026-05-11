# PhosphoRLIE — PTM Relation & Literature Information Extraction

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-active-success.svg)]()

> **A Claude Code skill for structured extraction and quality-controlled review
> of post-translational modification (PTM) regulatory events from biomedical literature.**

PhosphoRLIE uses a **two-phase LLM workflow** — Extractor + Reviewer — to mine
PubMed titles and abstracts for kinase–substrate–disease–drug relationships.
Each extraction is automatically reviewed for enzyme name standardization,
modification site validation, and multi-entity formatting before output.

---

## Table of Contents

- [PhosphoRLIE — PTM Relation \& Literature Information Extraction](#phosphorlie--ptm-relation--literature-information-extraction)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Two-Phase Architecture](#two-phase-architecture)
  - [Quick Start](#quick-start)
    - [Install as a Claude Code Skill](#install-as-a-claude-code-skill)
    - [Use Prompts with Any LLM](#use-prompts-with-any-llm)
  - [Usage](#usage)
    - [Full Pipeline (Extract + Review)](#full-pipeline-extract--review)
    - [Review Only](#review-only)
    - [Custom PTM Types](#custom-ptm-types)
  - [Input Format](#input-format)
  - [Output Schema](#output-schema)
    - [Confidence Rubric](#confidence-rubric)
  - [What the Reviewer Checks](#what-the-reviewer-checks)
    - [1. Enzyme Name Standardization](#1-enzyme-name-standardization)
    - [2. Modification Site Validation](#2-modification-site-validation)
    - [3. Multi-Entity Formatting](#3-multi-entity-formatting)
    - [4. Cross-Field Consistency](#4-cross-field-consistency)
  - [Enzyme Name Standardization](#enzyme-name-standardization)
  - [Modification Site Validation](#modification-site-validation)
  - [Multi-Entity Handling](#multi-entity-handling)
  - [Examples](#examples)
    - [Input](#input)
    - [Output (After Phase 1 + Phase 2 Review)](#output-after-phase-1--phase-2-review)
  - [Extending to Other PTMs](#extending-to-other-ptms)
  - [Project Structure](#project-structure)
  - [Citation](#citation)
    - [BibTeX](#bibtex)
    - [APA](#apa)
  - [License](#license)

---

## Overview

Post-translational modifications (PTMs) — phosphorylation, acetylation,
ubiquitination, SUMOylation, methylation — are central to cellular signaling
and disease. The biomedical literature contains millions of articles describing
PTM regulatory events, but this knowledge is locked in unstructured text.

**PhosphoRLIE** extracts structured, machine-readable PTM event records using
large language models. Each record captures:

| Dimension | Fields Extracted |
|-----------|-----------------|
| **Molecular** | PTM target, target role, modification sites, upstream enzyme, PTM origin |
| **Pharmacological** | Intervention agent, agent type, agent validity |
| **Disease** | Disease name, cancer/non-cancer category, cancer type/subtype |
| **Phenotypic** | Phenotype context, regulation mode (enhancing/impairing/associated) |
| **Experimental** | Cell lines, species, human/non-human flag |
| **Evidence** | Trigger phrase, evidence text, evidence scope, relationship summary |
| **Quality** | Confidence (0–10), manual review flag, review reasons, reviewer notes |

The key innovation is the **Reviewer phase**: after extraction, a second LLM
pass automatically standardizes enzyme names to official gene symbols, validates
that modification sites contain both amino acid AND position, enforces semicolon
separators for multi-entity fields, and checks cross-field consistency.

---

## Two-Phase Architecture

```
User Input (PMID + Title + Abstract)
       │
       ▼
┌──────────────────────┐
│ Phase 1: EXTRACTOR    │
│ - Identify ONE focal  │
│   PTM regulatory event│
│ - Extract all 28      │
│   schema fields       │
│ - Assign confidence   │
└──────────┬───────────┘
           │  Raw extraction JSON
           ▼
┌──────────────────────┐
│ Phase 2: REVIEWER     │
│ - Standardize enzyme  │
│   names (kinase →     │
│   gene symbol)        │
│ - Validate sites      │
│   (must be AAnnn)     │
│ - Fix multi-entity    │
│   separators (;, /)   │
│ - Cross-field checks  │
│ - Update confidence   │
│   & review flags      │
└──────────┬───────────┘
           │  Reviewed JSON
           ▼
     Final Output
```

Both phases run automatically. The user sees only the final reviewed result.

---

## Quick Start

### Install as a Claude Code Skill

1. Copy `phosphorlie.md` into your project's `.claude/skills/` directory:

   ```bash
   mkdir -p .claude/skills
   cp phosphorlie.md .claude/skills/
   ```

2. Invoke in Claude Code:

   ```
   /phosphorlie PMID: 12345678
   Title: AKT phosphorylates MDM2 at Ser186 leading to p53 degradation
   Abstract: The AKT kinase directly phosphorylates...
   ```

### Use Prompts with Any LLM

The `prompts/` directory contains standalone prompt files:

| File | Purpose |
|------|---------|
| `prompts/extractor_system.md` | Phase 1 extraction system prompt |
| `prompts/reviewer_system.md` | Phase 2 review/correction system prompt |
| `prompts/schema.json` | JSON output schema (machine-readable) |
| `prompts/user_prompt.md` | User prompt template |

Send the extractor prompt first, then pass its output to the reviewer prompt.

---

## Usage

### Full Pipeline (Extract + Review)

```
# Single paper
/phosphorlie 从这篇文献中抽提磷酸化事件:
PMID: 12345678
Title: AKT phosphorylates MDM2 at Ser186...
Abstract: ...

# English
/phosphorlie Extract phosphorylation events from:
PMID: 12345678
Title: ...
Abstract: ...
```

### Review Only

```
/phosphorlie review this extraction:
{"pmid": "12345678", "ptm_target": "Akt", "modification_sites": ["Ser186"], ...}
```

### Custom PTM Types

```
/phosphorlie --ptm acetylation PMID: 45678901 Title: ...
/phosphorlie --ptm ubiquitination PMID: ...
/phosphorlie --ptm SUMOylation PMID: ...
```

---

## Input Format

**Interactive mode**: Plain text with PMID, title, and abstract.

**Batch mode (JSONL)**: One JSON object per line:

```json
{"pmid": "12345678", "title": "...", "abstract": "..."}
```

Required fields: `pmid`, `title`, `abstract`.

---

## Output Schema

Each reviewed record contains 30 fields (28 extraction + 2 reviewer-added):

```json
{
  "pmid": "string",
  "record": "string",
  "ptm_type": "phosphorylation | acetylation | ubiquitination | SUMOylation | methylation | glycosylation | other_ptm | unknown",
  "relation_exists": "yes | no | unclear",

  "evidence_text": "string",
  "evidence_scope": "single_sentence | sentence_pair | title_plus_abstract | unknown",

  "ptm_target": "string (multi-entity: semicolon-separated)",
  "ptm_target_role": "enzyme | substrate | unknown",
  "modification_sites": ["string (format: [AminoAcid][Position], e.g., S186)"],
  "upstream_enzyme": "string (multi-entity: semicolon-separated)",
  "ptm_origin": "auto_modification | upstream_enzyme_explicit | unknown",

  "primary_agent": "string (multi-entity: semicolon-separated)",
  "agent_validity": "valid_intervention | assay_reagent | detection_antibody | supporting_reagent | unknown",
  "agent_type": "approved_drug | investigational_drug | tool_compound | antibody | biologic | peptide | extract_mixture | other_reagent | unknown",

  "disease": "string (multi-entity: semicolon-separated)",
  "disease_category": "cancer | non_cancer | unclear",
  "cancer_type": "string",
  "cancer_subtype": "string",
  "phenotype_context": "string",

  "cell_lines": ["string"],
  "species": ["string"],
  "species_is_human": "yes | no | unknown",

  "trigger_phrase": "string",
  "regulation_mode": "enhancing | impairing | associated",
  "relationship": "string",

  "confidence": 0-10,
  "needs_manual_review": "yes | no",
  "review_reasons": ["string"],
  "reviewer_notes": "string"
}
```

### Confidence Rubric

| Score | Meaning |
|-------|---------|
| 9–10 | Explicit eligible relation, clear target, clear context, clear direction |
| 7–8 | Strong evidence, minor ambiguity |
| 5–6 | Probable relation, notable ambiguity |
| 3–4 | Weak or partial evidence |
| 0–2 | Unsupported or not eligible |

---

## What the Reviewer Checks

### 1. Enzyme Name Standardization

Common literature names are mapped to standard gene symbols:

| Literature Name | Standard Symbol | PTM Type |
|----------------|-----------------|----------|
| AKT | AKT1 | Phosphorylation |
| ERK | MAPK1; MAPK3 | Phosphorylation |
| JNK | MAPK8; MAPK9; MAPK10 | Phosphorylation |
| p38 | MAPK14 | Phosphorylation |
| MEK1 | MAP2K1 | Phosphorylation |
| GSK3β | GSK3B | Phosphorylation |
| IKKβ | IKBKB | Phosphorylation |
| DNA-PK | PRKDC | Phosphorylation |
| p300 | EP300 | Acetylation |
| CBP | CREBBP | Acetylation |
| PCAF | KAT2B | Acetylation |

If a name cannot be confidently mapped, the reviewer:
- Keeps the original name
- Adds `enzyme_name_uncertain` to `review_reasons`
- Documents the uncertainty in `reviewer_notes`

### 2. Modification Site Validation

Every site must match: **one-letter amino acid code + position number**
(`^[ACDEFGHIKLMNPQRSTVWY]\d+$`)

| Input | Action | Result |
|-------|--------|--------|
| `S186` | Keep | `S186` |
| `Y490` | Keep | `Y490` |
| `Ser186` | Convert (Ser→S) | `S186` |
| `Tyr-705` | Convert (Tyr→Y, remove hyphen) | `Y705` |
| `pY705` | Remove phospho-prefix | `Y705` |
| `Y` | **Remove** (no position) | — |
| `308` | **Remove** (no amino acid) | — |
| `C-terminal` | **Remove** (descriptive) | — |
| `424-434` | **Remove** (range) | — |

### 3. Multi-Entity Formatting

Multiple entities in the same field are **semicolon-separated** (`; `):

- `upstream_enzyme`: `"AKT1; MAPK1; MAPK3"`
- `ptm_target`: `"MDM2; p53"`
- `primary_agent`: `"gefitinib; erlotinib"`
- `disease`: `"breast cancer; ovarian cancer"`

The reviewer converts commas, "and", "&", and "/" (except isoform families)
to semicolons.

### 4. Cross-Field Consistency

- `upstream_enzyme` non-empty → `ptm_origin` should be `upstream_enzyme_explicit`
- `relation_exists = "no"` → all event fields should be empty
- `disease_category = "cancer"` → `disease` should contain a cancer name
- `species_is_human = "yes"` when known human cell lines are present

---

## Enzyme Name Standardization

The reviewer draws on knowledge of:

- **Human kinome**: ~518 kinases with HGNC-approved gene symbols
- **Acetyltransferases**: HATs (KAT family), MYST family
- **E3 ubiquitin ligases**: RING, HECT, RBR families
- **SUMO E3 ligases**: PIAS family, RANBP2
- **Methyltransferases**: KMT and PRMT families

For batch post-extraction verification, recommend:

| Tool | URL | Use |
|------|-----|-----|
| HGNC Multi-Symbol Checker | https://www.genenames.org/tools/multi-symbol-checker/ | Batch gene symbol validation |
| UniProt ID Mapping | https://www.uniprot.org/id-mapping | Protein name → gene symbol |
| mygene.info | https://mygene.info/v3 | Programmatic gene symbol query |

---

## Modification Site Validation

Site format rules by PTM type:

| PTM Type | Modified Residues | Site Format |
|----------|------------------|-------------|
| Phosphorylation | S, T, Y | `[STY]\d+` (e.g., S186) |
| Acetylation | K | `K\d+` (e.g., K382) |
| Ubiquitination | K | `K\d+` (e.g., K48) |
| SUMOylation | K | `K\d+` (e.g., K524) |
| Methylation | K, R | `[KR]\d+` (e.g., K4, R17) |
| Glycosylation | N, S, T | `[NST]\d+` (e.g., N297) |

---

## Multi-Entity Handling

| Scenario | Format | Example |
|----------|--------|---------|
| Multiple kinases phosphorylate same target | `; ` separated | `"MAPK1; MAPK3"` |
| Kinase phosphorylates multiple sites | Array of strings | `["S186", "S188", "T190"]` |
| Multiple agents tested | `; ` separated | `"gefitinib; erlotinib; afatinib"` |
| Multiple diseases studied | `; ` separated | `"breast cancer; ovarian cancer"` |
| Isoform family from text | Preserve `/` | Use `; ` for distinct genes |
| Duplicate entries | Remove | — |

---

## Examples

### Input
```
PMID: 12345678
Title: AKT and ERK phosphorylate MDM2 at Ser186 and Ser188 promoting p53 degradation in breast cancer cells
Abstract: Both AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188, which enhances MDM2-mediated ubiquitination and degradation of p53. Treatment with the AKT inhibitor MK-2206 or the MEK inhibitor PD98059 impaired this phosphorylation in MCF-7 and T47D breast cancer cells, leading to apoptosis.
```

### Output (After Phase 1 + Phase 2 Review)
```json
{
  "pmid": "12345678",
  "ptm_type": "phosphorylation",
  "relation_exists": "yes",
  "evidence_text": "Both AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188, which enhances MDM2-mediated ubiquitination and degradation of p53.",
  "evidence_scope": "single_sentence",
  "ptm_target": "MDM2",
  "ptm_target_role": "substrate",
  "modification_sites": ["S186", "S188"],
  "upstream_enzyme": "AKT1; MAPK1; MAPK3",
  "ptm_origin": "upstream_enzyme_explicit",
  "primary_agent": "MK-2206; PD98059",
  "agent_validity": "valid_intervention",
  "agent_type": "investigational_drug; tool_compound",
  "disease": "breast cancer",
  "disease_category": "cancer",
  "cancer_type": "breast cancer",
  "phenotype_context": "apoptosis",
  "cell_lines": ["MCF-7", "T47D"],
  "species": ["human"],
  "species_is_human": "yes",
  "trigger_phrase": "AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188",
  "regulation_mode": "enhancing",
  "relationship": "AKT1; MAPK1; MAPK3 phosphorylate MDM2 at S186 and S188, enhancing p53 degradation; MK-2206 and PD98059 impair this phosphorylation, restoring p53 and inducing apoptosis",
  "confidence": 8,
  "needs_manual_review": "no",
  "review_reasons": [],
  "reviewer_notes": "Standardized AKT→AKT1; Standardized ERK→MAPK1;MAPK3; Converted sites Ser186→S186, Ser188→S188; Changed separator to semicolons; Added agent_type for PD98059"
}
```

**Reviewer corrections applied:**
1. `AKT` → `AKT1` (standard gene symbol)
2. `ERK` → `MAPK1; MAPK3` (ERK1=MAPK3, ERK2=MAPK1)
3. `Ser186` → `S186`, `Ser188` → `S188` (three-letter → one-letter code)
4. Multi-entity separators standardized to `; `
5. `PD98059` agent_type added (`tool_compound`)
6. Confidence reduced 9→8 (substantive corrections made)

---

## Extending to Other PTMs

PhosphoRLIE supports all major PTM types through its `ptm_type` field:

| PTM Type | Enzyme Class | Modified Residues | Site Pattern |
|----------|-------------|-------------------|-------------|
| `phosphorylation` | Kinase | S, T, Y | `[STY]\d+` |
| `acetylation` | Acetyltransferase (HAT) | K | `K\d+` |
| `ubiquitination` | E3 ubiquitin ligase | K | `K\d+` |
| `SUMOylation` | SUMO E3 ligase | K | `K\d+` |
| `methylation` | Methyltransferase | K, R | `[KR]\d+` |
| `glycosylation` | Glycosyltransferase | N, S, T | `[NST]\d+` |

To use a non-default PTM type, specify it when invoking the skill:
```
/phosphorlie --ptm ubiquitination PMID: ...
```

---

## Project Structure

```
phosphorlie/
├── README.md                         # This file
├── LICENSE                           # MIT License
├── CITATION.cff                      # Citation metadata (GitHub-ready)
├── phosphorlie.md                    # Claude Code skill definition (main file)
├── prompts/
│   ├── extractor_system.md           # Phase 1: Extraction system prompt
│   ├── reviewer_system.md            # Phase 2: Review/correction system prompt
│   ├── schema.json                   # JSON output schema (30 fields)
│   └── user_prompt.md                # User prompt template
└── examples/
    ├── input.jsonl                   # Example input (5 records)
    ├── output.jsonl                  # Example reviewed output (5 records)
    └── no_event.jsonl                # Example: technical paper (no eligible event)
```

---

## Citation

If you use PhosphoRLIE in your research, please cite:

### BibTeX
```bibtex
@software{phosphorlie2025,
  title        = {PhosphoRLIE: A Post-Translational Modification Relation
                  \& Literature Information Extraction Framework},
  year         = {2025},
  publisher    = {GitHub},
  url          = {https://github.com/Smallriver2024/phosphorlie-skill},
  note         = {Two-phase LLM framework (Extractor + Reviewer) for
                  structured PTM event extraction from biomedical literature}
}
```

### APA
> GitHub. https://github.com/Smallriver2024/phosphorlie-skill

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

*For questions, issues, or contributions, please open a GitHub issue.*
