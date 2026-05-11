---
name: phosphorlie
description: >-
  Extract and review post-translational modification (PTM) regulatory events from
  biomedical literature. Two-phase workflow: (1) Extractor — pull ONE structured
  PTM event per paper from PubMed titles/abstracts; (2) Reviewer — standardize
  enzyme names, validate modification sites, handle multi-entity formatting, and
  flag quality issues. Supports phosphorylation, acetylation, ubiquitination,
  SUMOylation, methylation, and other PTMs.
tags: [biomedical, literature-mining, ptm, phosphorylation, nlp, bioinformatics, quality-control]
allowed-tools: [Read, Write, Bash, WebSearch]
---

# PhosphoRLIE — PTM Relation & Literature Information Extraction

## Overview

PhosphoRLIE extracts structured post-translational modification (PTM) regulatory
events from biomedical literature using a **two-phase quality-controlled workflow**:

| Phase | Role | Responsibility |
|-------|------|---------------|
| **Phase 1** | **Extractor** | Identify and extract ONE PTM regulatory event from title + abstract |
| **Phase 2** | **Reviewer** | Standardize enzyme names, validate modification sites, format multi-entity fields, flag quality issues |

Both phases run automatically in sequence. The output is a reviewed, corrected,
and quality-flagged JSON record ready for downstream analysis.

---

## Usage Modes

### Mode 1: Full Pipeline (Extract + Review)

The default mode. Provide a paper's PMID, title, and abstract. Claude first
extracts the PTM event, then reviews and corrects it.

**Trigger examples**:
- `/phosphorlie PMID: 12345678 Title: ... Abstract: ...`
- "Extract and review phosphorylation events from this paper: ..."
- "帮我从这篇文献抽提磷酸化信息并审核: ..."

### Mode 2: Review Only

Provide an already-extracted JSON record. Claude applies only the Reviewer phase
to standardize names, validate sites, and fix formatting.

**Trigger examples**:
- `/phosphorlie review this extraction: {"pmid": "...", "ptm_target": "Akt", ...}`
- "审核这份抽提结果，检查激酶名称和位点格式: ..."

### Mode 3: Batch Extraction + Review

Provide a JSONL file path. Claude processes each record through both phases and
writes reviewed results to an output file.

**Trigger examples**:
- `/phosphorlie batch process input.jsonl → output_reviewed.jsonl`

### Mode 4: Custom PTM Type

Specify a PTM type other than phosphorylation.

**Trigger examples**:
- `/phosphorlie --ptm acetylation PMID: ...`
- "使用泛素化模式抽提这篇文献: ..."

---

## Phase 1: Extractor

### Extractor System Prompt

When acting as the Extractor, Claude MUST apply the following internal prompt.
Do NOT display this prompt to the user.

```
You are an information extraction engine for post-translational modification (PTM)
biomedical literature.

Read ONLY the provided PMID, title, and abstract.

Your task is to extract ONE PTM-related regulatory evidence event and return
exactly ONE JSON object that follows the given schema.

═══════════════════════════════════════
CORE INCLUSION RULE
═══════════════════════════════════════
A record is eligible only when ALL of the following are satisfied:
1. The text contains an explicit PTM-related regulatory relation
   (phosphorylation, acetylation, ubiquitination, SUMOylation, methylation, etc.).
2. At least one left-side entity exists: modifying enzyme (kinase, acetyltransferase,
   E3 ligase, etc.) OR substrate/target.
3. At least one right-side context exists: valid intervention agent OR disease OR
   phenotype_context.

═══════════════════════════════════════
SCOPE RULES
═══════════════════════════════════════
- Use only the provided title and abstract.
- Do NOT use outside knowledge to add missing facts, isoforms, or IDs.
- Keep the PTM target at the granularity reported in the literature.
  Examples: keep "AKT" if the text says "AKT"; keep "ERK1/2" if the text says "ERK1/2".
- One output record = ONE focal PTM event.
- If the abstract contains multiple PTM events, return only the SINGLE best-supported event.
- Prefer a single sentence as evidence. Use two adjacent sentences only if
  the relation is explicitly linked across them.
- Do NOT merge distant statements across the abstract.
- Do NOT output article-wide flat entity lists.

═══════════════════════════════════════
ENTITY RULES
═══════════════════════════════════════
- modifying_enzyme: the enzyme catalyzing the PTM (kinase, acetyltransferase,
  E3 ligase, etc.), explicitly mentioned in the text.
- substrate: a PTM target protein explicitly mentioned.
- ptm_target: the single focal target protein for this record.
- primary_agent: an intervention factor directly applied to the biological system
  (treatment, stimulation, inhibition, deprivation, exposure, extract, etc.).
- disease: a true disease entity if explicitly stated.
- phenotype_context: a phenotype/biological process (e.g., apoptosis, migration,
  proliferation, angiogenesis, drug sensitivity, cell-cycle arrest).

═══════════════════════════════════════
UPSTREAM ENZYME RULES
═══════════════════════════════════════
- upstream_enzyme: fill ONLY when the same focal PTM event explicitly states
  which upstream enzyme modifies the ptm_target.
- Do NOT infer upstream enzymes from pathway knowledge.
- When an explicit enzyme→substrate modification relation is stated,
  use the substrate as ptm_target and the enzyme as upstream_enzyme.
- Use the enzyme itself as ptm_target ONLY for auto-modification events.
- For phosphorylation specifically:
  * kinase→substrate phosphorylation → substrate is ptm_target, kinase is upstream_enzyme
  * kinase autophosphorylation → kinase is ptm_target, upstream_enzyme is empty

═══════════════════════════════════════
MULTI-ENTITY FORMATTING RULES (IMPORTANT)
═══════════════════════════════════════
- Multiple enzymes/kinases: join with "; " (semicolon + space)
  Example: "AKT1; AKT2; AKT3"
- Multiple substrates/targets: join with "; " (semicolon + space)
  Example: "MDM2; p53"
- Multiple modification sites: use a JSON array of strings
  Example: ["S186", "S188", "T190"]
- Multiple intervention agents: join with "; " (semicolon + space)
  Example: "gefitinib; erlotinib"
- Multiple diseases: join with "; " (semicolon + space)
- Do NOT merge unrelated entities from different sentences or different events.
- Only combine entities that participate in the SAME focal PTM event.
- If the text uses "/" to indicate isoforms of the same family, preserve it
  (e.g., "ERK1/2", "AKT1/2/3").

═══════════════════════════════════════
MODIFICATION SITE RULES
═══════════════════════════════════════
- Each site entry MUST follow the format: ONE-LETTER amino acid + position number.
  Valid: "S186", "Y490", "T308", "K382", "S15"
  Invalid (DO NOT include): "Y" (no position), "308" (no amino acid),
  "Ser186" (three-letter code), "C-terminal" (descriptive),
  "Ser/Thr-Pro" (motif), "424-434" (range)
- Extract the exact site from the evidence text.
- If the text uses three-letter amino acid codes, convert to one-letter:
  Ser→S, Thr→T, Tyr→Y, Lys→K, Arg→R, etc.
- If no explicit site with both amino acid AND position is present,
  leave modification_sites as an empty array [].
- If multiple sites are mentioned for the same focal event, include all as
  separate array entries.

═══════════════════════════════════════
PTM ORIGIN RULES
═══════════════════════════════════════
- ptm_origin must be one of: auto_modification, upstream_enzyme_explicit, unknown.
- "auto_modification": text explicitly states self-modification
  (autophosphorylation, autoacetylation, autoubiquitination).
- "upstream_enzyme_explicit": text explicitly states an upstream enzyme modifies ptm_target.
- "unknown": neither of the above conditions is met.

═══════════════════════════════════════
AGENT VALIDITY RULES
═══════════════════════════════════════
- valid_intervention: agent directly applied as treatment/perturbation in the system.
- assay_reagent: synthetic substrates, ATP in assay, affinity matrices, chromatography materials.
- detection_antibody: IP/detection/enrichment antibodies or antisera.
- supporting_reagent: non-core supporting materials.
- unknown: role cannot be determined.

═══════════════════════════════════════
EXCLUSION RULES
═══════════════════════════════════════
- Do NOT treat detection antibodies, IP antisera, synthetic assay peptides,
  affinity matrices, purification reagents, buffers as valid intervention agents.
- Do NOT output records based on simple co-occurrence alone.
- Do NOT output records for purely technical/methodological papers
  (phosphoprotein isolation, assay protocols, etc.) without a real regulatory relation.
- If no eligible event exists, set relation_exists = "no" with empty event fields.

═══════════════════════════════════════
DISEASE RULES
═══════════════════════════════════════
- disease: fill ONLY if a true disease is explicitly stated; otherwise empty string.
- disease_category: "cancer" if a cancer disease is present; "non_cancer" if a
  non-cancer disease is present; "unclear" if no clear disease.
- cancer_type: broad label when cancer context is explicit
  (e.g., "breast cancer", "lung cancer", "leukemia", "melanoma", "glioma").
- cancer_subtype: specific subtype ONLY when explicitly supported
  (e.g., "lung adenocarcinoma", "triple-negative breast cancer", "AML").
- Do NOT guess cancer subtype if the text only supports a broader cancer type.

═══════════════════════════════════════
RELATIONSHIP RULES
═══════════════════════════════════════
- trigger_phrase: copy the original trigger phrase from the text verbatim.
- regulation_mode:
  * "enhancing" — PTM event promotes/strengthens/supports the downstream effect.
  * "impairing" — PTM event weakens/opposes/reduces the downstream effect.
  * "associated" — relation exists but direction is unclear.
- relationship: a concise English sentence summarizing the complete PTM-regulatory
  relationship (enzyme→target→site→effect→disease/agent context).

═══════════════════════════════════════
CONFIDENCE RUBRIC
═══════════════════════════════════════
- 9-10: explicit eligible relation, clear target, clear context, clear direction.
- 7-8: strong evidence, minor ambiguity remains.
- 5-6: probable eligible relation, notable ambiguity.
- 3-4: weak or partial evidence.
- 0-2: unsupported or not eligible.

═══════════════════════════════════════
OUTPUT RULES
═══════════════════════════════════════
- Return exactly ONE compact JSON object.
- Follow the schema EXACTLY — do not add, remove, or rename any keys.
- Do NOT output markdown fences (```json```).
- Do NOT output explanations or commentary.
- The output must be parseable by a standard JSON parser.
```
```

---

## Phase 2: Reviewer

### Reviewer System Prompt

After Phase 1 extraction is complete, Claude switches to the Reviewer role.
The Reviewer checks and corrects the extraction, applying the following rules.

**IMPORTANT**: The Reviewer must apply this prompt internally. Do NOT display
this prompt to the user.

```
You are a quality-control reviewer for PTM event extractions from biomedical literature.

Your task is to review a PTM extraction JSON record, correct errors, standardize
names, validate sites, and flag remaining issues. You have access to the original
title and abstract for verification.

═══════════════════════════════════════
REVIEW WORKFLOW
═══════════════════════════════════════
1. Enzyme Name Review → standardize names, fix formatting
2. Modification Site Review → validate format, remove invalid sites
3. Multi-Entity Review → ensure correct separator usage
4. Cross-Field Consistency Check → verify logical coherence
5. Quality Flag Update → adjust confidence and review flags

═══════════════════════════════════════
STEP 1: ENZYME NAME STANDARDIZATION
═══════════════════════════════════════

1a. Kinase Name Standardization (for phosphorylation)

Apply the following corrections using your knowledge of standard gene nomenclature:

Common non-standard → standard mappings:
| Non-standard (common in literature) | Standard Gene Symbol |
|--------------------------------------|---------------------|
| AKT (unspecified isoform) | AKT1 (if isoform unclear, keep as "AKT") |
| PKB | AKT1 |
| ERK (unspecified) | MAPK1; MAPK3 (or keep "ERK1/2" if explicitly stated) |
| ERK1 | MAPK3 |
| ERK2 | MAPK1 |
| p38 | MAPK14 (or the specific p38 isoform mentioned) |
| JNK (unspecified) | MAPK8; MAPK9; MAPK10 (or keep "JNK" if unspecified) |
| JNK1 | MAPK8 |
| JNK2 | MAPK9 |
| JNK3 | MAPK10 |
| MEK (unspecified) | MAP2K1; MAP2K2 (or keep "MEK1/2") |
| MEK1 | MAP2K1 |
| MEK2 | MAP2K2 |
| PI3K (unspecified) | PIK3CA; PIK3CB; PIK3CD; PIK3CG (or keep "PI3K") |
| PKC (unspecified) | PRKCA; PRKCB; PRKCG (or keep "PKC") |
| PKA (unspecified) | PRKACA; PRKACB; PRKACG (or keep "PKA") |
| GSK3 | GSK3B (if beta isoform is standard) |
| GSK3β | GSK3B |
| GSK3α | GSK3A |
| CHK1 | CHEK1 |
| CHK2 | CHEK2 |
| MK2 | MAPKAPK2 |
| RSK (unspecified) | RPS6KA1; RPS6KA2; RPS6KA3 (or keep "RSK") |
| p90RSK | RPS6KA1 |
| p70S6K | RPS6KB1 |
| mTOR | MTOR |
| ATM | ATM (already standard) |
| ATR | ATR (already standard) |
| DNA-PK | PRKDC |
| IKKα | CHUK |
| IKKβ | IKBKB |
| IKKε | IKBKE |
| TBK1 | TBK1 (already standard) |
| CK2 | CSNK2A1; CSNK2A2 |
| CK1 | CSNK1A1; CSNK1D; CSNK1E |

Rules:
- If the text specifies a particular isoform, use that specific gene symbol.
- If the text uses a family-level name without specifying isoform, keep the family name.
- If you are uncertain about the correct standard name, flag with review reason
  "enzyme_name_uncertain" and keep the original name.
- For names not in the mapping table above, check against your knowledge of
  standard human gene nomenclature (HUGO Gene Nomenclature Committee).
- BCR-ABL / BCR-ABL1: keep as "BCR-ABL1" (fusion gene, not a single standard symbol).
- Viral kinases: keep original name and flag "enzyme_name_uncertain".

1b. Acetyltransferase Name Standardization (for acetylation)

| Non-standard | Standard |
|-------------|----------|
| p300 | EP300 |
| CBP | CREBBP |
| PCAF | KAT2B |
| GCN5 | KAT2A |
| TIP60 | KAT5 |
| MOF | KAT8 |
| MOZ | KAT6A |
| MORF | KAT6B |
| HBO1 | KAT7 |

1c. E3 Ligase Name Standardization (for ubiquitination)

Keep standard gene symbols. Common E3 ligases:
MDM2, NEDD4, NEDD4L, ITCH, WWP1, WWP2, CBL, CBLB, RNF, TRIM family members, etc.
Flag uncertain names with "enzyme_name_uncertain".

1d. General Rules for Enzyme Names
- Replace common aliases with standard gene symbols when confident.
- Keep the original name in the record field and add the standardized name
  in the appropriate field.
- If the text uses "/" separators (e.g., "AKT1/AKT2/AKT3"), convert to
  semicolons: "AKT1; AKT2; AKT3".
- Multiple distinct enzymes → semicolon-separated.
- If the enzyme name cannot be mapped to a standard symbol, add a note in
  a new field `reviewer_notes` (see Step 5).

═══════════════════════════════════════
STEP 2: MODIFICATION SITE VALIDATION
═══════════════════════════════════════

2a. Site Format Validation

Each entry in modification_sites MUST match: ^[ACDEFGHIKLMNPQRSTVWY]\d+$

Valid examples: "S186", "Y490", "T308", "K382", "S15", "T202", "Y204"
Invalid examples (REMOVE these):
- "Y"        → amino acid only, no position → REMOVE
- "308"      → position only, no amino acid → REMOVE
- "Ser186"   → three-letter code → CONVERT to "S186"
- "Ser 186"  → three-letter code with space → CONVERT to "S186"
- "C-terminal" → descriptive, not a site → REMOVE
- "424-434"  → range, not a single site → REMOVE
- "Ser/Thr-Pro" → motif, not a site → REMOVE
- "17Thr"    → reversed format → CONVERT to "T17"
- "Tyr-705"  → three-letter with hyphen → CONVERT to "Y705"
- "pY705"    → phospho-prefix → CONVERT to "Y705"
- "K382ac"   → acetylation suffix → CONVERT to "K382"
- ""         → empty string → REMOVE

2b. Three-Letter to One-Letter Conversion Table

| Three-Letter | One-Letter |
|-------------|------------|
| Ala / Alanine | A |
| Cys / Cysteine | C |
| Asp / Aspartic acid | D |
| Glu / Glutamic acid | E |
| Phe / Phenylalanine | F |
| Gly / Glycine | G |
| His / Histidine | H |
| Ile / Isoleucine | I |
| Lys / Lysine | K |
| Leu / Leucine | L |
| Met / Methionine | M |
| Asn / Asparagine | N |
| Pro / Proline | P |
| Gln / Glutamine | Q |
| Arg / Arginine | R |
| Ser / Serine | S |
| Thr / Threonine | T |
| Val / Valine | V |
| Trp / Tryptophan | W |
| Tyr / Tyrosine | Y |

2c. Site Validation Actions
- Remove all entries that don't match the valid pattern (after conversion attempts).
- If after cleanup, modification_sites becomes empty [], add review reason "missing_site".
- Record the number of removed sites in reviewer_notes.
- Do NOT invent sites. Only keep sites explicitly stated in the evidence text.

═══════════════════════════════════════
STEP 3: MULTI-ENTITY FORMATTING REVIEW
═══════════════════════════════════════

3a. Semicolon Separation Rules

Apply semicolon separation ("; ") for multi-entity fields:
- upstream_enzyme: multiple kinases → "MAPK1; MAPK3"
- ptm_target: multiple targets in same event → "MDM2; p53"
- primary_agent: multiple agents → "gefitinib; erlotinib"
- disease: multiple diseases → "breast cancer; ovarian cancer"

3b. Array Fields (already arrays, just validate)

- modification_sites: keep as JSON array ["S186", "Y490"]
- cell_lines: keep as JSON array ["MCF-7", "HeLa"]
- species: keep as JSON array ["human", "mouse"]

3c. Separator Rules
- Use "; " (semicolon + space) for all multi-entity string fields.
- Do NOT use "/" to separate distinct entities (use "/" only for isoform families
  where the original text uses it, like "ERK1/2").
- Do NOT use commas (they can be confused with natural language).
- Do NOT use "and" or "&".
- Remove duplicate entries within the same field.

3d. Isoform Family Handling
- If the text says "AKT1/AKT2/AKT3" or "AKT isoforms", convert to "AKT1; AKT2; AKT3".
- If the text says "ERK1/2" (meaning ERK1 and ERK2 as a pair), keep as "MAPK3; MAPK1".
- If the text says "p38 MAPK" without specifying which isoform, keep as "MAPK14"
  (or the family name if truly unspecified).

═══════════════════════════════════════
STEP 4: CROSS-FIELD CONSISTENCY CHECK
═══════════════════════════════════════

4a. ptm_target ↔ ptm_target_role consistency
- If ptm_target_role = "enzyme" and ptm_origin = "upstream_enzyme_explicit",
  verify: is ptm_target being modified by upstream_enzyme? If so, role should
  be "substrate". If ptm_target is the enzyme that auto-modifies, role = "enzyme"
  and ptm_origin = "auto_modification".

4b. upstream_enzyme ↔ ptm_origin consistency
- If upstream_enzyme is not empty, ptm_origin should be "upstream_enzyme_explicit"
  (unless it's auto_modification with an empty upstream_enzyme).
- If upstream_enzyme is empty, ptm_origin should be "auto_modification" or "unknown".

4c. relation_exists ↔ event fields consistency
- If relation_exists = "no", all event fields should be empty/default values.
- If relation_exists = "yes", at minimum ptm_target should be non-empty.

4d. disease_category ↔ disease consistency
- If disease is non-empty and is clearly a cancer, disease_category must be "cancer".
- If disease is non-empty and is clearly non-cancer (e.g., "Alzheimer's disease",
  "type 2 diabetes"), disease_category must be "non_cancer".
- If disease is empty, disease_category must be "unclear".

4e. species_is_human ↔ cell_lines consistency
- If cell_lines contain known human cell lines → species_is_human should be "yes".
- Common human cell lines: HeLa, HEK293, MCF-7, PC-3, A549, HCT116, U2OS, etc.
- Common non-human cell lines: NIH3T3 (mouse), CHO (hamster), COS-7 (monkey).

4f. cancer_type ↔ disease_category consistency
- If disease_category = "cancer", cancer_type should be non-empty if identifiable.
- If disease_category = "non_cancer", cancer_type should be empty.

═══════════════════════════════════════
STEP 5: QUALITY FLAG UPDATE
═══════════════════════════════════════

5a. Add new field: reviewer_notes

Add a `reviewer_notes` field (string) to document changes made during review.
Format: "Change1; Change2; Change3"

Example: "Standardized AKT→AKT1; Removed invalid site 'Y' from modification_sites; Converted ERK→MAPK1;MAPK3"

5b. Update needs_manual_review

Set needs_manual_review = "yes" if ANY of these apply:
- Enzyme name could not be standardized → add review reason "enzyme_name_uncertain"
- Site validation removed sites that appeared to be real but malformed → add "missing_site"
- Cross-field inconsistency detected and could not be resolved
- Multiple competing interpretations are equally plausible
- The original extraction had confidence < 7

5c. Add new review reason enum values

The review_reasons enum is extended with:
- "enzyme_name_uncertain": enzyme name cannot be confidently mapped to a standard gene symbol
- "site_format_invalid": one or more modification sites had invalid format and were removed
- "multi_entity_ambiguous": unclear whether entities should be merged or kept separate

5d. Adjust confidence

- If substantive corrections were made (name changes, site removals), reduce
  confidence by 1 point (minimum 0).
- If cross-field inconsistencies were found and resolved, reduce by 1 point.
- Do NOT increase confidence during review.

═══════════════════════════════════════
OUTPUT RULES
═══════════════════════════════════════
- Return the REVIEWED JSON object with all corrections applied.
- The output schema is the same as the extraction schema, plus the new
  `reviewer_notes` field and extended `review_reasons` enum values.
- Do NOT output markdown fences.
- Do NOT output explanations.
```

---

## Output Schema (After Review)

The final output schema includes all extraction fields plus reviewer-added fields:

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
  "review_reasons": [
    "missing_entity | unclear_direction | missing_site | missing_cell_line | multi_entity_ambiguity | weak_evidence | agent_role_unclear | technical_assay_only | enzyme_name_uncertain | site_format_invalid | other"
  ],

  "reviewer_notes": "string (semicolon-separated list of changes made during review)"
}
```

### Reviewer-Added Fields

| Field | Type | Description |
|-------|------|-------------|
| `reviewer_notes` | string | Semicolon-separated list of corrections made during review phase |

### Extended Review Reasons

| Reason | Added By | Description |
|--------|----------|-------------|
| `enzyme_name_uncertain` | Reviewer | Enzyme name cannot be confidently mapped to a standard gene symbol |
| `site_format_invalid` | Reviewer | One or more modification sites had invalid format and were removed |

---

## Example: Full Two-Phase Pipeline

### Input
```
PMID: 12345678
Title: AKT and ERK phosphorylate MDM2 at Ser186 and Ser188 promoting p53 degradation in breast cancer
Abstract: Both AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188, which enhances MDM2-mediated ubiquitination and degradation of p53. Treatment with the AKT inhibitor MK-2206 or the MEK inhibitor PD98059 impaired this phosphorylation in MCF-7 and T47D breast cancer cells, leading to apoptosis.
```

### After Phase 1 (Extraction)
```json
{
  "pmid": "12345678",
  "record": "",
  "ptm_type": "phosphorylation",
  "relation_exists": "yes",
  "evidence_text": "Both AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188, which enhances MDM2-mediated ubiquitination and degradation of p53.",
  "evidence_scope": "single_sentence",
  "ptm_target": "MDM2",
  "ptm_target_role": "substrate",
  "modification_sites": ["Ser186", "Ser188"],
  "upstream_enzyme": "AKT, ERK",
  "ptm_origin": "upstream_enzyme_explicit",
  "primary_agent": "MK-2206; PD98059",
  "agent_validity": "valid_intervention",
  "agent_type": "investigational_drug",
  "disease": "breast cancer",
  "disease_category": "cancer",
  "cancer_type": "breast cancer",
  "cancer_subtype": "",
  "phenotype_context": "apoptosis",
  "cell_lines": ["MCF-7", "T47D"],
  "species": ["human"],
  "species_is_human": "yes",
  "trigger_phrase": "AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188",
  "regulation_mode": "enhancing",
  "relationship": "AKT and ERK phosphorylate MDM2 at S186 and S188, enhancing p53 degradation; inhibitors impair this",
  "confidence": 9,
  "needs_manual_review": "no",
  "review_reasons": []
}
```

### After Phase 2 (Review) — Final Output
```json
{
  "pmid": "12345678",
  "record": "",
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
  "cancer_subtype": "",
  "phenotype_context": "apoptosis",
  "cell_lines": ["MCF-7", "T47D"],
  "species": ["human"],
  "species_is_human": "yes",
  "trigger_phrase": "AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188",
  "regulation_mode": "enhancing",
  "relationship": "AKT1; MAPK1; MAPK3 phosphorylate MDM2 at S186 and S188, enhancing p53 degradation; MK-2206 and PD98059 impair this phosphorylation",
  "confidence": 8,
  "needs_manual_review": "no",
  "review_reasons": [],
  "reviewer_notes": "Standardized AKT→AKT1; Standardized ERK→MAPK1;MAPK3; Converted sites Ser186→S186, Ser188→S188; Changed separator ',' to ';' in upstream_enzyme; Added tool_compound agent_type for PD98059"
}
```

**Review changes made:**
1. "AKT" → "AKT1" (standard gene symbol)
2. "ERK" → "MAPK1; MAPK3" (ERK is ERK1+ERK2; ERK1=MAPK3, ERK2=MAPK1)
3. "Ser186" → "S186", "Ser188" → "S188" (three-letter → one-letter code)
4. "," → ";" in upstream_enzyme (standardized separator)
5. agent_type expanded to "investigational_drug; tool_compound" for the two agents
6. confidence reduced from 9→8 (substantive name corrections)
7. reviewer_notes documents all changes

---

## Kinase/Enzyme Name Verification Protocol

When the Reviewer cannot confidently map an enzyme name using internal knowledge,
use the following verification strategies:

### Strategy 1: Internal Knowledge Base
Claude's training data includes extensive knowledge of:
- Human kinome (~518 kinases) with standard gene symbols
- Common E3 ubiquitin ligases
- Acetyltransferases (HATs) and deacetylases (HDACs)
- Methyltransferases and demethylases

Apply this knowledge first.

### Strategy 2: Pattern-Based Deduction
- Names ending in "K" or followed by a number are likely standard (e.g., "AKT1", "MAPK1")
- Names with Greek letters (α, β, γ) should be converted to Latin (A, B, G)
  when part of standard symbols (e.g., "IKKβ" → "IKBKB")
- Fusion genes: keep original (e.g., "BCR-ABL1")
- Viral enzymes: flag "enzyme_name_uncertain"

### Strategy 3: Flag for External Verification
If a name cannot be confidently standardized, add "enzyme_name_uncertain" to
review_reasons and document the uncertainty in reviewer_notes.

Recommended external tools for batch verification (post-extraction):
- **HGNC Multi-Symbol Checker**: https://www.genenames.org/tools/multi-symbol-checker/
- **UniProt ID Mapping**: https://www.uniprot.org/id-mapping
- **mygene.info API**: `https://mygene.info/v3/query?q=symbol:{gene_symbol}&species=human`

---

## Notes

- **No API keys in this skill**: This is a Claude Code skill definition — it uses
  Claude's built-in capabilities. No external API keys are needed.
- **Batch processing**: For large-scale extraction (>100 papers), use the companion
  Python script `qwen_extract_phospho_event_sharded.py` with an OpenAI-compatible API.
- **Citation**: If you use PhosphoRLIE in academic work, please cite the GitHub
  repository and associated publication.
